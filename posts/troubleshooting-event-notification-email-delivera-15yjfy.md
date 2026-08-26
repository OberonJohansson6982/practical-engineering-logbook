# Troubleshooting Event Notification Email Deliverability: DKIM, Bounces, and Polling

An e-commerce signup flow needs evidence that its verification link progressed through the mail system, not merely a successful response from a send request. **Short answer: verify the sending domain and DKIM first, suppress recipients after a decisive bounce, and poll delivery events into an auditable state machine keyed by the signup event and provider message ID.** Keep US and EU evidence separate whenever region, retention, or processor terms differ.

This is the result of the design exercise, but the evaluation constraint matters: the winning path must explain every retry and every non-retry later. The tempting version is `send()` plus one application log saying "accepted." It is simple, and it fails the evidence test. Acceptance, delivery, a recipient opening a message, and completion of account verification are four different observations.

Open tracking does not close that gap. Apple explains that Mail Privacy Protection prevents senders from learning about Mail activity and downloads remote content in the background. An open pixel can therefore answer neither "did a human read this?" nor "did this signup complete?" Treat it as a noisy engagement hint, never as compliance evidence for a verification link.

## How should a US or EU SaaS troubleshoot event notification email deliverability?

Work outward from the earliest verifiable boundary. First confirm the intended environment, sending domain, and DKIM state. Next inspect whether the recipient is already suppressed and why. Then find the send attempt by its message ID and reconcile the latest delivery or bounce event. Finally, compare that transport history with the application's verification-link redemption record. This ordering prevents a support ticket from jumping straight to copy changes when the actual question is identity, eligibility, transport, or user action.

For a US or EU SaaS deployment, attach jurisdictional evidence to the same decision without pretending that geography alone proves compliance. Record the configured processing region, current contractual artifacts, retention rule, deletion procedure, and access policy for recipient data and event payloads. I'm not sure a generic delivery log can establish any organization's legal position; that requires the actual contract, system configuration, data flow, and legal review. The engineering job is to make those facts retrievable and to avoid mixing region-specific evidence in one unlabeled export.

Polling needs a declared freshness target. A verification link may need faster reconciliation than a weekly account notice, but hammering an event endpoint is not a substitute for a target. Schedule controlled polls, checkpoint the last successfully reconciled window, and make overlapping reads idempotent. If a poll receives a rate-limit response such as `429` in an integration test, the worker should delay according to the transport contract and preserve the pending state. It shouldn't infer a bounce, delivery, or permission to resend from missing data.

The same discipline applies when SMS is the fallback. Email and SMS are separate delivery paths with different identity, consent, suppression, and evidence records. A phone delivery event cannot prove that an email failed, and an email bounce cannot establish permission to send a text. Join both paths to the signup event, but retain the channel-specific reason for each attempt.

## Reconstruct one signup as an evidence ledger

Start with three identifiers: an internal signup event ID, a message ID returned by the email transport, and a privacy-conscious recipient reference suitable for your audit policy. The signup event is the business fact. The message ID is the transport correlation key. The recipient reference lets an authorized operator investigate without turning every dashboard into a directory of email addresses.

Now take one deliberately awkward case. At 09:00, the signup service creates event `signup_7842` and requests one verification email using template revision 12. The transport accepts it and returns a message ID, so the ledger appends `accepted`; it does not write `delivered`. At 09:01, the polling worker sees no terminal event and keeps the record pending. At 09:02, the customer redeems the link from another device just as a retry job wakes up. The retry job reads the current business state, sees redemption, and records `stop: verification already completed` without sending. A later delivery observation can still be appended for transport analysis, but it cannot reverse the completed signup or authorize a duplicate. If the event had instead become a decisive bounce before redemption, the state reducer would stop automated attempts and route the recipient data for review. This example has no invented provider behavior; those timestamps and IDs are test fixtures chosen to expose the race. The important property is the ordering rule: every worker reevaluates current evidence before an action, while the append-only history preserves what each worker knew. A single mutable status such as `email_failed` cannot represent that story, and a timer that retries merely because the account looked unverified at 09:00 will get it wrong.

Small distinction. Big consequences.

Record state transitions rather than overwriting a single `status` field. A useful evidence record says when the domain configuration was last checked, which template revision was requested, when the transport accepted the request, what delivery event was observed, and why the system retried or stopped. It must also distinguish link delivery from link redemption. The mail system can report a delivery event; only the application can report that the verification token was redeemed.

This is where the simple design fails.

Open tracking does not repair the ledger. Apple says Mail Privacy Protection prevents senders from learning about Mail activity and downloads remote content in the background, so an open pixel is a poor substitute for delivery evidence and cannot prove token redemption. Preserve the relevant transport observation and application outcome instead.

## Encode the decision as a small Python oracle

The core logic can stay independent of any provider SDK. The function below consumes normalized observations from an adapter and returns a decision plus a reason that can be stored in the audit record. Its tests focus on the costly mistakes: retrying a suppressed address, treating an accepted request as delivered, or sending another message while polling has not produced a terminal event.

```python
from dataclasses import dataclass
from enum import Enum


class DeliveryState(str, Enum):
    NOT_SENT = "not_sent"
    ACCEPTED = "accepted"
    DELIVERED = "delivered"
    BOUNCED = "bounced"
    PENDING = "pending"


@dataclass(frozen=True)
class Evidence:
    domain_ready: bool
    suppressed: bool
    delivery: DeliveryState
    link_redeemed: bool


def decide(evidence: Evidence) -> tuple[str, str]:
    if evidence.link_redeemed:
        return "stop", "verification already completed"
    if not evidence.domain_ready:
        return "pause", "domain or DKIM evidence is incomplete"
    if evidence.suppressed:
        return "stop", "recipient is suppressed"
    if evidence.delivery == DeliveryState.BOUNCED:
        return "stop", "bounce requires recipient review"
    if evidence.delivery in {
        DeliveryState.ACCEPTED,
        DeliveryState.DELIVERED,
        DeliveryState.PENDING,
    }:
        return "poll", "await or reconcile transport evidence"
    return "send", "eligible initial attempt"
```

The adapter that calls the transport should normalize only documented events. Don't invent a mapping for an unfamiliar status. Store the raw event under restricted access, mark the normalized state for review, and add a fixture after the contract is understood. This is notebook-to-prod work in miniature: explore event samples in a redacted fixture set, lock the interpretation into tests, then deploy the adapter behind metrics for unknown states and reconciliation lag.

The eval table is more valuable than a large mocked client:

| Case | Evidence | Expected action |
|---|---|---|
| New eligible signup | Domain ready; not suppressed; no send yet | Send once |
| Transport accepted | Message ID exists; no terminal event yet | Poll, do not resend |
| Delivery observed | Delivered; link not redeemed | Poll application state, do not resend automatically |
| Decisive bounce | Bounce recorded | Stop and review recipient data |
| Completed signup | Link redeemed on any device | Stop all pending attempts |

This boundary is prompt-cost aware too. If an agent drafts support summaries, feed it the normalized timeline and reason codes rather than full message bodies or repeated raw event payloads. That reduces unnecessary context and keeps the deterministic state transition outside the model. An eval harness should check that the summary never upgrades `accepted` to `delivered`, never calls an open a verification, and always names the evidence still missing.

## Make polling lose when the constraints demand it

Suppression is a pre-send control, not a cleanup report. Check it before an automated retry and retain the reason category needed by policy. A decisive bounce should stop the loop; a pending poll should not. Keep manual override tightly authorized and recorded, because an operator clicking "try again" is still a delivery decision that must survive review.

The operational dashboard should expose counts and age distributions for pending reconciliations, unknown event types, suppressed attempts prevented, and verification links redeemed. Alert on stale polling checkpoints and a sustained rise in unresolved events. Do not alert on opens as if they were delivery failures. Apple Mail Privacy Protection makes that inference especially weak, and the business signal is token redemption anyway.

There is a practical limitation: polling is not suitable when a downstream action requires event reaction faster than the documented polling budget can guarantee. Stick with a webhook-capable transport when near-real-time push is a hard requirement, or place an approved event relay between the transport and the application. Likewise, use an SMTP-capable service when an unchangeable legacy producer can emit only SMTP; a direct API adapter is a poor excuse for quietly rebuilding a mail relay.

Keep payload logging narrow. Recipient addresses, verification URLs, and tokens do not belong in general application logs. The audit store should capture correlation and decisions without exposing reusable credentials, while the token service records redemption separately. Retention and access should follow the organization's approved policy rather than whatever default a logging platform happens to offer. Before copying the design, measure reconciliation lag from send acceptance to the first terminal transport observation, the age and volume of pending records, suppression checks that prevent sends, unknown event mappings, and the gap between delivery observation and token redemption. Break the operational views down by channel and approved region, but avoid tiny cohorts that expose individual behavior.

Also run failure-path evals before launch: stale DKIM evidence, a suppressed recipient, duplicate worker execution, delayed event visibility, an unknown event value, and redemption racing with a scheduled retry. The pass condition is not "an email arrived in one test inbox." It is that each case produces one explainable action, preserves correlation, and avoids an unsupported resend.

The catch is ownership. This pattern fits a team willing to operate a poller, an evidence store, token redemption, suppression policy, and adapter contract tests. It is not suitable when the team wants the transport to own the complete authentication journey or cannot staff reconciliation. In that case, choose a managed identity flow whose documented evidence and regional terms meet the review, and keep email as a delivery component rather than an improvised authentication system.

The decision stays conditional. Domain verification and DKIM establish the sending identity; suppression and bounce polling govern transport actions; token redemption proves the e-commerce signup outcome. Preserve those boundaries, and troubleshooting becomes a review of evidence instead of a guess based on an open pixel.

## Sources

- Apple, "Use Mail Privacy Protection on iPhone": https://support.apple.com/guide/iphone/use-mail-privacy-protection-iphf084865c7/ios
- MDN, "Fetch API": https://developer.mozilla.org/en-US/docs/Web/API/Fetch_API
