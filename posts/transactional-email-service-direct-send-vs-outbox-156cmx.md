# Transactional Email Service: Direct Send vs Outbox for Startup Receipts

The operational constraint is payment state: once a B2B SaaS order is settled, the receipt must remain sendable even if the transactional email service API is slow or unavailable. **Choose a transactional outbox over a direct API call** when a missed receipt is less acceptable than a little extra worker and database code. Direct send is still the easier fit for a startup prototype where payment state can be replayed safely and nobody treats the first response as proof of delivery.

TL;DR: commit the settled payment and a receipt job together, return success, then let a worker send through a replaceable HTTP adapter. Evaluate the design with duplicate events, timeouts after acceptance, permanent recipient failures, and worker restarts. The cheapest-looking API is irrelevant if the application can lose the intent to send.

## Should a startup transactional email service send onboarding receipts directly?

A direct call makes the request path pleasantly short: mark the order paid, render a receipt, call an email API, and report success. The trouble lives between those verbs. If the database commit succeeds and the process stops before the API call, the order is settled but no receipt intent exists. If the provider accepts the message and the client times out before receiving the response, retrying may create a duplicate. An acceptance response, where offered, describes acceptance by an API rather than inbox delivery.

Payment completion and message delivery are separate state machines. Treating them as one synchronous action hides that fact; it does not remove it.

That is the trade-off.

This is the point where notebook-to-production instincts matter. A happy-path request proves serialization and authentication. It says nothing about crash recovery. My evaluation target would be an invariant instead: every settled order has exactly one durable receipt intent, while sending is at least once and the visible message is duplicate-tolerant.

Email authentication is another independent layer. SPF lets a receiving system check whether a host is authorized to use a domain in the SMTP `MAIL FROM` identity. It does not make an application retry safe, and it does not prove that a receipt reached a person's inbox. The architecture has to keep application evidence separate from transport and domain-authentication evidence.

## The focused implementation

The smallest useful design needs two tables in one database transaction: the order transition and an outbox row. A worker claims pending rows, calls a generic HTTPS email adapter, records the provider's opaque message identifier when available, and retries only failures classified as transient.

Here is a deliberately narrow Python sketch. It shows the boundary, not a vendor SDK. The unique `event_key` turns a repeated payment-settled event into the same durable intent.

```python
from dataclasses import dataclass
from typing import Protocol


@dataclass(frozen=True)
class Receipt:
    order_id: str
    recipient: str
    amount_minor: int
    currency: str


class EmailPort(Protocol):
    def send_receipt(self, receipt: Receipt, idempotency_key: str) -> str:
        """Return an opaque acceptance identifier or raise a typed error."""


def record_settlement(db, receipt: Receipt) -> None:
    event_key = f"receipt:{receipt.order_id}"
    with db.transaction() as tx:
        tx.execute(
            "UPDATE orders SET status = ? WHERE id = ? AND status != ?",
            ("settled", receipt.order_id, "settled"),
        )
        tx.execute(
            """
            INSERT INTO email_outbox
                (event_key, order_id, recipient, amount_minor, currency, status)
            VALUES (?, ?, ?, ?, ?, ?)
            ON CONFLICT(event_key) DO NOTHING
            """,
            (
                event_key,
                receipt.order_id,
                receipt.recipient,
                receipt.amount_minor,
                receipt.currency,
                "pending",
            ),
        )


def deliver_one(db, email: EmailPort, row) -> None:
    receipt = Receipt(
        order_id=row.order_id,
        recipient=row.recipient,
        amount_minor=row.amount_minor,
        currency=row.currency,
    )
    acceptance_id = email.send_receipt(receipt, row.event_key)
    db.mark_sent(row.event_key, acceptance_id)
```

The adapter's idempotency key is useful only if the selected API defines and honors such a mechanism. The database uniqueness constraint remains the application's control. After an ambiguous timeout, the worker should reconcile using the provider's documented semantics rather than assume the message was rejected.

Keep rendering deterministic. Store the template revision and business inputs with the outbox record, or store the final rendered content if retention policy permits. Otherwise a retry after a template edit can send a materially different receipt for the same order. That is a subtle production failure because both versions may be individually valid.

One warning: do not put an access token, password, or other reusable authenticator in the receipt. NIST's authenticator guidance is a useful baseline for separating authenticator handling from ordinary notification content. A receipt can link to an authenticated account area; the email itself should not become the account's durable proof of identity.

## Direct call or outbox?

The choice is asymmetric. Direct sending removes a table, a worker, claiming logic, and operational metrics. That simplicity is real, so I would retain it for an internal experiment whose settled events are replayable and whose users do not depend on immediate receipts. It also works when the upstream system already provides a durable workflow with equivalent retry and deduplication guarantees.

For a production order flow, the outbox wins because it preserves intent across process failure and makes backlog visible. It does not promise exactly-once email delivery. No application-side transaction spans a local database, an external API, the receiving mail system, and a user's inbox. The practical goal is a durable intent plus controlled retries, duplicate-resistant content, and evidence for reconciliation.

The cost is operational ownership. A worker can stall. Rows can be claimed forever unless leases expire. Bad addresses can cycle endlessly unless permanent failures move to a terminal state. Schema retention can quietly accumulate recipient data. These are inspectable problems, which is precisely why I prefer them to an invisible gap between a commit and a network call.

**Use the outbox when the receipt is part of the order's operational record.** Use direct sending only when replayability and low consequence make its failure window acceptable.

The limitation is equally concrete: an outbox is not a fit for a throwaway onboarding demo, and it may duplicate durability already supplied by a workflow engine. A small team then owns polling or notifications, claim leases, retry classification, retention, dashboards, and deployment of another process. If those controls have no operator and no alert path, the design merely relocates uncertainty into a table. Direct send is a defensible choice in that narrow case, provided the application records enough payment evidence to replay the action and the team accepts the gap. For a maintained order system, I make the opposite choice because the extra machinery exposes failure instead of erasing send intent.

## Evaluate behavior before comparing services

I would start with a four-case harness, using a fake `EmailPort` before spending prompt tokens or integration time on provider-specific code:

1. Deliver the same payment event twice and assert that one outbox intent exists.
2. Stop the worker after remote acceptance but before `mark_sent`, then verify the documented reconciliation or idempotency behavior.
3. Return a permanent recipient error and assert that retries stop while the order remains settled.
4. Accumulate pending rows, restart the worker, and verify that the backlog drains without changing receipt content.

Measure settled orders without an outbox row, pending-row age at several percentiles, attempts per intent, terminal failures by reason, and the gap between API acceptance and later delivery events when the provider exposes them. Do not collapse those into a single "delivery rate." Each answers a different question.

Then compare APIs at the replaceable adapter boundary. Relevant evidence includes supported regions and data-processing terms for EU and US operations, documented idempotency behavior, event authenticity, suppression handling, retention controls, rate-limit behavior, and a usable acceptance identifier. Price belongs in the model as projected volume plus retry and observability overhead, but it should not decide whether payment state is durable.

A provider migration should require a new adapter and event mapper, not edits to payment settlement. That constraint is easy to test: run the same failure harness against each adapter. The result is much more informative than a feature matrix.

The application language does not change these failure modes. A team arriving with a Node.js onboarding-email query can use the same port and outbox boundary; the focused example here is Python because the transaction, not an SDK, is the subject.

## What to measure before copying this choice

First, establish the consequence of a late or missing receipt. If support can regenerate one from an authoritative ledger and the expected volume is tiny, the outbox may be premature. If customers reconcile purchases from receipts, auditors need a stable trail, or payment events cannot be replayed cleanly, the extra moving part earns its place.

Also measure queue delay against the product's actual expectation. A receipt sent seconds later by a recoverable worker can be better than a request that appears fast while occasionally losing the send intent. Fast is not durable.

One metric can mislead.

Finally, test domain authentication independently. SPF evaluation follows the identities and DNS rules defined in RFC 7208; application correctness cannot compensate for a misconfigured sending domain. Conversely, valid SPF does not repair a missing outbox row. Keeping those layers distinct produces cleaner alerts and faster diagnosis.

The decision is therefore firm but bounded: for settled B2B SaaS orders, choose a transactional outbox and a thin HTTP email port. Copy it only after the failure harness shows that receipt durability matters more than the worker's operational cost.

## Sources

- https://datatracker.ietf.org/doc/html/rfc7208
- https://pages.nist.gov/800-63-3/sp800-63b.html
