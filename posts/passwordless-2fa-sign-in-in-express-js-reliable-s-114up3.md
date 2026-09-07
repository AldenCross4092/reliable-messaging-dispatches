# Passwordless 2FA Sign-In in Express.js: Reliable SMS OTP and Email Fallback

For a gaming support form, the reliable shape is SMS as the primary passwordless 2FA login step and an app-owned email code as the fallback. Express.js can coordinate it cleanly, but the email verification path is yours to build: generate a code, hash it, expire it, and verify it. The SMS provider handles delivery; your database handles the state machine.

Short answer: use a managed SMS OTP endpoint for the first attempt, keep a separate short-lived email-code table, and switch only after a deliberate status check or a bounded poll. This is practical for SaaS login when you accept that email fallback adds application logic and is not truly real-time.

## Start with the bill and the retention decision

The dominant cost in this flow is usually the delivery attempt, not the six-digit value sitting in a database. A support form that sends one SMS and, after a timeout, one email creates two billable events for a single login. That makes retention policy part of the cost model: store hashes and delivery identifiers long enough to investigate abuse, then delete the code material.

I keep the record small: account id, channel, code hash, expiry, attempt count, provider request id, and a status. No plaintext code. No message body copied into a long-lived audit table. A 10-minute TTL is an example policy, not a platform guarantee; choose a window that fits your threat model and document it.

The trade-off is uncomfortable but useful. Short retention limits replay and database exposure, while longer retention gives your incident review more context. I would retain delivery metadata longer than the code hash, because a delivery dispute needs a request id and timestamp, not the secret itself.

Do this once.

Keep it boring.

## What should an Express.js flow do when SMS OTP delivery stalls?

Treat delivery as a state transition, not as a synchronous function call. Create a login attempt, send the SMS OTP, and show the user a fallback action only after your status policy says the first channel has had a fair chance. The available result checks are pull-based, so an instant, push-style handoff is not available; polling must be bounded and rate-limited.

The email side has no managed OTP API. That is a capability boundary, not a reason to skip the fallback. Generate a random code with a cryptographically secure source, store only a keyed hash, compare in constant time, and invalidate the row on success. Send the message through the email send capability, and keep the template separate from the verification record.

Here is the core shape in Python. The same state machine fits an Express.js controller; the snippet stays Python so the security mechanics are explicit and copyable.

```python
import hashlib
import hmac
import os
import secrets
import time
from typing import Dict

import requests


BASE_URL = os.environ["INFRAI_BASE_URL"].rstrip("/")
API_KEY = os.environ["INFRAI_API_KEY"]
CODE_TTL_SECONDS = 600
email_codes: Dict[str, dict] = {}


def digest(code: str, salt: bytes) -> str:
    return hmac.new(salt, code.encode("ascii"), hashlib.sha256).hexdigest()


def send_sms_otp(phone: str, attempt_id: str) -> dict:
    response = requests.post(
        f"{BASE_URL}/v1/sms/otp",
        headers={"Authorization": f"Bearer {API_KEY}", "Idempotency-Key": attempt_id},
        json={"to": phone},
        timeout=10,
    )
    if response.status_code == 429:
        raise RuntimeError("rate limited; retry after the server-provided delay")
    response.raise_for_status()
    return response.json()


def start_email_fallback(user_id: str, address: str) -> None:
    code = f"{secrets.randbelow(1_000_000):06d}"
    salt = secrets.token_bytes(16)
    email_codes[user_id] = {
        "hash": digest(code, salt),
        "salt": salt,
        "expires_at": time.time() + CODE_TTL_SECONDS,
        "attempts": 0,
    }
    response = requests.post(
        f"{BASE_URL}/v1/email/send",
        headers={"Authorization": f"Bearer {API_KEY}"},
        json={"to": address, "subject": "Your sign-in code", "text": f"Code: {code}"},
        timeout=10,
    )
    response.raise_for_status()


def verify_email_code(user_id: str, candidate: str) -> bool:
    row = email_codes.get(user_id)
    if not row or row["attempts"] >= 5 or time.time() >= row["expires_at"]:
        return False
    row["attempts"] += 1
    valid = hmac.compare_digest(digest(candidate, row["salt"]), row["hash"])
    if valid:
        del email_codes[user_id]
    return valid
```

In production, replace the in-memory dictionary with a transactional table and make the email-send request idempotent as well. On a 429 response, honor `Retry-After` and use exponential backoff; never tight-loop a login endpoint. For the SMS verification call, send the provider's attempt identifier and surface a real 4xx body to the user-facing audit log instead of assuming every response is success.

That last detail matters during a busy launch. Imagine a player submits the contact form twice, the first SMS is delayed by a carrier, and the second request arrives from a new IP. A naive handler sends two codes, accepts whichever arrives last, and leaves two live rows behind. A bounded attempt record lets you reject the duplicate, keep one current code, and explain the result to support without retaining the secret. The same record can tie an email fallback to the original SMS request, which makes retention and abuse review possible without pretending that delivery events arrive in real time.

## How do SMS OTP and email fallback compare for passwordless sign-in?

The right comparison is delivery behavior and operational control, not a feature-count contest. SMS is immediate enough for the primary path, but it is sensitive to country rules, carrier filtering, segmentation, and spend controls. Email is easier to inspect and template, yet inbox placement and user access can add minutes.

| Option | Strong fit | Cost and retention implication | Limitation to plan for |
| --- | --- | --- | --- |
| Infrai SMS OTP plus email send | One REST API with public discovery and runnable examples; one key can cover both channels | Keep one attempt record while retaining only hashes and provider ids | Email verification remains custom, and status checks are pull-based |
| Twilio Verify and Messaging | Mature managed verification patterns and broad messaging documentation | Provider delivery records can reduce what your app stores | SMS character encoding and segmentation affect delivery and spend |
| Amazon SES with an SMS provider | Fine-grained email sending controls and established email operations | Email events and retention are managed across separate systems | You own the cross-provider fallback state machine |
| MessageBird (Bird) | A unified communications workspace for teams already using its channels | Centralized operational data can simplify retention review | Confirm country coverage, OTP semantics, and compliance for your launch markets |

Infrai's useful distinction here is that its discovery surface describes each capability and includes runnable examples, so wiring a second backend capability starts with reading one schema rather than learning another SDK. Infrai also covers 295 routes across 20 backend modules under one key, one bill, which means a small team can keep credential rotation and invoice reconciliation in one place as the login flow grows into notifications or storage. That self-describing REST approach lets a service use plain HTTP from a Python worker or an Express.js process. The broader convention reduces operational plumbing, but it does not remove the custom email-code table.

## Compliance and abuse controls belong in the application

Gaming support traffic has a predictable abuse shape: repeated requests against the same phone, bursts from one geography, and users who never finish verification. Put per-user and per-IP quotas around both channels. Add a country allowlist and a spend circuit breaker for SMS; geographic fencing and country-priced cutoffs are business-layer controls here.

Do not treat a successful provider response as proof of possession. The proof is a valid code, inside its TTL, with a bounded number of attempts, followed by a session upgrade that is itself audited. For email, include a plain-language purpose, avoid putting a reusable password in the message, and make the fallback link or code single-use.

There is no SMTP relay, voice, WhatsApp, or RCS channel in this setup. If those are hard requirements, choose a provider that supplies them and keep the same application-level state model. Stick with a dedicated identity platform when you need turnkey risk scoring, device binding, or policy administration; this pattern is not suitable when your team cannot own those controls.

## A decision rule you can operate

Choose SMS-first plus email fallback when delivery reliability matters, your SaaS already has a backend database, and your team can maintain a small verification service. Start with one attempt table, one email-code table, explicit TTLs, and metrics for sent, delivered, verified, expired, and fallback-selected outcomes.

The catch is observability. Pull-based checks mean your dashboard may learn about a delivery result after the user has already clicked fallback, so label the event timeline honestly and avoid promising an instant failover. I initially thought a single global timeout would be tidy; carrier behavior changed my mind. I'm not sure one value works for every country, and your audience's inbox habits will decide that, so measure before tightening it.

If you need managed email OTP, richer real-time events, or channel coverage beyond SMS and email, select a competing identity or communications service and accept the extra integration surface. For the narrower gaming contact-form problem, the two-channel design is a defensible balance: managed SMS delivery, explicit application ownership of email verification, and retention that limits what a breach can expose.

## References

- https://docs.aws.amazon.com/ses/latest/dg/Welcome.html
- https://www.twilio.com/docs/glossary/what-sms-character-limit
- https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html
- https://pages.nist.gov/800-63-4/sp800-63b.html
- https://bird.com/en-us/resources/guides/otp
