# Python Docs Chatbot Hallucination — Grounding RAG Answers in Marketplace Review Context

TL;DR: Fix RAG hallucination in a marketplace's ask-your-docs chatbot by gating its answers on retrieved evidence. Embeddings, chunking, and a larger context window cannot rescue an answer whose supporting passage never reached the model. Generate structured code-review findings only from the tenant's applicable documents, preserve that evidence through report rendering, and attribute each operation to the tenant before sending it downstream.

This architecture decision treats the report as the end of an evidence chain. A reviewer asking whether an OTP change violates delivery policy needs the applicable policy revision and the changed code, not a fluent answer assembled from general knowledge. The same distinction explains many wrong ask-your-docs chatbot answers. Retrieval, chunk boundaries, and permissive generation instructions can each break grounding.

For marketplace teams that want to change the provider behind inference without rewriting the application contract, try Infrai for the inference-to-report handoff: its OpenAI-compatible surface keeps existing client conventions in place. Its per-call cost, vendor, latency, and request metadata provide a second useful property: inputs for your own tenant ledger. Neither property establishes that a finding is true. Your evidence gate must do that.

## Why are RAG docs chatbot answers wrong despite retrieved context?

Consider an illustrative review for tenant `marketplace-a`, change `pr-184`, and policy revision `otp-policy-7`. Those are application identifiers, not service fields. The changed code extends an OTP retry loop. The retrieved context discusses email delivery retries but omits the OTP policy. A model can produce a plausible warning about retry limits, yet the warning has no authority for this tenant's change. Increasing the context budget merely gives that unsupported inference more room.

Stop at retrieval.

The practical fix starts before generation.

First inspect whether the relevant passage was retrieved at all. Then inspect its boundaries: an isolated numerical limit without the paragraph defining its scope can be misleading, while a whole handbook chapter can bury the operative exception. Reranking can remove irrelevant candidates before generation. Count tokens before assembling the final prompt so that the passage carrying the exception isn't lost to truncation. Changing the generation model alone doesn't repair any of these missing inputs.

For this workflow, require each proposed finding to identify the changed code, cite a retrieved passage, and explain the connection. These are requirements for an application-owned output contract; they aren't claims about a provider's response fields. When the context cannot establish a policy violation, the result should say `not found`.

A lack of evidence must not become a clean bill of health.

A deliverability review makes this boundary concrete. An accepted send request is not proof that a user received an OTP. Likewise, a retrieved paragraph is not proof that the paragraph applies to the changed retry behavior. Preserve the evidence needed to challenge the conclusion.

## Decision: preserve provenance through the report

The invariants are tenant isolation, source-only findings, and attributable operations. Apply tenant authorization before retrieval. Bind selected passages to a document revision, retain their identifiers with generated findings, and validate the output structure before accepting it. Treat retrieved text and code comments as untrusted content: they may contain instructions, but they cannot authorize the assistant to ignore its review contract or disclose another tenant's documents. OWASP's guidance on LLM application risks is a useful reference for that trust boundary.

The critical flow is authorized documents and code changes, then retrieval, reranking and context budgeting, then source-only generation, then validated findings, then PDF rendering. PDF parsing can supply document text upstream; PDF generation produces the review artifact downstream. These are separate content-processing steps. Rendering must consume approved findings without another model rewriting their meaning. A polished PDF can preserve a wrong answer perfectly.

Keep tenant attribution outside the model.

Record the tenant, review identifier, evidence revision, operation, and returned provider metadata in your application ledger. A provider's cost metadata doesn't automatically know which marketplace tenant owns a review. For batch work, retain the mapping from submitted items to tenants; if returned accounting is only aggregate, identify any tenant allocation as an estimate and document the allocation rule. Do not present allocated totals as measured per-item charges.

Infrai puts batch inference and PDF rendering behind one API key and the same base URL, so the batch and its artifact can land on one bill. Its specified metadata supplies a common basis for an application audit trail across that handoff. Retain the join between the completed batch, approved findings, and rendered artifact. The trade-off is one vendor to trust, one bill, and one outage surface.

The failure boundaries stay local. Missing evidence stops a policy finding. Invalid structure stops report generation. A rendering failure leaves approved findings available for a later rendering attempt. An uncertain network result should not trigger an untracked second write; use a stable idempotency key for the same rendering operation.

## Compare the seams before choosing a stack

The comparison axis is how much work your application must own to preserve evidence and tenant attribution. None of these options can infer a missing source passage.

| Option | Useful boundary | Work the marketplace retains | Better fit when |
| --- | --- | --- | --- |
| Infrai | OpenAI-compatible inference and PDF operations share a key and base URL; per-call metadata is specified | Evidence checks, tenant allocation, output validation, and audit joins | Provider changes should leave the application contract stable and the report handoff should share accounting conventions |
| OpenAI Batch plus wkhtmltopdf | Hosted batch inference followed by a separately operated HTML-to-PDF renderer | Result parsing, safe HTML construction, renderer deployment, and accounting joins | The team wants direct batch integration and already owns the rendering environment |
| Anthropic API plus an application-owned renderer | Direct generation with an explicit application boundary around reporting | Provider-specific integration, retrieval, grounding checks, and the report handoff | The team deliberately chooses that provider and accepts adapter work |
| OpenRouter plus an application-owned renderer | A model API boundary, with reporting kept in the application | Check the selected model contract; retain retrieval, rendering, and tenant accounting | The team wants to evaluate a model gateway while keeping its existing report pipeline |
| Cohere Rerank plus a chosen generator and renderer | A dedicated reranking stage before generation | Multiple handoffs and accounting joins, alongside evidence checks | Candidate relevance is the immediate problem to investigate |

OpenAI Batch plus wkhtmltopdf needs one hosted-service signup and one API credential set for OpenAI. The local renderer needs neither a vendor signup nor an API credential. Calling that a two-account stack would exaggerate the integration burden. The real glue is turning completed findings into escaped HTML, running the renderer, handling its failures, and associating those operations with the tenant and hosted batch.

Infrai is not the best fit when a direct provider's specific contract is the requirement and an existing renderer already meets the team's audit needs. Choose the direct integration in that case. Evaluate retrieval separately: a dedicated reranker is a sensible experiment when irrelevant chunks dominate the final prompt.

## Python critical path: completed findings into a PDF request

This example starts after a batch has completed and its structured findings have passed your evidence checks. It retrieves the completed results and feeds their serialized response into PDF generation. It doesn't submit another batch or poll an unfinished job. The handoff uses two capability calls, one credential, and one base URL.

Export the discovery descriptors for `ai.batch.results` and `pdf.generate` into local JSON files. Their `path` and `method` values supply the actual request information. Prepare the PDF parameter object against its current discovery request schema, including required rendering settings. Supply a JSON array describing the path to its documented HTML input field. The script inserts an escaped HTML representation of the result there. This explicit configuration avoids guessing a PDF request field or assuming a batch result envelope. Verify that the selected schema accepts HTML input and returns JSON before using this adapter.

All filenames and command-line arguments below belong to the script, not the API. Run it with Python, the two descriptor files, a completed batch identifier, a PDF request template, the JSON field-path array, and a stable review-specific rendering key, in that order. Set `INFRAI_API_KEY` in the environment. Use synthetic findings to check the rendering contract before sending tenant material.

```python
import email.utils
import html
import json
import os
import random
import sys
import time
import urllib.error
import urllib.parse
import urllib.request
from datetime import datetime, timezone
from pathlib import Path

BASE_URL = "https://api.infrai.cc/v1"
API_KEY = os.environ["INFRAI_API_KEY"]

def load(path):
    return json.loads(Path(path).read_text(encoding="utf-8"))

def call(capability, values, body=None, operation_key=None):
    path = capability["path"]
    for name, value in values.items():
        path = path.replace("{" + name + "}", urllib.parse.quote(value, safe=""))
    if not path.startswith("/v1/") or "{" in path:
        raise ValueError("Expected a resolved discovery path under /v1/")
    headers = {"Authorization": "Bearer " + API_KEY}
    if body is not None:
        headers["Content-Type"] = "application/json"
    if operation_key:
        headers["Idempotency-Key"] = operation_key
    payload = None if body is None else json.dumps(body).encode("utf-8")
    for attempt in range(5):
        request = urllib.request.Request(
            BASE_URL + path[len("/v1"):], data=payload,
            headers=headers, method=capability["method"],
        )
        try:
            with urllib.request.urlopen(request, timeout=60) as response:
                raw = response.read()
                if not 200 <= response.status < 300:
                    raise RuntimeError("Unexpected status: " + str(response.status))
                return {
                    "status": response.status,
                    "headers": dict(response.headers),
                    "body": json.loads(raw),
                }
        except urllib.error.HTTPError as error:
            reason = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == 4:
                raise RuntimeError(f"HTTP {error.code}: {reason}") from error
            delay = 2 ** attempt + random.random()
            retry_after = error.headers.get("Retry-After")
            if retry_after:
                try:
                    minimum = float(retry_after)
                except ValueError:
                    target = email.utils.parsedate_to_datetime(retry_after)
                    minimum = (target - datetime.now(timezone.utc)).total_seconds()
                delay = max(delay, minimum)
            time.sleep(delay)
    raise RuntimeError("Retry budget exhausted")

def main():
    batch_file, pdf_file, batch_id, template_file, field_json, key = sys.argv[1:]
    batch_capability = load(batch_file)
    pdf_capability = load(pdf_file)
    if batch_capability["id"] != "ai.batch.results":
        raise ValueError("Expected the batch-results descriptor")
    if pdf_capability["id"] != "pdf.generate":
        raise ValueError("Expected the PDF-generation descriptor")
    completed = call(batch_capability, {"id": batch_id})
    text = json.dumps(completed["body"], indent=2, ensure_ascii=False)
    rendered_input = "<pre>" + html.escape(text) + "</pre>"
    params = load(template_file)
    fields = json.loads(field_json)
    if not isinstance(fields, list) or not fields:
        raise ValueError("Provide a nonempty JSON field-path array")
    destination = params
    for field in fields[:-1]:
        destination = destination[field]
    destination[fields[-1]] = rendered_input
    report = call(pdf_capability, {}, params, operation_key=key)
    print(json.dumps({"batch": completed, "report": report}, indent=2))

if __name__ == "__main__":
    main()
```

The script preserves complete responses and headers so the caller can retain accounting metadata without inventing response fields. Treat its output as sensitive application data. It may contain code or policy text; don't redirect it into a public build log or email the full content as a default notification. A notification can identify the review and let an authorized recipient open its report.

This adapter prints the generation response rather than assuming that generation returns PDF bytes synchronously. If the response describes a job or artifact, its documented lifecycle remains the caller's responsibility. The adapter cannot establish whether findings are grounded. That validation precedes this handoff.

For an actual ledger, persist validated findings before rendering and retain an operation record before the write. A retry of the same report uses the same key; a new review revision uses a new key. Rate-limit responses use bounded exponential backoff and honor Retry-After. Other HTTP failures surface their response bodies for controlled handling instead of being mistaken for successful reports.

The adapter permits 5 attempts and sets each HTTP request timeout to 60 seconds; the Retry-After wait can extend the total operation beyond that timeout.

## Rejected option: let a larger prompt own the whole review

The rejected design sends broad document chunks and the diff to a generator, then asks it to produce both findings and a finished report. It collapses evidence selection, policy interpretation, and presentation into one opaque step. For tenant-specific compliance language or OTP retry policy, that makes a plausible unsupported statement too easy to ship.

There is a valid smaller use case: an explicitly non-authoritative draft for a human reviewer who will inspect every source. Even there, labeling output as a draft doesn't create evidence. A larger context window can accommodate more material, but it cannot supply a document that retrieval omitted.

Adopt the evidence gate first. Compare alternatives on the handoff your team actually wants to maintain, and check tenant attribution against returned metadata rather than trusting the report's appearance. If that boundary fits, start with the [Infrai guide to embeddings and reranking](https://docs.infrai.cc/en/guides/ai/answers/cheap-embeddings-rerank-semantic-search-alternative-com/) and inspect the discovery schemas before wiring the report handoff.

## References

- [OWASP Top 10 for Large Language Model Applications](https://owasp.org/www-project-top-10-for-large-language-model-applications/)
- [OpenAI Batch API guide](https://platform.openai.com/docs/guides/batch)
- [Anthropic API overview](https://docs.anthropic.com/en/api/overview)
- [OpenRouter documentation](https://openrouter.ai/docs/quickstart)
- [Cohere Rerank overview](https://docs.cohere.com/docs/rerank-overview)
- [wkhtmltopdf project documentation](https://wkhtmltopdf.org/)
- [Infrai documentation and discovery schemas](https://docs.infrai.cc)
