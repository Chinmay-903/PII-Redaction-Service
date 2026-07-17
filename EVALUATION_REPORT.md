# Evaluation Report

Generated on 2026-07-18 using the 1,501-document Faker corpus and 1,501-document Presidio corpus. Matching is case-insensitive, whitespace-normalised, and requires both the entity type and text to match.

## Overall results

| Metric | Result |
|---|---:|
| Documents | 3,002 |
| Total entities | 15,935 |
| True positives | 14,132 |
| False positives | 644 |
| False negatives | 1,803 |
| Precision | 0.9564 |
| Recall | 0.8869 |
| F1 score | 0.9203 |
| Accuracy | 0.8524 |

This is an improvement over the preceding `0.8508` F1 result. Explicit contact-field detection, relaxed-but-validated name rules, and contextual company handling increased recall; headings such as `Phone` are excluded from company candidates to preserve precision.

## Results by PII type

| PII type | TP | FP | FN | Precision | Recall | F1 |
|---|---:|---:|---:|---:|---:|---:|
| Person | 1,768 | 115 | 591 | 0.9389 | 0.7495 | 0.8336 |
| Email | 1,551 | 0 | 0 | 1.0000 | 1.0000 | 1.0000 |
| Phone | 1,526 | 11 | 68 | 0.9928 | 0.9573 | 0.9748 |
| Company | 1,588 | 283 | 162 | 0.8487 | 0.9074 | 0.8771 |
| Address | 1,548 | 194 | 961 | 0.8886 | 0.6170 | 0.7283 |
| SSN | 1,518 | 0 | 0 | 1.0000 | 1.0000 | 1.0000 |
| Credit card | 1,617 | 1 | 20 | 0.9994 | 0.9878 | 0.9935 |
| DOB | 1,501 | 28 | 0 | 0.9817 | 1.0000 | 0.9908 |
| IP address | 1,515 | 2 | 1 | 0.9987 | 0.9993 | 0.9990 |
| LinkedIn | N/A | N/A | N/A | N/A | N/A | N/A |
| GitHub | N/A | N/A | N/A | N/A | N/A | N/A |
| Website | N/A | N/A | N/A | N/A | N/A | N/A |

The supplied benchmark labels do not contain LinkedIn, GitHub, or website entities. They are therefore excluded from the scored table rather than represented by misleading zero metrics. The evaluator API still returns their zero-labelled counters for transparency.

## Error discussion

- **False positives:** the remaining company false positives are ambiguous capitalised phrases (for example, publication titles and fictional groups). Twenty-eight DOB false positives are general dates in the imported Presidio source that are not labelled as DOB.
- **False negatives:** the main gap is international and multi-line addresses. Some labels are address fragments or standalone geographic names, whereas the detector deliberately requires an address cue to avoid redacting every location.
- **Strength:** regex provides near-perfect structured PII detection; spaCy supplies contextual candidates, and field/header rules recover common resume and contact-block PII without dataset-specific names.
- **Current limitations:** profile URLs are detected and redacted but are not benchmarked yet; PDF replacement may shrink text to fit its original rectangle; scanned PDFs remain unsupported.
- **Next steps:** add labelled profile-link samples, improve international address parsing, and use a stronger installed spaCy model where startup and memory budgets allow it.
