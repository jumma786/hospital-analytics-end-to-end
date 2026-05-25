# Hospital Operations & Readmission Analytics

End-to-end SQL analytics project on a synthetic US hospital dataset (Synthea), 
covering 974 patients, 27,891 encounters, and 47,701 procedures from 
2011 to early 2022. Built in SQL Server / SSMS with T-SQL.

## Project Goals

Answer three sets of business questions a hospital operations team would ask:
- **Operations**: encounter volume trends, mix by class, length of stay
- **Financial**: payer coverage gaps, procedure costs, cost variance by payer
- **Patient Risk**: quarterly patient activity, 30-day readmission patterns

## Tech Stack

- **Database**: SQL Server (Microsoft) via SSMS
- **Language**: T-SQL (window functions, CTEs, conditional aggregation)
- **Data**: Synthea synthetic EHR (Walonoski et al., JAMIA 2018)

## Repository Structure

| File | Purpose |
|---|---|
| `01_setup_sqlserver.sql` | Schema creation and BULK INSERT data load |
| `02_data_quality_checks.sql` | NULL audit, duplicate check, FK integrity, date sanity |
| `03_data_fixes.sql` | Empty-string-to-NULL conversion, patient reload via Import Wizard |
| `04_analytics_objective1.sql` | Encounters overview (volume, mix, duration) |
| `05_analytics_objective2.sql` | Cost & coverage analysis |
| `06_analytics_objective3.sql` | Patient behaviour & readmission analysis |
| `data_quality_findings.md` | Documented findings and decisions |

## Data Quality Process

Before any analysis, ran a full data quality audit. Key findings:

- **1 patient row failed BULK INSERT** (Mrs. Melaine933 Hintz995) — fixed via 
  SSMS Import Flat File Wizard.
- **All date and string fields loaded as empty strings, not NULLs** — 
  resolved with `NULLIF()` and `TRY_CAST()` during reload.
- **53 encounters dated after the patient's death date** — flagged as likely 
  admin/billing entries; kept for cost analysis, excluded from patient-level analysis.
- **1 chronic-care patient with 1,381 lifetime encounters** (mostly ambulatory) — 
  not a data error, but documented to prevent misinterpretation in readmission analysis.

Full details in `data_quality_findings.md`.

## Key Findings

### Operations

- **27,891 encounters across 11 years**, peaking in 2014 (3,885) and 2021 (3,530).
- **Ambulatory visits dominate every year (37–60%)** except 2021, when 
  outpatient surged to 40% of all encounters — likely a COVID-era coding shift.
- **95.87% of encounters complete within 24 hours**; only 4.13% exceed it.

### Financial

- **48.71% of encounters had zero payer coverage** (13,586 of 27,891).
- Critically, this includes 4,779 encounters where the patient *had* insurance 
  but the payer still covered nothing — likely deductibles or denied claims.
- **Electrical cardioversion is the largest single cost driver**: 1,383 procedures 
  at ~$25,903 each = ~$35.8M in base cost.
- **Average claim cost varies 3.7× across payers**, from $1,696 (Dual Eligible) 
  to $6,205 (Medicaid). Medicaid and uninsured patients carry the highest 
  average costs — consistent with delayed-care patterns in US healthcare research.

### Patient Risk

- **Quarterly unique patients are stable around 240** for most of the period, 
  with surges in 2014-Q1 (394) and 2021-Q1/Q2 (417, 414).
- **79% of patients had at least one encounter within 30 days of a prior one** — 
  using the broad definition (any encounter class).
- Under the stricter clinical definition (inpatient-to-inpatient only), only 
  **29 patients** meet the CMS-standard 30-day readmission criterion.

## Analytical Notes

Several questions in the brief required interpretation decisions, documented inline:

1. **"Zero payer coverage"** — interpreted literally as `PAYER_COVERAGE = 0`, 
   not as "uninsured". Both numbers reported.
2. **"Admitted each quarter"** — interpreted broadly as any encounter, since 
   the brief asks about patient activity. Strict inpatient definition included as a variant.
3. **"Readmission within 30 days"** — both broad and clinical-strict definitions 
   computed and reported.

## How to Reproduce

1. Place CSV files in `C:\HospitalData\` (or update path in `01_setup_sqlserver.sql`)
2. Run scripts 01 through 06 in order
3. See `data_quality_findings.md` for context on data quality decisions

## Data Source

Synthea: Walonoski J, Kramer M, Nichols J, et al. *Synthea: An approach, method, 
and software mechanism for generating synthetic patients and the synthetic 
electronic health care record.* JAMIA, March 2018. 
https://doi.org/10.1093/jamia/ocx079
