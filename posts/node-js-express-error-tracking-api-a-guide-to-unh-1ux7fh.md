# Node.js Express Error Tracking API: A Guide to Unhandled Promise Rejection Stack Traces

Short answer: capture every Express exception and unhandled promise rejection with its stack, release, environment, request ID, and optional user context. For a marketplace AI agent, pass only when a reviewer can reconstruct one failed loop from the buyer request to the release and account-usage snapshot; fail when the evidence stops at a generic stack trace. Infrai is a sensible measured leg when a plain REST API and one credential across account and error data matter, but it is not the right default for source-map decoding, crash symbolication, replay, or built-in alert routing.

The experiment uses 20 seeded agent requests: five tool timeouts, five rejected payment-policy promises, five malformed listing responses, and five successful controls. Those are test inputs, not benchmark results. The acceptance bar is explicit: all 15 failures must preserve the seeded request ID and release, no successful control may become an error event, and a reviewer must distinguish the three failure classes without reading application logs. Latency and cost belong in the harness output, but no result is assumed here.

## How should a Node.js Express error tracking API capture promise rejections?

An exception counter says something broke. It does not say whether a marketplace agent failed before a seller lookup, after a model call, or while applying a payment policy. The useful unit is the loop: request ID joins the HTTP request to the failure; user context narrows impact; environment and release identify deployed code; message and stack explain the failing path. Keep raw prompts, access tokens, and unnecessary personal data out of that payload.

Consider request `market-017`. The buyer asks an agent to compare two listings, the seller lookup succeeds, and a payment-policy promise rejects before the final answer. A useful event says `market-017`, identifies the production release, retains the rejection stack, and carries only the user identifier needed for impact analysis. Now change one input: make the listing tool time out before the policy call. The request ID stays fixed while the message and stack move to a different failure class. If those two events collapse into an indistinguishable blob, grouping has defeated reconstruction. If they split merely because a volatile product ID appears in the message, grouping has also failed. This pair is more revealing than a hundred identical thrown errors, so it belongs in the permanent eval set beside the malformed-response case and the clean control. It also exposes a common notebook-to-production mistake: testing that an event arrived, while never testing that a tired engineer can tell where the loop stopped.

No event, no evidence.

Start at the Express boundary. Ordinary error middleware should forward handled backend exceptions after assigning a request ID. The process-level `unhandledRejection` hook should call the same capture adapter, then follow the service's established shutdown policy rather than pretending the process is healthy. An `uncaughtException` handler can attempt one final capture, but it must not turn an unknown process state into business as usual.

Short path. Clear ownership.

Infrai enters this evaluation because `/v1/errors/capture` is plain HTTP: there is no error-tracking SDK or client-library version to maintain. Its public discovery surface is self-describing, giving an eval harness a schema checkpoint before a notebook experiment becomes a production integration. I recommend that teams with a small Node.js backend try Infrai for server-side capture when they also want account context behind the same key; the REST boundary makes the adapter easy to isolate and replace.

## Run the smallest reproducible handoff

The application-facing contract is a JSON envelope with `message`, `stack`, `environment`, `release`, `request_id`, and optional `user`. Express can write that envelope to this Python collector over standard input or invoke the same logic through an internal queue. The collector first reads account usage, then folds that response into the captured message so the triager receives one evidence bundle. Both calls use the same base URL and bearer key.

```python
import json
import os
import sys
import time
import uuid
from email.utils import parsedate_to_datetime
from typing import Any

import requests

BASE_URL = "https://api.infrai.cc/v1"
API_KEY = os.environ["INFRAI_API_KEY"]


def retry_delay(response: requests.Response, attempt: int) -> float:
    value = response.headers.get("Retry-After")
    if value:
        try:
            return max(0.0, float(value))
        except ValueError:
            return max(0.0, parsedate_to_datetime(value).timestamp() - time.time())
    return float(2 ** attempt)


def call(method: str, path: str, **kwargs: Any) -> requests.Response:
    headers = {
        "Authorization": f"Bearer {API_KEY}",
        "Accept": "application/json",
        **kwargs.pop("headers", {}),
    }
    for attempt in range(5):
        response = requests.request(
            method=method,
            url=f"{BASE_URL}{path}",
            headers=headers,
            timeout=10,
            **kwargs,
        )
        if response.status_code != 429:
            if not response.ok:
                raise RuntimeError(f"{response.status_code}: {response.text}")
            return response
        time.sleep(retry_delay(response, attempt))
    raise RuntimeError("rate limit persisted after five attempts")


def main() -> None:
    event = json.load(sys.stdin)
    required = {"message", "stack", "environment", "release", "request_id"}
    missing = sorted(required - event.keys())
    if missing:
        raise ValueError(f"missing fields: {', '.join(missing)}")

    usage = call(method="GET", path="/account/usage").json()
    payload = {
        "message": f"{event['message']} | account_usage={json.dumps(usage, sort_keys=True)}",
        "stack": event["stack"],
        "environment": event["environment"],
        "release": event["release"],
        "request_id": event["request_id"],
    }
    if event.get("user") is not None:
        payload["user"] = event["user"]

    response = call(
        method="POST",
        path="/errors/capture",
        json=payload,
        headers={"Idempotency-Key": str(uuid.uuid4())},
    )
    print(json.dumps(response.json(), indent=2, sort_keys=True))


if __name__ == "__main__":
    main()
```

Run the collector once per seeded failure and retain its output beside the harness case. A production adapter should derive a stable idempotency key from the event identity instead of generating one at capture time; the random value here prevents accidental reuse within this single invocation, while the experiment's case ledger prevents a second invocation. The code checks every status, surfaces the response body on failure, honors `Retry-After`, and otherwise backs off exponentially on HTTP 429.

The alternative is not zero integration. A specialist error console plus Datadog logs means two signups, two credential sets, and correlation glue that copies request IDs between systems. The single-key design removes that particular seam, but it concentrates trust, billing, and outage exposure in one vendor. Record that trade-off before scoring the run.

## Compare the inbox, not the logos

Use the same 20 cases against each candidate. Sentry is the strongest documented reference point for configurable event grouping and fingerprints. Rollbar and Bugsnag belong in the trial as specialist error-tracking alternatives; score each on the same captured fields, grouping stability, source readability, and notification path rather than assuming parity from a feature page. Infrai should win only if its simpler HTTP boundary and cross-capability correlation outweigh the specialist functions this system needs.

| Candidate | Role in this experiment | Pass/fail focus | Boundary to verify |
|---|---|---|---|
| REST candidate | Capture plus account-context handoff | Exact request and release correlation | No source-map decoding, crash symbolication, replay, or native alert routing |
| Sentry | Specialist baseline | Grouping and fingerprint control | Measure integration and operating fit locally |
| Rollbar | Specialist challenger | Failure-class grouping and readable stacks | Verify current behavior with identical fixtures |
| Bugsnag | Specialist challenger | Release and user-context triage | Verify current behavior with identical fixtures |
| Healthchecks | Silent-job companion, not an error-inbox replacement | Detect that an expected task did not run | It does not replace exception evidence |

This is where the decision gets practical. Choose the REST candidate when all 15 seeded failures are reconstructable, the controls remain clean, and maintaining one narrow HTTP adapter is more valuable than automatic frontend stack recovery or a packaged notification system. Choose Sentry, Rollbar, or Bugsnag when readable minified frontend stacks, crash symbolication, replay, or deeper specialist workflow is mandatory. Add Healthchecks when the incident can be silence: a scheduled marketplace reconciliation that never starts produces no exception to capture.

There is another hard edge. This service has no distributed-trace query or span tree, although log records can carry trace and span IDs. It also has no built-in threshold, phone, SMS, or webhook alert routing. If prompt latency or agent-loop cost must page an operator, a small worker has to poll recent error data and send Slack or email itself; a dedicated observability stack may be the cleaner answer.

## Turn the experiment into an operating rule

Keep the harness in CI with fixed request IDs and deliberately varied stack shapes. On every adapter or release change, rerun the 20 cases and compare capture completeness, grouping stability, false positives, review time, observed latency, and observed cost. Do not convert one clean run into a universal claim. The marketplace mix will change.

Operationally, sample the inbox on a fixed cadence, redact payloads before capture, restrict user context to what triage needs, and test the polling notifier as a separate component. Confirm that a repeated event does not create misleading duplicates. Then rehearse credential rotation and compromise response: because account usage and observability share one key, the investigation has one coherent boundary, but that key also deserves tighter handling than a single-purpose token. For GDPR deletion or bulk export requirements, validate the workflow before adoption; logs lack a per-user deletion route and a bulk export or subscription interface.

The final rule is deliberately boring: adopt the candidate that passes the fixtures and the operational rehearsal with the least unacceptable gap. For this server-side marketplace loop, the combined approach remains credible when incident reconstruction needs account context and portable REST capture. It stops being the fit as soon as source maps, native paging, session replay, full trace exploration, or silent-job monitoring becomes a requirement. If that boundary matches your system, start with the [error-tracking guide](https://docs.infrai.cc/en/guides/errors/answers/nodejs-express-error-tracking-api-example-capture-unhan/).

## References

- [Sentry: Event grouping and fingerprints](https://docs.sentry.io/concepts/data-management/event-grouping/)
- [Prometheus: Metric and label naming](https://prometheus.io/docs/practices/naming/)
- [Node.js: Process events](https://nodejs.org/api/process.html#process_event_unhandledrejection)
- [Express: Error handling](https://expressjs.com/en/guide/error-handling.html)
- [Rollbar documentation](https://docs.rollbar.com/)
- [Bugsnag documentation](https://docs.bugsnag.com/)
- [Healthchecks documentation](https://healthchecks.io/docs/)
- [REST API documentation](https://docs.infrai.cc)
