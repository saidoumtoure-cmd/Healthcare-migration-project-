[README.md](https://github.com/user-attachments/files/26986259/README.md)
# 🏥 Healthcare Data Migration Project

> **Simulated enterprise-grade ETL pipeline** migrating patient records from a legacy EHR system to a new platform — with full data cleaning, transformation, field mapping, and validation reporting.

---

## 📋 Project Overview

This project simulates a real-world healthcare data migration engagement. The scenario involves extracting 95 patient records from **LegacyEHR v3.2**, identifying and resolving multiple data quality issues, and loading 80 clean, standardized records into **NewEHR v1.0** — with a complete audit trail and data quality report.

The pipeline is intentionally designed to reflect common challenges found in production healthcare migrations:

- Mixed date formats across thousands of records
- Name format inconsistencies caused by manual data entry
- Duplicate records introduced by system errors and manual re-entry
- Missing contact information with no safe imputation path
- Inconsistently formatted phone numbers

---

## 🗂️ Repository Structure

```
healthcare-migration/
├── data/
│   ├── legacy_patients_raw.csv                   # Source: 95 records with issues
│   ├── legacy_patients_cleaned_intermediate.csv  # Post-cleaning, pre-rename
│   └── clean_patients_new_system.csv             # Final: 80 migrated records
│
├── scripts/
│   ├── 01_generate_legacy_data.py   # Generates raw dataset with injected issues
│   ├── 02_clean_transform.py        # ETL cleaning & transformation pipeline
│   └── 03_validate_report.py        # Validation checks & data quality report
│
├── sql/
│   └── migration_queries.sql        # Schema DDL, ETL inserts, validation queries
│
├── reports/
│   ├── audit_trail.json             # Step-by-step audit log (machine-readable)
│   └── data_quality_report.json     # Structured quality report with scores
│
└── README.md
```

---

## 🛠️ Tools & Technologies

| Layer | Tool |
|---|---|
| Language | Python 3.12 |
| Data manipulation | pandas |
| Database queries | PostgreSQL-compatible SQL |
| Serialization | JSON (audit trail & reports) |
| Version control | Git / GitHub |

---

## 🔍 Dataset Details

### Legacy System Fields (Source)

| Field | Type | Notes |
|---|---|---|
| `patient_id` | VARCHAR | e.g., PT0001 |
| `old_name` | VARCHAR | Mixed formats |
| `dob` | VARCHAR | Inconsistent date formats |
| `address` | TEXT | Some null |
| `phone_number` | VARCHAR | Mixed formats |
| `email` | VARCHAR | Some null, some invalid |
| `registration_date` | VARCHAR | Inconsistent formats |
| `insurance_provider` | VARCHAR | Some null |
| `blood_type` | VARCHAR | Consistent |
| `primary_physician` | VARCHAR | Consistent |

### Intentionally Injected Data Quality Issues

| Issue Type | Description | Count |
|---|---|---|
| Exact duplicates | Full row duplicates (system export error) | 10 |
| Near-duplicates | Same person, different patient_id | 5 |
| Mixed DOB formats | MM/DD/YYYY mixed with YYYY-MM-DD | ~30% of records |
| Name format mix | "LAST, FIRST" and "ALL CAPS" entries | ~25% of records |
| Missing emails | NULL contact_email | 8 |
| Missing phones | NULL phone_number | 6 |
| Missing address | NULL mailing_address | 5 |
| Missing insurer | NULL insurance_provider | 7 |
| Unformatted phones | Digits only, no formatting | ~20% of phone records |

---

## 🔄 Field Mapping (Legacy → New System)

| Legacy Field | New Field | Transformation Applied |
|---|---|---|
| `patient_id` | `legacy_id` | Preserved as reference key |
| `old_name` | `full_name` | Title case; "LAST, FIRST" → "First Last" |
| `dob` | `date_of_birth` | Normalized to ISO 8601 (YYYY-MM-DD) |
| `address` | `mailing_address` | Whitespace trimmed |
| `phone_number` | `contact_phone` | Normalized to (XXX) XXX-XXXX |
| `email` | `contact_email` | Lowercased; regex-validated |
| `registration_date` | `enrollment_date` | Normalized to ISO 8601 |
| `insurance_provider` | `payer_name` | NULLs filled with "Unknown" |
| `blood_type` | `blood_group` | No transformation needed |
| `primary_physician` | `attending_physician` | No transformation needed |
| *(generated)* | `new_patient_id` | Sequential: NP00001, NP00002… |
| *(generated)* | `migrated_at` | UTC timestamp at run time |
| *(generated)* | `data_quality_flag` | COMPLETE / INCOMPLETE |

---

## 🧹 Cleaning & Transformation Steps

### Step 1 — Name Normalization
All names converted to **"First Last" title case**.  
Patterns handled: `"DOE, JOHN"` → `"John Doe"` | `"JOHN DOE"` → `"John Doe"`

```python
if "," in raw:
    parts = [p.strip() for p in raw.split(",", 1)]
    name = f"{parts[1].title()} {parts[0].title()}"
else:
    name = " ".join(p.title() for p in raw.split())
```

### Step 2 — Date Standardization
All date fields parsed from multiple formats and unified to **ISO 8601 (YYYY-MM-DD)**.  
Formats handled: `MM/DD/YYYY`, `YYYY-MM-DD`, `YYYY/MM/DD`, `DD-MM-YYYY`

### Step 3 — Phone Number Normalization
All phone numbers standardized to **(XXX) XXX-XXXX**.  
Strips all non-digit characters, validates 10-digit US format.

### Step 4 — Email Validation
Lowercased and regex-validated against pattern `^[\w.+-]+@[\w-]+\.\w{2,}$`.  
Invalid entries set to NULL rather than attempting correction.

### Step 5 — Null Handling
- `insurance_provider` (now `payer_name`): NULL → `"Unknown"` (safe default)
- `phone`, `email`, `address`: NULLs retained (cannot safely impute patient contact data)

### Step 6 — Deduplication
Two-pass strategy:
1. **Exact duplicates** removed via `drop_duplicates()`
2. **Near-duplicates** detected by `(full_name, date_of_birth)` compound key → keep record with lowest `patient_id`

### Step 7 — Field Mapping & Schema Enrichment
Legacy fields renamed to new system schema. Three synthetic fields added:
`new_patient_id`, `migrated_at`, `data_quality_flag`

---

## ✅ Data Validation Report

| Metric | Value |
|---|---|
| Raw legacy records | 95 |
| Exact duplicates removed | 10 |
| Near-duplicates removed | 5 |
| **Final migrated records** | **80** |
| Missing address (final) | 5 |
| Missing phone (final) | 6 |
| Missing email (final) | 8 |
| Missing payer (final) | 0 |
| COMPLETE records | 72 (90%) |
| INCOMPLETE records | 8 (10%) |
| **Overall Data Quality Score** | **97.8% — Grade A** |

---

## ⚠️ Key Challenges

**1. Inconsistent date formats across the same column**  
The legacy `dob` column stored dates in at least three different formats with no schema enforcement. Solution: multi-format parser with ordered fallback attempts.

**2. Near-duplicate records without identical patient IDs**  
When the legacy system was migrated from an older platform (~5 years prior), some patient records were re-entered manually, generating new IDs for existing patients. These required fuzzy matching on `(name, date_of_birth)` rather than ID-based deduplication.

**3. Missing contact data with no imputation path**  
~10% of records had missing phone or email. Unlike insurance provider (which can default to "Unknown"), contact information cannot be safely imputed — doing so in a healthcare context risks HIPAA violations. These were preserved as NULL and flagged INCOMPLETE.

**4. Name format inconsistency from data entry staff**  
Two legacy departments used different conventions: one used "First Last" while another used "LAST, FIRST". The parser handles both without hardcoding department-specific rules.

---

## 🚀 How to Run

```bash
# 1. Install dependencies
pip install pandas

# 2. Generate raw legacy dataset
python scripts/01_generate_legacy_data.py

# 3. Run cleaning & transformation pipeline
python scripts/02_clean_transform.py

# 4. Generate data quality report
python scripts/03_validate_report.py
```

Outputs are written to `data/` and `reports/`.  
SQL DDL and validation queries are in `sql/migration_queries.sql`.

---

## 📊 Final Results

| | Before | After |
|---|---|---|
| Record count | 95 | 80 |
| Duplicate records | 15 | 0 |
| Date format variations | 3 | 1 (ISO 8601) |
| Name format variations | 3 | 1 (Title Case) |
| Phone format variations | 2 | 1 ((XXX) XXX-XXXX) |
| NULL insurance records | 7 | 0 |
| Data quality score | ~73% | **97.8%** |

---

## 📄 License

This project uses synthetic, generated data only. No real patient information is included. For portfolio and demonstration purposes.

---

*Built as a portfolio project demonstrating data migration analysis skills: ETL pipeline design, data quality management, field mapping, SQL schema design, and validation reporting.*
