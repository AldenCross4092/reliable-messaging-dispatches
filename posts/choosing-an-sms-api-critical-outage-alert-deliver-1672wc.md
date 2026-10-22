# Choosing an SMS API: Critical Outage Alert Delivery Status and Retry Evidence

TL;DR: Choose an SMS API by the evidence it returns and the state transitions your application can enforce. For a fintech platform sending restaurant waitlist updates, the least complex workable design is a local message ledger with four terminal outcomes, a separate recipient-suppression record, bounded polling, and idempotent retry. Keep compact compliance evidence longer than verbose provider payloads. That controls the dominant storage term without losing the proof needed to explain why an invalid number stopped receiving messages.

The bill is made of message attempts, status polling, retained event bytes, and operational review. The first two grow with traffic; retained event bytes grow with traffic and time. Before comparing API prices, estimate `attempts per day x events per attempt x retained bytes x retention days`. If the system emits five 2 KB records for 100,000 daily attempts and keeps them for 365 days, the raw event body alone is about 365 GB before indexes, replicas, and backups. The useful change is separating short-lived diagnostic payloads from durable evidence.

## What evidence must survive an invalid recipient?

A delivery callback is an observation, not permission to send again. Store the provider-neutral facts that support the decision: an internal message ID, a pseudonymous recipient key, purpose, consent or other applicable authority, payload version, attempt number, observed status, provider event ID, timestamps, and the suppression reason. Keep the phone number and message body out of the evidence row unless a documented requirement demands them. This narrows access and makes deletion rules easier to enforce.

A practical state model is `queued -> accepted -> delivered`, with `failed`, `suppressed`, `canceled`, and `expired` as terminal outcomes. Do not map every provider label directly into business logic. Map it once at the adapter boundary, retain the original code for investigation during the shorter diagnostic window, and let the application act on the normalized state.

The sharp edge is ambiguity. An accepted request proves that an upstream system accepted work; it does not prove handset delivery. A timeout proves even less. Retry only when the internal idempotency key has no recorded acceptance, or when policy explicitly permits a new attempt after a terminal failure. A late callback must be able to close the original attempt without reopening a suppressed recipient.

## How should you choose an SMS API for critical outage alerts?

Suppression should be its own durable decision, checked immediately before every send. A queue entry can be valid when created and invalid by the time a worker claims it. That race matters during an outage, when a delayed waitlist update may arrive after the guest has been seated, opted out, or supplied a corrected number. Cancellation belongs in the same final pre-send gate.

Suppression wins.

The following Python sketch keeps provider syntax outside the policy. The adapter returns normalized observations; the ledger owns retry and suppression.

```python
from dataclasses import dataclass
from datetime import datetime, timezone
from enum import Enum


class State(str, Enum):
    QUEUED = "queued"
    ACCEPTED = "accepted"
    DELIVERED = "delivered"
    FAILED = "failed"
    SUPPRESSED = "suppressed"
    CANCELED = "canceled"
    EXPIRED = "expired"


TERMINAL = {
    State.DELIVERED, State.FAILED, State.SUPPRESSED,
    State.CANCELED, State.EXPIRED,
}


@dataclass(frozen=True)
class SendIntent:
    message_id: str
    recipient_key: str
    purpose: str
    expires_at: datetime


def dispatch(intent, ledger, suppression, adapter):
    now = datetime.now(timezone.utc)
    current = ledger.state(intent.message_id)

    if current in TERMINAL:
        return current
    if now >= intent.expires_at:
        return ledger.transition(intent.message_id, State.EXPIRED, now)
    if suppression.contains(intent.recipient_key):
        return ledger.transition(intent.message_id, State.SUPPRESSED, now)

    idempotency_key = f"{intent.message_id}:1"
    result = adapter.send(intent, idempotency_key=idempotency_key)
    return ledger.record_acceptance(
        message_id=intent.message_id,
        provider_event_id=result.event_id,
        observed_at=now,
    )
```

There is no automatic resend in that function. Good. Resend is a policy decision that needs a fresh suppression check, an unexpired intent, an attempt ceiling, and evidence that another notification would still be useful. For a restaurant waitlist update, usefulness can expire in minutes even though the audit record must survive much longer.

I used to treat a timeout as a retry signal. Delivery gaps make the flaw obvious: the first attempt may already be moving through the network, so a blind retry can produce two late alerts while leaving no clean account of why either was sent. The safer trade-off is slower recovery for ambiguous attempts in exchange for deterministic evidence. Reconcile first; create a new attempt only after the policy can name the reason.

## Poll narrowly, reconcile broadly

Use callbacks as the normal status path and polling as bounded reconciliation. Poll records that remain nonterminal after a defined delay, back off between checks, and stop at a fixed deadline. A cancellation request should mark the local intent immediately; if upstream cancellation is available, treat its result as another observation rather than a guarantee that already-dispatched traffic can be recalled.

Run a periodic reconciliation job over gaps: accepted messages without a terminal observation, duplicate event IDs, terminal-state reversals, and sends attempted after suppression. Alert on the age and count of those gaps. A delivery-rate chart can look healthy while a small set of invalid recipients is repeatedly retried, so the suppression-check failure rate deserves its own signal.

Test ugly ordering. Feed the adapter duplicate callbacks, a `delivered` event before `accepted`, a terminal failure after cancellation, and an event arriving after diagnostic data has expired. The invariant is compact: terminal business state never moves backward, and suppression always wins over queued work.

Late is wrong.

## Compare contracts, not dashboards

A serious evaluation uses captured fixtures and a written scorecard. Ask each candidate to demonstrate stable event identifiers, documented status meanings, authenticated callbacks, idempotent submission behavior, status lookup for reconciliation, regional processing options relevant to the deployment, exportable logs, and deletion controls. Record unknowns as unknowns. Do not turn a sales promise into a control.

The API also needs an adapter small enough to replace. This is where structured tool definitions offer a useful design lesson: give the sending boundary a narrow name, an exact schema, and descriptions that remove ambiguity. The application should not need provider-specific status strings to decide whether a recipient is suppressed.

Run the same failure suite against every adapter. A candidate that returns rich statuses but cannot support deterministic deduplication may create more compliance work than a smaller status model. Cost still belongs in the scorecard, but compare total attempts, polling volume, evidence storage, review labor, and cross-region data handling rather than a headline unit price.

## Retain the decision, expire the transcript

Use at least two retention classes. Diagnostic records include request and callback payloads, provider labels, and transport details; keep them only for the approved troubleshooting window. Evidence records include the normalized transition, policy version, purpose, timestamps, pseudonymous recipient key, and suppression decision; keep them for the period established by counsel and the organization's retention schedule. No universal duration can be inferred from an API feature list.

Deletion must cover replicas, exports, and dead-letter queues. Verify it with sampled jobs and immutable deletion receipts rather than assuming a time-to-live setting reached every copy. Access to evidence should be logged, and the lookup path should begin with an internal ID instead of a raw phone number.

This design deliberately stops keeping full message bodies, complete callback payloads, and provider request metadata after the diagnostic window. The trade-off is real: an investigation months later may establish the policy and state transition but lack the original carrier detail. Accept that loss only after legal, security, and operations agree that the durable evidence is sufficient. The resulting choice is defensible: select the SMS API whose contract preserves those invariants, then keep the minimum record that proves the system honored them.

## Further reading

- https://datatracker.ietf.org/doc/html/rfc6376
- https://platform.claude.com/docs/en/agents-and-tools/tool-use/overview
