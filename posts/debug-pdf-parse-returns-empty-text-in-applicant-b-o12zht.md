# Debug PDF Parse Returns Empty Text in Applicant Bundles (Under Batch Pressure)

TL;DR: When a PDF parse returns empty text, first decide whether the suspect document page contains text-showing operators or only scanned pixels. If it has text operators, debug decoding, clipping, page geometry, and parser behavior; if it has only an image, route that page to OCR. In a high-volume resume-bundle pipeline, make that decision per page, preserve the original bytes until verification completes, and avoid sending every page through the most expensive path.

The bill for a merge-and-split service is mostly work multiplied by pages: bytes retained, pages decoded, pages rasterized, and retries performed. A 40-page applicant bundle should not become 40 OCR jobs because page 17 is a scan. The change that moves the dominant compute term is page-level classification before OCR, while the change that controls storage is a retention clock tied to successful output validation.

This matters in game hiring, where a batch may combine resumes, portfolio cover sheets, and consent pages, then split them for downstream review. Throughput is the primary constraint, but a fast empty string is still a failed result. Treat `text == ""` as an observation that opens a decision tree, never as permission to discard the input.

## How should I debug a PDF parse that returns empty text?

A PDF page is a set of drawing instructions, not a promise of extractable reading order. ISO 32000-2 defines the document format; extraction is an interpretation performed by software. A page can therefore look perfectly readable in a viewer while its visible letters are pixels inside an image. There are no characters for a text extractor to return.

The opposite case is harder: the content stream does show text, but the parser still returns nothing or unusable output. Fonts may map character codes through encoding information that the extractor cannot interpret as expected. Text can also be present outside the visible crop, transformed to an unexpected location, clipped, or painted in a way that defeats a particular parser. A malformed page tree or damaged object reference can narrow the failure to a subset of pages. Encryption and access controls belong near the front of the check as well; do not mistake denied processing for a scan.

So I use three buckets: no text objects, text objects with failed interpretation, and document-structure failure. That root-cause classification is more useful than setting scanned versus digital as one file-level flag. Mixed bundles are common enough that one label for the whole document hides the page that matters.

The distinction controls the queue.

## Measure the page before choosing the recovery path

The first probe should collect evidence without mutating the source. Record the input digest, byte length, page count, per-page extracted character count, image count, and parser outcome. Do not log resume content. Names, phone numbers, email addresses, and portfolio links are exactly the fields that should stay out of operational logs.

Here is the shape of the classifier I put around a parser. The adapter methods are intentionally generic; bind them to a parser you have tested against your corpus.

```python
from dataclasses import dataclass
from enum import Enum


class PageRoute(str, Enum):
    TEXT = "text"
    OCR = "ocr"
    REVIEW = "review"


@dataclass(frozen=True)
class PageEvidence:
    extracted_chars: int
    image_count: int
    has_text_operators: bool
    parse_error: str | None


def choose_route(evidence: PageEvidence) -> PageRoute:
    if evidence.parse_error is not None:
        return PageRoute.REVIEW
    if evidence.extracted_chars > 0:
        return PageRoute.TEXT
    if evidence.has_text_operators:
        return PageRoute.REVIEW
    if evidence.image_count > 0:
        return PageRoute.OCR
    return PageRoute.REVIEW
```

The conservative branch is deliberate. Zero characters plus text operators is not evidence that OCR is correct; it is evidence that decoding or layout interpretation needs attention. Rasterizing immediately may create plausible text while concealing the original cause. For an applicant record, plausible is dangerous. An incorrect digit in a phone number can pass a superficial non-empty check and still break contact delivery.

Run the probe page by page before merging, and retain source-to-output lineage through the split. A useful record needs only opaque identifiers: bundle ID, source digest, source page index, output document ID, output page index, route, parser version, and validation state. That lets an operator reconstruct where a page went without exposing its contents.

Stop there.

## Make throughput a routing problem

A single queue for parsing, rasterization, OCR, merging, and splitting couples cheap work to expensive work. Separate the stages and apply bounded concurrency at each boundary. Parsing can usually proceed independently by page; merge order cannot. OCR should consume only pages classified for it, while review receives ambiguous pages and hard failures.

For each batch, measure page throughput and latency distributions by route, not just one end-to-end average. Also count empty pages after extraction, OCR-routed pages, retry attempts, and quarantine outcomes. A sudden rise in the OCR share may indicate a new scanner source. A rise in the review share after a parser release points somewhere else. These are diagnostic signals, not content, so they fit a privacy-conscious telemetry design.

Backpressure matters. If OCR capacity is saturated, keep text-bearing pages moving and hold image-only pages in a bounded queue rather than retrying the whole bundle. Make work idempotent with the source digest, page index, operation, and processing-policy version. The output assembler can wait for all required page results and then preserve source order.

Use two explicit validation gates. Before processing, confirm that the container can be opened and that the declared page set is reachable. After assembly, confirm the expected page count, page order, output readability, and a disposition for every page: extracted, OCR-processed, intentionally blank, or quarantined. Blank pages need an explicit state because a game portfolio packet may contain divider sheets; silently treating every blank as corruption creates needless retries.

That is the operational checklist: identify the root class, isolate the page, choose one route, validate the assembled document, and only then advance retention state.

## What should the system retain, and for how long?

Retention is both a cost control and a compliance boundary. Keep the immutable source while the job is in flight and until the assembled outputs pass validation. Keep compact provenance and operational metrics according to the system's approved retention policy. Delete intermediate raster images as soon as the OCR result and final bundle are verified, unless an authorized review case requires them.

The useful cost model is simple: source bytes multiplied by source retention time, intermediate raster bytes multiplied by their much shorter lifetime, output bytes multiplied by the downstream retention period, plus page-work for parsing and selective OCR. Put measured values from your own workload into that model. Published per-operation prices are a distraction here because page mix, resolution, retries, and retention dominate differently in every pipeline.

I would deliberately stop keeping successful-job rasters and duplicate split fragments. That lowers the largest avoidable storage term and reduces the copies of applicant data under management. The price is narrower forensic visibility: after deletion, an operator can replay from the retained source and policy version, but cannot inspect the exact transient bitmap. Once the source itself reaches its approved deletion deadline, replay is impossible. That is a real trade-off, so deletion must follow verified output and an auditable policy, not a best-effort cleanup task.

Access should follow job roles, and deletion should cover backups and derived artifacts under the same policy. Avoid embedding extracted text in queue messages; pass opaque object references with short-lived authorization instead. This resembles the discipline needed in OTP delivery systems: payloads are sensitive, retries must be bounded, and logs should explain state without becoming a second database of secrets.

## Prove the fix with a hostile corpus

A happy-path resume is weak evidence. Build a versioned corpus containing an image-only page, ordinary extractable text, a mixed document, an intentionally blank divider, rotated content, unusual page boxes, clipped text, protected input, a damaged object reference, and a page whose fonts stress character mapping. Expected outcomes should state the route and validation result, not merely "non-empty."

Then test invariants around merge and split: output page count equals the selected source-page count; ordering matches the manifest; every output page maps to one source digest and page index; no page disappears when another page enters review; a retry creates the same logical result; and rejected input never produces a partial bundle marked complete. Use synthetic identities in fixtures. Real resumes do not belong in a test repository.

Compare parser releases against that corpus before deployment. Roll out with a small batch slice, watch route proportions and quarantine counts, and retain the ability to process new jobs with the previous validated version. Do not declare recovery merely because character count increased. Sample structure-aware assertions such as expected field labels, while keeping human review for ambiguity rather than pretending an automated score proves semantic correctness.

The final operating rule is compact: inspect each page, use direct extraction where character evidence is sound, OCR only image-only pages, quarantine ambiguous interpretation failures, and assemble only after every page has a recorded disposition. That keeps batch throughput high without converting silent loss into fast silent loss.

No shortcuts.

## Further reading

- ISO 32000-2, Portable Document Format: https://www.iso.org/standard/75839.html
