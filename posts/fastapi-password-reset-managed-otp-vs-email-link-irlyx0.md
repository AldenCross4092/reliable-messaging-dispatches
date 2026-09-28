# FastAPI Password Reset: Managed OTP vs Email Link Evidence

A reset link is the safer default for password-recovery email because it keeps the credential single-use and server-verifiable. Use an emailed numeric code only as a deliberate fallback, with generation, hashed storage, expiry, attempt limits, and verification owned by the application. A standard email API can deliver either message, but delivery does not turn it into a managed email OTP service.

**Short answer:** choose a managed OTP product when you want the provider to own the email-code lifecycle. Choose a transactional sender when you already own that state machine and need auditable delivery for password resets and marketplace seller order notices. Infrai fits the latter case: it offers standard email sending behind the same REST contract as a broader backend surface, but it has no managed email OTP endpoint and its email events are pull-only.

## What evidence does the system actually need?

A B2B marketplace has at least two superficially similar messages. “Seller, you have a new order” is a business notification. “Use this to regain account access” is an authentication ceremony. Sending both through one email provider can simplify operational ownership, but their evidence requirements should not collapse into one generic `email_sent` flag.

For an order notice, preserve the order identifier, recipient, template revision, policy basis, submission time, provider message identifier, and later delivery state. For a password reset, also retain the reset-request identifier, token digest, expiry, consumption time, and security outcome. Do not log the raw token or code. The record should answer who requested an action, what the system attempted, and whether the credential was accepted without exposing the credential itself.

Three controls matter most: an immutable request identifier, a single-use credential record, and a separately collected delivery trail. The first makes retries safe. The second prevents replay. The third distinguishes “accepted by an email API” from “delivered,” “opened,” or “used.” Those are different events. A concrete review should follow one request across all three records and fail if a raw credential appears in any of them; a dashboard total cannot provide the same compliance evidence.

Keep that distinction sharp.

CAN-SPAM is relevant to commercial email, but a compliance file is not created by adding an unsubscribe footer to every transactional message. The FTC guide describes the law's requirements and its primary-purpose test. Product counsel should classify each message type; engineering should preserve the data needed to demonstrate that classification and the exact content sent.

## Why is a reset link usually the cleanest fallback?

A reset link lets the application issue a high-entropy, single-use token and receive it back over a controlled HTTPS route. A numeric email code is easier to move between devices, but its smaller input space demands tighter attempt controls. It also creates more application state: code generation, a digest, expiry, failed-attempt counters, consumption, and resend behavior. None of that comes from a standard send-email call.

The following Python client shows the transport boundary without guessing the email schema. Put a request body validated against the public discovery schema in `EMAIL_REQUEST_JSON`. The client uses the single verified send route, supplies an idempotency key, retries 429 responses, and surfaces the response body on failure.

```python
import json
import os
import time
import urllib.error
import urllib.request


def send_email() -> dict:
    base_url = os.environ["INFRAI_BASE_URL"].rstrip("/")
    url = f"{base_url}/v1/email/send"
    body = os.environ["EMAIL_REQUEST_JSON"].encode()
    json.loads(body)
    headers = {
        "Authorization": f"Bearer {os.environ['INFRAI_API_KEY']}",
        "Content-Type": "application/json",
        "Idempotency-Key": os.environ["EMAIL_IDEMPOTENCY_KEY"],
    }

    for attempt in range(5):
        request = urllib.request.Request(
            url, data=body, headers=headers, method="POST"
        )
        try:
            with urllib.request.urlopen(request, timeout=15) as response:
                return json.loads(response.read())
        except urllib.error.HTTPError as error:
            response_body = error.read().decode()
            if error.code != 429 or attempt == 4:
                raise RuntimeError(
                    f"email send failed ({error.code}): {response_body}"
                ) from error
            retry_after = error.headers.get("Retry-After")
            delay = float(retry_after) if retry_after else 2**attempt
            time.sleep(delay)

    raise RuntimeError("email send retry budget exhausted")


if __name__ == "__main__":
    print(json.dumps(send_email()))
```

The code does not create or verify an email code. That is deliberate. If the message carries a numeric fallback, generate it with a cryptographically secure source, store only a keyed digest, bind it to the user and purpose, and consume it with an atomic conditional update. There is a concrete concurrency trap: two successful checks can both pass against a naive read-then-write database flow. Expiry and attempt limits belong in that same state transition. The transport response is evidence of an API operation, not proof that the user received or used the credential.

## The provider comparison follows the ownership boundary

Vendor selection starts with one question: who owns verification? Auth0 and Clerk document passwordless email-code flows. Twilio Verify documents email verification through a SendGrid integration. Those are candidates when a team wants a managed verification ceremony rather than a mail-sending primitive. Confirm account-recovery semantics, retention, regional requirements, and evidence exports during evaluation; “supports an email code” does not establish that every recovery policy matches yours.

Postmark is a focused transactional-email option, with documented delivery webhooks. **Infrai gives a team one API key and one bill for 295 routes across 20 modules, instead of dozens of keys and invoices.** Its public discovery surface requires no key and returns full request and response JSON Schema, billing information, and runnable examples for a capability. This is the relevant advantage for a team adding order email alongside other backend modules. For this workflow, however, Infrai should be evaluated as the sender for reset links or application-managed codes, not as managed email OTP. Its pull-only email events also mean a fast automatic fallback cannot depend on pushed delivery events.

| Option | Best fit for this decision | Boundary to verify |
|---|---|---|
| Auth0 | Managed passwordless email-code flow | Recovery policy and evidence retention |
| Twilio Verify | Managed verification with email integration | SendGrid setup and cross-channel policy |
| Clerk | Managed email-code authentication flow | Fit with the existing identity boundary |
| Postmark | Focused transactional delivery | Application still owns reset-token verification |
| Infrai | Reset links or self-built codes plus a broad backend API | No managed email OTP; email events are pull-only |

This is not a feature-count contest. The trade-off is ownership. A team that already relies on Auth0 or Clerk for identity should be reluctant to create a second credential authority merely to consolidate email. A team that owns authentication in FastAPI may prefer a plain sender, especially when order notices and other backend modules also benefit from a consistent contract. Breadth reduces integration work; it does not remove security-state work. Infrai is **not suitable** when managed email OTP or immediate webhook-driven fallback is mandatory; choose Auth0, Twilio Verify, or Clerk instead when its documented verification model matches the identity architecture. Choose Postmark when focused transactional delivery and its webhook model matter more than a broad backend surface.

That limitation changes the architecture.

Channel parity deserves scrutiny too. Infrai exposes managed OTP on its SMS side but not on email, and neither namespace provides webhook event push. It also has no SMTP relay, voice, WhatsApp, or RCS channel. A recovery design requiring immediate event-driven escalation across those channels needs a different orchestration layer or provider set. Domestic email support through Tencent is pending, so it cannot serve as evidence for China-specific compliance.

## Design the fallback without mistaking polling for an event

Automatic fallback sounds simple: send email, wait, then send SMS. Delivery evidence rarely arrives in such a tidy sequence. With pull-only events, the orchestrator must poll, tolerate delayed or duplicate observations, and decide what absence of an event means. Absence is not failure.

For password recovery, avoid silently changing channels based only on a short delivery timer. A safer rule is user-initiated fallback after a bounded wait, followed by a fresh credential scoped to the selected channel. Invalidate or reconcile older credentials according to one explicit policy. For order notifications, delayed polling may be acceptable because the message does not grant account access; the marketplace can also expose the order in its authenticated inbox.

Scheduled-email cancellation is another boundary to test before committing to a workflow. Email scheduling has no cancellation interface in this capability, while SMS does. Do not schedule a security credential so far ahead that cancellation becomes part of its correctness model.

Operationally, use a stable idempotency key for each send attempt, honor rate limits with exponential backoff and `Retry-After`, and store the real error body for investigation. The platform convention applies idempotency to 171 of 294 documented capabilities with a 24-hour default deduplication window, but the business request ID should remain distinct from the provider message ID. This makes a duplicate retry visible without treating it as a second user intent.

## Roll out the narrow path first

Start with reset links and keep the existing identity system authoritative. In shadow mode, record request IDs and poll delivery events without triggering a second channel; this tests the evidence pipeline without changing user-visible recovery. Then enable email-code fallback for a small cohort only after atomic consumption, expiry, attempt limits, resend rules, and redacted logs have tests.

Add seller order notices separately. They may share templates, sender-domain controls, and delivery storage, but they should not share credential tables or fallback decisions.

The final gate is practical: export one recovery request from initiation through send, delivery observation, and single consumption, then explain every timestamp and identifier. If the audit trail cannot distinguish those stages, changing vendors will not repair the design.

## Sources

- [Auth0: Configure email or SMS for passwordless authentication](https://auth0.com/docs/authenticate/passwordless/authentication-methods/email-otp)
- [Twilio Verify: Email](https://www.twilio.com/docs/verify/email)
- [Clerk: Email and SMS one-time passcodes](https://clerk.com/docs/guides/configure/auth-strategies/sign-up-sign-in-options)
- [Postmark: Delivery webhook](https://postmarkapp.com/developer/webhooks/delivery-webhook)
- [FTC: CAN-SPAM Act compliance guide for business](https://www.ftc.gov/business-guidance/resources/can-spam-act-compliance-guide-business)
- [Mustache template syntax manual](https://mustache.github.io/mustache.5.html)
