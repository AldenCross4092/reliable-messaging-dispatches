# Secure Marketplace Account Deletion with Consent Cleanup and Session Revocation

**Short answer:** Delete a marketplace account only after fresh authentication, then make local session revocation the first irreversible security action and treat consent cleanup plus user removal as a retryable workflow. This order favors session security without forcing the user to wait for every downstream cleanup call.

For a marketplace that accepts Google and GitHub sign-in, the dangerous design is a single `DELETE` handler that calls every dependency and hopes they all finish. A safer decision is an account-state transition backed by durable work: verify the deletion request, block new sessions, revoke existing sessions, record cleanup jobs, and let idempotent workers remove identity and profile data. External consent revocation may lag; local access must not.

## What should an account deletion workflow revoke before user removal?

The workflow has four invariants. First, a request must be tied to a recently authenticated session rather than possession of an old browser cookie. OWASP recommends reauthentication for sensitive features and after risk events, with session invalidation and token rotation around reauthentication. Second, once deletion starts, no refresh, passwordless callback, or social sign-in callback may create a new active session for that account. Third, retries must not restore access or duplicate side effects. Fourth, retained records must be deliberately separated from active identity data instead of leaving a half-usable account.

Reauthentication is the visible friction point. For a user who signed in through Google or GitHub, ask for a fresh provider authentication or another already enrolled factor, bind the proof to this deletion intent, and expire that proof quickly according to the service's risk policy. Don't silently treat a weeks-old session as consent to delete. Equally, don't require a password that a social-only account never created.

The failure boundary sits between local authorization and downstream cleanup. Local account state and session validity are under the marketplace's control, so they belong in the committed security boundary. A provider's consent endpoint, an email suppression store, a payout ledger, or an analytics erasure queue may be temporarily unreachable without being allowed to reopen the account. Store each obligation durably and retry it. This distinction matters during an ambiguous retry: the client may not know whether its confirmation arrived, but the server can still return the pending state, preserve the original cleanup plan, and refuse any attempt to mint a session. The workflow moves forward from durable state; it never reconstructs security state from a browser's guess about the last response.

Retries will happen.

No new sessions.

That rule must also cover races. A social callback can arrive while deletion is being confirmed; a mobile refresh token can be redeemed at nearly the same moment; a seller may still have an active dashboard tab. Every session-creation path therefore checks the same account state, and session records carry an account security version so a version increment invalidates cached credentials even before individual session rows are swept. A stale callback may complete protocol validation, but it must not cross the final local account-state check.

## Invariants and failure boundaries

Model deletion as a small state machine: `active -> deletion_pending -> deleted`. The transition to `deletion_pending` freezes authentication and creates a durable cleanup plan in one database transaction. The worker can then revoke local sessions, remove stored provider credentials and grants, detach marketplace roles, apply the service's documented retention policy, and finally remove the login identity. `deleted` means the required jobs reached their terminal outcomes; it should not mean that an HTTP request happened to return success.

Consent cleanup has two layers. Remove the marketplace's stored provider access and refresh credentials so they can never be used again. If the provider offers a standards-compatible revocation operation for the credential type in use, invoke it through a provider adapter. Those actions are related but not interchangeable: deleting a local token copy stops this service from using its copy, while provider-side revocation addresses the provider's grant. The adapter keeps provider-specific mechanics out of the account state machine.

Be careful with retained data. Order history, dispute evidence, financial records, marketing consent, notification suppression, and authentication identifiers do not necessarily share one lifecycle. The deletion plan should name the owner, retention rule, and terminal action for each category. This is a design requirement, not an excuse to retain everything. In particular, preserving an email solely in a suppression list should not leave it usable as a login identifier; the authentication lookup and the compliance record need distinct namespaces and permissions. The exact retention periods depend on the marketplace's jurisdiction and contracts, and I'm not sure a universal number would be defensible without that context.

Use generic outcomes at the API edge. A repeated deletion request should reach the same safe state rather than reveal whether a particular identity still exists. Internally, keep enough audit data to answer which authenticated actor requested deletion, which policy version created the plan, and which cleanup step remains, while excluding reusable secrets.

## Option comparison

| Option | Security boundary | User friction | Failure behavior | Best fit |
| --- | --- | --- | --- | --- |
| One synchronous request | Depends on every cleanup dependency | User waits for the slowest step | Partial completion is difficult to classify | Small, self-contained systems with no remote cleanup |
| Transactional state change plus durable jobs | Local access closes in the first commit | Confirmation can return after durable acceptance | Each cleanup obligation retries independently | Marketplaces with social identity and multiple data owners |
| Cooling-off period before freeze | Access remains available until the delay ends | Easy cancellation | A compromised session keeps a wider window | Low-risk communities where recovery outweighs immediate revocation |

The middle option is the default here because a marketplace account crosses identity, sessions, listings, messaging, and transaction records. The catch is operational weight: it needs a worker, an outbox or equivalent durable queue handoff, idempotency keys, dead-letter review, and a status model that support staff can interpret. A compact application whose data lives in one database may reasonably choose a single transaction instead. Stick with a cooling-off period only when the product explicitly accepts the longer access window; it is not suitable when the deletion request may follow account takeover.

## Critical path in Python

The following focused example shows the boundary. It omits web-framework plumbing and provider-specific wire formats; those belong in adapters whose contracts can be tested separately. The transaction changes account state, increments the security version, revokes server-side sessions, and inserts cleanup jobs atomically.

```python
from dataclasses import dataclass
from datetime import datetime, timezone
from typing import Protocol
from uuid import UUID


@dataclass(frozen=True)
class DeletionCommand:
    account_id: UUID
    actor_id: UUID
    reauth_proof_id: UUID
    request_id: UUID


class DeletionStore(Protocol):
    def verify_fresh_proof(
        self, account_id: UUID, actor_id: UUID, proof_id: UUID
    ) -> None: ...

    def begin(self) -> None: ...
    def mark_pending(self, account_id: UUID, requested_at: datetime) -> bool: ...
    def increment_security_version(self, account_id: UUID) -> None: ...
    def revoke_sessions(self, account_id: UUID) -> None: ...
    def enqueue_once(
        self, request_id: UUID, account_id: UUID, task_name: str
    ) -> None: ...
    def commit(self) -> None: ...
    def rollback(self) -> None: ...


def request_account_deletion(
    command: DeletionCommand, store: DeletionStore
) -> str:
    store.verify_fresh_proof(
        command.account_id, command.actor_id, command.reauth_proof_id
    )
    store.begin()
    try:
        changed = store.mark_pending(
            command.account_id, datetime.now(timezone.utc)
        )
        if not changed:
            store.commit()
            return "deletion_pending"

        store.increment_security_version(command.account_id)
        store.revoke_sessions(command.account_id)
        for task_name in (
            "revoke_provider_consent",
            "apply_data_retention_policy",
            "remove_login_identity",
        ):
            store.enqueue_once(
                command.request_id, command.account_id, task_name
            )
        store.commit()
    except Exception:
        store.rollback()
        raise

    return "deletion_pending"
```

The `mark_pending` result makes a retry boring: an already pending or deleted account does not receive a second state transition. The outbox uniqueness constraint should use the request, account, and task identity represented by `enqueue_once`. Workers also need idempotent handlers because delivery can repeat after a worker performs a side effect but loses its acknowledgement. None of this replaces authorization at the edge; `verify_fresh_proof` must validate that the proof belongs to this actor, this account, and this specific sensitive action.

Test the races, not just the happy path. Hold a refresh request open while committing deletion and assert that its final account-state check denies session creation. Replay the same deletion command twice and assert one set of jobs. Interrupt each worker after its external side effect, run it again, and verify the terminal state. Check that a deleted social identity cannot be relinked by an old callback. Then inspect logs to ensure tokens, authorization codes, and reauthentication proofs never appear. A `401` after the security-version change is expected; an accepted refresh is the test failure.

Prove the race.

## Rejected option and its valid use case

A fully synchronous cascade was rejected for this marketplace. It couples the user's confirmation request to unrelated stores and remote consent operations, encourages long database transactions, and makes an ambiguous client retry harder to reason about. Fast response time isn't the main concern; a crisp boundary is.

The synchronous design still has a valid use case. If an application owns one transactional database, has no remote grant to revoke, and can remove every active credential and identity row within that transaction, the smaller design is easier to operate and easier to prove correct. Keep the same invariants: fresh authorization, atomic session closure, idempotent requests, and a final state check on every login path. Do not adopt a worker fleet merely to look sophisticated.

For the marketplace case, the acceptance criteria are plain: after the first commit, neither Google nor GitHub callbacks nor existing sessions can restore access; every cleanup obligation is observable and retryable; retained records cannot authenticate; and support can distinguish pending cleanup from completed removal without seeing reusable credentials. This gives session security priority while containing friction to one explicit reauthentication step and an immediate acknowledgement.

## References

- OWASP Authentication Cheat Sheet: https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html
