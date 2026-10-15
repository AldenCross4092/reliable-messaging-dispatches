# One API Key Across Multiple LLM Providers: Text Classification Governance Explained

TL;DR: A single gateway credential can simplify access to OpenAI, Claude, Gemini, and other model backends, but it does not make classification requests interchangeable. For a fintech code-review service, the controlling constraint should be attribution: record every attempt against the tenant, validate the structured finding before accepting it, and permit fallback only under an explicit policy. Cheapest-first routing is a poor default because an invalid response, a duplicate retry, or an untraceable failover can cost more than the successful call and leave the review impossible to audit.

The useful abstraction is not one universal model API. It is one governed boundary around several imperfect upstreams. That boundary owns identity, schema validation, retry limits, and an append-only usage trail; each provider adapter owns the details needed to make a request. Keep those responsibilities separate.

## Can one API key govern multiple LLM providers for text classification?

A gateway-issued key answers one narrow question: who may call the gateway? A production code-review flow has harder questions. Which fintech tenant caused the work? Which repository and commit were reviewed? Which policy selected an upstream? Did the first attempt return a valid finding? Was a second attempt a retry or a fallback? Who pays for both?

Those fields must exist before routing begins. If tenant identity arrives only as free-form prompt text, it is unsuitable for authorization or accounting. Bind the credential to an allowed tenant, then carry an internal request ID and review ID through every attempt. Never accept a caller-supplied tenant label without checking it against that binding.

Identity comes first.

Delivery systems make the analogy sharp: sending an OTP is not complete merely because an upstream accepted a request. Classification is similar. A 200 response with malformed JSON, an unknown severity, or a finding that points outside the submitted diff is not a successful review. Count it as an attempt, preserve its outcome, and decide whether policy allows another one.

## Derive the ledger before the router

Start with two records. The review record describes the business operation; the attempt record describes one upstream interaction. One review can have several attempts, but only one accepted result. This keeps a timeout followed by a successful fallback from looking like one cheap, clean call.

For each attempt, capture stable dimensions such as tenant ID, review ID, adapter name, model identifier returned or selected, policy version, start and finish timestamps, outcome class, input and output usage when reported, and an immutable pricing-snapshot identifier. Store money as an integer in the smallest accounting unit chosen by your finance contract, not a floating-point value. If usage or price is unavailable, mark the cost unresolved instead of silently writing zero.

Zero is not unknown.

The ledger also needs an idempotency key. A client retry after a lost response should retrieve the existing review or resume its bounded state machine; it should not launch an unrelated second review. The same rule applies inside the gateway. Persist the attempt transition before dispatch where the storage design permits it, then reconcile ambiguous timeouts rather than pretending they never happened.

A practical state sequence is short: `queued`, `running`, `accepted`, `rejected`, or `exhausted`. Attempts have their own outcomes, including transport failure, timeout, invalid structure, policy rejection, and success. Separating review state from attempt outcome is what makes per-tenant chargeback explainable.

## Validate findings, then consider fallback

The output contract should be smaller than the model's prose capability. For code review, a finding can require `path`, `line`, `severity`, `category`, `message`, and `confidence`. The gateway should reject extra fields, unknown enum values, nonexistent paths, and line numbers outside the reviewed patch. Schema-valid nonsense is still nonsense, so deterministic checks belong after parsing.

OpenAI documents function calling as a way to connect models to application-defined tools and structured arguments. That can help an adapter request a specific shape, but the gateway must still validate returned data. Other adapters may express the same intent differently. Normalize only after provider-specific decoding; do not assume that a flag with a similar name carries identical behavior across OpenAI, Claude, and Gemini.

Fallback needs a reason matrix, not a catch-all exception handler. A connection timeout may justify one move to a different adapter. Invalid JSON may justify one constrained repair attempt or a fallback, depending on the tenant's policy. Authentication failure should stop immediately. Content or compliance rejection should not be routed around automatically, because changing providers can turn an explicit control into an accidental bypass.

Keep the limits boring: a fixed deadline, a fixed maximum attempt count, and no cycles. The numbers are policy inputs, not universal constants. For example, a tenant might allow at most two attempts for pull-request tagging while requiring manual review after a policy rejection. Another tenant may prohibit cross-provider fallback entirely. Record the rule version either way.

```python
from dataclasses import dataclass
from typing import Any, Protocol


class Adapter(Protocol):
    name: str

    def classify(self, text: str, schema: dict[str, Any]) -> dict[str, Any]: ...


@dataclass(frozen=True)
class Route:
    adapter: Adapter
    model: str


def review_change(
    *,
    tenant_id: str,
    review_id: str,
    diff: str,
    routes: list[Route],
    schema: dict[str, Any],
    ledger: Any,
) -> dict[str, Any]:
    for sequence, route in enumerate(routes, start=1):
        attempt_id = ledger.begin_attempt(
            tenant_id=tenant_id,
            review_id=review_id,
            sequence=sequence,
            adapter=route.adapter.name,
            model=route.model,
        )
        try:
            raw = route.adapter.classify(diff, schema)
            finding = validate_finding(raw, schema=schema, diff=diff)
            ledger.accept(attempt_id, usage=raw.get("usage"))
            return finding
        except RetryableUpstreamError as error:
            ledger.reject(attempt_id, reason=type(error).__name__)
            continue
        except (InvalidFinding, PolicyViolation) as error:
            ledger.reject(attempt_id, reason=type(error).__name__)
            raise

    raise ReviewExhausted(review_id)
```

The sketch deliberately leaves price calculation out of the request path. Usage collection and monetary valuation are different operations. Pin the price table used for a report, retain the raw usage evidence, and let reconciliation update unresolved entries without rewriting the history of which calls occurred.

## Compare control boundaries, not logos

There are three defensible operating shapes. They differ mainly in who owns adapters, credentials, policy enforcement, and usage evidence.

| Shape | Credential boundary | Main advantage | Main liability | Cost evidence to verify |
|---|---|---|---|---|
| Direct adapters for OpenAI, Claude, and Gemini | Your service holds separate upstream credentials | Maximum control over provider-specific behavior | More adapter, secret, and reconciliation work | Provider usage fields plus your attempt ledger |
| Self-hosted compatibility proxy | Your proxy holds upstream credentials | One internal endpoint with code-level control | Your team owns availability and semantic drift | Proxy logs reconciled with upstream records |
| Hosted aggregation gateway | The gateway credential fronts multiple upstreams | Fast access through one commercial boundary | Policy and observability depend on the gateway contract | Exportable attempt-level usage and provider identity |

Each shape has a real limitation. Direct adapters are not suitable for a small team that cannot maintain three authentication paths, response decoders, and usage reconcilers; a hosted aggregator may be the more manageable boundary if its exports and controls pass review. A self-hosted proxy is a poor fit when the team cannot operate another critical service, so direct adapters or a hosted gateway are clearer choices. Hosted aggregation is not suitable when a tenant requires credentials, routing decisions, or code data to remain inside infrastructure the team controls; direct adapters are the cleaner option in that case. The trade-off is ownership. No row wins without the tenant's compliance constraints, staffing, and evidence requirements.

OpenRouter's documentation is an example of the hosted aggregation shape, while OpenAI's function-calling guide illustrates a provider-specific structured interaction. Neither fact decides the architecture. Claude and Gemini can be evaluated through direct adapters or through an aggregation layer, but the acceptance test should remain yours.

Ask each candidate the same questions. Can it return the actual upstream and model for every attempt? Can usage be exported with your tenant and review correlation IDs? Does fallback happen only when your policy requests it, or can the service reroute implicitly? Can you distinguish a provider error from a gateway error? Can you disable data paths that violate a tenant's residency or retention rules?

Cheap is contextual. Compare the cost per accepted, policy-compliant finding, including rejected attempts and reconciliation gaps, rather than a headline unit rate. Run a fixed evaluation set of representative diffs and measure schema acceptance, deterministic-validation failures, latency distribution, and total attempts. Do not put live secrets or sensitive production code in that test corpus.

## Roll out with evidence

Begin in shadow mode: select the route and write the proposed ledger entry, but keep the existing review result authoritative. Next, enable the new boundary for one tenant cohort with fallback disabled. Reconcile attempt counts and usage against upstream records before expanding. Then allow one narrowly defined fallback reason and alert on any review that consumes more than one attempt.

The rollback unit is the policy version, not a hurried code edit. Keep adapter releases and routing policies independently identifiable so an operator can tell whether a change in accepted findings came from a model, an adapter, or a decision rule.

One key can reduce secret sprawl. The durable result comes from the controls around it: tenant-bound authorization, explicit routing, strict finding validation, bounded fallback, and an attempt ledger that finance and engineering can both reconstruct.

## Sources

- https://platform.openai.com/docs/guides/function-calling
- https://openrouter.ai/docs
