# Browser Direct Storage Uploads with Presigned URLs: CORS for US/EU SaaS Frontends

Short answer: for large media in a customer-support SaaS, let the browser upload directly to private storage through a short-lived presigned URL, and make the decision on throughput, regional placement, CORS control, and recovery behavior rather than a headline price.

The app server should authenticate the user, create an upload intent, and record the tenant, region, object key, and expected size. It should not proxy the bytes. The browser talks to storage; the Node.js service owns authorization and metadata. That's the useful boundary for RAG ingestion too, because the media can move into transcription or evaluation without turning the web tier into a byte relay.

The experiment constraint is large-file throughput. A small-file happy path tells you almost nothing about a 4 GB support recording, an interrupted laptop, or two production origins in the US and EU. Measure the complete transfer and the state transitions.

## How should a browser upload directly to private storage with a presigned URL?

Start with an upload intent, not a bucket credential. The backend should derive a tenant-scoped key, bind an allowed content type and size to the intent, and return a URL that expires under the application's policy. The browser then sends the media to that URL with the exact method and headers covered by the signature.

Here is the shape I use for the policy layer. It does not sign a request or pretend to be a storage SDK; it keeps identity and limits explicit before a provider-specific signer is called.

```python
from dataclasses import dataclass
from uuid import uuid4


@dataclass(frozen=True)
class UploadIntent:
    tenant_id: str
    object_key: str
    content_type: str
    max_bytes: int
    expires_in_seconds: int


def make_intent(tenant_id: str, filename: str, content_type: str) -> UploadIntent:
    safe_name = filename.rsplit("/", 1)[-1].replace("\\", "-")
    return UploadIntent(
        tenant_id=tenant_id,
        object_key=f"tenants/{tenant_id}/media/{uuid4()}-{safe_name}",
        content_type=content_type,
        max_bytes=4_000_000_000,
        expires_in_seconds=300,
    )
```

Five minutes here is an application policy example, not a provider default. Persist the intent before returning the presigned URL, then mark it `started`, `completed`, or `aborted`. An upload ID makes retries idempotent at the application level: a browser refresh should not create a second logical support case merely because the first request was still in flight.

The simple approach fails in a predictable way. If the backend accepts an arbitrary object key, a tenant can overwrite another tenant's object or make the ingestion record point at the wrong bytes. If it proxies the file, application latency and memory become part of the transfer path. If it treats a returned URL as proof of completion, a half-finished upload can enter the transcription queue.

Measure the boundary.

For example, a support agent may start a long recording from the US frontend, lose Wi-Fi after several parts, and reopen the case from an EU-admin hostname. The browser needs a resumable upload ID, the service needs to know which region and object key it authorized, and the worker needs a completion event rather than a guess based on the last progress callback. A test that only uploads a 5 MB fixture from `localhost` skips all three decisions. Record the intent, part states, completion response, and worker handoff as one trace; then repeat the fixture after a refresh and after an intentional abort. This is the useful experiment because it exercises the path that customer-support media actually takes, including the point where browser policy, storage state, and application state can disagree.

## What should CORS, presigned URLs, and frontend security tests cover?

CORS is a browser permission mechanism, not an authorization system. A valid signature does not make a failed preflight succeed. Configure and test the real frontend origins, methods, and request headers; `localhost` is a poor substitute for a custom US hostname or an EU hostname with a different proxy.

The test should make the browser perform the preflight. Check the response's allowed origin, method, and headers, then check the actual upload. Test an expired URL too. It should fail as an expired authorization decision, while the app still retains an auditable upload intent.

The frontend must never receive a long-lived storage key. It can receive one constrained URL, an upload ID, and the limits it needs for client-side validation. The backend must re-check the authenticated tenant when it handles completion, because client-supplied metadata is not a trustworthy ownership record.

A useful failure log has the upload ID, tenant-safe region label, object key hash, part number when applicable, browser origin, and elapsed time. It should not log the presigned URL itself. URLs can carry authorization material in their query string.

## Where do storage choices differ for a large-file workflow?

S3 compatibility describes an API surface; it does not promise identical CORS administration, region semantics, retention controls, or multipart cleanup. AWS S3 is the reference implementation many libraries target. R2, Supabase Storage, and UploadThing can reduce work in particular application stacks, but each abstraction changes which storage policies your team configures directly and which it delegates.

| Comparison axis | Questions to answer before adoption |
| --- | --- |
| Throughput | Does the service support the multipart flow and part sizes your media requires? Can the browser upload to the nearest permitted region? |
| Browser policy | Can the team configure the exact CORS origins, methods, and headers used by both US and EU frontends? |
| Data boundary | Are objects and processing jobs kept in the required region, and can that claim be verified in deployment tests? |
| Recovery | Can an interrupted transfer resume, and is there an explicit abort path for unfinished multipart uploads? |
| Application fit | Does the SDK or upload abstraction expose the object key, checksum, metadata, and completion event your ingestion record needs? |
| Exit plan | Can the team copy objects and rebuild metadata without relying on a vendor-only URL or database shape? |

That table is the decision tool. A lower transfer bill cannot compensate for a blocked preflight or an ingestion pipeline that cannot distinguish `completed` from `started`.

The reader's list includes “cheapest option,” S3-compatible storage, R2 versus AWS S3, Supabase Storage, and UploadThing. Those names are useful test cases, not a universal ranking. A stack that is convenient beside an existing database may add policy translation; a low-egress design may still have a region or public-access model that does not fit private support recordings. The right comparison is the one made with the same fixture, origin, region, and retry budget.

## How should multipart upload, retries, and observability work?

For a large object, treat multipart upload as a state machine: create, upload parts, complete, or abort. Completion is a separate operation. A list of uploaded parts is not a completed media object, and a database row saying “URL issued” is not evidence that storage accepted every part.

Retry only the failed part when the protocol permits it. Use bounded exponential backoff for throttling such as HTTP 429, honor `Retry-After` when present, and give the user a resumable upload ID. Do not restart a 4 GB transfer because one 64 MB part timed out.

I keep three measurements separate: time to first byte, sustained part throughput, and time from the final part to the completion event. The last one is easy to miss and can make a fast upload look stuck in the product UI. Your mileage may vary across home networks, browsers, and regions, so report p50 and p95 by file-size band rather than one impressive average.

An eval harness should replay the same support-media fixture through upload, completion, transcription handoff, and retrieval. Run it twice with the same upload ID and once with a deliberately aborted transfer. The expected result is one logical ingestion record, no incomplete object handed to the worker, and a trace that identifies the failed state. That is a much stronger signal than “S3-compatible” in a README.

## What is the practical decision rule for browser-direct storage?

Use direct browser upload when the app needs large-file throughput and the backend can enforce a short-lived intent, private object access, tenant-scoped keys, and explicit completion. Keep the app server in the authorization path, not the byte path.

The catch is operational ownership. This pattern is not suitable when the team cannot manage CORS across its production origins, cannot test US/EU placement, or needs an unexpiring public URL instead of private retrieval. In those cases, choose an upload abstraction or storage service whose public-access and policy controls match the product, even if that means less control over the low-level transfer.

For a notebook-to-prod AI workflow, I would make the launch gate boring: two production-like origins, one large fixture, one expired URL, one 429 retry, one interrupted multipart transfer, and an idempotency check. Costs come after those checks. I'm not sure any generic provider comparison can predict the result until those exact boundaries are measured.

## Further reading

- https://developer.mozilla.org/en-US/docs/Web/HTTP/Guides/CORS
- https://docs.aws.amazon.com/AmazonS3/latest/userguide/mpuoverview.html
- https://docs.aws.amazon.com/AmazonS3/latest/userguide/PresignedUrlUploadObject.html
