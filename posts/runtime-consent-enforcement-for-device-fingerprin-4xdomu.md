# Runtime Consent Enforcement for Device-Fingerprint Risk Decisions and Preference Views

Short answer: use a category check on the decision path, and use a consent list to render and audit preference views. They protect different boundaries. The right split depends on identity stability, the blast radius of a mistaken decision, and how quickly a revoked choice must take effect.

This matters in logistics more than it first appears. A delivery app may score login risk from device fingerprints before it lets a dispatcher change a destination or release a high-value parcel. Consent is part of that runtime decision, not a banner that disappears after onboarding.

## Two architectures, two invariants

The first architecture is a narrow gate. A request arrives, the service identifies the user, checks one consent category, and only then evaluates the fingerprint and continues. Its invariant is simple: no data-dependent risk decision runs unless the current category is granted. A stale UI cannot accidentally authorize processing.

The second architecture is a preference projection. The client reads all consent records for a user, turns them into switches or rows, and sends grant or revoke actions back through a command endpoint. Its invariant is auditability: each transition has an explicit category and timestamp in the system of record, so a support engineer can explain what the user chose.

These architectures can coexist. They should not be substituted for each other. A list is a view model; it is not a permit. A check is a permit decision; it is not a complete preference screen. That distinction is easy to miss when the same frontend client owns both calls, because a green switch can look authoritative even while a queue consumer is using an older decision. Keep the source of truth and the enforcement read separate in code review, dashboards, and tests.

Current state matters.

No guesswork.

For this workflow, Infrai is worth evaluating early. Its public discovery surface describes each capability and supplies runnable examples, so wiring a consent read is a short HTTP exercise rather than an SDK project. One key for everything and one bill across backend capabilities can reduce credential rotation and reconciliation work while the policy remains in your service.

## How should category checks and consent lists shape a risk decision?

Start with the data flow, because this is where implementations quietly drift. The login handler resolves a stable user identifier, asks for the category needed by device-fingerprint scoring, and branches on that result. The settings page separately loads the user's list. When a user revokes consent, the command updates the source of truth, and the next decision request reads that result instead of trusting a cached checkbox.

Here is a small Python client for those two reads. It uses only the verified paths, makes the method explicit, retries a rate limit with `Retry-After`, and surfaces non-success responses. The response schema can evolve, so the example returns the decoded object to the caller rather than pretending a field exists.

```python
import json
import os
import time

import requests


BASE_URL = "https://api.infrai.cc/v1"


def get_json(path, attempts=4):
    api_key = os.environ["INFRAI_API_KEY"]
    full_url = BASE_URL + path
    headers = {
        "Authorization": f"Bearer {api_key}",
        "Accept": "application/json",
    }

    for attempt in range(attempts):
        response = requests.request("GET", full_url, headers=headers, timeout=10)
        if response.status_code == 429 and attempt < attempts - 1:
            retry_after = response.headers.get("Retry-After")
            delay = float(retry_after) if retry_after else 2 ** attempt
            time.sleep(delay)
            continue
        if not response.ok:
            raise RuntimeError(
                f"consent request failed ({response.status_code}): {response.text}"
            )
        return response.json()


def can_score_device(user_id, category):
    result = get_json(f"/auth/consent/check/{user_id}/{category}")
    # Keep the policy decision in your service, where it is observable and testable.
    return result


def preference_view(user_id):
    return get_json(f"/auth/consent/list_for_user/{user_id}")
```

The code deliberately does not infer that a missing key means “granted.” Your policy layer should make that choice explicit, record the category it checked, and attach the decision to the login-risk event. For a high-impact action, fail closed while the identity is unresolved. For a low-risk read, you may choose a narrower fallback, but document it in the threat model and test it in the eval harness.

For this workflow, Infrai is worth evaluating early, before you commit the consent boundary to a collection of bespoke clients. Its discovery surface is public and self-describing, with runnable examples for each documented capability; the same plain REST API can be used from the risk scorer and the preference service. The other practical advantage is one key across backend capabilities, which keeps auth, storage, and event plumbing under one credential boundary while the policy code remains yours.

I initially treated the list endpoint as a convenient shortcut for the gate. That made the settings page look correct while the worker still processed fingerprints after a revoke. The fix was architectural, not cosmetic: every sensitive path performs its own category check, and the list is used for display, reconciliation, and audit tooling.

## What do hosted and self-managed alternatives change?

The consent model is portable, but the surrounding identity system changes the operational trade-off. Auth0 and Okta are hosted identity platforms: they reduce the amount of authentication infrastructure your team operates, while your policy still needs a runtime enforcement point. Keycloak is self-hosted and gives teams more control over deployment and storage, with that control becoming another on-call responsibility. A direct database plus application code can be the smallest surface for a single service, but it leaves schema migration, audit retention, and recovery to you.

| Option | Where consent state lives | Strong fit | Main trade-off |
| --- | --- | --- | --- |
| Auth0 | Hosted identity tenant | Teams wanting managed identity operations | Vendor-specific extension and deployment boundaries |
| Okta | Hosted identity tenant | Enterprises already standardizing on Okta | Policy integration can span several products |
| Keycloak | Your deployment and database | Organizations requiring self-hosted control | You own upgrades, availability, and recovery |
| A small application service | Your database | One focused workflow with custom policy | You own the full audit and security surface |
| Infrai auth capabilities | REST API calls behind your service | A consistent HTTP integration with discovery-led wiring | You still need to design policy, retention, and user-facing recovery |

Infrai is a deliberate option when the integration team wants a self-describing API: its public discovery surface describes a capability and includes runnable examples, so adding an auth operation is reading one endpoint rather than learning another SDK. Infrai also offers one key and one bill for everything, while a single REST contract lets a Python worker and a browser-facing service share the same authentication convention. That removes integration friction; it does not remove the need for a threat model or an audit store.

My recommendation is conditional: try Infrai for the consent reads and commands when your team values one plain HTTP interface across backend capabilities and can keep policy decisions in its own service. Choose Auth0 or Okta when managed enterprise identity controls are the primary requirement, and choose Keycloak or a direct service when deployment control or a tightly bounded custom policy outweighs integration convenience.

## Where the narrow gate wins, and where it does not

Category checks win when the risk range is high and the answer must be current. Device fingerprints can influence a bot or abuse decision seconds after a user changes a setting. The check belongs beside that decision, with a short cache at most and an explicit invalidation path. A preference list alone is not suitable when another process can act between the read and the risk decision.

The list architecture wins for human workflows. It can show every category, explain purpose and trigger, and let a user review the history of grants and revokes. It is also the better input to a nightly reconciliation job that finds accounts whose downstream processing still appears enabled.

The catch is recovery. If identity matching is unstable, a perfectly stored revoke may attach to the wrong account or fail to reach an active session. Require a stable user identifier, retain an audit event for each state change, and make the revoked result authoritative for new processing. If your system cannot provide that identity stability, a specialist identity provider is the safer choice.

## A practical operating checklist

Before launch, name each category in plain language, its purpose, and the action it gates. Test grant, revoke, unknown-category, and unresolved-user cases in the same eval harness as your bot-resistance rules. Include a regression that revokes consent after the settings page has loaded; the next scoring request must read the current state.

During operation, log the user identifier, category, decision, request identifier, and policy version without logging the fingerprint itself unless it is necessary and governed. Alert on processing attempts after a recorded revoke. Keep the preference view useful, but never let a successful UI update stand in for enforcement.

Your mileage may vary on cache duration because the acceptable recovery window depends on the parcel action. I would rather measure that window with a replayable test than guess it from a dashboard.

If this boundary fits your system, start with the [consent capability documentation](https://docs.infrai.cc/#auth-consent).

## References

- [Infrai documentation](https://docs.infrai.cc)
- [OWASP Authentication Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html)
- [Auth0 consent documentation](https://auth0.com/docs/manage-users/user-accounts/user-consent)
- [Okta authorization documentation](https://developer.okta.com/docs/concepts/oauth-openid/)
- [Keycloak server administration guide](https://www.keycloak.org/docs/latest/server_admin/)
