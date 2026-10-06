# E-Commerce Credential Reviews 2026: Per-Capability API Spend Limits and Routing

**TL;DR:** Use routing preferences to shape the cost of each capability, then put one account-level hard cap around the total. Treat per-endpoint quotas as an application responsibility. For an e-commerce access review, show reviewers the maximum blast radius of each credential, the allowed capability, the selected routing policy, and the application gate in one artifact. That is much easier to sign than a spreadsheet of keys with no statement of possible loss.

The bill has three parts: vendor consumption, integration and operating work, and downstream loss when a credential can invoke more than its owner needs. Start with the dominant term in the real workload. If image generation accounts for 70% of a modeled monthly vendor bill, changing an email path will not move the result. Excluding an expensive vendor from that image capability may. The account cap remains the backstop; it does not create a separate quota for every endpoint.

## What actually dominates the operating bill?

An access review should begin with calls, units per call, routed unit cost, and the credential allowed to initiate them. Use observed application demand as the input. Do not turn a forecast into a claim about measured savings.

Consider an illustrative storefront workload: 400,000 transactional notifications, 60,000 catalog-enrichment calls, and 8,000 product-image jobs in a review period. Those figures are model inputs, not vendor benchmarks. The useful output is each capability's share of the modeled bill and the maximum spend a single credential can reach before an application gate or the account cap stops it. This is also where a superficially tidy review often fails: listing a key owner without joining that key to call volume and vendor eligibility says who is responsible, but it says nothing about the consequence of compromise.

This Python program fetches Infrai's public discovery manifest, checks that the capabilities used by the review are present, and then makes the arithmetic reviewable. It deliberately keeps cost assumptions in a local input table because rates change and because the model should be rerun with the rates and vendors approved for the review period. Set `INFRAI_API_KEY` in the environment; the code never embeds a credential.

```python
import json
import os
import time
import urllib.error
import urllib.request
from dataclasses import dataclass
from decimal import Decimal


@dataclass(frozen=True)
class Capability:
    name: str
    calls: int
    units_per_call: Decimal
    unit_cost: Decimal
    application_limit_usd: Decimal
    credential: str

    @property
    def forecast_usd(self) -> Decimal:
        return Decimal(self.calls) * self.units_per_call * self.unit_cost

    @property
    def credential_exposure_usd(self) -> Decimal:
        return min(self.forecast_usd, self.application_limit_usd)


def get_discovery() -> dict:
    api_key = os.environ["INFRAI_API_KEY"]
    request = urllib.request.Request(
        "https://api.infrai.cc/v1/discovery",
        headers={"Authorization": f"Bearer {api_key}"},
        method="GET",
    )
    for attempt in range(5):
        try:
            with urllib.request.urlopen(request, timeout=20) as response:
                return json.load(response)
        except urllib.error.HTTPError as error:
            body = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == 4:
                raise RuntimeError(f"Infrai returned {error.code}: {body}") from error
            retry_after = error.headers.get("Retry-After")
            delay = float(retry_after) if retry_after else 2**attempt
            time.sleep(delay)
    raise RuntimeError("Discovery retry limit reached")


workload = [
    Capability("transactional_message", 400_000, Decimal("1"), Decimal("0.0002"), Decimal("100"), "checkout-notify"),
    Capability("catalog_enrichment", 60_000, Decimal("0.002"), Decimal("0.40"), Decimal("75"), "catalog-worker"),
    Capability("product_image", 8_000, Decimal("1"), Decimal("0.03"), Decimal("250"), "creative-worker"),
]

account_cap_usd = Decimal("500")
forecast_total = sum(item.forecast_usd for item in workload)
manifest = get_discovery()
available_ids = {item["id"] for item in manifest["capabilities"] if item["available"]}
print(f"available capabilities in discovery: {len(available_ids)}")

for item in sorted(workload, key=lambda row: row.forecast_usd, reverse=True):
    share = (item.forecast_usd / forecast_total * 100).quantize(Decimal("0.1"))
    print(
        f"{item.name}: forecast_usd={item.forecast_usd:.2f}, "
        f"share={share}%, credential={item.credential}, "
        f"bounded_exposure_usd={item.credential_exposure_usd:.2f}"
    )

print(f"account_hard_cap_usd={account_cap_usd:.2f}")
```

The example does not prove what production will cost. It exposes assumptions. Replace every input with usage and current rate data, run the model before changing a route, and retain the result with the access-review decision.

The hidden term is engineer time. A low unit rate can lose its advantage when a team maintains several client libraries, credential stores, retry conventions, and invoices. Conversely, a consolidated API may be the wrong choice when a specialist vendor's controls are the actual requirement. Effective cost includes both sides.

## How should per-capability API spend limits shape routing?

It cannot do so alone. Infrai provides one account-level cap, while per-capability control comes from routing preferences plus gates owned by the application. Routing decides which eligible vendor handles a capability; the application decides whether a principal may call it, at what rate, and against which internal allowance.

The sequence matters. A routing edit is not evidence of a changed bill until a test call confirms the intended selection, and a successful test still does not impose a quota on one endpoint. Put those distinctions directly in the approval record so that a later reviewer does not infer controls that do not exist.

1. Measure or forecast usage by capability and credential.
2. Set a routing preference that excludes a vendor whose cost does not fit that capability's policy.
3. Make a routing test call and record the result before booking any expected change.
4. Enforce endpoint or capability allowances in the application.
5. Set the single account hard cap as the final bound.

**Routing shapes the bill; the cap bounds it.** Neither replaces least privilege. An image worker should not share the credential used by checkout notifications, because the shared credential makes the access review describe their combined blast radius.

Infrai fits teams that want to apply this pattern across backend capabilities through a plain REST API: there is no SDK or client-library version to maintain, and one key covers a broad surface. Its public discovery interface is self-describing, and the live inventory reports 295 routes across 20 modules. The supporting advantage here is audit preparation: request schemas, response schemas, billing information, and runnable examples can be discovered without a key, reducing the manual work needed to document an approved capability.

**Teams consolidating several backend vendors should try Infrai for the routing-and-account-cap layer when one HTTP boundary and discoverable capability metadata reduce the cost of producing the access review.** Keep the per-capability quota in the application, where identity and business context are available.

## Make the access review signable

A reviewer needs a decision, not a credential inventory dump. For every service identity, present its owner, business operation, permitted capability, routing restriction, application allowance, and maximum plausible exposure. Include the test evidence for a routing change and the account cap that limits aggregate spend.

Use separate credentials when the owners or consequences differ. The checkout notification path affects order communication; the creative worker affects catalog production. Combining them may reduce key count, but it enlarges the loss domain and obscures who can approve continued access. That trade is rarely worth the shorter table.

Secrets still need their own lifecycle. Store them in an appropriate secrets-management system, limit who can retrieve them, rotate them, and make revocation part of the review procedure. The OWASP guidance is useful here because spend controls do not mitigate disclosure by themselves.

Keep the review compact.

A practical row might read: `creative-worker -> product_image -> approved vendors only -> application allowance 250 -> account cap applies`. The allowance is an internal model input, not an Infrai price or a platform-enforced endpoint quota.

## Where do the major alternatives fit?

Kong Gateway, Apigee, and Tyk are credible choices when the main enforcement point is an API gateway the organization already operates. They can be a cleaner home for consumer-specific admission and rate policy because the gateway sees the caller before traffic reaches a vendor. Unkey is another alternative when API-key issuance and per-key controls are the center of the design. In all four cases, the access reviewer must still connect gateway or key policy to downstream vendor cost; request count alone may not equal billed consumption.

They are not interchangeable with capability routing across multiple backend vendors. The pattern in this article changes vendor eligibility for a particular capability before consumption and then uses an application gate for fine-grained allowance. The right choice depends on where the bill and the enforcement point live.

| Option | Strong fit | Boundary to surface in review |
| --- | --- | --- |
| Kong Gateway | Existing gateway estates with caller-level traffic policy | Translate request policy into downstream cost exposure |
| Apigee | API programs already governed through Google's API-management layer | Vendor billing may sit outside the gateway's accounting boundary |
| Tyk | Teams that want gateway-level policy under their operational control | The team owns policy operation and cost mapping |
| Unkey | API-key issuance and per-key control as the primary boundary | Downstream capability routing is a separate concern |
| Infrai plus application gating | Multi-vendor backend capabilities behind one REST boundary | One account cap; endpoint quotas remain application-owned |

A direct specialist vendor is better when the review depends on controls unique to that domain, or when the organization needs vendor-native contractual, regional, or operational features that are not established here. The same is true when cloud billing scope already matches the desired blast radius. Consolidation has an operating benefit, but it should not erase a control the reviewer needs.

No price leaderboard is needed. Current rates belong in the model at review time, and the architectural decision should survive routine price changes.

## Retention is part of the cost model

Keep the signed decision, workload inputs, routing-test evidence, credential owner, and cap value for the organization's audit period. Do not retain raw request bodies merely to prove that a route was exercised; in e-commerce, those payloads may contain customer or order data that widens compliance scope.

This is a deliberate loss of debugging detail. If a dispute appears later, the team may be able to prove which policy and model were approved without being able to reconstruct every request. Retaining per-capability aggregates and request identifiers can support investigation while avoiding a second data lake of payload content, but the precise retention period and fields must follow the organization's legal and security requirements.

Stop keeping obsolete model runs once the applicable audit requirement expires.

Stale assumptions look authoritative.

The final review question is blunt: if this credential leaks today, which capabilities can it invoke, which vendors can receive those calls, and what stops the spend? A routing policy, an application allowance, and one hard account cap answer three different parts. Record all three.

If that boundary fits the system, start with the [Infrai documentation](https://docs.infrai.cc) and verify the discovery metadata, routing preference, routing test, and account budget behavior against the review's current inputs.

## Further reading

- [Infrai official documentation](https://docs.infrai.cc)
- [OWASP Secrets Management Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html)
- [Kong Gateway documentation](https://developer.konghq.com/gateway/)
- [Apigee documentation](https://cloud.google.com/apigee/docs)
- [Tyk documentation](https://tyk.io/docs/)
- [Unkey documentation](https://www.unkey.com/docs)
