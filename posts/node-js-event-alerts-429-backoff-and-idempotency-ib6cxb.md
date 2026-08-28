# Node.js Event Alerts: 429 Backoff and Idempotency for Email/SMS APIs

Short answer: put email and SMS event notifications behind durable, purpose-specific queues; assign one stable idempotency key before enqueueing; and let a shared dispatcher handle HTTP 429 responses with bounded backoff while uncertain outcomes go to reconciliation instead of an immediate retry.

The important boundary isn't the API client. It's the point where an accepted application event becomes a durable delivery intent. In a Node.js service, the request handler should commit that intent and return. A worker can then enforce rate limits across replicas, protect OTP traffic from bulk-email bursts, and preserve enough state to answer the uncomfortable question after a timeout: did the provider accept this message or not?

This note records that architecture decision. The example is in Python because the delivery state machine is easier to inspect without framework machinery; the same states belong in a Node.js worker, and no runtime-specific retry package changes the underlying failure boundaries.

## What must remain true at every delivery boundary?

Four invariants matter.

First, an application event creates at most one delivery intent for each `(event, channel, recipient, purpose)` tuple. The unique key belongs in durable storage, not process memory. Redelivering an event, restarting a worker, or deploying a second replica must collide with the same record rather than create a new message.

Second, message priority is set by purpose. A login code and a product digest may both use SMS, but they don't have the same latency budget or compliance consequences. Put OTP, transactional, and bulk work in separate lanes with separate concurrency and admission controls. A global provider limit can still sit above those lanes; the point is to shed or defer low-priority traffic before it consumes the capacity reserved for a short-lived code.

Priority is policy.

Third, provider acceptance and recipient delivery are different states. An accepted API call can move a job out of the send queue, while a later delivery receipt updates the final status. Don't turn a delayed receipt into another send. Email authentication belongs on this boundary too: RFC 7489 defines DMARC policy and identifier alignment, so retries cannot compensate for a domain-authentication configuration that prevents trustworthy handling by receivers.

Fourth, consent is evaluated again at dispatch time. An unsubscribe or SMS opt-out can arrive after enqueueing but before a worker obtains its lease. Checking only when the original event was created leaves a compliance race — and a perfectly functioning retry loop will amplify it. The dispatch transaction should reject suppressed recipients before any external call.

These invariants define the failure boundaries. A duplicate event is handled at insertion. A worker crash is handled by an expiring lease. A 429 is scheduler input. A definitive client-side rejection becomes terminal. A network timeout after bytes were sent is ambiguous and moves to reconciliation, because blindly retrying an unknown result can send twice.

## How should a Node.js event notification API handle rate limits and retry backoff?

Treat a rate limit as shared capacity feedback, not as an exception local to one request. The dispatcher should honor a valid server-provided delay when one is available; otherwise it should calculate capped exponential backoff with jitter. It must persist `next_attempt_at` before releasing the job. Sleeping inside an Express handler or an individual worker hides demand, occupies resources, and lets every replica make the same optimistic decision at once.

I've watched a broad `catch` block turn HTTP 429 into noise: it retried quickly, logged only the final success, and made the throttle invisible to operators. The painful edge wasn't the status code itself — it was that the code deciding to retry had no view of other workers. Keep counters for attempted sends, throttled sends, age of the oldest ready job, time from event creation to acceptance, and reconciliation backlog. Split them by channel and purpose; combining OTP and newsletters into one latency chart erases the signal that matters.

The idempotency key should be deterministic and versioned. Canonicalize an email address or phone number with the same rules used by the recipient store, then hash the event ID, channel, canonical recipient, purpose, and key version. Don't include an attempt number or current time. If the external API supports its own idempotency mechanism, send the same key on every attempt, but retain the local uniqueness constraint because provider retention and semantics may differ.

Backoff needs an end.

Set a maximum attempt count and an event expiry based on purpose, then fail closed when either is reached. Retrying an expired OTP is worse than marking it expired: the user may already have requested a newer code, and arrival order can reverse. Bulk messages can tolerate a longer schedule, while account-security notifications may need an explicit alternate path. Those are product rules, so they should be visible configuration rather than constants buried in an HTTP wrapper.

I'm not sure what scope a particular provider applies to a published limit unless its contract says so. Assume account-wide capacity until measurement proves a narrower boundary, and key the limiter by the documented scope. Your mileage may vary across regions or message classes, which is exactly why `limit_scope` and queue age belong in telemetry.

## Which queue shape makes the failure states visible?

The choice is less about fashionable infrastructure than ownership of concurrency, leases, and replay. This comparison uses failure visibility as the axis:

| Shape | Duplicate control | Rate-limit coordination | Operational fit |
| --- | --- | --- | --- |
| Direct calls from the event handler | Depends on request lifetime and upstream replay behavior | Per process unless separately coordinated | Low-volume internal alerts where a missed or duplicate message is acceptable |
| One durable queue for every message | Stable key and lease can be centralized | Global control is straightforward, but bulk work can block OTP | Modest transactional traffic with similar urgency |
| Purpose-specific queues behind one dispatcher | Stable key remains global; leases remain per job | Shared ceiling plus reserved capacity per lane | Mixed OTP, transactional, and bulk workloads |
| Separate dispatchers by channel and account | Isolation is strongest but deduplication must span dispatchers | Each documented scope gets its own limiter | Large multi-tenant systems that can support more moving parts |

For event notifications that include OTP, the decision here is purpose-specific queues behind one dispatch boundary. It keeps a common idempotency ledger while preventing a scheduled campaign from consuming every ready slot. The catch is operational weight: teams must tune lane capacity, alert on starvation, and test leases during deployment. It is not suitable when traffic is tiny and duplicates have no material effect; stick with a direct call for a disposable internal alert if the caller can surface failure and nobody will mistake API acceptance for human delivery.

Deployment deserves a deliberate test. Pause a worker after it acquires a lease, start the new version, and verify that the job becomes eligible exactly once when the lease expires. Feed the dispatcher a sequence containing 429, acceptance, a delayed delivery receipt, an opt-out update, and an ambiguous timeout. The assertions should cover database states and outbound call count, not merely whether a retry helper was invoked.

No live recipient belongs in that test dataset.

## What does the critical delivery state machine look like?

The focused path below leaves transport and storage behind interfaces. `claim()` must atomically lease a ready row, while every transition must compare the lease token so an old worker cannot overwrite a newer result. The configured numbers are examples, not claims about any external API.

```python
from __future__ import annotations

from dataclasses import dataclass
from datetime import datetime, timedelta, timezone
from enum import Enum
import hashlib
import random


class ResultKind(Enum):
    ACCEPTED = "accepted"
    RATE_LIMITED = "rate_limited"
    REJECTED = "rejected"
    UNKNOWN = "unknown"


@dataclass(frozen=True)
class SendResult:
    kind: ResultKind
    retry_after_seconds: int | None = None
    receipt_id: str | None = None


@dataclass(frozen=True)
class Delivery:
    event_id: str
    channel: str
    canonical_recipient: str
    purpose: str
    attempt: int
    expires_at: datetime
    lease_token: str


def delivery_key(delivery: Delivery, version: int = 1) -> str:
    material = "|".join(
        (
            str(version),
            delivery.event_id,
            delivery.channel,
            delivery.canonical_recipient,
            delivery.purpose,
        )
    )
    return hashlib.sha256(material.encode("utf-8")).hexdigest()


def retry_delay(attempt: int, retry_after_seconds: int | None) -> timedelta:
    if retry_after_seconds is not None and retry_after_seconds >= 0:
        seconds = min(retry_after_seconds, 300)
    else:
        ceiling = min(2 ** attempt, 120)
        seconds = random.uniform(0, ceiling)
    return timedelta(seconds=seconds)


def dispatch_one(store, transport, limiter, now: datetime | None = None) -> None:
    now = now or datetime.now(timezone.utc)
    delivery = store.claim(now=now)
    if delivery is None:
        return

    if delivery.expires_at <= now:
        store.mark_expired(delivery.lease_token)
        return

    if store.is_suppressed(delivery.channel, delivery.canonical_recipient):
        store.mark_suppressed(delivery.lease_token)
        return

    if not limiter.reserve(delivery.channel, delivery.purpose):
        store.reschedule(delivery.lease_token, now + timedelta(seconds=1))
        return

    result = transport.send(
        channel=delivery.channel,
        recipient=delivery.canonical_recipient,
        idempotency_key=delivery_key(delivery),
    )

    if result.kind is ResultKind.ACCEPTED:
        store.mark_accepted(delivery.lease_token, result.receipt_id)
    elif result.kind is ResultKind.RATE_LIMITED:
        store.reschedule(
            delivery.lease_token,
            now + retry_delay(delivery.attempt, result.retry_after_seconds),
        )
    elif result.kind is ResultKind.UNKNOWN:
        store.mark_for_reconciliation(delivery.lease_token)
    else:
        store.mark_rejected(delivery.lease_token)
```

Notice the boring branch: `UNKNOWN` does not call `send()` again. A reconciliation worker should query by the stable key or receipt information when the external contract permits that lookup; if the outcome cannot be established, a product-specific policy decides whether duplicate risk or omission risk wins. For OTP, expiring the intent and allowing the user to request a new code is often clearer than delivering two generations out of order.

WebOTP changes the browser handoff, not the delivery guarantees. MDN documents that it is available only in secure contexts and that the user agent extracts a specially formatted code from an SMS after user consent. Use it as a progressive enhancement: keep a manual input path, bind the code to the intended authentication transaction, and don't interpret autofill as proof that every mobile browser or SMS route behaves identically.

## Why reject automatic email-to-SMS failover?

Automatic cross-channel failover looks attractive because it converts a delayed email into an SMS attempt. I would reject it as the default. Channel consent, message length, sender identity, suppression state, cost controls, and user expectation are different; a timeout also leaves the first channel's outcome unknown, so failover can produce two messages with inconsistent wording.

There is a valid use case. A security alert with explicit consent for both channels can use a policy-driven escalation after a defined wait, provided both intents share an incident correlation ID and each channel keeps its own idempotency key. That is escalation, not a transport retry. Keep it out of the generic dispatcher so compliance review and delivery reporting can see the decision.

The final ADR decision is therefore narrow: durable intent before I/O, stable keys, purpose isolation, centrally coordinated throttling, and reconciliation for ambiguity. It won't make authentication or consent mistakes disappear. It does make them observable at the boundary where a team can act on them.

## References

- RFC 7489, Domain-based Message Authentication, Reporting, and Conformance (DMARC): https://datatracker.ietf.org/doc/html/rfc7489
- MDN, WebOTP API: https://developer.mozilla.org/en-US/docs/Web/API/WebOTP_API

## Further reading

The two primary references above are the useful next stops: RFC 7489 for email-domain policy and alignment, and MDN's WebOTP API guide for the browser-side OTP contract.
