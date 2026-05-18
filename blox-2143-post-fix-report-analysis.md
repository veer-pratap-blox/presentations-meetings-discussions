# BLOX-2143 Post-Fix Report Analysis

This report compares the pre-fix reports with the latest post-fix reports generated from `BLOX-2143-pr2951-report-flow-resolved-20260511`.

Compared files:

- Pre-fix: `/Users/veerpratapsingh/Desktop/correctness_comparison1.xlsx`
- Pre-fix: `/Users/veerpratapsingh/Desktop/correctness_comparison1.json`
- Post-fix: `/Users/veerpratapsingh/Desktop/compare_script_issues_fixed_newer_rust_impl_15May2026.xlsx`
- Post-fix: `/Users/veerpratapsingh/Desktop/compare_script_issues_fixed_newer_rust_impl_15May2026.json`

PPT references:

- `/Users/veerpratapsingh/Desktop/compare script rust vs python/rust-vs-python-compare-script-omni-calc-meeting-deck-with-report-examples12-updated-analysis.pptx`
- `/Users/veerpratapsingh/Desktop/compare script rust vs python/rust-vs-python-compare-script-omni-calc-meeting-deck-with-report-examples12.pptx`

## High-Level Result

Post-fix report status:

- `883 COMPLETE_MATCH`
- `15 DATA_MATCH_WITH_WARNINGS_DIFF`
- `206 DIFF`
- `4 ERROR`
- `58,556` mismatch diagnosis rows
- `81.05%` correctness pass rate
- `99.64%` endpoint success rate

Compared with the pre-fix report:

- Complete matches increased from `869` to `883`.
- Value diffs reduced from `209` to `206`.
- Endpoint/error blocks reduced from `11` to `4`.
- `13` common blocks moved from bad status to good status:
  - `7 ERROR -> COMPLETE_MATCH`
  - `5 DIFF -> COMPLETE_MATCH`
  - `1 DIFF -> DATA_MATCH_WITH_WARNINGS_DIFF`

## What Is Resolved Now

These PPT issues are resolved or materially improved in the latest report.

### Compare Script Issue 1: DeepDiff Path Like `root[9][name]` Is Not Enough

Resolved. Latest report now has business-level issue paths like:

```text
indicator:126150 / period:Aug-23 / field:total_values.value / occurrence:1
```

This is much clearer than raw array paths because it identifies the affected indicator, period, and field.

### Compare Script Issue 2: `ignore_order` / Sorting Hides Wrong Value Pairing

Resolved in code. Latest report compares canonical business-keyed rows instead of relying on array position or order masking.

### Compare Script Issue 3: Float Truncation Can Hide Differences

Resolved. Latest post-fix report does not show `rust_precision_or_rounding_mismatch` as a remaining block-level issue. Precision-only differences are no longer being counted the same way as real business mismatches.

### Compare Script Issue 4: Missing Rows Treated As Metadata/Warning

Resolved. Missing/value row issues are now classified as real data diffs unless they are explicitly warning/metadata-only.

### Compare Script Issue 6: Empty Valid Output Becomes Failure

Improved. Pre-fix had `11 ERROR`; post-fix has `4 ERROR`. Also, `7 ERROR -> COMPLETE_MATCH`, so several false/error-style failures are now clean matches.

### Compare Script Issue 7 / Issue 15: DIFF Does Not Say Which Side Is Wrong / Diagnosis Columns Unknown

Resolved strongly. Latest report has diagnosis populated for all bad blocks:

- `206 DIFF + 4 ERROR = 210` bad blocks
- `210 / 210` bad blocks have `diagnosis_summary`
- `58,556` mismatch diagnosis rows
- Only `1` diagnosis row has unknown side, and that is the valid `both_endpoint_error` case where both endpoints failed.

### Compare Script Issue 8: Timing Is Full HTTP Time

Improved. Latest XLSX now labels timing more clearly:

- `Python API Round Trip`
- `Omni API Round Trip`
- `Python Engine Time`
- `Omni Engine Time`
- `Engine Speedup`

### Compare Script Issue 11: Summary Undercounts Correctness Failures

Resolved. Latest JSON summary now explicitly has:

- `complete_match_blocks`
- `warning_diff_blocks`
- `value_diff_blocks`
- `endpoint_error_blocks`
- `correctness_pass_rate`
- `endpoint_success_rate`

## Fix Evidence From Latest Report

The clearest fixed functional bug is **Model `14752`: stale input dimension parity**.

Before fix:

- `14752 / 36778 AI Operator Revenue`: `DIFF`, `15 value changes`
- `14752 / 36783 COGS`: `DIFF`, `12 value changes`
- `14752 / 36786 Cashflow`: `DIFF`, `12 value changes`
- `14752 / 36787 Profit & Loss`: `DIFF`, `17 value changes`
- `14752 / 36788 Key SaaS Metrics`: `DIFF`, `12 value changes`

After fix:

- All above are `COMPLETE_MATCH`.

Meaning: the stale removed-dimension issue is fixed in both Python and omni-calc paths for that model.

### Detailed Example: Model `14752`, Block `36778`, `AI Operator Revenue`

Before fix:

- Block `36778 / AI Operator Revenue`: `DIFF`, `values_changed: 15`
- Python had empty output for indicators like `Total customer number`
- Rust had values like `Apr-26 = 120`
- This matched the stale input dimension bug: a removed dimension still existed in old `DataInputs.data_values`

After fix:

- Block `36778 / AI Operator Revenue`: `COMPLETE_MATCH`
- Block `36783 / COGS`: `DIFF 12 -> COMPLETE_MATCH`
- Block `36786 / Cashflow`: `DIFF 12 -> COMPLETE_MATCH`
- Block `36787 / Profit & Loss`: `DIFF 17 -> COMPLETE_MATCH`
- Block `36788 / Key SaaS Metrics`: `DIFF 12 -> COMPLETE_MATCH`

This is the clearest parity fix to show in a sprint demo.

### Another Good Example: Model `14870`, Block `37969`, `Financing`

Before:

- `DIFF`
- `values_changed: 2`

After:

- `DATA_MATCH_WITH_WARNINGS_DIFF`
- Both endpoints returned `200`
- `values_match = true`
- `diff_count = 1`
- Difference is only:

```text
iterable_item_removed: root['id:233195']['warning'][0]
```

Meaning: the actual data mismatch is gone. The remaining difference is warning metadata only.

## Post-Fix ERROR Root Causes

Post-fix has `4 ERROR` blocks.

### 1. Model `14662`, Block `35992`, `Calculate Vintagewise Tied SM HC`

Issue type:

```text
both_endpoint_error
```

Root cause evidence:

Both Python and omni-calc failed with:

```text
Error calculting summary ratio with formula: block35993___ind214912
for indicator: (215789)
```

This is not a Rust-vs-Python value mismatch. Both endpoints fail before meaningful comparison.

PPT mapping:

- `NEW ISSUE: Shared Summary-Ratio Failure Is Post-Processing`

Likely area:

- shared API/model/reporting post-processing path
- summary-ratio formula calculation
- `modelAPI/calc_engine/data_agreggator.py` ratio-total calculation area

### 2. Model `14754`, Block `36932`, `NRE (Projects) Revenue`

Issue type:

```text
rust_endpoint_error
```

Root cause evidence:

Python returned a meaningful response, but omni-calc failed with:

```text
Calculation failed: Result extraction failed:
Block b36932 not found in results. Available: []
```

PPT mapping:

- `Rust Issue 6: Requested Block Dataframe Missing`

Likely area:

- Rust result materialization
- requested block dataframe extraction
- `block_kpi_v4_rust.py` result extraction path
- omni-calc execution/dataframe output path

### 3. Model `14789`, Block `37089`, `Financing`

Issue type:

```text
rust_endpoint_error
```

Root cause evidence:

```text
Calculation failed: Result extraction failed:
Block b37089 not found in results. Available: []
```

PPT mapping:

- `Rust Issue 6: Requested Block Dataframe Missing`

Likely area:

- Rust result materialization / requested block dataframe not emitted

### 4. Model `14796`, Block `37162`, `Financing`

Issue type:

```text
rust_endpoint_error
```

Root cause evidence:

```text
Calculation failed: Result extraction failed:
Block b37162 not found in results. Available: []
```

PPT mapping:

- `Rust Issue 6: Requested Block Dataframe Missing`

Likely area:

- Rust result materialization / requested block dataframe not emitted

## About The Remaining `206 DIFF` Blocks

The latest report explains the remaining `206 DIFF` blocks much better than the pre-fix report.

Block-level diagnosis for bad blocks:

- `127` blocks: `rust_numeric_value_mismatch`
- `42` blocks: `rust_zeroed_reference_value`
- `37` blocks: `rust_unfiltered_nonzero_value`
- `3` blocks: `rust_endpoint_error`
- `1` block: `both_endpoint_error`

For the `206 DIFF` blocks, the report now includes:

- correct side
- wrong side
- confidence
- issue path
- likely cause
- suspected code path
- evidence summary

So the report no longer only says `values_changed`; it now tells where to inspect.

Important nuance: these are high-quality diagnosis classifications, but they are still automated likely-cause labels. They correctly group issue patterns, but they are not a human-confirmed root cause for every single row.

## Are Remaining Issues Covered In The PPTs?

Mostly yes. The main remaining bug families are represented in the updated PPT. The PPT does not list all `206` blocks individually, but it does cover the categories causing them.

### `rust_zeroed_reference_value`

Maps to:

- `Rust Issue 2: Forecast-Start Input Zeroing`
- `Rust Issue 3: Actuals Length Mismatch Zeroes Everything`
- `Rust Issue 4: Missing Filter Metadata Returns Zeros`
- `Newer Rust Issue 7: Preload Snapshot Misses Runtime Metadata`
- `Newer Rust Issue 8: Dimension-Type Filter Semantics Changed`
- `Newer Rust Issue 9: Bulk Property Preload Drops Null/Missing Semantics`

Meaning:

Omni-calc returned zero where Python reference returned a non-zero value. This points to Rust dropping or zeroing a valid row.

### `rust_unfiltered_nonzero_value`

Maps to:

- `Rust Issue 10: Rust Keeps Non-Zero Rows That Python Zeros Out`
- `NEW ISSUE: Extra Output Rows Need Zero vs Non-Zero Split`
- filter/source-data/actual-period semantic mismatch issues

Meaning:

Omni-calc returned a non-zero value where Python reference returned zero. This points to Rust retaining rows that Python filters or zeros out.

### `rust_numeric_value_mismatch`

Maps to:

- `Rust Issue 11: Model 14712 Rollups Differ Across Statement Blocks`
- formula evaluation
- source input alignment
- filtering
- dependency materialization areas

Meaning:

Both engines returned numeric values, but omni-calc differs from Python reference.

### `rust_endpoint_error`

Maps to:

- `Rust Issue 6: Requested Block Dataframe Missing`

Meaning:

Omni-calc endpoint failed while Python returned a meaningful response. The main observed error is:

```text
Block b<block_id> not found in results. Available: []
```

### `both_endpoint_error`

Maps to:

- `NEW ISSUE: Shared Summary-Ratio Failure Is Post-Processing`

Meaning:

Both endpoints failed, so the script cannot decide which engine output is correct.

## Report Improvements

Pre-fix XLSX had only:

- `Summary`
- `Block Details`
- `Statistics`

Post-fix XLSX now has:

- `Summary`
- `Block Details`
- `Iteration Details`
- `Mismatch Diagnosis`
- `Statistics`

Post-fix report also added `58,556` mismatch diagnosis rows. This is a major reporting improvement.

Before, a mismatch looked like:

```text
root[9][name] changed
values_changed: 108
```

After, it now shows useful business/debug fields:

- `Correct Side`
- `Wrong Side`
- `Confidence`
- `Issue Path`
- `Likely Cause`
- `Suspected Code Path`
- `Evidence`

Example from post-fix report:

- Model `12966`, block `25392`, `Cashflow`
- Indicator `Loans in`
- Period `Aug-23`
- Python value: `300000`
- Omni value: `0`
- Correct side: `python`
- Wrong side: `omni-calc`
- Issue type: `rust_zeroed_reference_value`
- Suspected path: `filter_utils.rs`, `actuals_filter.rs`, `input_handler`, `preload.rs`

That is much better than the previous generic DeepDiff output.

## Not Fully Resolved / Caveats

- `Compare Script Issue 5`: multi-iteration drift is fixed in code, but this all-model report used `1` iteration per block, so this report does not prove multi-iteration behavior.
- Indicator debug summaries are still not clearly present in this all-model workbook unless generated with the relevant flag.
- Branch/build provenance is still not fully solved if the report does not store branch, commit, `.so` hash, and endpoint config directly.
- The remaining `206 DIFF` blocks are real investigation targets, not compare-script false positives based on the latest report structure.
