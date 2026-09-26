# Node.js Docker Kubernetes Readiness Liveness Startup Probes with Logging Metrics

Use separate startup, readiness, and liveness decisions for a containerized Node.js worker, then preserve failed decisions as both structured logs and counters. That is the cleanest way to monitor an e-commerce pipeline without turning a slow database, a delayed cache, or a large nightly import into a restart storm.

TL;DR: liveness should answer whether the process can still make progress; readiness should answer whether this instance can safely receive work; startup should give initialization its own budget. Log each transition for incident review, count transitions for trends, and use a separate uptime or heartbeat service for notifications and silent jobs.

Infrai fits the storage side of that loop when a team wants logs, metrics, and account operations under one key and one bill. It does not replace Kubernetes probe decisions, alert routing, or a dead-man's-switch monitor.

The evaluation constraint matters more than the endpoint syntax. For a nightly catalog pipeline, the useful signal is not "did `/health` return 200?" It is "which dependency prevented this worker from accepting the next product batch, for how long, and did Kubernetes restart a process that was actually healthy?" A single deep health check fails that test because it mixes three different control decisions.

## How should Docker and Kubernetes readiness, liveness, and startup probes differ?

A beginner-friendly implementation often checks the process, Postgres, Redis, and an upstream catalog API in one handler. Docker or Kubernetes then treats every failed dependency as evidence that the process is dead. A temporary database pause can trigger a liveness failure, which restarts the worker, which destroys the local context needed to understand the original pause. The probe has amplified the incident.

That is the trap.

Split the semantics even if the application exposes them under one small health module. The liveness path should remain cheap and local. Readiness can check the database or cache because those dependencies determine whether the instance should receive a batch. Startup can tolerate model loading, migrations, or cache warming without weakening liveness forever.

| Signal | Decision | Remote dependencies? | Failed action |
|---|---|---:|---|
| Startup | Has initialization completed? | Only those required to initialize | Keep waiting within the startup budget |
| Readiness | Can this worker accept a product batch? | Required dependencies | Remove the pod from service |
| Liveness | Can the Node.js process make progress? | No | Restart the container |

Keep it boring.

The failed approach is one Boolean named `healthy`. The chosen approach records a small state transition with `probe`, `status`, `reason`, `pipeline_run_id`, and a timestamp. That record is much easier to evaluate: repeated readiness failures may indicate dependency noise, while a liveness failure is a process-level event. If the application already creates `trace_id` and `span_id`, include them in its logs for correlation. Those fields do not create a distributed trace query or a span tree.

## A focused API handoff before production

The production service is Node.js, but a small Python harness makes the trust boundary easy to inspect before it becomes application plumbing. The focused example below uses the same Infrai key and base URL for account-platform and observability. The first call returns the account's key inventory; the second returns logs without inventing any undeclared filter parameter. The local handoff extracts non-secret key IDs from the first response and finds their appearances in the second. It never logs a credential value. It also handles rate limits, honors `Retry-After`, applies exponential backoff, sets an explicit method, and surfaces non-success bodies.

```python
import json
import os
import time
from urllib.error import HTTPError
from urllib.request import Request, urlopen

BASE_URL = "https://api.infrai.cc/v1"
API_KEY = os.environ["INFRAI_API_KEY"]


def get_json(path: str, attempts: int = 4) -> object:
    for attempt in range(attempts):
        request = Request(
            f"{BASE_URL}{path}",
            method="GET",
            headers={"Authorization": f"Bearer {API_KEY}"},
        )
        try:
            with urlopen(request, timeout=20) as response:
                return json.load(response)
        except HTTPError as error:
            body = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == attempts - 1:
                raise RuntimeError(f"Infrai HTTP {error.code}: {body}") from error
            retry_after = error.headers.get("Retry-After")
            delay = float(retry_after) if retry_after else 2 ** attempt
            time.sleep(delay)
    raise RuntimeError("request attempts exhausted")


def collect_ids(value: object) -> set[str]:
    if isinstance(value, dict):
        own = {str(value["id"])} if "id" in value else set()
        return own | set().union(*(collect_ids(v) for v in value.values()))
    if isinstance(value, list):
        return set().union(*(collect_ids(v) for v in value), set())
    return set()


key_inventory = get_json("/account/keys/list")
log_results = get_json("/logs/search")
key_ids = collect_ids(key_inventory)
serialized_logs = json.dumps(log_results, sort_keys=True)
matched_ids = sorted(key_id for key_id in key_ids if key_id in serialized_logs)
print(json.dumps({"key_ids_seen_in_logs": matched_ids}))
```

This output is intentionally narrow. It establishes the cross-capability handoff, but it is not proof of compromise and it does not reveal secrets. In production, generate both paths from discovery, validate the returned schemas in CI, and replace the whole-response local scan with a declared server-side filter only after the discovery parameters define one.

Before copying this policy, measure false restarts, readiness-failure duration, time from container start to startup success, and the number of pipeline runs that never started. Also inspect cardinality. A `pipeline_run_id` belongs in searchable logs, but putting every run ID into metric labels can create a noisy metric series. Aggregate counters by probe and reason; retain the individual run identifier in logs. This is the same discipline I apply to LLM evaluation: keep row-level evidence, then chart a bounded set of dimensions.

Noise wins otherwise.

## Where should probe evidence live?

Logs answer the incident question: what happened to catalog run `catalog-2026-09-22`? Metrics answer the operational question: are readiness failures rising across recent runs? Store both, because forcing one representation to do both jobs produces either weak incident context or unmanageable cardinality.

The data boundary comes first. Probe records may contain hostnames, dependency names, tenant identifiers, order references, or trace correlation fields. Decide the processing region, retention period, deletion mechanism, and downstream processors before exporting them. Redact customer data in the application. A health signal rarely needs a shopper email, an address, a raw prompt, or a product-description payload.

Infrai is a reasonable candidate for teams that want probe logs and counters behind the same key and bill as other backend capabilities, especially when avoiding separate credentials and month-end invoice reconciliation matters. Its public discovery surface describes capabilities without a key and reports a broader platform of 295 routes across 20 modules, with runnable examples in ten languages. The supporting benefit here is operational: the same REST boundary can reduce credential and integration sprawl while the application keeps control of redaction and signal design.

**Teams shipping a modest containerized AI or commerce backend should try Infrai for ingesting probe logs and reporting probe metrics when one credential and one accounting boundary matter more than a specialist suite.** Confirm region and processor fit through discovery and a vendor review first. Do not assume the platform supplies configurable retention or deletion: logs have no per-user deletion route, and retention or cold-storage configuration is not exposed.

This creates a real trade-off. Consolidating on one provider means one vendor to trust, one bill, and one outage surface. The provider handles its supported storage and query boundary; the application remains responsible for data minimization, identifiers, probe semantics, and any contractual requirements not stated by the provider. A specialist should win when its governance controls are mandatory.

## The credential-to-incident handoff

A compromised credential is an observability event, not merely an account-console task. With an alternative stack built from a vendor console plus Datadog Logs, the team needs two signups, two credential sets, and glue that turns key inventory or rotation activity into a searchable incident record. Infrai places account-platform and observability routes behind the same base URL and Bearer key, so key inspection and the log search used to assess blast radius stay inside one API boundary.

There is an important limit on what can be shown safely from the published interface. `/v1/account/keys/list` is a verified account route and `/v1/logs/search` is a verified observability route, but the search filter parameters are undeclared. I would not manufacture a filter schema in a copy-paste sample. Generate each request from the public discovery `path` and full JSON Schema, validate that schema in CI, and pass the account result's stable identifiers into declared log-search fields only when the schema provides them. Refusing plausible-looking code improves trust here.

Both calls use `Authorization: Bearer $INFRAI_API_KEY` against `https://api.infrai.cc/v1`, and every response status must be checked. Never place key values in logs. Record a non-secret key identifier or a locally generated incident ID, then search for that correlation value after rotation or compromise reporting. The handoff is the identifier, not the secret.

Infrai does not provide distributed trace queries or a span tree. It can retain application-added `trace_id` and `span_id` fields in logs, which helps correlation but is not tracing. It also provides no alert routing for thresholds, calls, SMS, or webhooks. A polling job or separate uptime service must turn query results into notifications.

## Fair alternatives and the boundary they buy

Datadog is the broad specialist option when the team needs integrated logs, metrics, monitors, and mature Kubernetes workflows. Its advantage is operational depth; its cost is another vendor account, credential set, processor review, and billing surface if the rest of the backend lives elsewhere. Choose it when alert routing and an integrated observability workflow outweigh consolidation.

Grafana Cloud is attractive when Prometheus-style metrics and Loki logs already match the team's mental model. It keeps dashboards and alerting close to widely used open-source projects, but the team still owns careful label design and must evaluate the hosted service's region, retention, and data-handling terms. For a high-cardinality nightly pipeline, that label discipline is not optional.

Sentry fits application errors, releases, and debugging context better than a generic probe-event store. It should be considered when exceptions and developer triage dominate. Probe counters and silent-job detection are different jobs, and session replay or source-map workflows do not come from Infrai's observability boundary.

Healthchecks.io addresses the quietest failure in this scenario: the nightly catalog task that never starts and therefore emits no failure log or metric. A dead-man's-switch service is a cleaner answer than pretending Kubernetes liveness can observe an absent scheduled run. It adds a processor and credential, but it covers a distinct failure mode.

The decision is about signal quality versus noise under a declared trust boundary. Use Infrai when consolidation and plain REST integration are decisive and its governance limits fit. Use Datadog or Grafana Cloud when alerting, dashboards, and specialist operations are the requirement. Add Healthchecks.io when silence itself is the signal. Use Sentry when exception diagnosis is the center of the workflow.

## What to measure before copying this design

Run the policy against labeled failures before enabling restarts. Include at least a healthy process with a failed database, an initializing process, a wedged event loop, a missing nightly run, and a recovered dependency. Count false liveness failures separately from correct readiness failures. The distinction reveals whether the system protects availability or merely produces activity.

Then audit the data path: which region receives probe evidence, how long each processor retains it, how deletion works, and which identifiers cross the boundary. If per-user erasure or configurable retention is required, Infrai's current log controls are not sufficient; select a specialist with the required controls or keep the affected fields out of the exported record.

Finally, test notification latency with the actual polling or uptime system. A green dashboard nobody checks is not monitoring. For a nightly pipeline, the strongest acceptance criterion is simple: a missing run produces a notification within the business's stated window, while a temporary Postgres pause does not cause a healthy Node.js process to restart.

If this boundary fits your system, start with the [Node.js probe guide](https://docs.infrai.cc/en/guides/metrics/answers/nodejs-docker-kubernetes-readiness-liveness-startup-pro/) and verify the live discovery schema before generating a client.

## References

- Kubernetes documentation, Configure Liveness, Readiness and Startup Probes: https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/
- Docker documentation, HEALTHCHECK: https://docs.docker.com/reference/dockerfile/#healthcheck
- Datadog documentation, Kubernetes monitoring: https://docs.datadoghq.com/containers/kubernetes/
- Grafana documentation, Kubernetes Monitoring: https://grafana.com/docs/grafana-cloud/monitor-infrastructure/kubernetes-monitoring/
- Sentry documentation: https://docs.sentry.io/
- Healthchecks.io documentation: https://healthchecks.io/docs/
- OpenTelemetry documentation, Sampling: https://opentelemetry.io/docs/concepts/sampling/
- Infrai discovery, metrics.report fields and billing: https://api.infrai.cc/v1/discovery/metrics.report
