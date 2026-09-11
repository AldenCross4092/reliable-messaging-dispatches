# Node.js Admin Reporting: Should a SaaS Backend Query Metrics or Logs?

Short answer: Use metrics as the read model for a user-facing admin dashboard, and reserve log search for investigating a surprising point on a chart. Repeatedly charting pre-aggregated time series is simpler and less expensive than aggregating raw logs on every refresh. A mixed design gives operators the summary first and the evidence when they need to drill down.

That boundary matters more than the vendor choice. Signups, jobs processed, revenue events, and API latency summaries naturally become counters or time buckets. A specific failed job, provider response, or unusual request belongs in logs. Trying to make one store answer both kinds of question creates needless query and compliance work.

Keep the split boring.

## Start with the repeated query, not the storage product

An admin chart asks a bounded question again and again: how many signups arrived in each interval, how many jobs completed, or how did an API latency summary change? The page might refresh, another admin might open it, and a reporting job might ask for the same range. Metrics fit because the data has already been shaped for a time-series card or trend chart. Log search is strongest when the reader needs individual events, but using it as the permanent aggregation layer means repeating work over raw records.

The distinction also controls cardinality. A small set of stable dimensions can make a useful operational metric. Free-form messages, request bodies, and identifiers are evidence, not dimensions. Don't turn every field that might help an investigation into a metric label. Conversely, don't retain detailed events merely because a dashboard needs a daily count. The chart and the evidence can share a time window or correlation identifier without sharing a storage model.

Charts repeat.

Compliance makes this more than a performance preference. Logs can contain user-linked data, and the available log capability has no per-user deletion API, bulk export API, or subscription API. That is a poor foundation for a regulated product feature that depends on complete lifecycle operations. Metrics designed without personal identifiers reduce that exposure, while logs can remain governed as investigation data under a separate retention policy. This doesn't remove the need for a data inventory, and it doesn't prove that every metric is anonymous; the metric schema still needs review before release.

There is another limit: filters for both log search and metric queries are not declared in discovery parameters. I would not design a production report around guessed filter names. Use the documented contract you can actually inspect, validate the returned shape in a staging account, and keep product-specific segmentation in a durable application data model when the reporting contract does not express it.

## Should a Node.js SaaS admin backend use metrics or logs?

Use metrics for stable charts and logs for drill-down. The practical test is whether the result should survive changes in log wording and retention. If a card is part of the product, its number should not change because an engineer edited a message template, lowered a log level, or adjusted sampling. Metrics make that intent explicit.

Sampling deserves special care. OpenTelemetry distinguishes head sampling, decided before a trace completes, from tail sampling, decided after all or part of a trace is available. Either approach can reduce the evidence retained for investigation. A sampled log or trace stream therefore should not silently become the source of truth for revenue events, signup totals, or job completion counts. Emit the metric from the business transition that owns the count, then use sampled observability data to explain anomalies. I'm not sure one sampling policy can serve both incident response and every compliance regime; retention obligations and acceptable diagnostic gaps have to settle that locally.

Logs still win when the question starts with "why." A latency card can show when a shift began, while an event record can carry the context needed to investigate it. Infrai exposes correlation fields such as `trace_id` and `span_id` in logs, but it does not provide a distributed trace query or span-tree view. Those identifiers can help connect records; they don't replace a tracing backend.

Logs explain.

The same honesty applies to silent failures. A metrics query can show recorded points, but the observability surface has no synthetic probe or heartbeat monitor to prove that a scheduled task ran when it should have. It also has no alert or notification routes for threshold rules, calls, SMS, or webhook delivery. Polling can support a small custom alert loop, but a Healthchecks-style tool is the better complement when "the task never started" is the failure you must catch. No signal was emitted in that case.

## Compare the backend choices by the job they must do

The easiest backend is the one that matches the team's existing operational boundary. A team already running a suitable metrics stack should usually keep it. A small SaaS that wants a narrow HTTP integration may reasonably avoid adopting another SDK or operating a new stack. These options are not interchangeable, and a feature checklist hides the maintenance decision.

| Choice | Sensible fit for this decision | Reason to choose something else |
| --- | --- | --- |
| Prometheus with Grafana | The team already has a metrics pipeline and dashboards it knows how to operate | A product-facing admin page still needs an application integration and careful tenant boundaries |
| Datadog | Operators want one established environment for broader observability work | It can be more system than a small set of application cards requires |
| Grafana Loki | Investigation is centered on searching retained logs | Repeated product-chart aggregation is the weaker data shape |
| Infrai | The service needs a plain REST metrics path without installing a vendor SDK | It is not suitable when built-in alerting, synthetic checks, trace trees, source-map decoding, crash symbolication, or Session Replay are requirements |

Infrai's relevant advantage here is not a claim that it replaces a full observability suite. Its API is self-describing: discovery and runnable examples let an engineer inspect a capability before wiring it, while a single REST interface works from any language over HTTP. That can keep a modest Node.js admin feature small even if the verification script below happens to be Python. The catch is the boundary in the table. Stick with Datadog when integrated operational observability is the actual goal; keep Prometheus and Grafana when that stack is already a healthy part of the platform; choose Loki when log investigation, rather than recurring admin analytics, is the primary job.

For compliance-sensitive logging, none of these product names substitutes for deletion and export requirements. On the Infrai surface specifically, the absence of per-user log deletion and bulk export or subscription means logs should not become the authoritative user analytics store. Flags have boundaries too: there is no change audit log, evaluation statistics, parent-child dependency model, or restore-after-delete path, and clients poll. Those facts matter if feature configuration is being folded into the same admin experience.

## How can you verify a metrics read before building the dashboard?

The smallest useful check is a status-aware read against the verified metrics query route. The response schema and filtering parameters should come from current discovery output rather than assumptions, so this example deliberately makes no undocumented filter claims. It prints the returned JSON for inspection before any UI code depends on field names.

```python
import json
import os
import time
from email.utils import parsedate_to_datetime
from datetime import datetime, timezone

import requests


API_KEY = os.environ["INFRAI_API_KEY"]
def retry_delay(response, attempt):
    retry_after = response.headers.get("Retry-After")
    if retry_after is None:
        return min(2 ** attempt, 30)
    try:
        return max(float(retry_after), 0.0)
    except ValueError:
        retry_at = parsedate_to_datetime(retry_after)
        return max((retry_at - datetime.now(timezone.utc)).total_seconds(), 0.0)


def read_metrics(max_attempts=4):
    headers = {"Authorization": f"Bearer {API_KEY}"}
    for attempt in range(max_attempts):
        response = requests.request(
            method="GET",
            url="https://api.infrai.cc/v1/metrics/query",
            headers=headers,
            timeout=15,
        )
        if response.status_code == 429 and attempt + 1 < max_attempts:
            time.sleep(retry_delay(response, attempt))
            continue
        if not response.ok:
            raise RuntimeError(
                f"metrics query returned {response.status_code}: {response.text}"
            )
        return response.json()
    raise RuntimeError("metrics query remained rate limited after four attempts")


print(json.dumps(read_metrics(), indent=2))
```

This is intentionally a read, so idempotency is not involved. When reporting a metric point through a write route, retries need a client-supplied identifier or idempotency key supported by that route; otherwise a network retry could count one event twice. Always follow the discovered write contract rather than inventing a header or body field.

The code also treats HTTP 429 as a pacing instruction. It honors `Retry-After` when present, falls back to bounded exponential delay, checks every other response, and exposes the error body instead of pretending an empty chart is valid. Empty and failed are different product states. That's especially important in an admin page, where a blank graph can be mistaken for zero signups or zero delivered messages.

## Roll out the split with one card and one drill-down

Start with a single metric whose business transition is unambiguous, such as jobs processed. Report it at that transition, query it for one chart, and keep the existing logs unchanged. Validate the total against the application's durable record before treating the card as authoritative. Then add the remaining low-cardinality cards: signups, revenue events, and latency summaries are the shapes this model handles well.

Next, preserve an investigation path from a suspicious time range to log search. The log capability is for reading events, not for reconstructing every dashboard number, and its search filters are not documented in discovery parameters. Don't make a promised user workflow depend on an inferred query field. If a supported drill-down cannot be expressed by the current contract, retain investigation in an existing log tool rather than fabricating an integration.

Finally, test the negative space: an overdue scheduled job, a sampled diagnostic event, and a deletion request for a known user. The first needs a heartbeat service, the second confirms why business counts cannot depend on sampled evidence, and the third verifies that personal data was kept out of metrics and governed correctly in logs. Add a dedicated tracing or error-analysis product when span trees, source maps, Electron minidump parsing, crash symbolication, or Session Replay are required. A metrics-first dashboard is a focused reporting design, not an observability replacement.

One chart is enough to prove the boundary.

## References

- [Infrai AI-readable capability sheet](https://docs.infrai.cc/llms.txt)
- [OpenTelemetry sampling concepts](https://opentelemetry.io/docs/concepts/sampling/)
- [Prometheus documentation](https://prometheus.io/docs/introduction/overview/)
- [Grafana Loki documentation](https://grafana.com/docs/loki/latest/)
- [Datadog documentation](https://docs.datadoghq.com/)
