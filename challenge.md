# Detection Improvement Log

## Memory Optimization for Cloud Deployment

### Previous implementation

PDF processing extracted the text for every page into `page_texts`, then used
`detect_many()` to create a second document-wide collection of entity lists
before any page redactions were applied.

### Identified memory bottlenecks

For a large PDF, all extracted page strings and all detected entities remained
live at once, in addition to PyMuPDF's open document. The production model
loader also attempted progressively larger spaCy model names before selecting a
model.

### Modifications made

The PDF writer now loads, extracts, detects, redacts, and releases one page at
a time. It calls `detector.detect()` per page, deletes that page's temporary
text, entity, replacement, and page objects, and runs garbage collection once
after the page completes. On Render, the documented `RENDER=true` environment
variable selects only `en_core_web_sm`; local development retains the existing
transformer/large/small fallback order. Progress logs record page numbers and
save completion without recording document contents.

### Estimated memory reduction

The page-text and entity working set changes from proportional to the entire
document to proportional to one page. For similarly sized pages, a 127-page
PDF no longer retains up to 127 pages of these temporary objects at once.
PyMuPDF's document and spaCy's single loaded model still contribute fixed
resident memory, so the exact reduction depends on the source PDF.

### Render deployment stability

Streaming prevents accumulated extracted text and NER entities from competing
with the PDF renderer and spaCy model for Render's limited memory. This reduces
the chance of a worker being killed while preserving page order, detection,
replacement mapping, coordinates, formatting, filenames, and API responses.

## Challenge

Contextual PII recall collapsed to zero after selecting `en_core_web_trf`.

## Root Cause

The transformer model was not installed. The previous `OSError` fallback silently created a blank spaCy pipeline, so no `PERSON`, `COMPANY`, or `ADDRESS` entities could be emitted.

## Solution

The detector now tries the transformer, large, and small English spaCy models in that order. The installed `en_core_web_sm` model is used when larger models are unavailable. A warning is emitted only when no NER model is available.

## Result

Contextual detection is restored without requiring a large download. This removes the blank-pipeline regression and keeps startup resilient.

## Challenge

Every spaCy `ORG`, `GPE`, `LOC`, and `FAC` span was previously accepted as sensitive data, producing false positives in skills, technologies, and generic location-like words.

## Root Cause

Raw NER labels are candidates, not business or address validation.

## Solution

Company candidates now require a recognised business suffix or an explicit company/organisation context. Address candidates require structural address regex matches or nearby address cues. Person candidates require a credible name shape, with support for common suffixes such as `MD` and `PhD`.

## Result

The final benchmark achieves company precision of `0.8193`, person precision of `0.7548`, and address precision of `0.8886` while restoring contextual recall.

## Challenge

spaCy sometimes returns only part of an explicit field value, such as a partial company name after `Company:`.

## Root Cause

NER span boundaries do not necessarily align with a document's key-value field boundaries.

## Solution

For explicit common fields (`Name`, `Customer`, `Employee`, `Company`, `Organisation`, and similar), the detector expands a valid NER candidate to the full line value and validates the result before accepting it.

## Result

Partial detections are reduced without relying on dataset-specific names or templates.

## Challenge

Equivalent credit-card values were counted as both false positive and false negative when a detector span contained trailing whitespace.

## Root Cause

The evaluator matched only case-folded raw text and treated whitespace differences as distinct values.

## Solution

Evaluation now case-folds, trims, and collapses whitespace before Counter-based matching. Labels, entity types, and repeated occurrence counts remain required.

## Result

Whitespace-only mismatches no longer distort the score; unsupported 12- and 19-digit card formats remain honest false negatives.

## Challenge

Original PDF output reflowed all text and destroyed résumé layout.

## Root Cause

The old writer extracted text and regenerated a plain-text PDF.

## Solution

PDFs are redacted at original coordinates with PyMuPDF, and contextual PDF candidates are filtered conservatively.

## Result

Surrounding PDF layout is preserved. Replacement text can still use a fallback font or a smaller size when necessary.

## Challenge

Names, companies, and contact details in resume-style documents were under-detected when they were not expressed as conventional sentences.

## Root Cause

NER is trained primarily on prose. It can miss names in a document header and values in compact `Label: value` contact blocks; it may also return only part of a company value.

## Solution

The detector now recognises common person and company fields directly, accepts credible one-to-four-token names with titles, initials, hyphens, and suffixes, and treats a safe first-line header as a name. Company candidates are accepted from explicit business context, suffixes, or credible capitalised spans, while generic headings such as `Phone` are rejected.

## Result

Person recall increased to `0.7495` and company recall to `0.9074`; company precision remains `0.8487` after excluding the high-volume heading false positives.

## Challenge

Resume contact sections can contain Indian mobile numbers and public profile URLs that are not covered by the original PII list.

## Root Cause

The original phone regex was US-oriented and no detector registered LinkedIn, GitHub, or portfolio links as sensitive data.

## Solution

An additional Indian mobile pattern and narrowly scoped LinkedIn, GitHub, and website recognisers were added. Profile usernames receive consistent Faker usernames; full portfolio URLs receive consistent fake URLs.

## Result

These PII types are redacted in text and coordinate-preserved PDFs. The included benchmark has no labels for them, so a future labelled profile dataset is needed before reporting meaningful precision and recall.
