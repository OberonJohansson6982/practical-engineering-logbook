# SMS OTP Two-Factor Authentication — Contact Queue Compliance Through Atomic Challenges

Use a server-owned challenge record, not the SMS delivery receipt, as the authority for staff access to a media contact queue. The record should bind a short-lived code to one account and one login attempt, enforce resend and verification budgets atomically, and emit a sanitized audit event for every state change. Recovery codes belong in the same control plane, but each must be independently hashed and consumed once.

TL;DR: Treat SMS as an untrusted transport. A successful send does not prove possession, and a correct code is insufficient after expiry, budget exhaustion, prior use, or supersession. For compliance evidence, preserve the decision inputs and outcome without preserving the OTP, recovery code, or full phone number.

## How should NestJS verify SMS OTP for two-factor authentication?

Imagine a newsroom contact form that labels submissions and routes them to legal, subscriptions, corrections, or general support. The form itself may be public, but the queue contains names, email addresses, message text, and sometimes sensitive allegations. A staff login therefore needs more than a happy-path `send` followed by `verify`.

The useful evidence is a sequence: an authenticated account requested a second-factor challenge; policy either allowed or denied issuance; a particular challenge was verified, rejected, expired, or superseded; and access to a named queue followed from that decision. Keep a random correlation ID across those events. Do not put message bodies, plaintext codes, or complete phone numbers into the audit payload.

There are two separate clocks. The challenge lifetime limits how long a captured code remains useful. The attempt window limits how quickly an actor can guess or trigger messages. They should not share a counter, because repeated-send traffic and verification guesses are different abuse signals.

Keep them separate.

**The login decision belongs to the backend transaction, while delivery remains an external side effect.** That separation makes retries understandable. It also means an operator can explain why access was granted without treating a carrier receipt as authentication evidence.

## A small, runnable challenge core

The following Python module is intentionally narrower than a full web application. A NestJS controller can expose equivalent operations, but the security boundary is the state machine and its storage transaction, not the framework decorator. The example uses SQLite so the invariants are visible: one current challenge per account, a five-minute lifetime, no more than three sends in ten minutes, and five verification attempts. Those numbers are example policy inputs, not universal recommendations; tune them with abuse data and support impact. In a NestJS build, the controller validates the request and passes the authenticated account ID to a service; the service starts the transaction shown here; a queue worker handles delivery after commit. Keep the challenge ID opaque at the controller boundary. Never accept an account ID from the request body when the authenticated session already supplies it. That mapping is deliberately boring, which is useful: framework concerns stay outside the code that decides whether a guess can open the contact queue.

```python
from __future__ import annotations

import hashlib
import hmac
import secrets
import sqlite3
import time
import uuid


TTL_SECONDS = 300
SEND_WINDOW_SECONDS = 600
MAX_SENDS = 3
MAX_ATTEMPTS = 5


class ChallengeService:
    def __init__(self, db: sqlite3.Connection, pepper: bytes) -> None:
        self.db = db
        self.pepper = pepper
        self.db.execute("PRAGMA foreign_keys = ON")
        self.db.executescript(
            """
            CREATE TABLE IF NOT EXISTS challenges (
                id TEXT PRIMARY KEY,
                account_id TEXT NOT NULL,
                code_digest BLOB NOT NULL,
                expires_at INTEGER NOT NULL,
                attempts_left INTEGER NOT NULL,
                used_at INTEGER,
                superseded_at INTEGER
            );
            CREATE TABLE IF NOT EXISTS send_events (
                account_id TEXT NOT NULL,
                sent_at INTEGER NOT NULL
            );
            CREATE TABLE IF NOT EXISTS audit_events (
                event_id TEXT PRIMARY KEY,
                occurred_at INTEGER NOT NULL,
                account_id TEXT NOT NULL,
                challenge_id TEXT,
                action TEXT NOT NULL,
                outcome TEXT NOT NULL
            );
            """
        )

    def _digest(self, challenge_id: str, value: str) -> bytes:
        message = f"{challenge_id}:{value}".encode()
        return hmac.new(self.pepper, message, hashlib.sha256).digest()

    def _audit(
        self, account_id: str, challenge_id: str | None,
        action: str, outcome: str, now: int,
    ) -> None:
        self.db.execute(
            "INSERT INTO audit_events VALUES (?, ?, ?, ?, ?, ?)",
            (str(uuid.uuid4()), now, account_id, challenge_id, action, outcome),
        )

    def issue(self, account_id: str, now: int | None = None) -> tuple[str, str]:
        now = int(time.time()) if now is None else now
        with self.db:
            recent = self.db.execute(
                "SELECT COUNT(*) FROM send_events "
                "WHERE account_id = ? AND sent_at >= ?",
                (account_id, now - SEND_WINDOW_SECONDS),
            ).fetchone()[0]
            if recent >= MAX_SENDS:
                self._audit(account_id, None, "challenge.issue", "throttled", now)
                raise PermissionError("challenge request denied")

            self.db.execute(
                "UPDATE challenges SET superseded_at = ? "
                "WHERE account_id = ? AND used_at IS NULL AND superseded_at IS NULL",
                (now, account_id),
            )
            challenge_id = str(uuid.uuid4())
            code = f"{secrets.randbelow(1_000_000):06d}"
            self.db.execute(
                "INSERT INTO challenges VALUES (?, ?, ?, ?, ?, NULL, NULL)",
                (
                    challenge_id,
                    account_id,
                    self._digest(challenge_id, code),
                    now + TTL_SECONDS,
                    MAX_ATTEMPTS,
                ),
            )
            self.db.execute(
                "INSERT INTO send_events VALUES (?, ?)", (account_id, now)
            )
            self._audit(account_id, challenge_id, "challenge.issue", "allowed", now)
        return challenge_id, code

    def verify(
        self, account_id: str, challenge_id: str, candidate: str,
        now: int | None = None,
    ) -> bool:
        now = int(time.time()) if now is None else now
        with self.db:
            row = self.db.execute(
                "SELECT code_digest, expires_at, attempts_left, used_at, superseded_at "
                "FROM challenges WHERE id = ? AND account_id = ?",
                (challenge_id, account_id),
            ).fetchone()
            if row is None:
                self._audit(account_id, challenge_id, "challenge.verify", "unknown", now)
                return False

            digest, expires_at, attempts_left, used_at, superseded_at = row
            active = (
                used_at is None
                and superseded_at is None
                and expires_at >= now
                and attempts_left > 0
            )
            matches = hmac.compare_digest(
                digest, self._digest(challenge_id, candidate)
            )
            if not active or not matches:
                if active:
                    self.db.execute(
                        "UPDATE challenges SET attempts_left = attempts_left - 1 WHERE id = ?",
                        (challenge_id,),
                    )
                self._audit(account_id, challenge_id, "challenge.verify", "denied", now)
                return False

            updated = self.db.execute(
                "UPDATE challenges SET used_at = ? "
                "WHERE id = ? AND used_at IS NULL AND superseded_at IS NULL",
                (now, challenge_id),
            ).rowcount
            outcome = "allowed" if updated == 1 else "denied"
            self._audit(account_id, challenge_id, "challenge.verify", outcome, now)
            return updated == 1
```

The delivery adapter receives the returned code only after the database commit. If sending fails, record a delivery outcome keyed by the challenge ID; do not roll back the abuse-budget event. A retry should reuse an idempotency key and follow an explicit policy rather than silently minting another valid code.

Delivery is not proof.

This sample returns the code to its caller to keep transport out of the core. Production code must pass it directly to a message-rendering and delivery boundary, never to an HTTP response. A Mustache template is adequate for a plain message such as `Your sign-in code is {{code}}`, provided the application supplies the value and does not allow untrusted users to author templates.

## Recovery is another authentication path

Recovery codes are often treated as a printable appendix to enrollment. Operationally, each code is a second-factor credential. Generate them with a cryptographically secure random source, show the plaintext set once, store only salted password-hash outputs, and consume exactly one row in the same transaction that grants access. A batch identifier lets the backend revoke the entire old set when a new set is generated.

Do not log which recovery code was typed. Log the batch ID, code record ID, actor account, outcome, timestamp, request correlation ID, and the policy version that evaluated it. The same rule applies to the phone number: a stable internal destination ID is more useful for correlation and creates less exposure than duplicating the full number across logs.

Recovery also needs a deliberate support boundary. An agent who can read contact-form submissions should not automatically be able to reset another agent's second factor. Manual reset should require its own authorization, record the approver and subject, revoke active challenges and recovery-code batches, and notify the account through a separately maintained channel.

Shortcuts hurt here. If the recovery endpoint bypasses the normal audit writer, the system produces its weakest evidence exactly where account takeover risk is highest.

## Throttling without creating an oracle

Apply limits at several scopes: account, destination, source network, and a broader service budget. A single IP limit punishes offices and mobile carrier gateways; an account-only limit lets an attacker distribute requests across many accounts. The enforcement store must support atomic increments or transactional inserts, and all application instances must observe the same budget. This is a genuine trade-off: aggressive limits reduce automated guessing and message flooding, but they also increase lockouts during breaking-news traffic, when many editors may arrive through the same corporate network. A bypass for that shared network would erase much of the protection. A better policy weighs several scopes, returns the same public response, and lets an authorized operator inspect the internal reason without manually editing counters.

Public responses should avoid confirming whether an account or phone number exists. Internally, preserve distinct outcomes such as `unknown_account`, `destination_blocked`, `send_budget_exhausted`, and `delivery_failed` in access-controlled telemetry. That split keeps the external surface quiet while leaving enough resolution for incident review and evals.

Build an evaluation table before tuning policy. Include a normal login, two rapid resends, the fourth send inside the example window, an expired code, a superseded code, five wrong guesses, concurrent verification of one correct code, and recovery-code replay. Assert both the API result and the audit rows. This is the notebook-to-production move that matters: turn every policy claim into a deterministic state transition before adding a carrier or UI.

Prompt-assisted routing belongs after authentication. If a model classifies a contact message into `legal`, `corrections`, or `subscriptions`, expose only a small enum-returning tool and validate its output server-side. Tool descriptions can shape model behavior, but they do not replace authorization. The authenticated staff member still needs permission for the selected queue, and the audit event should record the model or ruleset version used for routing without copying the contact message into a general authentication log.

## Operating the evidence trail

An audit record should answer who attempted what, against which account and challenge, under which policy version, at what time, and with what result. It should not become a credential store. Restrict write paths, make later alteration detectable according to the organization's retention design, synchronize clocks, and separate access to security evidence from ordinary application debugging.

SMS has a real assurance boundary. NIST SP 800-63B treats use of the public switched telephone network for out-of-band authentication as restricted and calls for risk indicators such as device swap, SIM change, and number porting to be considered. That makes SMS a policy choice, not a universal high-assurance answer. **It is not suitable as the sole second factor for a queue whose threat model requires phishing resistance.** Document the accepted risk for this queue and provide a migration path to phishing-resistant authenticators where the data classification requires one. The Python sample has another deliberate limitation: SQLite exposes the transaction rules clearly, but a multi-instance deployment needs a shared transactional store and tested contention behavior. Moving storage does not relax the one-time-consumption invariant.

The operational review can stay compact. Issuance and verification mutations need atomicity; expired and superseded challenges must fail; repeated-send and guess budgets remain independent; secrets and contact content stay out of logs; recovery replay is covered by tests; delivery callbacks never grant access; queue authorization follows the second-factor check; retention and deletion follow the evidence policy; and alerts rely on aggregate failure patterns rather than raw message contents. Those assertions should run again whenever policy, storage, templates, delivery adapters, or routing models change.

The durable design is pleasantly plain: one challenge state machine, one separately controlled recovery path, and one sanitized event vocabulary. Delivery providers can change. The evidence and access decision remain intelligible.

## References

- https://pages.nist.gov/800-63-4/sp800-63b.html
- https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html
- https://cheatsheetseries.owasp.org/cheatsheets/Logging_Cheat_Sheet.html
- https://mustache.github.io/mustache.5.html
- https://platform.claude.com/docs/en/agents-and-tools/tool-use/overview
