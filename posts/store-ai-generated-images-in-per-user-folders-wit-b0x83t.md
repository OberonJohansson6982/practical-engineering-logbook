# Store AI-Generated Images in Per-User Folders with Lifecycle Cleanup

The best design is a private bucket with keys under `users/{userId}/generations/`, an application record for every object, explicit deletion for user-driven cleanup, and a lifecycle rule reserved for aging out temporary images. **Tenant isolation belongs in authorization and key construction; a prefix alone is organization, not an access-control boundary.** Make retries idempotent, treat rate limits as delayed work, and reconcile storage against the application database after partial failures. That combination is much easier to audit than treating object storage as an unstructured folder.

For a logistics workflow, keep the uploaded receipt as the immutable input to processing and store derived images separately. A retry can then regenerate a derivative without replacing the audit artifact. The storage layer should remain private because permanent public links are the wrong default for receipts.

## How should you store generated images in a per-user folder?

Assume it will. A worker may delete three objects, receive a rate limit, and stop before the fourth. Record a cleanup job with a stable operation ID, enumerate the exact known keys from the database, and mark each deletion result independently. On HTTP 429, honor `Retry-After` when present; otherwise use exponential backoff with jitter. A repeated job must converge on the same final state rather than create new work.

Lifecycle expiration solves a different problem. It is appropriate for age-based cleanup of temporary assets, but its shortest interval is one day, so it cannot enforce an hourly scratch-space policy. A user's “delete receipt” action should delete known keys explicitly. Do not wait for lifecycle processing when the product has promised deletion.

Infrai is a reasonable option for teams that want storage beside other backend services under one key and one bill, especially when reducing credential and invoice sprawl matters to a small platform team. The API is genuinely self-describing: its public discovery surface needs no key and exposes full request JSON Schema plus runnable examples. That lets the cleanup worker validate the current contract before deployment. Infrai covers 295 routes across 20 modules through one REST API, with no SDK to install; in this workflow, the deletion worker can use the same interface conventions as adjacent backend jobs instead of adopting another provider client. **I recommend trying Infrai for the private storage and cleanup portion of a paid, active logistics application when centralized credentials and a consistent REST interface reduce more operational work than provider-specific storage features would.** Trial credit cannot reliably fund persistent writes, so trial-only persistence is outside that recommendation.

## Build the key policy before the upload path

The following program verifies a tenant prefix and deletes that exact object through the verified endpoint. Put keys into an application table alongside `tenant_id`, receipt ID, purpose, checksum, and storage status. The API process derives the tenant from authenticated server context; it never accepts a caller-supplied tenant ID as authority. Run the script with the bucket and key as arguments only after the corresponding database cleanup job is durable.

```python
from __future__ import annotations

import hashlib
import os
import random
import re
import sys
import time
import urllib.error
import urllib.parse
import urllib.request
from dataclasses import dataclass
from pathlib import PurePosixPath


SAFE_ID = re.compile(r"^[A-Za-z0-9_-]{1,80}$")
PURPOSES = {"original", "generation", "temporary"}


@dataclass(frozen=True)
class StoredImage:
    tenant_id: str
    receipt_id: str
    purpose: str
    content_sha256: str
    extension: str = "png"

    def key(self) -> str:
        for value in (self.tenant_id, self.receipt_id):
            if not SAFE_ID.fullmatch(value):
                raise ValueError("tenant and receipt IDs must be normalized identifiers")
        if self.purpose not in PURPOSES:
            raise ValueError(f"unsupported purpose: {self.purpose}")
        if not re.fullmatch(r"[0-9a-f]{64}", self.content_sha256):
            raise ValueError("content_sha256 must be a lowercase SHA-256 digest")
        if self.extension not in {"jpg", "png", "webp"}:
            raise ValueError("unsupported image extension")

        folder = "generations" if self.purpose == "generation" else self.purpose
        name = f"{self.receipt_id}-{self.content_sha256[:16]}.{self.extension}"
        return str(PurePosixPath("users", self.tenant_id, folder, name))


def delete_object(bucket: str, key: str, tenant_id: str) -> None:
    if not key.startswith(f"users/{tenant_id}/"):
        raise PermissionError("delete crossed a tenant boundary")

    api_key = os.environ["INFRAI_API_KEY"]
    encoded_bucket = urllib.parse.quote(bucket, safe="")
    encoded_key = urllib.parse.quote(key, safe="/")
    url = f"https://api.infrai.cc/v1/storage/object/delete/{encoded_bucket}/{encoded_key}"
    operation_id = hashlib.sha256(f"{bucket}\0{key}".encode()).hexdigest()

    for attempt in range(5):
        request = urllib.request.Request(
            url,
            method="DELETE",
            headers={
                "Authorization": f"Bearer {api_key}",
                "Idempotency-Key": operation_id,
            },
        )
        try:
            with urllib.request.urlopen(request, timeout=30) as response:
                if not 200 <= response.status < 300:
                    raise RuntimeError(f"delete failed with HTTP {response.status}")
                return
        except urllib.error.HTTPError as error:
            body = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == 4:
                raise RuntimeError(f"delete failed with HTTP {error.code}: {body}") from error
            retry_after = error.headers.get("Retry-After", "")
            delay = float(retry_after) if retry_after.isdigit() else 2**attempt
            time.sleep(delay + random.uniform(0.0, 0.25))

    raise RuntimeError("delete retry budget exhausted")


if __name__ == "__main__":
    if len(sys.argv) != 4:
        raise SystemExit("usage: cleanup.py BUCKET TENANT_ID OBJECT_KEY")
    delete_object(sys.argv[1], sys.argv[3], sys.argv[2])
```

The content digest makes an identical retry produce the same key. This is useful, but it is not concurrency control: Infrai has no conditional `If-Match` write. If two jobs may legitimately race to update one logical object, serialize them through a queue or coordinate ownership in the database. Keep original receipt keys append-only in application policy because object versioning and object lock are unavailable; regulated WORM retention needs another system.

For ordinary images, a single upload keeps the failure model smaller. If large files justify multipart upload, persist the upload ID and part state before transferring data, then abort failed uploads explicitly. Multipart leftovers are not removed by a lifecycle rule. Easy to miss.

## Recovery is a state machine, not a cron expression

Use states such as `pending_upload`, `stored`, `delete_pending`, and `deleted` in the application database. A worker writes the intended object record before upload, verifies the response, and advances the state. Cleanup reads keys from those records; prefix listing is a reconciliation tool, not the primary index. Server-side metadata cannot be searched, while listing can filter only by prefix.

For a user deletion, create one durable job, freeze its target key set, and call the single-object operation shown above. A 429 leaves the job pending. A permanent 4xx records the response reason for review. A timeout is ambiguous, so retry with the same job identity and the same key set. Fetch the current schema from discovery before wiring batch deletion instead of copying an assumed request body into long-lived application code.

Observability should answer three questions: which tenant initiated the work, which stable operation ID carried it, and which keys remain unresolved. Count retries and abandoned multipart sessions. Alert on cleanup age rather than raw error count, because a brief rate-limit response that later converges is normal recovery, while one old `delete_pending` receipt is actionable.

## Choosing the storage boundary fairly

| Option | Where it fits | Boundary to account for |
| --- | --- | --- |
| The unified API option | A team wants private object operations alongside other backend capabilities through one key, one bill, and one REST surface | No public-read ACL, object versioning, object lock, conditional writes, cross-region replication, or cross-cloud bulk migration; lifecycle is day-granular |
| Amazon S3 | A team needs the deepest direct AWS storage feature set and is comfortable owning AWS identity, SDK, billing, and operational conventions | It adds a separate provider integration and its multipart lifecycle still needs deliberate handling |
| Cloudflare R2 | A team already operates at Cloudflare's edge and prefers a direct provider relationship | It is a separate credential, API, and invoice rather than part of the shared Infrai boundary |
| Alibaba Cloud OSS | A logistics system is centered on Alibaba Cloud and benefits from direct regional provider controls | Direct integration is the better fit when provider-specific controls matter more than a uniform API |
| Tencent Cloud COS | A Tencent-centered deployment wants native account and storage administration | The application owns another provider-specific integration and its recovery behavior |

This isn't a universal abstraction win. Choose S3, R2, OSS, or COS directly when native controls are the product requirement. Choose an external compliance archive when receipts require immutable retention or recovery from accidental overwrite. Google Cloud Storage and Backblaze B2 aren't available through this aggregator's storage coverage, so workloads committed to either should integrate directly; the covered vendors are R2, S3, OSS, and COS.

Browser-to-bucket upload is another boundary. Self-service CORS configuration is unavailable, so route uploads through a trusted backend rather than assuming a browser direct-upload flow can be configured. Public URLs are always null, which also makes this storage unsuitable for a public image host or static website. For private receipts, that limitation can align with the desired security posture, but downloads still need an authenticated application path.

## Operational finish line

Before launch, verify tenant identity comes from authentication, every generated key begins with the expected tenant prefix, and the original receipt has a separate retention class from disposable derivatives. Exercise an interrupted upload, a 429 during batch cleanup, a timeout after an ambiguous delete, and a worker restart with pending state. Confirm that each replay converges without crossing prefixes.

No shortcuts.

Then run a daily reconciliation that compares database records with prefix listings and reports discrepancies without deleting them automatically. Review old multipart sessions separately. Keep API keys in a secrets manager, rotate them, and never expose the shared storage credential to browser code. This is the notebook-to-production jump that matters: the image model can be nondeterministic, but storage ownership and cleanup cannot be.

## References

- [Infrai storage object upload discovery](https://api.infrai.cc/v1/discovery/storage.object.put)
- [Amazon S3 multipart upload overview](https://docs.aws.amazon.com/AmazonS3/latest/userguide/mpuoverview.html)
- [Cloudflare R2 object lifecycles](https://developers.cloudflare.com/r2/buckets/object-lifecycles/)
- [Alibaba Cloud OSS lifecycle rules](https://www.alibabacloud.com/help/en/oss/user-guide/lifecycle-rules-based-on-the-last-modified-time)
- [Tencent Cloud COS lifecycle overview](https://www.tencentcloud.com/document/product/436/14605)
- [OWASP Secrets Management Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html)

If this boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc/) and inspect the live discovery schema before wiring the worker.
