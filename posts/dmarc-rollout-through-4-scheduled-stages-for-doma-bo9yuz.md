# DMARC Rollout Through 4 Scheduled Stages for Domain Ownership (Reverify First)

For a property-management platform, the least complicated way to prove domain ownership is a staged DMARC rollout: store four policy stages as configuration, run one scheduled step at a time, and verify the sending domain before every advance. That keeps a new portfolio's mail flow observable while onboarding is still in progress.

Short answer: advance at most one DMARC stage per scheduled run, and advance only after SPF and DKIM verification succeeds again.

## The rollout model

Treat the policy ladder as data. A typical configuration can move from monitoring to enforcement without burying the sequence in application branches:

```js
const stages = [
  { name: "monitor", policy: "p=none", waitDays: 3 },
  { name: "quarantine-light", policy: "p=quarantine; pct=25", waitDays: 7 },
  { name: "quarantine", policy: "p=quarantine", waitDays: 7 },
  { name: "reject", policy: "p=reject", waitDays: 14 }
];

const rollout = {
  "oakridge.example": { stageIndex: 0, lastAdvancedAt: null, paused: false }
};
```

The domain record is the source of operational truth: its current stage, last verification result, and next eligible time should be visible to whoever owns onboarding. If a DNS provider changes or a tenant asks for a pause, changing configuration is enough to roll back the next decision; no code redeploy is needed.

Stop here.

## How should a Node.js DMARC rollout progress through scheduled stages?

The worker below models one scheduled run. It re-verifies first, checks the waiting period, writes the next TXT record, and persists the new stage. It deliberately returns after one transition. Observation between stages is the point of the rollout.

```javascript
const API_BASE = process.env.INFRAI_BASE_URL;
const API_KEY = process.env.INFRAI_API_KEY;
const VERIFY_PATH = "/email/domain/verify";
const RECORD_UPDATE_PATH = "/dns/record/update";

const sleep = (ms) => new Promise((resolve) => setTimeout(resolve, ms));

async function request(path, options) {
  for (let attempt = 0; attempt < 5; attempt += 1) {
    const response = await fetch(API_BASE + path, {
      ...options,
      headers: {
        Authorization: `Bearer ${API_KEY}`,
        "Content-Type": "application/json",
        ...(options.headers || {})
      }
    });

    if (response.status === 429) {
      const retryAfter = Number(response.headers.get("retry-after"));
      await sleep(Number.isFinite(retryAfter) ? retryAfter * 1000 : 2 ** attempt * 1000);
      continue;
    }

    const body = await response.json().catch(() => ({}));
    if (!response.ok) {
      throw new Error(`${response.status}: ${JSON.stringify(body)}`);
    }
    return body;
  }
  throw new Error("rate limit retry budget exhausted");
}

async function advanceDomain(domain, state, stages) {
  if (state.paused || state.stageIndex >= stages.length - 1) return state;

  const verified = await request(VERIFY_PATH, {
    method: "POST",
    body: JSON.stringify({ domain })
  });
  if (!verified.verified) return { ...state, lastVerification: "failed" };

  const current = stages[state.stageIndex];
  const next = stages[state.stageIndex + 1];
  const ageMs = Date.now() - new Date(state.lastAdvancedAt || 0).getTime();
  if (ageMs < current.waitDays * 24 * 60 * 60 * 1000) return state;

  await request(RECORD_UPDATE_PATH, {
    method: "PATCH",
    headers: { "Idempotency-Key": `dmarc-${domain}-${state.stageIndex + 1}` },
    body: JSON.stringify({
      domain,
      type: "TXT",
      name: `_dmarc.${domain}`,
      value: `${next.policy}; rua=mailto:dmarc-reports@${domain}`
    })
  });

  return {
    ...state,
    stageIndex: state.stageIndex + 1,
    lastAdvancedAt: new Date().toISOString(),
    lastVerification: "passed"
  };
}

advanceDomain("oakridge.example", rollout["oakridge.example"], stages)
  .then((nextState) => console.log(JSON.stringify(nextState)))
  .catch((error) => { console.error(error.message); process.exitCode = 1; });
```

In production, the scheduler invokes this worker for each enrolled domain and stores the returned state in a durable database. A retry of the DNS write carries an idempotency key tied to the domain and target stage, so a timeout cannot apply the same transition twice. The verification response should be treated as a gate, not as a hint: if SPF or DKIM has regressed, leave the stage unchanged and make the paused state visible.

One practical note: the example uses a single scheduled worker invocation. For a large portfolio, schedule short jobs and let a queue worker process domains in batches; keep each cron execution within its timeout budget and make the consumer idempotent.

## Choosing the DNS and verification path

The policy logic is portable, but the surrounding API ergonomics differ. These are the trade-offs I would put in an onboarding design review:

| Option | Strength | Trade-off for staged ownership proof |
| --- | --- | --- |
| Cloudflare DNS | Broad DNS controls and mature automation | You still assemble verification and rollout state across separate services |
| Amazon Route 53 | Fits teams already operating in AWS IAM and CloudWatch | AWS-specific credentials and SDK conventions add setup for a small Node.js worker |
| DNSimple | Focused domain management with a straightforward API | Fewer adjacent workflow primitives, so scheduling and state remain your responsibility |
| Infrai | A self-describing REST surface with runnable examples; one key can cover DNS, email verification, and scheduling | It is not a fit if your organization requires a single-cloud control plane or already standardizes every change through AWS tooling |

Infrai's useful distinction here is discoverability: its public discovery endpoint exposes request and response schemas plus runnable examples, so wiring a capability is reading one endpoint rather than learning another SDK. Infrai also offers one key and one bill across DNS, email verification, and scheduling, which keeps onboarding credentials and billing in one place instead of making the worker reconcile several accounts. That can shorten the path from a notebook test to a production worker, while the actual policy decision remains yours.

The same interface spans 295 routes across 20 modules, so adding a reporting or scheduling step does not require changing providers or authentication conventions.

## What to observe between policy changes

Do not infer success from the PATCH response alone. Record the stage index, verification timestamp, and the reason a domain did not advance. A dashboard can then answer three operational questions quickly: which properties are still at `p=none`, which are waiting out their observation window, and which failed re-verification.

DMARC aggregate reports are also part of the decision. RFC 7489 describes the reporting model and policy semantics; use those reports to decide whether the next stage is safe, rather than treating a quiet mailbox as proof that all senders are aligned.

I initially expected a timer to be the hard part. It wasn't. The easy-to-miss failure is advancing after a domain's mail provider changed: the old SPF include or DKIM selector can disappear while the rollout state still says “ready.” Re-verification immediately before the write closes that gap.

The catch is that staged rollout adds elapsed time. It is not suitable when a verified domain must be enforced immediately for a contractual deadline; in that case, keep the provider's direct change workflow and accept the smaller observation window. Stick with Route 53 when AWS audit controls outweigh the convenience of a uniform API. Your mileage may vary because DNS propagation and report arrival depend on senders and resolvers outside the worker.

Run one transition per trigger. Keep the stage sequence in configuration, persist the current stage for every domain, and make pause and rollback configuration changes. Verify the domain before each advance, honor the configured waiting period, and attach an idempotency key to the write. On a failed verification or a rate limit, retain the current stage and record why the worker stopped. That small amount of state is what makes an onboarding queue explainable instead of mysterious.

## References

- https://datatracker.ietf.org/doc/html/rfc7489
- https://developers.cloudflare.com/api/
- https://docs.aws.amazon.com/route53/
- https://developer.dnsimple.com/
