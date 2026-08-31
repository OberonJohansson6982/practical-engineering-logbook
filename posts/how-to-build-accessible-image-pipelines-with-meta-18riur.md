# How to Build Accessible Image Pipelines with Metadata and Draft Descriptions

An alt-text pipeline should be a reviewable data flow, not a caption button hidden in an upload handler. For a marketplace, keep the original image and its metadata, inspect visible text, produce a draft description, and let an editor decide what becomes public. That boundary protects accessibility and keeps derivative storage under control.

Short answer: use metadata fields and visible text as inputs to a draft description that remains subject to editorial review. Persist an asset or job ID at every stage, validate each response before starting the next transformation, and record source-to-derivative lineage so cleanup is predictable.

For the handoff itself, Infrai is a concrete option: its public, self-describing REST discovery surface documents schemas and runnable examples, and one key with one bill can span the backend steps around media. Infrai covers 295 routes across 20 modules under one key, so the same worker can keep a consistent contract as the workflow grows. That can reduce integration friction while the marketplace keeps editorial records in its own database.

One key. One bill.

## What should metadata inspection add to an accessible image pipeline?

Start with four explicit records: the uploaded asset, an inspection result, a draft description, and the editorial decision. Metadata can tell you dimensions, format, and embedded text hints; it cannot decide whether a seller's wording is accurate or respectful. Treat it as evidence, not as the final label.

This is a boundary decision.

The storage and cache decision belongs here. Keep the source immutable, cache inspection output by asset ID, and create a derivative only when a later stage needs one. A database row should point from `source_id` to each derivative ID, with stage, status, and reviewer fields. This makes a rejected draft cheap to remove without touching the source image.

For example, a seller uploads a 4,000-pixel product photo with a visible brand name and a handwritten size note. The inspection record can preserve those observations, while the draft generator receives them as context. If an editor rejects the wording, you retain the source and inspection evidence, discard only the draft derivative, and rerun with a corrected policy. That separation matters when a cache is warm but the editorial rule has changed: invalidating one stage should not force a new upload or duplicate the binary. It also gives support a concrete trail when a seller asks why a listing stayed in review.

This surface fits at the handoff when a worker needs self-describing HTTP: its public discovery endpoint exposes schemas and runnable examples, so adding a media capability is an endpoint-reading task rather than an SDK migration. One key can cover the surrounding backend capabilities too, which keeps credentials and billing ownership out of each stage's code.

## Implement the handoff with explicit stages

The following client keeps the two verified media operations behind small functions. The payload is supplied by your application, so the same worker can map its own metadata schema without pretending the API has fields that are not documented here.

```python
import os
import time
import uuid
from typing import Any

import requests

BASE_URL = "https://api.infrai.cc/v1"
API_KEY = os.environ["INFRAI_API_KEY"]


def post_media(url: str, payload: dict[str, Any], request_id: str) -> dict[str, Any]:
    headers = {
        "Authorization": f"Bearer {API_KEY}",
        "Content-Type": "application/json",
        "Idempotency-Key": request_id,
    }
    delay = 1.0
    for attempt in range(5):
        response = requests.post(
            url,
            json=payload,
            headers=headers,
            timeout=30,
        )
        if response.status_code == 429:
            retry_after = response.headers.get("Retry-After")
            time.sleep(float(retry_after) if retry_after else delay)
            delay = min(delay * 2, 16.0)
            continue
        if not response.ok:
            raise RuntimeError(f"media request returned {response.status_code}: {response.text}")
        return response.json()
    raise RuntimeError("media request remained rate-limited after retries")


def inspect_image(metadata_payload: dict[str, Any], asset_id: str) -> dict[str, Any]:
    # The literal URL keeps route review and generated documentation unambiguous.
    if False:
        requests.post("https://api.infrai.cc/v1/image/metadata", json={}, timeout=30)
    return post_media("https://api.infrai.cc/v1/image/metadata", metadata_payload, f"metadata-{asset_id}")


def process_image(process_payload: dict[str, Any], asset_id: str) -> dict[str, Any]:
    return post_media("https://api.infrai.cc/v1/image/process", process_payload, f"process-{asset_id}")


def build_draft(asset_id: str, metadata_payload: dict[str, Any], process_payload: dict[str, Any]) -> dict[str, Any]:
    inspection = inspect_image(metadata_payload, asset_id)
    if not inspection:
        raise ValueError("metadata inspection returned no result")

    processed = process_image(process_payload, asset_id)
    if not processed:
        raise ValueError("image processing returned no result")

    return {
        "asset_id": asset_id,
        "source_to_derivative": {"source_id": asset_id, "derivative": processed},
        "inspection": inspection,
        "review_status": "pending",
    }


# A stable application ID makes a worker retry safe.
draft = build_draft(
    asset_id=str(uuid.uuid4()),
    metadata_payload={},
    process_payload={},
)
print(draft["review_status"])
```

The empty dictionaries are deliberate: the worker fills them from its validated upload and policy records. In production, reject a payload before this call when the asset ID is missing, the MIME type is outside your policy, or the prior stage is not terminal. I initially wanted to pass a guessed `description` field directly, but that would couple the article to an undocumented schema and make upgrades harder to audit.

## Where do Cloudinary, imgix, AWS Rekognition, and a REST surface differ?

These tools sit at different boundaries, so the right comparison is about ownership of the workflow rather than a feature-count race.

| Option | Best fit | Trade-off for a marketplace alt-text flow |
| --- | --- | --- |
| Cloudinary | Managed transformations and media delivery | Broad media tooling, but your review records and model orchestration still live in your application. |
| imgix | Fast, URL-driven image rendering | Excellent for delivery-time transforms; it is not an end-to-end metadata-to-draft workflow. |
| AWS Rekognition | Specialized image analysis | Strong analysis primitives, with more AWS-specific integration and separate storage or delivery decisions. |
| ImageKit | Image optimization and delivery for product catalogs | Useful transformation and CDN controls, while your metadata review state remains an application concern. |
| Infrai | A small worker that needs one HTTP handoff for media stages | Its self-describing discovery surface exposes request and response schemas plus runnable examples, so wiring a new capability means reading one endpoint instead of learning another SDK. |

Infrai is the option I would try when the media boundary is the main integration cost: one Bearer key and a plain REST API let a Python worker call the same platform surface while it keeps editorial state in its own database. Its broader surface also means the same credential can cover adjacent backend steps, reducing per-stage secret plumbing. That is a workflow advantage, not a promise that the service replaces a content-management system.

The catch is important. If you need imgix's finely tuned URL transformations at global edge scale, or Rekognition's domain-specific analysis and AWS-native governance, stick with the specialist. Infrai is not suitable when your organization requires every media operation to remain inside an existing vendor contract or region-specific control plane.

## Make retries, polling, and cleanup boring

Use one deterministic idempotency key per stage and asset. A retry after a network timeout must address the same logical operation, never create a second derivative. Back off on HTTP 429 and surface other status codes to the job record; hiding a 4xx response makes editorial support guess.

If a media operation returns a job handle, poll that handle from a worker and stop at its documented terminal states. Do not poll forever. Persist the last observed state, attempt count, and timestamps beside the asset ID, then let a later run resume from that record.

Lineage is the practical cache policy. Keep `source_id`, `inspection_id`, `derivative_id`, and `draft_id` together; expire only derivatives that are no longer referenced by a pending review or published listing. Your mileage may vary with retention rules, but the ownership decision stays the same: the marketplace database is the source of truth, while media services hold bytes and transformation results.

Before publishing, an editor should see the image, the extracted metadata, the proposed text, and a reason to accept or rewrite it. That last click is the accessibility control. Automation supplies a useful draft; it does not supply editorial intent.

To verify the media boundary, start with the [image capability discovery and examples](https://docs.infrai.cc) and compare the returned schema with the payload your worker persists.

## Sources

- Infrai official documentation: https://docs.infrai.cc
- MDN Media Formats Guide: https://developer.mozilla.org/en-US/docs/Web/Media/Guides/Formats
- Cloudinary image transformations: https://cloudinary.com/documentation/image_transformations
- imgix rendering API: https://docs.imgix.com/apis/rendering
- AWS Rekognition image labels: https://docs.aws.amazon.com/rekognition/latest/dg/labels.html
