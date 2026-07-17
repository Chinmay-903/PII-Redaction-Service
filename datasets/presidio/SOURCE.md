# Dataset source

Imported from [`data-privacy-stack/presidio-research`](https://github.com/data-privacy-stack/presidio-research), `data/synth_dataset_v2.json`, cloned on 2026-07-18.

- Source licence: MIT.
- Imported samples: 1,500 synthetic, labelled text records.
- Compatible labels retained: `PERSON`, `STREET_ADDRESS`, `GPE`, `ORGANIZATION`, `CREDIT_CARD`, `PHONE_NUMBER`, `EMAIL_ADDRESS`, `US_SSN`, and `IP_ADDRESS`.
- Label mapping: `STREET_ADDRESS`/`GPE` → `ADDRESS`, `ORGANIZATION` → `COMPANY`, `PHONE_NUMBER` → `PHONE`, `EMAIL_ADDRESS` → `EMAIL`, and `US_SSN` → `SSN`.

The source also contains entity types outside this application’s Version 1 scope (such as titles, ages, ZIP codes, domains, IBANs, and driver licences); those labels are intentionally not included in this evaluator’s ground truth.
