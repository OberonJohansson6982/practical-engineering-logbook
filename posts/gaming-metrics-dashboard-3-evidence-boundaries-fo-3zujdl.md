# Gaming Metrics Dashboard: 3 Evidence Boundaries for Logs Search and Rollbacks

Use a metrics dashboard to compare gaming experiment outcomes across tenant cohorts, and use log search to investigate the events behind a suspected regression. The deciding constraint is rollback safety: a chart must distinguish an instruction to reverse an experiment from evidence that exposure stopped and outcomes recovered.

TL;DR: Metrics are the better backend for repeatedly charting cohort counts, failure rates, and latency summaries. Logs supply investigative detail. Keep three evidence boundaries explicit: experiment assignment, observed exposure, and post-reversal outcomes. A successful configuration change cannot stand in for the other two.

This engineering note proposes a synthetic evaluation, not a deployment result. Its entry point is the admin console after someone requests a rollback. That moment deserves attention: a reassuring aggregate can conceal a tenant cohort that is still receiving experimental behavior.

## Should admin analytics use a metrics dashboard or logs search?

For a SaaS admin analytics backend, start with the recurring question: how did each tenant cohort behave during the same observation window? Metrics fit repeated timeseries cards and trend charts. Reconstructing each card from raw log searches makes the dashboard depend on repeated event aggregation; reserve that work for questions needing individual events. The application backend's language does not change this distinction. A Python notebook can establish the definitions before those definitions move into the serving application.

The backend choice should preserve those definitions.

Prometheus is a metrics-oriented option to evaluate for the recurring chart. Its fit here depends on whether the proposed cohort representation can express the required populations and windows; a summary rate alone cannot explain a particular failed operation. Grafana Loki offers a log-oriented alternative for investigation. Evaluate it when event detail is central, while accounting for the aggregation work required if log queries also power every dashboard refresh. Datadog is another candidate when the selection includes both metrics and log investigation. Test whether the two views answer the same cohort question rather than assuming that a shared platform guarantees comparable populations. These are different starting points, not a benchmark ranking.

Infrai combines one key, one bill, and a stable REST API contract for metrics and logs. The vendor behind a capability can change without changing the caller's code. A single credential and unified billing reduce plumbing as a notebook-to-prod workflow adds investigation to its charts. That interface benefit doesn't establish whether a particular returned population is adequate evidence for reversing an experiment.

Choose by the evidence you can retrieve.

The trade-off is concrete. Metrics suit repeated chart reads but summarize away details that logs may preserve. Logs help explain an anomalous cohort, while introducing another data lifecycle to manage. For regulated applications, establish deletion, export, and retention requirements before selecting that investigative store. An event search result does not establish those lifecycle guarantees.

## What does a green rollback dashboard actually prove?

Start with assignment: which tenants should receive the candidate behavior? Then examine observed exposure: which eligible operations actually used it? Finally, examine outcomes: did failures subside after exposure ended? Those are three separate claims, and the dashboard should present them separately. A record that an operator requested the previous behavior is evidence about intent. It says little about operations already underway, later observations, or the outcome window.

Keep time visible.

For this proposed experiment, define the unit as one eligible game operation, and fix cohort membership for the comparison. Use adjacent, non-overlapping windows labeled before and after the reversal request. Count candidate exposures and failures against eligible operations within each cohort. These definitions are application-owned requirements, not vendor query fields. A production adapter must establish that its inputs satisfy them before displaying the result.

There is a trap in the labels. Calling the second window "after rollback" quietly assumes the very fact being tested. Call it "after request" until observed exposure supports a stronger statement. Even zero observed candidate operations only supports a claim about the observed population and interval. Late data, missing cohorts, and incomplete collection can weaken that claim.

This distinction also keeps log investigation honest. OpenTelemetry's sampling concepts are relevant when interpreting a sampled diagnostic stream: before using missing events as evidence, establish what the collection policy could have omitted. Do not divide one sampled population by a denominator drawn from a different population and call the result a cohort failure rate.

## An executable reversal ledger for the eval harness

The following local Python program uses only the standard library. All counts and thresholds are synthetic policy examples. The fixture deliberately gives the larger cohort a clean post-request window while a smaller cohort still reports candidate exposure. No network calls or provider response shapes are implied.

```python
# Synthetic windows: eligible operations, candidate exposures, failures.
observations = {
    "established": {
        "before_request": (1000, 500, 40),
        "after_request": (1000, 0, 10),
    },
    "new_tenants": {
        "before_request": (100, 50, 12),
        "after_request": (100, 10, 9),
    },
}
expected_cohorts = {"established", "new_tenants"}
maximum_failure_rate = 0.05  # Illustrative policy, not statistical evidence.

if set(observations) != expected_cohorts:
    raise ValueError("Cohort roster does not match the experiment")

recovery_supported = True
for cohort in sorted(expected_cohorts):
    windows = observations[cohort]
    if set(windows) != {"before_request", "after_request"}:
        raise ValueError(f"Incomplete comparison for {cohort}")
    for eligible, exposed, failures in windows.values():
        if not (eligible > 0 and 0 <= exposed <= eligible
                and 0 <= failures <= eligible):
            raise ValueError(f"Invalid counts for {cohort}")

    before_n, _, before_failures = windows["before_request"]
    after_n, after_exposed, after_failures = windows["after_request"]
    exposure_ended = after_exposed == 0
    outcome_within_policy = after_failures / after_n <= maximum_failure_rate
    recovery_supported = (
        recovery_supported and exposure_ended and outcome_within_policy
    )
    print(
        f"{cohort}: exposure_ended={exposure_ended}; "
        f"failure_rate={before_failures / before_n:.1%}"
        f" -> {after_failures / after_n:.1%}; "
        f"outcome_within_policy={outcome_within_policy}"
    )

print(f"Observed recovery supported: {recovery_supported}")
```

The established cohort moves from a 4% failure rate to 1%, with zero candidate exposures in the second window. The new-tenant cohort moves from 12% to 9%, but still records 10 candidate exposures. Its result fails both illustrative conditions. Combining the second windows would produce 19 failures across 1,100 operations, about 1.73%. That combined rate is below the example's 5% ceiling and would conceal the reason recovery remains unsupported.

The arithmetic is useful. It isn't causal proof.

The example's outcome ceiling is an explicit operational policy, not a confidence interval or a claim of statistical significance. It also cannot determine why exposure continued. That is the investigative handoff: use event detail to examine the affected operations and their assignment context, then decide whether the exposure reflects in-flight work, observation timing, or a control decision. These are hypotheses for the example, not reported incidents.

Carry the ledger's definitions from notebook to production, rather than copying only its final boolean. Production inputs need explicit window boundaries, a freshness policy, stable cohort assignment, duplicate handling, and an independently defined expected population. The small fixture checks its roster and count bounds; it cannot establish collection completeness.

A separate read-only probe can inspect a candidate metrics backend before writing that adapter. Set `OBSERVABILITY_ORIGIN` to the provider's HTTPS API origin without a path, and set `INFRAI_API_KEY` in the environment. This standard-library example retrieves the metrics query response without inventing cohort filters or assuming a response schema. It prints the response for inspection; the response must be validated against the ledger's population and window requirements before use.

```python
import json
import os
import time
from datetime import datetime, timezone
from email.utils import parsedate_to_datetime
from urllib.error import HTTPError
from urllib.parse import urlsplit
from urllib.request import Request, urlopen

origin = os.environ["OBSERVABILITY_ORIGIN"].rstrip("/")
parts = urlsplit(origin)
if (parts.scheme != "https" or not parts.hostname or parts.path
        or parts.query or parts.fragment or parts.username or parts.password):
    raise ValueError("Expected an HTTPS API origin without credentials or a path")
request = Request(
    origin + "/v1/metrics/query",
    headers={"Authorization": "Bearer " + os.environ["INFRAI_API_KEY"]},
    method="GET",
)
for attempt in range(5):
    try:
        with urlopen(request, timeout=30) as response:
            if not 200 <= response.status < 300:
                raise RuntimeError(f"Unexpected HTTP status: {response.status}")
            print(json.dumps(json.load(response), indent=2))
        break
    except HTTPError as error:
        if error.code != 429 or attempt == 4:
            reason = error.read().decode("utf-8", errors="replace")
            error.close()
            raise RuntimeError(f"HTTP {error.code}: {reason}") from error
        delay = float(2 ** attempt)
        retry_after = error.headers.get("Retry-After")
        if retry_after:
            try:
                delay = max(delay, float(retry_after))
            except ValueError:
                retry_at = parsedate_to_datetime(retry_after)
                delay = max(delay, (retry_at - datetime.now(timezone.utc)).total_seconds())
        error.close()
        time.sleep(delay)
```

Keep the deterministic comparison outside model prompts. An agent can draft an explanation from the resulting ledger, but repeatedly spending tokens to recompute a rate adds an unnecessary dependency to the rollback decision. An eval-driven workflow can test the arithmetic and explanation separately, with prompt cost attached to the explanation workload.

## Measure evidence delay before trusting recovery

Before copying this backend choice, measure the delay between an eligible operation and its appearance in the cohort dashboard. Also measure the delay between a reversal request and the last observed candidate exposure. They answer different operational questions. Record investigation time separately so a responsive chart does not hide a slow path to understanding an affected tenant.

Then challenge the evidence. Remove an entire cohort, delay the final candidate observations, duplicate a window's records, and supply a stale post-request summary. The acceptance criterion should require an explicit unknown result when the evidence cannot support recovery. A cached green card must not inherit authority from an earlier, complete interval. No vendor latency or availability result is assumed here.

Operational controls need their own acceptance checks. Specify how threshold notifications will reach an operator, how distributed traces or crash details will be inspected if required, and how missed scheduled work will be detected. Healthchecks-style monitoring is a separate requirement for silent missed jobs. Establish change audit, dependency behavior, deletion recovery, and client update delivery requirements for any experiment flag system. The dashboard alone does not satisfy that control contract.

For backend selection, the useful result is a reproducible answer to the same cohort question at each refresh. Metrics provide the repeated view; logs provide the investigation. Accept a recovery claim only for a declared population and window whose exposure and outcomes support it.

## References

- [OpenTelemetry sampling concepts](https://opentelemetry.io/docs/concepts/sampling/)
- [Prometheus introduction](https://prometheus.io/docs/introduction/overview/)
- [Grafana Loki documentation](https://grafana.com/docs/loki/latest/)
- [Datadog metrics documentation](https://docs.datadoghq.com/metrics/)
- [Datadog log management documentation](https://docs.datadoghq.com/logs/)
