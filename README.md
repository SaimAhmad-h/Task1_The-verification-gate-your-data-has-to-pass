# sales-order-data-cleaning

Data Cleaning & Preparation pipeline with a formal Change Log and automated Verification Gate — built for the DecodeLabs Data Analytics Internship (2026 batch).

---

## Table of Contents

1. [Problem Statement](#problem-statement)
2. [Dataset Overview](#dataset-overview)
3. [Methodology](#methodology)
4. [Missing Value Treatment](#missing-value-treatment)
5. [Duplicate Audit](#duplicate-audit)
6. [Text & Category Standardization](#text--category-standardization)
7. [Date Standardization](#date-standardization)
8. [Numeric Precision & Logic Verification](#numeric-precision--logic-verification)
9. [The Verification Gate](#the-verification-gate)
10. [Change Log](#change-log)
11. [Output Artifacts](#output-artifacts)
12. [Key Findings](#key-findings)
13. [Recommendations](#recommendations)
14. [Limitations](#limitations)
15. [Repository Structure](#repository-structure)
16. [How to Run](#how-to-run)
17. [Tools Used](#tools-used)

---

## Problem Statement

Raw sales order data cannot be trusted for analysis, reporting, or modeling until it passes a formal quality check. This project builds a repeatable cleaning pipeline that:

1. Identifies and resolves missing values without silently discarding data.
2. Detects and removes duplicate records at both the full-row and unique-ID level.
3. Standardizes text formatting, categorical values, and date formats to a single consistent standard (ISO 8601).
4. Verifies internal business logic (that `TotalPrice` actually equals `Quantity × UnitPrice`).
5. Proves — via an automated "Verification Gate" — that the cleaned dataset meets a 0%-error bar before it is exported.
6. Documents every change made in a Change Log, exported as both Excel and a formatted PDF, so the cleaning process is auditable rather than a "black box."

## Dataset Overview

- **Raw input file:** `Dataset for Data Analytics.xlsx`
- **Rows:** 1,200 orders
- **Columns:** 14
- **Fields:** `OrderID`, `Date`, `CustomerID`, `Product`, `Quantity`, `UnitPrice`, `ShippingAddress`, `PaymentMethod`, `OrderStatus`, `TrackingNumber`, `ItemsInCart`, `CouponCode`, `ReferralSource`, `TotalPrice`
- **Data types on load:** `Date` loads natively as `datetime64` (since the source is Excel, not CSV); `Quantity` and `ItemsInCart` are integers; `UnitPrice` and `TotalPrice` are floats; the remaining 9 columns are text/object type.

## Methodology

The notebook (`The_verification_gate_your_data_has_to_pass.ipynb`) runs a linear, gated pipeline:

1. **Imports** — pandas, NumPy, `datetime`.
2. **Load the raw dataset** from Excel and keep an untouched copy (`df_raw`) for before/after comparison.
3. **Initial inspection** — `df.info()` and a full column list.
4. **Identify missing values** — build a `missing_summary` table of null counts and percentages per column.
5. **Impute missing values** using a tiered strategy: explicit business-logic labeling for `CouponCode`, median imputation for any numeric column with nulls, and mode imputation for any other categorical column with nulls — every action is logged to an in-memory `change_log` list as it happens.
6. **Duplicate audit** — check full-row duplicates, duplicate `OrderID`s, and duplicate `TrackingNumber`s; also run a SQL-style `GROUP BY ... HAVING COUNT(*) > 1` equivalent to find any duplicated `OrderID`s.
7. **Remove duplicates** — drop full-row duplicates, then collapse any duplicate `OrderID`s to a single canonical row (keep first).
8. **Standardize text columns** — trim whitespace and apply consistent Title Case to all text/categorical columns (except ID-like columns: `OrderID`, `CustomerID`, `TrackingNumber`, `CouponCode`).
9. **Standardize dates to ISO 8601** — convert `Date` to `datetime`, coercing unparseable values to null, then format as `YYYY-MM-DD` strings.
10. **Enforce numeric precision** — round `UnitPrice` and `TotalPrice` to 2 decimal places.
11. **Verify business logic** — recompute `Quantity × UnitPrice` and compare it against the existing `TotalPrice`; auto-correct any row where they don't match within a 1-cent tolerance.
12. **Run the Verification Gate** — a hard `assert`-based check that duplicate ID rate, bad date format rate, and remaining null count are all exactly 0 before the pipeline is allowed to proceed.
13. **Build the Change Log** — assemble every logged change into a structured `change_log_df` table.
14. **Export the "gold standard" dataset** — save the cleaned data to `Cleaned_Dataset_for_Data_Analytics.xlsx` and the change log to `Change_Log.xlsx`.
15. **Generate a formal PDF Change Log** — using ReportLab, render the change log as a styled, wrapped-text PDF table (`Change_Log.pdf`) alongside the Verification Gate's final error rates.

## Missing Value Treatment

**Before cleaning**, only one of the 14 columns had missing values:

| Column | Missing Count | Missing % |
|---|---|---|
| CouponCode | 309 | 25.75% |

No numeric column and no other categorical column had any nulls, so the notebook's median-imputation and mode-imputation branches were present in the code but did not trigger for this dataset.

**Treatment applied:** `CouponCode` nulls were filled with the explicit label `"No Coupon"` rather than dropped or guessed at, since a blank coupon code legitimately means no coupon was used at checkout — not that data was lost. This preserved all 309 affected rows rather than discarding them.

**Result:** `Total missing values remaining: 0` (confirmed by the notebook's own printed output immediately after imputation).

## Duplicate Audit

The notebook's own printed output:

```
Full duplicate rows: 0
Duplicate OrderID values: 0
Duplicate TrackingNumber values: 0
```

The SQL-style `GROUP BY OrderID HAVING COUNT(*) > 1` check returned an **empty DataFrame** — confirming zero duplicate `OrderID`s existed even before the explicit de-duplication step ran.

**De-duplication step result:**

```
Rows before: 1200 | Rows after: 1200 | Removed: 0
```

The dataset was already free of duplicates; the de-duplication logic ran successfully but had nothing to remove.

## Text & Category Standardization

All text/categorical columns were trimmed of whitespace and converted to Title Case (excluding ID-like columns). The notebook's own output confirms the categories were **already clean** before this step — 0 cells required formatting correction — and lists the standardized (and already-consistent) category sets:

- **Product:** Chair, Desk, Laptop, Monitor, Phone, Printer, Tablet
- **PaymentMethod:** Cash, Credit Card, Debit Card, Gift Card, Online
- **OrderStatus:** Cancelled, Delivered, Pending, Returned, Shipped
- **ReferralSource:** Email, Facebook, Google, Instagram, Referral

## Date Standardization

`Date` was converted to `datetime` (coercing any unparseable values to null) and reformatted as ISO 8601 strings (`YYYY-MM-DD`). Sample of the standardized output:

```
        Date
0 2023-01-04
1 2024-08-23
2 2024-02-27
3 2023-10-15
4 2025-05-08
```

Zero dates were unparseable — no rows were lost or flagged as bad dates during this conversion.

## Numeric Precision & Logic Verification

- **Precision enforcement:** `UnitPrice` and `TotalPrice` were rounded to 2 decimal places across all 1,200 records (sample: `570.62 → 2853.10`, `151.35 → 302.70`, etc. — values were already at 2-decimal precision, so no visible rounding occurred).
- **Business logic check:** the notebook recalculated `Quantity × UnitPrice` for every row and compared it against the stored `TotalPrice` (within a $0.01 tolerance).

```
TotalPrice mismatches found & fixed: 0
```

All 1,200 `TotalPrice` values were already internally consistent with `Quantity × UnitPrice` — no corrections were necessary.

## The Verification Gate

Before exporting, the pipeline runs a hard, `assert`-enforced gate that fails the notebook outright if any of the three conditions below are not met. The notebook's own printed result:

```
Duplicate ID error rate:   0.00%  (target: 0%)
Bad date format error rate: 0.00%  (target: 0%)
Remaining missing values:  0  (target: 0)

VERIFICATION GATE: PASSED ✅
```

This is not a soft warning — the `assert` statements would halt execution with an `AssertionError` if any condition failed, meaning the cleaned file could not be exported until the dataset passed.

## Change Log

Full change log as built and exported by the notebook:

| No. | Change ID | Description | Impact | Status |
|---|---|---|---|---|
| 1 | CR001 | Imputed `CouponCode` missing values with explicit label "No Coupon" | Preserved 309 records (avoided deletion) | Resolved |
| 2 | CR_DEDUPE | Removed full-row duplicates and collapsed duplicate `OrderID` records to a single canonical row | Removed 0 duplicate record(s); 1,200 unique records remain | Resolved |
| 3 | CR_TEXT_STD | Trimmed whitespace and applied consistent Title Case across text/categorical columns | Corrected formatting on 0 cell(s) | Resolved |
| 4 | CR_DATE_ISO | Converted `Date` column to ISO 8601 format (YYYY-MM-DD) | Standardized 1,200 records; 0 unparseable date(s) flagged | Resolved |
| 5 | CR_NUMERIC_PRECISION | Rounded `UnitPrice`, `TotalPrice` to 2 decimal places for numeric consistency | Applied to 1,200 records | Resolved |
| 6 | CR_LOGIC_TOTALPRICE | Verified `TotalPrice` equals `Quantity × UnitPrice` for all records | No corrections needed — 0 mismatches found | Resolved |

Six change types were logged in total. Only one of them (`CR001`) involved an actual data correction (imputing 309 missing `CouponCode` values); the remaining five ran as verification/standardization passes that confirmed the data was already compliant.

## Output Artifacts

Running the notebook end-to-end produces three files:

| File | Description |
|---|---|
| `Cleaned_Dataset_for_Data_Analytics.xlsx` | The final "gold standard" cleaned dataset — same 1,200 rows × 14 columns, with `CouponCode` nulls resolved and `Date` in ISO 8601 format |
| `Change_Log.xlsx` | The six-row change log table shown above, exported as a spreadsheet |
| `Change_Log.pdf` | A formatted PDF version of the change log, built with ReportLab, including a title block, a styled/wrapped table, and the Verification Gate's final error rates printed at the bottom |

**Note:** installing `reportlab` required two attempts in the original run (`!pip install reportlab` failed due to a network/DNS error; `!{sys.executable} -m pip install reportlab` succeeded and installed `reportlab-5.0.0`). Both install cells are retained in the notebook.

## Key Findings

| # | Finding | Supporting Evidence |
|---|---|---|
| 1 | The dataset had exactly one column with missing data. | `CouponCode`: 309 nulls (25.75%). All other 13 columns were 100% complete on load. |
| 2 | The dataset contained **zero duplicate records** of any kind before cleaning even began. | Full duplicate rows: 0; duplicate `OrderID`s: 0; duplicate `TrackingNumber`s: 0 — confirmed both by direct checks and a SQL-style `GROUP BY` query. |
| 3 | Categorical/text formatting was already fully consistent. | 0 cells required whitespace trimming or case correction across all text columns. |
| 4 | All 1,200 dates were valid and parseable. | 0 unparseable dates after ISO 8601 conversion. |
| 5 | `TotalPrice` was internally consistent with `Quantity × UnitPrice` for every single row. | 0 mismatches found in the business-logic verification step. |
| 6 | The dataset passed the Verification Gate on the first run, with no failed assertions. | Duplicate ID rate 0.00%, bad date rate 0.00%, remaining nulls 0 — all three gate conditions met simultaneously. |
| 7 | Despite the dataset being nearly pristine, the pipeline still executed all six planned cleaning/verification steps rather than skipping them. | The Change Log documents all six change types (`CR001` through `CR_LOGIC_TOTALPRICE`), even where the "impact" was zero corrections — proving the checks ran rather than assuming cleanliness. |

## Recommendations

- **Treat `CouponCode`'s missingness as expected, not exceptional**, in any downstream analysis — the "No Coupon" label should be used directly as a valid category rather than re-investigated as a data quality issue.
- **Keep the Verification Gate's `assert` statements in place** even as the dataset evolves — since this run passed cleanly, the gate hasn't yet been tested against a genuinely dirty version of the data; it should remain strict rather than relaxed.
- **Re-run this pipeline whenever a new raw extract arrives**, rather than assuming future extracts will be as clean as this one — the low error rates here reflect this specific file, not a guarantee about the data source going forward.
- **Fix the hardcoded local file path** (`C:\Users\SAIM\Downloads\Dataset for Data Analytics.xlsx`) before sharing or re-running this notebook elsewhere — see [How to Run](#how-to-run).
- **Consider adding at least one deliberately "dirty" test case** (e.g., a synthetic duplicate row or malformed date) to a test copy of the dataset, to confirm the Verification Gate actually fails when it should — the current run only proves it passes clean data, not that it correctly catches dirty data.

## Limitations

- This notebook cleans and verifies the data but does **not** analyze it — there are no summary statistics, charts, or business insights here; that work belongs in a separate EDA notebook.
- Because the source dataset was already very clean (0 duplicates, 0 bad dates, 0 logic mismatches), this run does not demonstrate what the pipeline does when it actually encounters and must correct dirty data — only that the checks execute and pass.
- The PDF generation step depends on the third-party `reportlab` package, which is not part of the standard `pandas`/`numpy` stack and must be installed separately (see [How to Run](#how-to-run)).
- The notebook loads the raw file from a local Windows path (`C:\Users\SAIM\Downloads\Dataset for Data Analytics.xlsx`); this must be changed to a relative path before the notebook will run in a cloned repository.
- Median/mode imputation logic exists in the code for numeric and other categorical columns but was never actually exercised in this run, since only `CouponCode` had missing values — this logic is therefore unverified against real missing numeric/categorical data.





**Before running:** update the file path in Cell 2 from the original local path to a relative path so it works in any cloned copy of this repo:

```python
RAW_FILE = "Dataset for Data Analytics.xlsx"
```

`Dataset for Data Analytics.xlsx` must be placed in the same folder as the notebook. Run all cells top to bottom; the three output files will be generated automatically in the working directory.

## Tools Used

Python, pandas, NumPy, `datetime`, ReportLab, Jupyter Notebook.

---
*Data Cleaning & Preparation — DecodeLabs Data Analytics Internship, 2026 Batch.*
