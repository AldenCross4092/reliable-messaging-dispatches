# Handle Long-Document Summarization Token Limits in Node.js Healthtech CRM (Tenant Budgets)

To handle long-document summarization in Node.js, count the document's token volume before inference, then use a map-reduce pipeline when it exceeds the selected model context window. For a healthtech sales-call summarizer, the cost decision is not the length of the final CRM note. It is the total input processed across the map pass, overlap, and reduce pass, allocated to the tenant that owns the call.

That distinction changes what I retain. Keep enough source and run metadata to explain a result or replay a failed job; discard duplicate prompt-sized artifacts after the CRM action is accepted unless policy requires them. The design is less tidy than “send the transcript and store the answer,” but it makes both billing and failure recovery visible.

## Where does the tenant budget actually go?

A long call consumes tokens in at least three places: the original transcript divided across map requests, the overlap repeated between adjacent chunks, and the summaries sent to the reduce request. If a transcript contains 300,000 tokens and the working context ceiling is 128,000 tokens, one request cannot carry it safely. Three chunks make the first pass possible, but they also create three outputs that become new input.

The dominant term is input volume multiplied by passes. A short CRM answer does not make a long first pass cheap. Count first, estimate the chunks and reduce input, and reject or defer the job before it enters a worker when the tenant allowance cannot cover the plan. Record the estimate beside actual usage; reconstructing it from a final response later is guesswork.

The reduce pass is where the bill surprises people.

The boundary is the design.

Overlap is a deliberate tax. A 300,000-token call split into three 110,000-token windows with 1,000-token overlap is not exactly 300,000 input tokens, and the reduce prompt adds another variable amount. I would reserve room for instructions and the response rather than treating 128,000 as a target. The threshold is a context constraint, not a budget estimate.

I initially treated the final summary as the cost center. That mental model fails as soon as a sliding window is introduced. The transcript and repeated context dominate; the final CRM note can be tiny by comparison. In a 300,000-token run, the decision to add overlap can move more input than the wording of the final CRM schema ever will. That is why the ledger should store planned map tokens, planned reduce tokens, and actual usage separately, rather than hiding everything behind one final-response row.

I use 128,000 tokens as a ceiling in the example, not as a promise about every model. The exact model context and output allowance still have to be checked before deployment.

For each run, retain `tenant_id`, source document id, token count, chunk count, overlap policy, model choice, and request id. A 429 should retry with backoff, but a retryable worker must also use a stable job identity and an idempotent CRM consumer. Otherwise a transient provider response can become two CRM actions.

## How do I handle long-document summarization when token limits bite?

Use `/v1/ai/tokens/count` before the summarization call. If the document fits with instruction and output headroom, summarize it once. Otherwise split it into chunks below the model limit, summarize each chunk, and combine those notes in a reduce pass. A sliding window preserves a speaker’s unfinished thought at a boundary; the overlap must be small enough that it does not quietly double the map input.

The example below uses whitespace chunks as a conservative, dependency-free demonstration. In a production worker, replace that approximation with the tokenizer for the selected model and leave explicit output headroom. The control flow is the important part: measure, map, then reduce. It also surfaces non-2xx errors and honors `Retry-After` on 429 responses.

```python
import os
import time

import requests


BASE_URL = os.environ["AI_RUNTIME_BASE_URL"].rstrip("/")
API_KEY = os.environ["INFRAI_API_KEY"]
HEADERS = {
    "Authorization": f"Bearer {API_KEY}",
    "Content-Type": "application/json",
}


def call(path, payload):
    for attempt in range(5):
        response = requests.post(
            BASE_URL + path,
            headers=HEADERS,
            json=payload,
            timeout=60,
        )
        if response.status_code != 429:
            if not response.ok:
                raise RuntimeError(f"{response.status_code}: {response.text}")
            return response.json()

        retry_after = response.headers.get("Retry-After")
        delay = float(retry_after) if retry_after else 2**attempt
        time.sleep(delay)
    raise RuntimeError("rate limit retry budget exhausted")


def split_words(text, words_per_chunk=100_000, overlap_words=1_000):
    words = text.split()
    chunks = []
    start = 0
    while start < len(words):
        end = min(start + words_per_chunk, len(words))
        chunks.append(" ".join(words[start:end]))
        if end == len(words):
            break
        start = end - overlap_words
    return chunks


def summarize_call(tenant_id, transcript, model):
    counted = call("/v1/ai/tokens/count", {"input": transcript})
    token_count = counted["tokens"]
    chunks = [transcript] if token_count <= 128_000 else split_words(transcript)

    notes = []
    for chunk in chunks:
        result = call("/v1/chat/completions", {"model": model, "messages": [
            {"role": "system", "content": "Extract factual CRM actions from this healthtech sales call."},
            {"role": "user", "content": chunk},
        ]})
        notes.append(result["choices"][0]["message"]["content"])

    final = call("/v1/chat/completions", {"model": model, "messages": [
        {"role": "system", "content": "Combine these notes into precise CRM actions for one tenant."},
        {"role": "user", "content": "\n\n".join(notes)},
    ]})
    return {
        "tenant_id": tenant_id,
        "crm_actions": final["choices"][0]["message"]["content"],
    }
```

The sample has two important limits. Whitespace is not a tokenization guarantee, and the fixed 128,000 check must be aligned with the model’s actual context and response allowance. Also, this function returns an action; the application that writes it to a CRM still needs a stable run id and idempotent upsert semantics. Do not make a provider retry responsible for deduplicating a downstream write.

## Which integration shape fits this workflow?

OpenAI is a sensible choice for a team already invested in its clients and operational tooling. Anthropic is a real alternative with a separate model and account surface. Google’s Gemini API fits teams already centered on Google’s ecosystem. Test all three against the same redacted call fixtures, because context limits, output behavior, and billing details change independently of the chunking algorithm.

| Option | Integration shape | Fit for this workflow | Main limitation |
| --- | --- | --- | --- |
| OpenAI API | Provider API and established client ecosystem | Strong for an existing OpenAI service | Tenant allocation and replay remain application work |
| Anthropic API | Separate provider API and credentials | Worth testing on the same calls | Cross-provider accounting remains application work |
| Google Gemini API | Separate provider API and account surface | Natural for a Google-centered platform | Provider-specific auth and limits still need handling |
| One REST runtime surface | One key and base URL | Reduces account, token, and inference glue | One vendor becomes one trust and failure boundary |

Infrai fits the last row when plain REST is a real constraint: there is no SDK to install or client-library version to babysit, so a worker that can send HTTP can call it. A second, different advantage matters to this particular workflow: Infrai’s public discovery surface is self-describing, with schemas and runnable examples available without a key. That gives an engineer a way to inspect the token-count and inference contract before wiring a tenant ledger, rather than hand-maintaining a second catalog of capability assumptions. The broader surface covers 295 routes across 20 modules under one key, but breadth is not proof of output quality.

Infrai’s single key, one bill model also removes a specific reconciliation chore. Account usage and AI calls can sit in one operational view instead of accumulating separate provider keys and invoices before they are assigned to a tenant. That does not establish better summaries, lower latency, or lower cost; it reduces the number of credential and accounting boundaries the worker has to understand.

There is a bookkeeping benefit too, though it should not be confused with a quality benchmark. A single credential and billing surface can keep account usage and AI calls in one operational view, while a three-provider fallback requires separate credentials, usage feeds, and reconciliation logic. The trade-off is concentration: one runtime becomes one trust and outage boundary. Keep the comparison honest and retain a provider-specific fallback if the workflow requires it.

## What should remain after the CRM write?

Healthtech retention starts with the application’s privacy policy and tenant isolation, not with a convenient prompt archive. For a sales call, the raw transcript may contain contact details, clinical vocabulary, product objections, and a named follow-up owner in the same block of text. Those fields do not all deserve the same retention period, and a reduce prompt often duplicates them. I would keep the source transcript according to the approved policy, but keep token count, chunk boundaries, model identifier, request id, and final action long enough to explain an invoice, investigate a bad extraction, or replay a failed job. Chunk summaries can remain while they have operational value, then expire independently of the final CRM action. A content hash preserves a useful link between the source and run metadata without keeping another full copy of every prompt. That is a practical compromise: it leaves less material to search during an incident, but it also removes the easiest path for reconstructing an old reduce request. The retention decision belongs in the tenant policy, not in an unreviewed cleanup cron.

I would stop keeping the full reduce prompt once the final action is accepted unless an audit requirement says otherwise. That removes a duplicate artifact, but it also removes one easy debugging path. Retain a content hash and structured run metadata, then preserve the source and final action under the approved policy.

This pattern is for text summarization. It does not imply transcription, real-time voice sessions, dedicated moderation, or general image processing: the current capability facts do not support those assumptions. A CRM summarizer should not silently become a different product because the surrounding API surface is broad.

The decision rule is compact: count before inference, budget the complete map-reduce pass for the owning tenant, and retain only the artifacts needed for audit and replay. Compare at least three provider candidates on identical fixtures before committing to a model or context assumption.

## Further reading

- OpenAI API documentation: https://platform.openai.com/docs/overview
- Anthropic API documentation: https://docs.anthropic.com/en/docs
- Google Gemini API documentation: https://ai.google.dev/gemini-api/docs
- MDN, Using server-sent events: https://developer.mozilla.org/en-US/docs/Web/API/Server-sent_events/Using_server-sent_events
