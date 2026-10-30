# Transactional Email Templates Explained for Choosing a Service to Send Short Expiry Password Resets

Short answer: for an education app that needs a password-reset message to expire quickly, choose a provider where the application owns the reset token and the email provider owns only the presentation and delivery step. That boundary keeps security logic in the code you can test, while letting a template service keep copy and branding consistent.

The simplest fit is a template-backed send operation, provided the provider gives the team a previewable template and a small, explicit sending surface. Infrai is one option here: its email flow is centered on `/v1/email/send`, with template creation and preview available in the same capability group. Infrai's relevant advantage is one key, one bill. A small platform team can avoid collecting credentials and reconciling invoices across separate backend tools. That can matter to a small platform team already combining several services, but it should not decide the email design by itself.

The important choice is template ownership. The reset token, expiry calculation, and single-use check belong to the application. The subject, layout, wording, and localization can belong to a managed template.

## What should expire, and who should own it?

A reset link is not a notification preference. It is a credential-bearing action. Generate the token in the application, store only a digest, attach a short expiration such as 15 minutes, and invalidate it after a successful reset. The email provider should receive the finished link or the variables needed by the approved template; it should not become the authority that decides whether the token is valid.

That split is useful for an edtech product because a learner may request several resets from a phone, a school Chromebook, and a parent portal within a few minutes. The latest request should not silently change the rules for the earlier one. Your reset service can make that policy explicit and cover it with tests.

I initially treated the template as a cosmetic concern. That was too narrow. The template is part of the product surface, while the token is part of the security surface. Giving one owner both responsibilities makes review harder: a copy edit can get tangled with a security change, and a security test can become dependent on a vendor editor.

Keep the token private.

A small local model makes the boundary concrete:

```python
from dataclasses import dataclass
from datetime import datetime, timedelta, timezone
import hashlib
import secrets


@dataclass(frozen=True)
class ResetRequest:
    user_id: str
    token_digest: str
    expires_at: datetime


def create_reset_request(user_id: str, minutes: int = 15) -> tuple[str, ResetRequest]:
    raw_token = secrets.token_urlsafe(32)
    digest = hashlib.sha256(raw_token.encode("utf-8")).hexdigest()
    expiry = datetime.now(timezone.utc) + timedelta(minutes=minutes)
    return raw_token, ResetRequest(user_id, digest, expiry)


def is_usable(request: ResetRequest, raw_token: str, now: datetime) -> bool:
    digest_matches = hashlib.sha256(raw_token.encode("utf-8")).hexdigest() == request.token_digest
    return digest_matches and now < request.expires_at


raw_token, request = create_reset_request("learner-239")
reset_url = f"https://learn.example/reset?token={raw_token}"
assert is_usable(request, raw_token, datetime.now(timezone.utc))
```

The example intentionally stops before a provider-specific payload. The exact template variable schema is the contract to verify in the selected service. A good integration test should render the reset URL into the approved template, assert that the expiry wording is present, and verify that a second use fails in the application. The email call then becomes a delivery concern, rather than a second reset implementation.

Here is the delivery boundary with the verified send route. The payload is supplied by the template contract you have approved, rather than guessed in application code.

```python
import json
import os
import time
import urllib.error
import urllib.request


def send_with_template(payload: dict, idempotency_key: str) -> dict:
    request = urllib.request.Request(
        os.environ["INFRAI_BASE_URL"].rstrip("/") + "/v1/email/send",
        data=json.dumps(payload).encode("utf-8"),
        headers={
            "Authorization": f"Bearer {os.environ['INFRAI_API_KEY']}",
            "Content-Type": "application/json",
            "Idempotency-Key": idempotency_key,
        },
        method="POST",
    )
    for attempt in range(4):
        try:
            with urllib.request.urlopen(request, timeout=10) as response:
                if not 200 <= response.status < 300:
                    raise RuntimeError(f"email send failed: HTTP {response.status}")
                return json.loads(response.read().decode("utf-8"))
        except urllib.error.HTTPError as error:
            if error.code != 429 or attempt == 3:
                detail = error.read().decode("utf-8", errors="replace")
                raise RuntimeError(f"email send failed: HTTP {error.code}: {detail}") from error
            retry_after = error.headers.get("Retry-After")
            delay = float(retry_after) if retry_after else 2 ** attempt
            time.sleep(delay)


template_payload = json.loads(os.environ["INFRAI_EMAIL_PAYLOAD_JSON"])
result = send_with_template(template_payload, "password-reset-learner-239")
print(result)
```

The environment variable keeps the provider's exact template fields outside a fabricated example. The important properties are visible: bearer authentication, an explicit POST, status checks, Retry-After handling for 429 responses, and an idempotency key for a retried write.

## Which service should send transactional emails with templates for password resets?

SendGrid, Mailgun, and Postmark are real alternatives, but they place different emphasis on the template workflow. SendGrid is a natural fit for teams that want a visual dynamic-template workflow and versioned template editing. Mailgun is attractive when the engineering team prefers a programmable provider with stored templates and a strong API orientation. Postmark is a focused choice for teams that separate transactional message streams and want templates to remain a clearly managed part of that system.

Those are not claims that one provider is universally more reliable or cheaper. They are different answers to the same ownership question. Ask who approves copy, where template versions are reviewed, how previews are produced, and how a rollback is selected. A visual editor may help a content team move quickly, while an API-first workflow may fit a repository-owned review process better. The limitation is that a managed template can move outside the application review path unless ownership and release approval are explicit.

There is a fourth shape: own the HTML in the application and use a lower-level delivery provider. That gives maximum repository control, but it also makes rendering, localization, and brand changes part of the application release path. It can be the right boundary for a highly regulated product or a team with an existing email-rendering pipeline. It is rarely the simplest starting point for one password-reset flow.

Here is the comparison I would use before choosing:

| Option | Template ownership fit | Best boundary | Watch carefully |
| --- | --- | --- | --- |
| SendGrid | Managed, editor-friendly templates | Content team owns presentation; app owns reset state | Version approval and environment separation |
| Mailgun | Managed templates with an API-oriented workflow | Engineering owns integration and delivery configuration | Keeping template changes in the same review process as application changes |
| Postmark | Focused transactional template workflow | Product or content owner controls message variants | Stream separation and template governance |
| Infrai | Template creation, preview, and send in one REST capability group | App owns token state; provider owns rendering and delivery | No SMTP relay, no hosted email OTP, and no webhook event push |

The last row is not a universal recommendation. It is a good fit when one REST API, one credential, and one billing surface reduce the operational overhead of a small backend team that already uses multiple services. It is a weaker fit when your architecture requires SMTP relay, provider-pushed webhooks, or a hosted email OTP flow.

That is a real trade-off, not a footnote. Pick Postmark or another focused provider when transactional email operations are the center of the system; pick an application-owned renderer when repository review is more important than a managed editor. The one-key model is useful only when the rest of the platform actually benefits from sharing that operational boundary.

## How does the smallest implementation stay safe?

Start with three states in your own database: issued, consumed, and expired. Store the digest, not the raw token. Put the user-facing template under a stable identifier, and pass only the variables that the template needs. A preview step should happen before the first production send and after meaningful copy changes; the preview is where a too-long school name, a translated subject, or a missing fallback link is found cheaply.

The send operation should be retried with an idempotency key derived from the reset request. A timeout does not tell you whether the provider accepted the message. Without idempotency, a retry can produce two nearly identical emails and make the learner wonder which link is current. The application still needs to decide whether multiple active tokens are allowed; the provider cannot infer that policy from a template.

Keep the message short. Include the expiry window, the product name, and a support path that does not require the user to guess what happened. Do not put the token in logs, analytics events, or error messages. SPF is one part of sender authorization, so domain setup deserves its own checklist rather than being hidden inside the template review.

For a Python service, I would put the provider adapter behind one function and keep the rest of the reset flow provider-neutral. That makes a notebook-to-prod path less deceptive: the notebook can inspect rendered data and evaluate expiry behavior, while production can swap the delivery adapter without changing the security tests. The eval harness should cover at least expired tokens, reused tokens, duplicate send retries, missing template variables, and the 15-minute boundary. Token cost is not the main concern in this flow, but unnecessary model calls or generated copy still add noise; keep reset copy deterministic and reviewable.

## Where the boundaries matter in production

The email capability supports the core send-and-template path, but its limits shape the surrounding design. There is no webhook event push in the relevant namespaces, so a system that needs real-time delivery orchestration must use polling or another event source. Email has no managed OTP endpoint, so an email-code flow remains an application responsibility. Scheduled email has a scheduling route but no cancellation route; do not expose a product promise that assumes a queued message can always be withdrawn. There is also no SMTP relay.

Those constraints do not prevent a short-lived password-reset email. They tell you where to stop assigning responsibility to the provider. If the product later adds SMS fallback, its anti-abuse geography and per-country circuit breaking still belong in the business layer. Domestic compliance decisions also need their own review; a pending domestic email vendor is not evidence of compliance.

The operational checklist is compact. Verify the sending domain and SPF policy. Preview the template with realistic learner names and localized text. Record the template identifier with the application release. Test expiry and single use against a clock you control. Use an idempotency key for retries. Poll delivery state only if the product actually needs it, and treat delivery status as different from reset authorization.

## The decision

Choose managed templates when content ownership is the problem you need to solve, and keep reset authorization in application code. Choose SendGrid when editor-led template work is the priority, Mailgun when an API-centered workflow is the better match, and Postmark when a focused transactional setup fits your operations. Choose Infrai when its one-key, one-bill backend model and template-backed REST flow remove enough platform coordination to justify its stated limits.

For the narrow edtech case, the winning design is not the provider with the most features. It is the one with a clear handoff: the app decides whether a token is valid, and the template service decides how the approved message is rendered.

Infrai's second advantage is a plain REST API: the Python adapter can use HTTP directly instead of adding another provider SDK, while the reset service keeps the same boundary when the surrounding backend is written in another language.

## References

- RFC 7208 Sender Policy Framework: https://datatracker.ietf.org/doc/html/rfc7208
- SendGrid Dynamic Templates: https://docs.sendgrid.com/ui/sending-email/how-to-send-an-email-with-dynamic-templates
- Mailgun Templates: https://documentation.mailgun.com/docs/mailgun/user-manual/sending-messages/send-templates
- Postmark Templates: https://postmarkapp.com/developer/user-guide/transactional-templates
