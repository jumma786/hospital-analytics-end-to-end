# Data Quality Findings

Documentation of data quality issues encountered during the hospital analytics 
project, and the decisions made to handle each one. This document accompanies 
the SQL scripts in `/sql/` and should be read alongside them.

## Source Data

- **Origin**: Synthea synthetic EHR data (Walonoski et al., JAMIA 2018)
- **Period**: 2 January 2011 to 5 February 2022
- **Volume**: 974 patients, 27,891 encounters, 47,701 procedures, 10 payers, 1 organisation
- **Files**: 5 CSVs delivered with UTF-8 BOM and Windows line endings

## Loading Issues

### Issue 1: One patient row failed BULK INSERT

**What happened**: After running `BULK INSERT` on `patients.csv`, only 973 of 
974 rows loaded. A LEFT JOIN check from encounters to patients revealed 19 
orphan encounters linked to patient ID `204f8028-72f8-d6f8-761f-79ebf9f02311` 
(Mrs. Melaine933 Hintz995).

**Likely cause**: Embedded special characters in a quoted CSV field that 
SQL Server 2019's BULK INSERT misparsed despite `FORMAT = 'CSV'`.

**Resolution**: Used SSMS's Import Flat File Wizard (which handles quoted 
fields more robustly than BULK INSERT) to load patients into a temp table, 
then merged into the main table with proper NULL and date conversion.

**Impact if unfixed**: 19 encounters (0.07% of all data) would have been 
analytically valid but unjoinable to patient demographics. Acceptable to 
proceed without fixing, but fixed for completeness.

### Issue 2: Empty strings instead of NULLs

**What happened**: After load, queries showed zero NULLs in optional columns 
(DEATHDATE, ZIP, MARITAL, SUFFIX, etc.). The source CSV has 154 patients 
with a populated deathdate and 142 with no ZIP — but the loaded table showed 
all 973 with NULL deathdate, which would have implied every patient is alive. 
Inspection revealed empty CSV fields had loaded as empty strings (`''`), 
not NULL.

**Resolution**: 
- For the patients table: reloaded via Import Wizard, using 
  `NULLIF(column, '')` and `TRY_CAST(NULLIF(column, '') AS DATE)` to 
  convert empty strings to true NULLs and parse dates safely.
- For encounters and procedures: applied UPDATE statements to convert 
  empty strings in REASONCODE and REASONDESCRIPTION to NULL.

**Impact if unfixed**: Every patient would have appeared alive in the data. 
All mortality-related analysis would have been wrong, and the data quality 
check for "encounters after death" would have falsely returned 0.

## Data Quality Checks

After fixes, the following checks confirmed a clean dataset:

| Check | Result | Status |
|---|---|---|
| Patient row count | 974 | ✅ |
| Patients with deathdate | 154 (~16%) | ✅ matches source |
| Patients missing ZIP | 142 (~14.6%) | ✅ flagged for geo analysis |
| Duplicate primary keys | 0 | ✅ |
| Duplicate procedure rows | 0 | ✅ |
| Orphan foreign keys (encounters → patients) | 0 | ✅ |
| Orphan foreign keys (procedures → encounters) | 0 | ✅ |
| Encounters with STOP < START | 0 | ✅ |
| Procedures with STOP < START | 0 | ✅ |
| Negative cost values | 0 | ✅ |
| PAYER_COVERAGE > TOTAL_CLAIM_COST | 0 | ✅ |
| Patients with death date before birth date | 0 | ✅ |

## Findings Documented but Not Fixed

These findings represent real characteristics of the data, not errors. 
Documented here so they don't get misinterpreted in downstream analysis.

### Finding A: 70% of encounter reasoncodes are NULL

**Context**: Routine visits (wellness checks, screenings, ambulatory care) 
have no specific diagnosis associated. This is normal for EHR data and 
matches real-world clinical documentation patterns.

**Decision**: Leave as NULL. Analyses that group by reason should explicitly 
include or exclude NULL depending on the question.

### Finding B: 53 encounters dated after the patient's death date

**Context**: Identified by joining encounters to patients on death date. 
These are likely administrative or billing entries posted after a patient's 
passing — for example, final billing reconciliation, certificate processing, 
or estate-related items.

**Decision**: Keep for financial and cost analyses (the encounters represent 
real spend). Exclude from patient behaviour analysis (Objective 3) where 
"patient activity" implies the patient was present.

### Finding C: One chronic-care patient skews readmission analysis

**Context**: Patient ID `1712d26d-822d-1e3a-2267-0a9dba31d7c8` 
(Kimberly627 Collier206) has 1,381 lifetime encounters — almost entirely 
ambulatory. The next-highest patient has 877. This is a chronic-care 
profile, not an error: a dialysis or similar long-term care patient would 
generate many routine ambulatory visits.

**Decision**: Keep in analysis but flag explicitly. In any "top readmitters" 
visualisation, either filter Kimberly627 out (with a footnote) or split 
ambulatory from non-ambulatory readmissions so the chronic-care signal 
doesn't dominate.

### Finding D: 4,779 zero-coverage encounters belong to insured patients

**Context**: When examining encounters with `PAYER_COVERAGE = 0`, only 
8,807 of 13,586 belong to NO_INSURANCE. The remaining 4,779 are linked 
to active insurers (Humana, Aetna, UnitedHealthcare, Cigna, Anthem, etc.) 
who covered nothing on that encounter.

**Interpretation**: This is consistent with real US healthcare patterns — 
high-deductible plans (patient hasn't met their deductible yet) or denied 
claims. It is not a data error.

**Decision**: When reporting "zero payer coverage" use the literal 
interpretation (PAYER_COVERAGE = 0, 13,586 encounters). When reporting 
"uninsured" use the payer-name filter (PAYER = NO_INSURANCE, 8,807). 
Both numbers are valuable; reporting only one can mislead.

## Interpretation Decisions in the Analytics Brief

Several questions in the original brief admit more than one interpretation. 
Each was resolved as below; alternative interpretations are noted in the 
SQL files as commented variants.

| Question | Interpretation used | Alternative |
|---|---|---|
| "Zero payer coverage" (2a) | `PAYER_COVERAGE = 0` | `PAYER = NO_INSURANCE` |
| "Admitted each quarter" (3a) | Any encounter | `ENCOUNTERCLASS = 'inpatient'` |
| "Readmitted within 30 days" (3b) | Any encounter within 30 days of any prior | Inpatient-to-inpatient only (CMS-standard) |

For 3b, both definitions were computed: broad gives 772 patients (~79%), 
strict inpatient-only gives ~29 patients.

## Lessons Learned

1. **Always sanity-check NULLs after a BULK INSERT.** Empty strings vs 
   NULLs is a silent failure that can corrupt every downstream analysis.
2. **A LEFT JOIN check is the fastest way to find load failures.** 
   Comparing row counts alone wouldn't have surfaced the missing patient.
3. **Document interpretation decisions in writing.** "Zero coverage" 
   sounds unambiguous until you look at the data. Recording these choices 
   is what separates analysis from query writing.
