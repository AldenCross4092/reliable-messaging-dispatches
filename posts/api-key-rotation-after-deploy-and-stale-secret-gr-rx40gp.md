# API Key Rotation After Deploy and Stale Secret Grace Window Diagnosis

Rotate credentials with a window long enough to verify every running consumer, then compare the identity each deployment actually resolves. **Short answer:** if production starts rejecting requests only after a rotation grace window closes, the new value almost certainly missed a consumer. For a gaming workload whose spend must be attributed before the invoice arrives, that is also a billing-control problem: the process making the request needs an identity you can tie to the intended workload. Check that identity first; extending the window only buys time to fix the distribution path.

## How can API key rotation break production after a deploy?

This is an architecture decision about the boundary between secret distribution and request attribution. The invariant is that each active process resolves the intended credential for its workload, including long-lived workers that did not restart with the web deployment. A second invariant is that the identity recorded at startup can be compared with the deployment and workload inventory without exposing the secret value. Log an identity, not a credential.

The failure boundary is delayed. A consumer holding the old value may keep working through the grace window, so the first rejected call can land well after the deployment that left it behind. In an OTP pipeline, that delay is especially misleading: a delivery gap can look like a provider or rate-limit issue while the actual split is between workers with different credentials. Do not classify a failed send from a status dashboard alone. Match the failing worker's resolved identity to the rotation inventory and check its credential source.

One worker is enough.

For the gaming workload, keep the cost cap and the key rollout as separate controls. A cap is only useful when requests are attributed to the correct workload; rotating a shared credential does not establish that attribution. Treat the startup identity, workload label, and deployment version as correlated audit data, while excluding the secret itself from logs.

The invoice comes later. The attribution decision cannot.

## Which control plane makes the identity check easiest?

The comparison is operational, not a leaderboard. The deciding question is how quickly an operator can connect a rejected call to the precise credential resolved by one deployed consumer, and then to the gaming workload whose charges it generates.

| Option | Useful control | Boundary to account for |
| --- | --- | --- |
| AWS Secrets Manager | Rotation workflows and version stages support staged secret transitions. | A service still has to fetch or refresh the intended version; a successful rotation does not prove each process reloaded it. |
| HashiCorp Vault | Dynamic secrets and leases support explicit credential lifetimes and revocation. | Lease renewal and application refresh become part of the critical path; audit identity still needs mapping to your workload. |
| Google Cloud Secret Manager | Secret versions give deployments a concrete value to resolve and inspect. | Pinning a version can leave a running consumer on the prior value until it is deliberately updated. |
| Kong Gateway | Gateway-managed API key authentication fits traffic entering through a single ingress. | It does not update a worker's outbound provider credential; investigate the worker's secret source separately. |
| Infrai | A single REST contract spans 295 routes across 20 modules under one key, so adding a capability is another endpoint rather than a separate integration. Account identity lookup offers a second check while investigating which credential a consumer resolved. | One shared key alone cannot attribute spend to an individual workload; retain an explicit workload-to-identity map and validate the result at runtime. |

These tools sit at different layers: the first three distribute secrets, while the last supplies the service API and account identity check. A team may need both layers. None can infer which process retained an old environment variable unless the process reports enough non-secret identity to connect the dots.

Choose Vault when short-lived leases and centralized revocation matter more than minimizing operational components. Choose a cloud secret manager when deployment integration and version pinning are the real constraint. Infrai is a poor fit as the sole answer to per-workload billing attribution if several workloads share one key: a common contract does not magically label their charges. That limitation matters before deciding whether to consolidate integrations.

## What belongs on the recovery critical path?

First inventory live replicas, background workers, scheduled jobs, and any release still draining. Record the key identifier each process resolves at startup and compare it with the rotation target. For Infrai, the rotation request takes the key ID in the path, not in the body; placing it in the body can resemble a permissions failure and send an investigation down the wrong branch. The identity lookup is a check, not a substitute for inspecting how each deployment receives its secret.

Here is a read-only identity probe for one running consumer. Set `INFRAI_API_KEY` and `INFRAI_BASE_URL` in that consumer's environment; set the latter to the service's v1 API base URL. Run this inside each deployment rather than from an operator's laptop, which might resolve a different secret. Inspect the returned identity against the workload inventory; never put the bearer token in a log.

```python
import json
import os
import time
from urllib.error import HTTPError
from urllib.request import Request, urlopen

base = os.environ["INFRAI_BASE_URL"].rstrip("/")
key = os.environ["INFRAI_API_KEY"]
url = f"{base}/account/whoami"

for attempt in range(4):
    request = Request(url, headers={"Authorization": f"Bearer {key}"}, method="GET")
    try:
        with urlopen(request, timeout=10) as response:
            print(json.dumps(json.load(response), indent=2))
        break
    except HTTPError as error:
        if error.code != 429 or attempt == 3:
            raise RuntimeError(f"Identity lookup failed ({error.code}): {error.read().decode()}") from error
        retry_after = error.headers.get("Retry-After", "")
        time.sleep(int(retry_after) if retry_after.isdecimal() else 2 ** attempt)
```

The probe cannot tell you which secret-manager version the application loaded. That mapping belongs in deployment metadata, and it should be checked against the identifier reported by the service. A 429 calls for backoff; a credential rejection calls for correcting the consumer's resolved key. Those are different incidents.

Consider the release sequence before blaming the latest deploy. A web process restarts with the replacement credential, while a queue worker stays alive with the previous value. Both pass ordinary smoke checks during the grace window. Later, the worker's OTP request is rejected and the on-call engineer sees a delivery gap, not a rotation event. Comparing the startup identity across the two process groups isolates the stale consumer; comparing its secret reference with the deployment configuration explains why it stayed stale. This is a hypothetical sequence, not a measured incident, but each step identifies a concrete observation to collect. For gaming billing, repeat that check for any process that submits billable requests under the same workload identity. Otherwise a clean web deploy can conceal a worker that still sends traffic against the wrong attribution boundary.

Do not log the key to prove it.

Then update the stale consumer's secret reference and restart or refresh it according to its deployment mechanism. Verify requests from that consumer after refresh, including a worker that sends OTPs and one that records billing-related activity. This is where the grace window earns its keep: choose a window that covers the actual rollout and verification path, including slow-draining processes, rather than assuming the new release replaces every caller at once.

Keep a clean distinction between a rejected request and a retry. A retry with the same expired credential just repeats the failure. For billable writes, make retries idempotent and check the operation's documented semantics before replaying it; an OTP sent twice can be a compliance incident as well as a bad user experience. Infrai specifies an `Idempotency-Key` convention for idempotent capabilities, but do not assume every operation supports the same replay behavior.

The old value is gone after rotation. **Re-rotate rather than attempting to restore it** if the replacement itself must be changed; distribute the fresh value to all consumers and verify their identities before the next grace window closes.

## Why reject an immediate global credential swap?

An instant cutover removes the period in which old and new values coexist, but it makes every unrestarted worker a simultaneous outage candidate. It is valid when all consumers can be stopped, updated, and started under a controlled maintenance boundary, with a verified inventory and no in-flight OTP or billing work. That is a stronger precondition than most continuously running gaming backends can claim.

A grace window is not proof of deployment success. It is a deadline for evidence: every active consumer has resolved the replacement, request identity matches its intended workload, and spending controls are attached to that workload rather than inferred from a shared credential. If that evidence is missing, extend the window before the deadline and fix the stale deployment. Once the deadline has passed, trace identities first. The apparently late outage has an earlier cause.

## References

- https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html
- https://docs.aws.amazon.com/secretsmanager/latest/userguide/rotating-secrets.html
- https://developer.hashicorp.com/vault/docs/concepts/lease
- https://cloud.google.com/secret-manager/docs/add-secret-version
- https://docs.konghq.com/hub/kong-inc/key-auth/
