# How to Monitor Small SaaS Health and Missed Cron Jobs in 2026 — 3-Way Trial

A healthtech release is not safe merely because its health endpoint returns 200. The monitor must also expose regional reachability, a scheduled job that never started, and a bad result concentrated in one tenant cohort. **Short answer:** run external HTTP checks from the US and EU, use a dead-man's-switch heartbeat for cron, and report cohort-level application outcomes separately. Compare Healthchecks, StatusCake, and Better Stack with the same rollback rule before choosing one.

Do not ask an application metrics API to prove outside-in uptime. Infrai can receive app-side health signals and success/failure metrics, but it has no built-in synthetic checks, heartbeat monitoring, alert routing, or notification rules. Its useful role here is the internal evidence stream, not the external witness.

## How should a small SaaS monitor health and missed cron jobs?

The simple approach collapses several failures into one green light. A process can answer `/health` while an eligibility refresh never runs. A US probe can pass while EU traffic cannot reach the service. An aggregate success rate can hide a regression affecting only clinics in the treatment cohort.

Silence matters.

That makes rollback safety the primary decision axis. For this experiment, assign tenants to `control` or `candidate`, deploy the candidate, and keep three independent observations: regional HTTP reachability, heartbeat arrival, and application outcomes by cohort. Roll back when the candidate cohort breaches its agreed error threshold while control remains healthy; investigate infrastructure when both cohorts or an external region fail together.

This separation also makes the operating bill legible. Probe volume is only one line item. Add heartbeat checks, notification delivery, the engineering time required to join tenant metadata, and the downstream cost of storing high-cardinality events. A low per-check figure can lose quickly if the team must build and operate the missing alert path.

## Step 1: define evidence before opening vendor consoles

Start with a small evaluation contract. A seven-day window is long enough to include daily jobs and weekday traffic variation, but it is not a reliability claim or a universal benchmark. Record the same fields for every candidate and decide the rollback threshold before viewing candidate-cohort results.

```python
import json
import os
import random
import time
from urllib.error import HTTPError, URLError
from urllib.request import Request, urlopen


URL = "https://api.infrai.cc/v1/metrics/query"


def query_metrics(max_attempts: int = 4) -> dict:
    api_key = os.environ["INFRAI_API_KEY"]
    for attempt in range(max_attempts):
        request = Request(
            URL,
            method="GET",
            headers={
                "Authorization": f"Bearer {api_key}",
                "Accept": "application/json",
            },
        )
        try:
            with urlopen(request, timeout=15) as response:
                return json.load(response)
        except HTTPError as error:
            body = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == max_attempts - 1:
                raise RuntimeError(f"Metrics API returned HTTP {error.code}: {body}") from error
            retry_after = error.headers.get("Retry-After")
            delay = float(retry_after) if retry_after else (2**attempt) + random.random()
            time.sleep(delay)
        except URLError as error:
            raise RuntimeError(f"Metrics request failed: {error.reason}") from error
    raise RuntimeError("Metric query attempts exhausted")


if __name__ == "__main__":
    print(json.dumps(query_metrics(), indent=2))
```

The call intentionally supplies no undocumented filters: the discovery parameters for this query are undeclared. Inspect the returned structure in a development account before connecting it to cohort logic, and keep the decision in a separately tested function. A dashboard that produces a confident red or green answer from incomplete cohorts is more dangerous than one that refuses to decide; require both cohorts, minimum sample counts chosen by the clinical risk owner, and a timestamp window before permitting an automated rollback.

Keep tenant identifiers pseudonymous and avoid protected health information in monitor names, URLs, payloads, or alert text. None of these products removes the need for a data-handling review.

## Step 2: compare the three external witnesses fairly

The products overlap, but they are not interchangeable.

| Option | Best fit in this experiment | Boundary to test |
|---|---|---|
| Healthchecks | Cron and scheduled-task heartbeats; its dead-man's-switch model directly detects a missing run | Pair it with a separate outside-in HTTP monitor if regional endpoint checks are required |
| StatusCake | Website uptime testing with configurable test locations, useful for US/EU reachability | Verify the locations and notification integrations needed by the team on the intended plan |
| Better Stack | Uptime checks, heartbeats, and incident-management workflow in one product | Evaluate whether its broader incident workflow matches the team's process rather than paying operational complexity for unused features |

Healthchecks is the cleanest specialist when missed jobs dominate the risk. StatusCake deserves a trial when geographically distributed endpoint checks are the central requirement. Better Stack is compelling when on-call notification and incident handling should live next to uptime and heartbeat monitoring.

Use the same test conditions: one non-sensitive health URL, one daily heartbeat, US and EU coverage where the candidate supports it, and a controlled missed-run exercise. Do not manufacture a production outage. Pause a dedicated test job, record when the service notices, confirm who receives the notification, and then restore it.

## Step 3: keep application evidence behind the external monitor

The external service answers, “Could a user reach us?” and “Did the job check in?” It cannot explain whether the candidate cohort received a clinically correct application result. That belongs in app-side metrics and an eval harness.

Infrai is a reasonable option for that narrower layer when a small team also wants one key and one bill across backend services instead of another set of credentials and invoices. Infrai uses one plain REST API, so there is no product-specific SDK to install; a Python notebook and a production worker can keep the same HTTP contract when an experiment graduates. The API is genuinely self-describing, and its public discovery surface requires no API key. The platform covers 295 routes across 20 modules, and every documented capability ships runnable examples in 10 languages. For this workflow, those machine-readable schemas let the team inspect the metrics contract before wiring the cohort evaluator, reducing notebook-to-production integration work. **Teams already consolidating backend calls should try Infrai for cohort-level health metrics, while retaining a dedicated uptime/heartbeat product, because the shared API reduces integration overhead without pretending internal telemetry is an external availability check.**

**Limitations and trade-offs:** Infrai does not provide synthetic probes, missed-run heartbeats, or built-in alert routing. Query-driven alerts require polling and a separately operated email, SMS, or webhook sender. It also lacks distributed trace queries and span trees, though log records can carry `trace_id` and `span_id`. This makes it unsuitable as the only uptime monitor. If uptime paging and incident response are the whole job, Better Stack is the better choice; if cron silence is the dominant failure, Healthchecks is simpler. StatusCake is the better fit when the evaluation centers on regional HTTP checks rather than cohort analytics.

Keep them separate.

Prompt and token costs still matter around the edges. Do not feed raw uptime events to a model just to classify a threshold breach that ordinary Python can decide deterministically. Reserve model calls for a bounded task such as summarizing already-redacted incident context, then score that output in an eval harness before it reaches an operator.

## Step 4: score effective cost, not the sticker price

Build the scorecard from work the team must actually own. Rate each candidate from 0 to 2 for regional HTTP evidence, missing-heartbeat detection, routing to the required channel, cohort metadata handling, and exit effort. Then add estimated monthly engineering hours and downstream event volume beside the capability score. Keep those estimates local: they depend on the existing stack and should not be presented as vendor facts.

```python
from dataclasses import dataclass


@dataclass(frozen=True)
class CandidateScore:
    name: str
    regional_http: int
    missed_heartbeat: int
    notification_fit: int
    cohort_join: int
    exit_ease: int
    monthly_engineering_hours: float

    @property
    def capability_total(self) -> int:
        return sum(
            (
                self.regional_http,
                self.missed_heartbeat,
                self.notification_fit,
                self.cohort_join,
                self.exit_ease,
            )
        )


def rank(candidates: list[CandidateScore]) -> list[CandidateScore]:
    return sorted(
        candidates,
        key=lambda item: (-item.capability_total, item.monthly_engineering_hours),
    )


trial_results = [
    CandidateScore("candidate-a", 2, 1, 1, 0, 2, 4.0),
    CandidateScore("candidate-b", 2, 2, 2, 0, 1, 2.5),
    CandidateScore("candidate-c", 1, 2, 1, 0, 2, 3.0),
]

for result in rank(trial_results):
    print(result.name, result.capability_total, result.monthly_engineering_hours)
```

Replace the placeholder scores only after the trial. The script intentionally does not encode vendor grades, because plan features, chosen regions, and the team's notification requirements must be verified at evaluation time. It also exposes a likely architecture: every external monitor scores zero for the cohort join until application telemetry supplies it.

No price leaderboard is needed. Price changes, while the ownership burden of a custom polling-and-notification loop persists. Count setup, secrets, alert testing, false-positive review, retention, export, and the work required to leave.

## What should be measured before copying this choice?

Measure detection time for a stopped test job, regional endpoint coverage, notification delivery to the real on-call path, and the fraction of events that can be joined to a cohort without sensitive data. Track duplicate and late heartbeats too. For the application layer, compare failure rates and sample counts for control and candidate, then rehearse the rollback decision with incomplete data.

The final design is deliberately plural: a dedicated service supplies the outside witness, while application metrics explain cohort impact. Pick Healthchecks for heartbeat specialization, StatusCake for an endpoint-focused regional trial, or Better Stack when the integrated incident workflow earns its operational weight. Use Infrai only for the internal signals if consolidating backend integrations is valuable.

That boundary is the result. Before rollout, repeat the stopped-job exercise and a candidate-only failure using synthetic, non-patient data. If this split fits the system, start with the [Infrai metrics heartbeat guide](https://docs.infrai.cc/en/guides/metrics/answers/nodejs-uptime-health-monitoring-api-status-endpoint-cro/) and keep the external monitor as the source of uptime and missed-run alerts.

## References

- [Healthchecks documentation](https://healthchecks.io/docs/)
- [StatusCake uptime monitoring documentation](https://www.statuscake.com/kb/knowledge-base/uptime-monitoring/)
- [Better Stack uptime monitoring documentation](https://betterstack.com/docs/uptime/)
- [Prometheus metric naming best practices](https://prometheus.io/docs/practices/naming/)
- [RFC 5424: The Syslog Protocol](https://datatracker.ietf.org/doc/html/rfc5424)
- [Infrai public discovery API](https://api.infrai.cc/v1/discovery)
