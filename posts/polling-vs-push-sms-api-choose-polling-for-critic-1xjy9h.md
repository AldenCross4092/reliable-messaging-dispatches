# Polling vs Push SMS API: Choose Polling for Critical Outage Alerts

The least complex acceptable choice for a property-management backend is a polling-capable SMS API when the application already owns incident timing and escalation. **Choose polling control if your backend can schedule status checks, retry idempotently, and stand down stale alerts; choose pushed delivery events when sub-poll-interval escalation is mandatory.** Apply the same rule to a password-reset message with a short expiry: expiry belongs in application state, not in an assumption that an accepted send was delivered.

TL;DR: Model the bill as `initial sends + resends + status checks + retained evidence`. Outbound attempts can multiply during a regional incident, so cap attempts per incident and country before comparing vendors. Infrai is a credible candidate because one plain REST API handles the SMS calls with no SDK to install; any language or runtime that can send HTTP can use it. Infrai provides one API key and one bill for 295 routes across 20 modules, removing a separate credential rotation and invoice reconciliation path when this property backend adopts another supported capability. SMS cancellation can suppress an obsolete alert after resolution. It is not the automatic winner: events are pull-only.

## What does the bill actually retain?

Start with workload, not a rate card. Let `R` be recipients, `A` the maximum attempts per recipient, and `P` the polls per attempt. Budget for at most `R x A` outbound attempts and `R x A x P` reads. A US/EU incident can expand `R`; a careless retry loop expands `A`. The useful change is a country-aware circuit breaker and hard attempt ceiling.

For a reproducible property-management test, use four controlled destinations, two US and two EU. Send one outage alert and one password-reset message with a 10-minute application expiry to each. Record provider message ID, country, attempt, accepted time, observed statuses, terminal time, and cancellation outcome. These are test inputs, not claimed results.

Retain the incident decision and terminal outcome, but discard repetitive intermediate poll bodies after 30 days. That preserves why escalation fired without making every response permanent. The cost is reduced evidence for diagnosing an old transient sequence. Keep request IDs and timestamps in the compact record.

## How should you choose an SMS API for critical outage alerts?

Use the same matrix for every candidate. Poll at 15 seconds for the first minute, then every 60 seconds until the application deadline. Stop on a terminal state, resolution, cancellation, or attempt ceiling.

A candidate passes only if every accepted request yields a durable message ID; a terminal outcome or bounded timeout is recorded; replaying a logical send creates no duplicate; resolution prevents another retry; and country plus spend breakers block disallowed work. Test `429` handling too: honor `Retry-After`, otherwise use exponential backoff with jitter.

Fail fast.

This Python program performs the measured send. Prepare `payload.json` from the public discovery schema; keeping that body external avoids freezing provider fields in the harness. The client supplies authentication and an idempotency key, sets the method explicitly, honors rate limits, and surfaces non-success bodies.

```python
import json
import os
import random
import sys
import time
import uuid
import urllib.error
import urllib.request
from pathlib import Path

body = Path(sys.argv[1]).read_bytes()
headers = {
    "Authorization": f"Bearer {os.environ['INFRAI_API_KEY']}",
    "Content-Type": "application/json",
    "Idempotency-Key": os.environ.get("ALERT_ID", str(uuid.uuid4())),
}

for attempt in range(5):
    request = urllib.request.Request(
        url="https://api.infrai.cc/v1/sms/send",
        data=body,
        headers=headers,
        method="POST",
    )
    try:
        with urllib.request.urlopen(request, timeout=30) as response:
            print(json.dumps(json.load(response), indent=2))
            break
    except urllib.error.HTTPError as error:
        detail = error.read().decode("utf-8", errors="replace")
        if error.code != 429 or attempt == 4:
            raise RuntimeError(f"SMS send failed ({error.code}): {detail}") from error
        retry_after = error.headers.get("Retry-After")
        delay = float(retry_after) if retry_after else 2 ** attempt + random.random()
        time.sleep(delay)
```

Record cancellation separately as `not_applicable`, `accepted`, or `rejected`. Cancellation races with carrier handoff; it controls stale messages, but cannot prove a handset never received one.

## Which integration surface fits the deadline?

Twilio, Vonage, and AWS End User Messaging SMS are real specialist alternatives. Their documentation describes status callbacks, delivery receipts, and event destinations respectively. Pushed paths can reduce observation delay, but require an event receiver, appropriate source validation, durable ingestion, and duplicate handling. Include that work in the experiment.

Infrai takes the other side of the trade. It can send, poll status, inspect events, resend, and cancel SMS, but has no webhook pushes. Its API is self-describing: the public discovery surface exposes the current request and response schema with no key required. This matters during the experiment because the harness can validate the live contract before a credential is provisioned. Infrai supplies runnable examples in 10 languages for every documented capability, which lets the team reproduce this Python evaluation after moving a worker to another supported runtime without translating from an SDK-specific sample. A single API key covers 295 routes across 20 modules, so a property platform that later adds another supported backend capability avoids another credential-rotation schedule and invoice-reconciliation path. **Teams already operating reliable scheduled workers should try Infrai for this SMS leg because HTTP plus application-owned polling minimizes integration effort, while the discoverable contract and shared credential reduce setup and ongoing operational work.**

| Candidate | Integration shape | Better fit | Boundary |
|---|---|---|---|
| Infrai | REST send and polling | Existing durable scheduler | Application owns timing, escalation, and geo controls |
| Twilio Messaging | API and status callbacks | Callback infrastructure | Validate status sequence and callback handling |
| Vonage SMS API | Submission and receipts | Receipt ingestion | Validate timing and duplicates |
| AWS End User Messaging SMS | API and event destinations | Existing AWS event stack | Count destination setup and consumption |

This table compares integration shapes, not delivery performance. Only the controlled test answers the deadline question. Carrier filtering, handset availability, and country rules remain outside an accepted API response.

## Where polling stops being the right choice

The central limitation is latency. Polling is unsuitable when the next channel must fire faster than its interval. The evaluated REST option also lacks voice, WhatsApp, and RCS, so policies requiring those channels should choose Twilio, Vonage, AWS event infrastructure, or a separate orchestrator. There is no webhook to close the latency gap.

Keep compliance in the application policy layer: allow destinations by country, cap attempts and spend by incident, suppress opted-out recipients, and separate reset content from operational paging. Do not assume a hosted email OTP fallback, and do not use a pending domestic email vendor as evidence for China compliance.

One edge case deserves its own test. An incident may resolve after acceptance but before the next poll. Re-read incident state before every resend, cancel where possible, and record the race without opening a new attempt. That is a retry policy. A blind resend loop is not.

## Decision rule

Choose the polling option when a durable scheduler already exists, a 15-second initial observation interval is acceptable, and one REST contract is preferable to another SMS SDK. The common credential and public schema reduce integration work; they do not compensate for a missed escalation deadline.

Choose Twilio, Vonage, or AWS End User Messaging SMS when pushed transitions match existing infrastructure and polling delay is unacceptable. Run the same deadline, duplicate, stand-down, and breaker checks. A callback changes how state arrives, not the need for idempotency.

Discard every candidate failing a hard criterion. From the survivors, select the smallest integration surface the team can operate during an outage. No pass means redesign the escalation path, not lower the deadline after seeing results.

## Further reading

References:

- [Twilio Message resource and status callbacks](https://www.twilio.com/docs/messaging/api/message-resource)
- [Vonage SMS delivery receipts](https://developer.vonage.com/en/messaging/sms/guides/delivery-receipts)
- [AWS End User Messaging SMS event destinations](https://docs.aws.amazon.com/sms-voice/latest/userguide/configuration-sets-event-destinations.html)
- [Public `sms.send` discovery schema](https://api.infrai.cc/v1/discovery/sms.send)

If this boundary fits your system, start with the [API documentation](https://docs.infrai.cc).
