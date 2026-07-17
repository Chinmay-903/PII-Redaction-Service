# Error Analysis — Current Detector Baseline

## Scope and method

This report evaluates the current implementation before any detector changes. It uses the existing `Evaluator` matching rule: case-insensitive exact `(label, text)` matches with repeated values counted independently. The benchmark contains the Faker and Presidio datasets.

## Baseline result

| Metric | Value |
|---|---:|
| Documents | 3,002 |
| Ground-truth entities | 15,935 |
| True positives | 9,171 |
| False positives | 112 |
| False negatives | 6,764 |
| Precision | 0.9879 |
| Recall | 0.5755 |
| F1 score | 0.7273 |
| Accuracy | 0.5715 |

The unusually high precision does **not** indicate an improved detector. The configured model is `en_core_web_trf`, but it is not installed. The `OSError` fallback creates `spacy.blank("en")`, whose pipeline is empty. Therefore no contextual `PERSON`, `COMPANY`, or `ADDRESS` entities are emitted.

## Precision and recall by PII type

| PII type | TP | FP | FN | Precision | Recall | F1 |
|---|---:|---:|---:|---:|---:|---:|
| Person | 0 | 0 | 2,359 | 0.0000 | 0.0000 | 0.0000 |
| Email | 1,551 | 0 | 0 | 1.0000 | 1.0000 | 1.0000 |
| Phone | 1,526 | 24 | 68 | 0.9845 | 0.9573 | 0.9707 |
| Company | 0 | 0 | 1,750 | 0.0000 | 0.0000 | 0.0000 |
| Address | 0 | 0 | 2,509 | 0.0000 | 0.0000 | 0.0000 |
| SSN | 1,518 | 0 | 0 | 1.0000 | 1.0000 | 1.0000 |
| Credit card | 1,560 | 58 | 77 | 0.9642 | 0.9530 | 0.9585 |
| DOB | 1,501 | 28 | 0 | 0.9817 | 1.0000 | 0.9908 |
| IP address | 1,515 | 2 | 1 | 0.9987 | 0.9993 | 0.9990 |

## Top false-positive causes

There are four recurring false-positive root causes. Splitting them further would manufacture dataset-specific distinctions rather than reveal a useful engineering cause.

| Rank | Count | Root cause | Example |
|---:|---:|---|---|
| 1 | 58 | Credit-card regex consumes a trailing separator/space, so an otherwise identical ground-truth value fails exact matching. | Predicted `4007070753690781 ` vs label `4007070753690781` in `presidio_0033.txt`. |
| 2 | 28 | The imported Presidio source has general `DATE_TIME` values that are outside this tool's mapped ground truth, while the DOB regex correctly recognises the date-shaped value. | `2/8/1935` in `presidio_0112.txt` is labelled only with `PERSON: Clark`. |
| 3 | 24 | Phone regex interprets pairs of address/building numbers as phone-like values. | `17151 2450` from the address `2450 Crown St` / `17151` in `presidio_0078.txt`. |
| 4 | 2 | IPv4 regex matches the first four groups of a dotted telephone-like number. | `03.93.92.16` in `03.93.92.16.85` in `presidio_0356.txt`. |

## Top false-negative causes

The table lists every meaningful recurring cause (fewer than 20 exist in this baseline). Counts are grouped by root cause rather than artificially split by individual sample.

| Rank | Count | Root cause | Example |
|---:|---:|---|---|
| 1 | 1,501 | PERSON labels in the Faker corpus are missed because the active spaCy pipeline has no NER component. | `Jožef Albin` in `faker_0001.txt`. |
| 2 | 1,500 | COMPANY labels in the Faker corpus are missed for the same missing-NER reason. | `Shaffer-Sims` in `faker_0001.txt`. |
| 3 | 1,500 | Structured-looking Faker street addresses are missed because no address regex exists and NER is unavailable. | `22038 Lewis Isle, New Isabella, MI 54350` in `faker_0001.txt`. |
| 4 | 1,009 | Presidio multi-line and international addresses are missed because NER is unavailable and the detector has no address parser. | `6750 Koskikatu 25 Apt. 864\nArtilleros\n, CO\nUruguay 64677` in `presidio_0001.txt`. |
| 5 | 858 | Presidio person names, including diacritics and initials, are missed because NER is unavailable. | `Mijail C Adomo` in `presidio_0002.txt`. |
| 6 | 250 | Presidio organisation names are missed because NER is unavailable. | `Bender LLC` in `presidio_0020.txt`. |
| 7 | 57 | Exact-match evaluation counts a card as missed when detector text differs only by a trailing space. This is an evaluation-normalisation defect, not a detection failure. | Label `4007070753690781`; predicted `4007070753690781 `. |
| 8 | 68 | Phone regex is primarily North-American and misses international, parenthesised, extension, and short local forms. | `+41 (0)96 471 07 95`, `07700 063 966`, and `60-56-85-91`. |
| 9 | 10 | Credit-card regex excludes 19-digit card values. | `4131034282458809939` in `presidio_0032.txt`. |
| 10 | 10 | Credit-card regex excludes 12-digit card values. | `630427373398` in `presidio_0038.txt`. |
| 11 | 1 | IP regex supports IPv4 only. | IPv6 `6e40:4041:c617:e898:c11:40d2:c669:2eb4`. |

## Example mistake summary

| Category | Current behaviour | Why it is wrong |
|---|---|---|
| Person | `Jožef Albin` is not detected. | The configured NER model is absent and the silent fallback has no NER. |
| Company | `Bender LLC` is not detected. | Same empty-NER regression. |
| Address | `22038 Lewis Isle, New Isabella, MI 54350` is not detected. | No structured address recogniser; NER is unavailable. |
| Card | `4007070753690781 ` is predicted but misses the label without its trailing space. | Matching does not normalise whitespace. |
| Phone | `17151 2450` is predicted as a phone. | Address numbers satisfy the broad numeric pattern. |
| DOB | `2/8/1935` is reported as DOB although its source label is not mapped as DOB. | Benchmark label mapping and detector scope are inconsistent. |
| IP | `03.93.92.16` is predicted from `03.93.92.16.85`. | Regex accepts the first four dotted groups without a stronger boundary check. |

## Evidence-led improvement priorities

1. **Restore a real NER model and fail clearly if it is unavailable.** Do not silently use a blank pipeline for contextual PII; this is responsible for 6,618 of 6,764 false negatives.
2. **Normalise evaluation whitespace** before exact comparison while preserving label/type and duplicate counts. This removes 57 evaluation artefacts without inflating unrelated matches.
3. **Add conservative address and company validation.** Address recognition should require structural cues (street number, street/postal keywords, ZIP/postcode) and company recognition should require suffix/context checks; NER should provide candidates, not final labels.
4. **Extend structured patterns deliberately.** Add robust boundaries for IP values, avoid address-number phone matches, and support internationally formatted phone/card values only with validation.
5. **Re-run this report after each detector change.** A meaningful improvement must increase PERSON/COMPANY/ADDRESS recall without recreating the broad NER false positives seen in earlier runs.
