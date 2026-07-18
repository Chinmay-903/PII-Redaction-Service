# PII Redaction Service

A compact Flask application that detects sensitive information, replaces it with consistent realistic fake data, and downloads a redacted file in the original format. The implementation favors small, isolated classes over framework-heavy architecture.

## Features

- Input: `.docx`, `.doc`, text-based `.pdf`, `.txt`, `.csv`, `.xlsx`, `.xls`, `.json`, and `.xml`.
- PII: people, email addresses, phone numbers, companies, physical addresses, SSNs, credit cards, dates of birth, IP addresses, LinkedIn/GitHub handles, and public website URLs.
- Hybrid detection: regex handles structured data; spaCy NER handles people, organisations, and locations/facilities.
- Consistent Faker replacements: identical type/value pairs always receive the same replacement within one file.
- Evaluation endpoint: compares detections with `datasets/*/labels.json` before any redaction and returns overall plus per-type metrics.

## Install and run

```powershell
python -m pip install -r requirements.txt
python -m spacy download en_core_web_sm
python pii_redactor.py
```

Open [![Live Demo](https://img.shields.io/badge/Live-Demo-brightgreen?style=for-the-badge)](https://pii-redaction-service.onrender.com/). Upload a supported document and use **Redact & Download**, or click **Run Accuracy Evaluation** for the included 1,500-record Faker dataset and 1,500-record Presidio Research dataset. Dataset provenance and label mapping are recorded in [datasets/faker/SOURCE.md](datasets/faker/SOURCE.md) and [datasets/presidio/SOURCE.md](datasets/presidio/SOURCE.md).

## Deploy on Render

Use the included `render.yaml` when creating a Render Blueprint. For an existing
Web Service, set its Build Command and Start Command to the values below, then
redeploy:

```text
pip install -r requirements.txt && python -m spacy download en_core_web_sm
gunicorn --bind 0.0.0.0:$PORT --workers 1 --timeout 600 --graceful-timeout 30 pii_redactor:app
```

The ten-minute Gunicorn timeout is intentional: PDF redaction is synchronous
and a large document can exceed Gunicorn's 30-second default timeout. One
worker prevents simultaneous large PDFs from exhausting the service's memory.

Each labels file is a JSON array. Every item contains `file` and an `entities` array with `label` and `text` fields. Add a document under the matching `original/` directory and its labels to extend the benchmark.

## Design

`FileExtractor`, `PIIDetector`, `PIIReplacer`, `FileWriter`, and `Evaluator` each own one concern. The detector builds a candidate set from compiled structured rules, profile rules, labelled fields, document headings, and spaCy entities; one ranking pass then removes duplicates and overlaps. To add a structured PII type, add one regex entry and one Faker generator entry. Contextual types are registered through `NER_LABELS`; resume/contact-block rules remain generic field and profile patterns rather than hard-coded names.

## Evaluation methodology

Predictions and labels are compared by normalised `(PII type, entity text)` pairs. True positives are pairs in both sets, false positives are predictions without a label, and false negatives are labels without a prediction. Precision, recall, F1, and the reported entity accuracy (`TP / (TP + FP + FN)`) are then calculated overall and per type. See [EVALUATION_REPORT.md](EVALUATION_REPORT.md).

## Limitations and tradeoffs

Development issues, their resolutions, and planned improvements are maintained in [challenge.md](challenge.md). The evidence behind the current detector improvements is in [error_analysis.md](error_analysis.md).

- Text-based PDFs are redacted in place, preserving surrounding page layout. Replacement text may use a fallback font or a smaller size when the fake value is longer than the source value; scanned PDFs are intentionally unsupported.
- DOCX paragraph and table text is preserved, but inline run formatting may be simplified where a replacement crosses runs.
- Legacy `.doc` conversion requires LibreOffice installed and accessible as `soffice` or `libreoffice`.
- NER can confuse companies, short locations, or names with ordinary words; regex patterns can also identify a non-PII number that resembles a phone/card value.
- NER can miss unusual names, companies, incomplete addresses, or languages beyond the installed English model. For production, use a domain-labelled benchmark, tune rules, add validation (such as Luhn checks), and consider review workflows.
