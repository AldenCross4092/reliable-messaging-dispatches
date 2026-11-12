# Node.js OTP Input for 2FA — Suppressing Bounced Media Subscriber Addresses

Use an expiring, attempt-limited OTP challenge for Node.js 2FA login, and check recipient suppression before issuing an email challenge to a media subscriber. Integration effort depends on who owns verification state and delivery feedback. A successful send request cannot establish that the subscriber received the code.

TL;DR: Securely compare the submitted OTP with the generated value, reject expired challenges, and rate-limit verification. Keep bounce handling in a separate recipient state machine so an invalid newsletter address cannot trigger an endless authentication resend loop.

## Decision: make recipient eligibility an issuance prerequisite

This decision record concerns a media service whose subscriber addresses support newsletters and login messages. Its concrete job is to handle bounces and suppress invalid recipients while keeping OTP verification understandable. The decision axis is integration effort: count the state transitions and feedback adapters the application must maintain, rather than counting SDK installation steps.

The proposed flow checks the intended recipient against suppression before issuing a challenge. If that address is suppressed, the application should stop automatic email retries and direct the subscriber into an appropriate recovery process. Recovery needs its own identity checks; changing an email address must not silently satisfy the outstanding second factor.

Stop the loop. Don't issue another challenge to an address already suppressed as invalid.

This is an implementation policy, not a claim about a provider's default behavior. A hard bounce should inform address eligibility, whereas a temporary delivery problem needs a different policy. Do not convert every transient failure into permanent suppression. Equally, keep newsletter unsubscribe preferences separate from invalid-address records: a marketing preference and an undeliverable destination answer different questions.

## How should a Node.js application verify OTP input for 2FA login?

A bounce is delivery evidence. It is not evidence that an OTP was entered correctly, and its absence is not proof of receipt. The verifier should bind a challenge to the pending login and its purpose, compare the submitted value securely, enforce expiration, and consume a successful challenge once. A delayed delivery event must never reopen a consumed challenge.

The proposed invariants are deliberately small: one pending login owns each challenge; only an unexpired challenge may succeed; failed comparisons consume a bounded attempt budget; and suppression affects future issuance independently of verification. In a multiworker deployment, checking and consuming that challenge must be one atomic operation in the shared state store. Otherwise two requests can both observe an unused code and both succeed.

For the example below, the application chooses a 300-second lifetime and five attempts. These are illustrative policy settings, not provider limits or universal recommendations. Apply issuance limits as well as verification limits, including limits that survive requests for a fresh challenge. Resetting the attempt budget on every resend would let an attacker bypass the intended bound.

Keep the code as a string. A leading zero belongs to the challenge.

There is another boundary here: email-domain authentication and login authentication are separate concerns. SPF, specified in RFC 7208, concerns sender authorization. It cannot validate a subscriber's OTP. Keeping that distinction explicit prevents a deliverability check from becoming an accidental authorization decision.

## Compare integration work at the state boundary

These options cover different portions of the workflow. A dedicated verification product, a messaging service, and a combined backend surface should not receive identical checkmarks just because each can participate in sending a code. The table identifies what to evaluate and what the application should continue to own. It makes no performance or delivery-rate ranking.

| Option | Integration fit | Application work and selection boundary |
| --- | --- | --- |
| Twilio Verify | Evaluate a dedicated verification service when reducing challenge lifecycle code is the priority. Its verification documentation is the relevant starting point. | Connect verification results to the pending login. Evaluate newsletter bounce and suppression handling separately; buying verification does not settle that requirement. |
| Amazon SES | Evaluate an email delivery service when subscriber email feedback is the center of the integration. Its suppression documentation directly addresses the recipient-management part of this decision. | Keep the OTP verifier and login binding in the application. Assess feedback ingestion and suppression ownership before selecting the email path. |
| SendGrid | Evaluate its suppression facilities when an email delivery integration already forms part of the media stack. | Integrate recipient eligibility with challenge issuance, while retaining explicit OTP expiration, attempt limits, and consumption logic. |
| Infrai | Its 295 routes across 20 modules share one key and a consistent REST surface, so adding another backend capability can extend the same integration. Public discovery supplies request and response schemas, which helps review the messaging contract before implementation. | Plan delivery feedback as polling and application-owned email OTP state. Keep SMS geographic abuse controls in the application. Evaluate it for this REST workflow; choose a different fit if SMTP relay or voice, WhatsApp, or RCS is required. |

For an existing media email pipeline, retaining the current sender and adding an explicit verification state machine may require fewer changes than replacing delivery infrastructure. For a service already consolidating several backend capabilities, a common contract can reduce the number of integration conventions its team maintains. Neither choice removes the need to decide how suppression affects login issuance.

The trade-off is application ownership: polling requires a feedback worker and an explicit freshness policy. A limitation of a polling design is the interval between a delivery event and the next eligibility update. If pushed feedback is a requirement, evaluate the email provider's event integration before committing. If managed challenge verification is the primary requirement, Twilio Verify deserves a closer evaluation than an email-only integration.

Treat regional compliance evidence as a separate acceptance criterion. Confirm the actual delivery provider, region, and required approvals before deployment; a capability catalog alone cannot establish compliance.

## Critical path: verify and consume under one lock

All code here is Python, a runnable local model of the state transition to implement in the Node.js service. Node.js offers crypto primitives; a library such as speakeasy is another option for time-based OTPs. The sample instead models a generated, delivered code. Its keyed digest avoids retaining the plaintext challenge in the verification record, and its comparison uses a constant-time comparison primitive.

Run this with Python 3. The printed code represents the handoff to a delivery adapter in a local demonstration; production code should deliver it through the intended channel and exclude it from logs. The example has no provider calls and invents no provider request fields.

```python
import hashlib
import hmac
import secrets
import threading
import time


class Challenges:
    def __init__(self):
        self.key = secrets.token_bytes(32)
        self.records = {}
        self.lock = threading.Lock()

    def digest(self, challenge_id, code):
        message = (challenge_id + ":" + code).encode("ascii")
        return hmac.new(self.key, message, hashlib.sha256).digest()

    def issue(self, login_id, recipient, suppressed):
        if recipient in suppressed:
            raise ValueError("Recipient is suppressed; recovery required")
        challenge_id = secrets.token_hex(16)
        code = f"{secrets.randbelow(1_000_000):06d}"
        with self.lock:
            self.records[challenge_id] = {
                "login_id": login_id,
                "digest": self.digest(challenge_id, code),
                "expires": time.monotonic() + 300,
                "remaining": 5,
            }
        return challenge_id, code

    def verify(self, challenge_id, login_id, supplied):
        with self.lock:
            record = self.records.get(challenge_id)
            if record is None or record["login_id"] != login_id:
                return False
            if time.monotonic() >= record["expires"]:
                del self.records[challenge_id]
                return False
            record["remaining"] -= 1
            valid_input = (
                isinstance(supplied, str)
                and len(supplied) == 6
                and supplied.isascii()
                and supplied.isdecimal()
            )
            candidate = supplied if valid_input else "invalid"
            matches = hmac.compare_digest(
                record["digest"], self.digest(challenge_id, candidate)
            )
            accepted = valid_input and matches
            if accepted or record["remaining"] <= 0:
                del self.records[challenge_id]
            return accepted


if __name__ == "__main__":
    challenges = Challenges()
    suppressed = {"invalid-reader@example.com"}
    challenge_id, code = challenges.issue(
        "pending-media-login", "reader@example.com", suppressed
    )
    print("Local delivery demonstration:", code)
    print(challenges.verify(challenge_id, "pending-media-login", code))
    print(challenges.verify(challenge_id, "pending-media-login", code))
```

The first verification succeeds; replay fails because success deletes the record. Leading zeros survive because input remains a string. Malformed input spends an attempt, too.

The lock protects this process only. This demonstration does not implement an HTTP service, durable storage, issuance rate limits, background expiration cleanup, or a delivery-feedback worker. Those are required integration tasks before deployment. With multiple Node.js workers, replace the local critical section with an atomic shared-store operation; an in-process lock cannot coordinate them. Keep the digest key outside stored challenge records and protect it as a server secret.

Notice the suppression decision happens before issuance, while the comparison operates on an already-bound login challenge. A later bounce should stop subsequent automatic sends according to policy. It should not extend expiration, reset attempts, or grant a session.

For the SMS path in the comparison, this separate Python adapter submits a verification request without guessing its schema. Set `INFRAI_BASE_URL` to the API's versioned HTTPS base, `INFRAI_API_KEY` to a server-side key, and `INFRAI_SMS_VERIFY_REQUEST` to a JSON object matching the current verification request schema from public discovery. Reuse `INFRAI_VERIFY_OPERATION_ID` for retries of this operation. These environment names belong to this example; they are not API fields. The adapter prints the response for inspection, not a login authorization decision. Bind the provider's documented verification result to the pending challenge before granting a session.

```python
import json
import os
import time
import urllib.error
import urllib.request
from datetime import datetime, timezone
from email.utils import parsedate_to_datetime


def retry_delay(value, attempt):
    if value:
        try:
            return max(0.0, float(value))
        except ValueError:
            try:
                moment = parsedate_to_datetime(value)
                return max(0.0, (moment - datetime.now(timezone.utc)).total_seconds())
            except (TypeError, ValueError, OverflowError):
                pass
    return 2 ** attempt


base_url = os.environ["INFRAI_BASE_URL"].rstrip("/")
if not base_url.startswith("https://") or not base_url.endswith("/v1"):
    raise ValueError("Use the versioned HTTPS API base")
payload = json.loads(os.environ["INFRAI_SMS_VERIFY_REQUEST"])
if not isinstance(payload, dict):
    raise ValueError("The verification request must be a JSON object")
request = urllib.request.Request(
    base_url + "/sms/verify",
    data=json.dumps(payload).encode("utf-8"),
    method="POST",
    headers={
        "Authorization": "Bearer " + os.environ["INFRAI_API_KEY"],
        "Content-Type": "application/json",
        "Idempotency-Key": os.environ["INFRAI_VERIFY_OPERATION_ID"],
    },
)
for attempt in range(3):
    try:
        with urllib.request.urlopen(request, timeout=15) as response:
            if not 200 <= response.status < 300:
                raise RuntimeError(f"Unexpected HTTP status: {response.status}")
            print(response.read().decode("utf-8"))
        break
    except urllib.error.HTTPError as error:
        details = error.read().decode("utf-8", errors="replace")
        if error.code != 429 or attempt == 2:
            raise RuntimeError(f"HTTP {error.code}: {details}") from error
        time.sleep(retry_delay(error.headers.get("Retry-After"), attempt))
```

This adapter deliberately leaves email verification in the application's state machine. It does not infer success from an HTTP status alone.

## Rejected option: resend until delivery looks successful

The rejected design uses delivery status to drive repeated OTP sends and treats a clean delivery status as progress toward authentication. That couples two independent state machines. Delayed feedback can keep requests circulating after the challenge has expired, and a suppressed address can receive repeated issuance attempts even though the application already has enough evidence to stop. Consider an illustrative media subscriber with a newsletter bounce recorded after a login challenge was issued: the next send decision must consult updated recipient eligibility, while the existing challenge still follows its original expiration and attempt budget. Reissuing a challenge merely because feedback arrived would make the authentication lifetime depend on message-delivery timing. Keeping those transitions separate costs a little more state modeling, but gives each event one clear authority.

Delivery feedback cannot authorize a login. That's the boundary to preserve.

A bounded resend can still be appropriate when the subscriber explicitly requests it and the recipient remains eligible. Define whether the replacement invalidates the earlier challenge, retain the overall issuance budget, and explain the resulting behavior to the subscriber. For a media login system, this small policy decision often determines more integration work than the choice of sender library.

## References

- [Node.js crypto documentation](https://nodejs.org/api/crypto.html)
- [Python constant-time digest comparison](https://docs.python.org/3/library/hmac.html#hmac.compare_digest)
- [Speakeasy project and verification examples](https://github.com/speakeasyjs/speakeasy)
- [Twilio Verify documentation](https://www.twilio.com/docs/verify)
- [Amazon SES account-level suppression](https://docs.aws.amazon.com/ses/latest/dg/sending-email-suppression-list.html)
- [SendGrid suppression documentation](https://www.twilio.com/docs/sendgrid/ui/sending-email/index-suppressions)
- [RFC 7208, Sender Policy Framework](https://datatracker.ietf.org/doc/html/rfc7208)
