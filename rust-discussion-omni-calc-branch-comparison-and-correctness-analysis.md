# Rust Discussion: Omni-Calc Branch Comparison And Correctness Analysis

## 1. Purpose

This document is the full technical analysis for the Rust omni-calc comparison work using only these inputs:

- `feature/update-omni-vs-python-compare-script`
- `codex/BLOX-2143-pr2951-report-flow-resolved-20260511`
- `ANALYSIS_REPORT_COMPARE_SCRIPT.txt`

The goal is to explain what the comparison report is really saying, which issues are compare-script/reporting problems, which issues are real Rust engine correctness problems, and where the newer branch improves or regresses behavior compared with the older Rust implementation.

This is analysis and documentation only. It does not propose code changes as already implemented work.

## 2. High-Level Finding

The current comparison work is useful, but the output should not be read as a simple pass/fail proof of Rust correctness.

There are three different types of issues mixed together:

1. Compare-script issues: the script can misclassify, under-report, or poorly explain mismatches.
2. Real Rust engine issues: Rust can return incorrect zeros, missing rows, missing block outputs, or wrong filter results.
3. Branch behavior differences: the newer branch fixes one older formula parsing blocker, but introduces new risks through preload metadata, dimension-type property filtering, and bulk property handling.

The most important practical conclusion is:

> The newer Rust implementation is not simply better or worse than the older one. It fixes one execution blocker, but it also changes runtime metadata/filter behavior in ways that create new value mismatches.

## 3. Branches Compared

### 3.1 Older Rust Branch

Branch:

```text
feature/update-omni-vs-python-compare-script
```

This branch represents the older Rust implementation used as a comparison baseline. It has a Rust bridge path, Rust input/filter handling, and the compare script, but it still uses older formula parsing fallback behavior and less complete preload behavior.

Important older-branch characteristics:

- Formula fallback used `bnf_parser(raw_formula)` and only consumed `parsed_result[0]`.
- Some raw formulas could remain in a form Rust did not understand.
- Some Rust engine bugs already existed, especially zero fallback behavior.
- Several materialization/filter/input bugs are shared with the newer branch.

### 3.2 Newer Rust Branch

Branch:

```text
codex/BLOX-2143-pr2951-report-flow-resolved-20260511
```

This branch contains a newer Rust implementation and the analysis report. It improves formula parsing by using `FormulaParser`, adds or changes preload-oriented execution paths, and changes connected-dimension/property filter behavior.

Important newer-branch characteristics:

- Raw formula parsing is improved.
- New preload snapshot behavior is introduced.
- Metadata loading is more eager but may be incomplete.
- Dimension-type property filter semantics changed.
- New value diffs appear that did not exist in the older branch.

## 4. Comparison Script Audit

The comparison script is valuable because it automates Rust-vs-Python checks across many models and blocks. However, the current logic still compares API JSON too structurally and does not yet give enough business-level mismatch explanation.

### 4.1 Issue 1: List/order normalization is not a real canonical comparison

Reference:

```text
modelAPI/scripts/compare_v2_vs_rust_pro.py
CalcEngineComparator._normalize_block_output_lists
CalcEngineComparator.compare_data
```

Current behavior:

The script sorts some lists by `id` or `name`, then calls `DeepDiff(..., ignore_order=True)`.

Why this is a problem:

- If Python and Rust return the same rows in different order, the script should not report a real mismatch.
- If values move to the wrong indicator or wrong period, the script must catch that clearly.
- Current output can say something like `root[9]['name'] changed`, which is hard to map back to the actual business value.

The deeper issue:

The script compares nested JSON structure instead of comparing canonical calculation keys.

The better comparison key should be:

```text
model_id + block_id + indicator_id + period + dimension item path
```

Expected improvement:

Flatten both Python and Rust output first, then compare rows by stable business key.

Example target row:

```json
{
  "model_id": 12966,
  "block_id": 25393,
  "indicator_id": 126150,
  "indicator_name": "Loans in",
  "period": "May-24",
  "python_value": 300000,
  "rust_value": 0,
  "delta": -300000,
  "classification": "value_mismatch"
}
```

Impact:

This does not create Rust bugs, but it makes the report harder to trust for exact diagnosis.

### 4.2 Issue 2: Float truncation hides real numeric differences

Reference:

```text
compare_v2_vs_rust_pro.py
_truncate_floats
```

Current behavior:

```python
return int(round(obj, 2))
```

Why this is wrong:

The code rounds a float and then converts it to an integer. That destroys decimal precision.

Example:

```text
Python: 100.49
Rust:   100.01
Current normalized values: 100 and 100
Current result: match
```

Correct behavior:

Use tolerance with `math.isclose`, preserving decimal differences and handling `NaN` / `inf` explicitly.

Impact:

Small or medium numeric mismatches can be hidden. This does not explain large zeroing bugs, but it can overstate correctness.

### 4.3 Issue 3: Missing or extra rows can be marked as metadata match

Reference:

```text
compare_v2_vs_rust_pro.py
_determine_match_status
```

Current behavior:

Some `iterable_item_added` and `iterable_item_removed` differences can become `METADATA_CHANGES`, and `METADATA_CHANGES` is counted as `values_match=True`.

Why this is wrong:

Missing indicator rows, missing time rows, and missing dimension rows are calculation-output failures, not harmless metadata.

Correct behavior:

Row additions/removals under calculation payloads should be `DIFF`, unless the field is explicitly allowlisted as metadata.

Impact:

The report can make incomplete Rust output look like a pass.

### 4.4 Issue 4: Only the first successful iteration is compared

Reference:

```text
compare_v2_vs_rust_pro.py
compare_block
```

Current behavior:

The script may run multiple iterations for timing, but correctness compares only the first successful Python response and first successful Rust response.

Why this is wrong:

If preload/cache/parallel behavior creates intermittent mismatches, this can miss them.

Correct behavior:

Compare every successful iteration and classify inconsistent results as either:

- `DIFF`
- `NON_DETERMINISTIC`
- `INTERMITTENT_RUST_FAILURE`

Impact:

The report can falsely pass a block when only later iterations fail.

### 4.5 Issue 5: Valid empty responses can be treated as endpoint failures

Reference:

```text
compare_v2_vs_rust_pro.py
_is_meaningful_data
```

Current behavior:

```python
if isinstance(data, (list, dict)) and len(data) == 0:
    return False
```

Why this is wrong:

An empty JSON response can be a valid output. The script should distinguish:

- HTTP/request failure
- valid empty payload
- invalid missing payload

Impact:

The report can create false `ERROR` results for empty-but-valid outputs.

### 4.6 Issue 6: The script compares equality but cannot prove which engine is correct

Reference:

```text
compare_v2_vs_rust_pro.py
compare_data
```

Current behavior:

The script says whether Python and Rust differ, but it does not say which side is likely correct.

Why this matters:

For engineering work, the useful question is not only:

```text
Do the responses differ?
```

It is:

```text
Which value is likely wrong, why, and which code path caused it?
```

Correct direction:

Add mismatch-cause classification:

- `rust_engine_bug`
- `python_engine_bug`
- `comparison_normalization_bug`
- `reporting_bug`
- `unknown_needs_manual_review`

Impact:

Without this, every mismatch still requires manual inspection.

## 5. Speedup And Performance Reporting Audit

### 5.1 Issue 1: Timing measures full HTTP request time, not pure engine runtime

Reference:

```text
compare_v2_vs_rust_pro.py
call_endpoint
```

Current behavior:

The timer wraps the full HTTP call:

```text
HTTP + Flask routing + auth + cache/DB + engine execution + serialization
```

What this means:

The current speedup is API latency speedup, not pure Rust engine speedup.

Correct reporting:

Keep the current number, but label it accurately:

- `python_api_ms`
- `rust_api_ms`
- `api_latency_speedup`

Then optionally add engine timings if runtime perf fields are available.

### 5.2 Issue 2: Warmup/cache/preload effects are not handled strongly enough

Current behavior:

When `--iterations 1` is used, Python always runs before Rust. This can warm shared DB/cache state before Rust runs.

Correct behavior:

Use warmup runs and measured iterations:

```text
warmup Python
warmup Rust
measured Python/Rust alternating order
report median/p95
```

### 5.3 Issue 3: Speedup is calculated for incorrect blocks

Current behavior:

The script can include speedup for blocks where Rust output is wrong.

Why this matters:

A faster wrong answer is not a useful performance win.

Correct reporting:

Separate:

- `raw_api_speedup`
- `correctness_passed_speedup`
- `excluded_due_to_diff`

### 5.4 Issue 4: Average of per-block ratios can mislead

Current behavior:

The report averages each block's speedup ratio.

Problem:

Small blocks can exaggerate the average.

Correct reporting:

Report:

- mean speedup
- median speedup
- weighted speedup: `sum(python_ms) / sum(rust_ms)`

### 5.5 Issue 5: Slowdowns are not clearly labeled

Current behavior:

`0.50x speedup` can be confusing.

Better output:

```text
Rust is 2.00x slower
```

## 6. Report Output Audit

### 6.1 Issue 1: JSON summary undercounts actual failures

The newer report says:

```json
{
  "total_blocks": 1103,
  "matched_blocks": 883,
  "failed_blocks": 4
}
```

But the detailed status distribution includes:

```text
COMPLETE_MATCH: 869
DATA_MATCH_WITH_WARNINGS_DIFF: 14
DIFF: 209
ERROR: 11
```

Problem:

`failed_blocks` currently means endpoint errors, not correctness failures.

Correct summary should include:

```json
{
  "complete_match_blocks": 869,
  "warning_diff_blocks": 14,
  "value_diff_blocks": 209,
  "error_blocks": 11
}
```

### 6.2 Issue 2: Report does not provide structured mismatch rows

Current output is based on DeepDiff paths like:

```text
root[2]['total_values'][4]['value']
```

Better output should show:

```json
{
  "model_id": 12966,
  "block_id": 25393,
  "indicator_name": "Loans in",
  "period": "May-24",
  "python_value": 300000,
  "rust_value": 0,
  "likely_cause": "dimension-type property filter semantics changed"
}
```

### 6.3 Issue 3: Indicator debug summaries are disabled in parallel model mode

In parallel mode, the script forces:

```python
include_indicator_summary=False
```

This removes useful formula/source context exactly where it is most needed: mismatched blocks.

Better approach:

Run the main comparison in parallel, then run a second debug-enrichment pass only for mismatched blocks.

### 6.4 Issue 4: Report labels still use v2/v32 naming

The command may compare:

```text
test-python vs test-omni
```

But JSON fields still say:

```text
v2_response
v32_response
```

This should be renamed or aliased to:

```text
python_response
omni_response
```

### 6.5 Issue 5: Excel dependency failure blocks JSON fallback

If Excel output is requested and `openpyxl` is missing, the script exits. Since JSON export exists, it should fall back to JSON instead of blocking the run.

### 6.6 Issue 6: Report does not capture branch/build provenance

The report should store:

- git branch
- git commit
- omni-calc `.so` hash
- Python endpoint
- Omni endpoint
- API process start time
- runtime env flags

This matters because branch/runtime mismatch can invalidate the comparison.

## 7. Older Rust Implementation Issues

### 7.1 Older Issue 1: Raw formula parsing uses incomplete BNF fallback

Branch:

```text
feature/update-omni-vs-python-compare-script
```

Reference:

```text
modelAPI/calc_engine/rust_bridge.py
_build_block_dict
```

Problem:

The older branch uses a fallback like:

```python
parsed_result = bnf_parser(raw_formula)
parsed_formula = parsed_result[0]
```

Why this is wrong:

This does not always resolve dimension/property references into Rust-compatible tokens.

Example:

```text
Raw formula:
'Members'.'Monthly Amount' * 12

Expected Rust-compatible formula:
dim123___prop456 * 12

Possible older fallback:
Members$$Monthly__Amount * 12
```

Rust may not understand the unresolved token. The result can be an execution error or zero output.

Classification:

```text
Older-branch Rust bridge bug.
Improved in newer branch.
```

Branch comparison evidence:

The report notes that model `14894`, block `38360` changed from `ERROR` in the older branch to `DIFF` in the newer branch. That means the newer formula parsing allowed execution to continue, but other correctness problems still remain.

#### Testing guide

Use this guide only for validating the old formula parsing issue.

- Relevant branch: `feature/update-omni-vs-python-compare-script`
- Model id: `14894`
- Block id: `38360`
- Exact localhost app route: not safely determined from the provided report.
- Reproduction steps:
  1. Run the backend with the older branch and rebuilt Rust extension for that branch.
  2. Open `http://localhost:3000`.
  3. Navigate to model `14894`.
  4. Find block `38360` in the model builder or relevant block output view.
  5. Compare Python output and Rust omni-calc output for the block.
- Wrong behavior:
  - Older Rust can fail the block or produce missing/zero output because the raw formula is not converted into Rust-compatible parsed formula.
- Correct behavior:
  - Rust should parse the formula using the same semantic resolution as Python and produce comparable values.
- Why this maps to the code bug:
  - The old bridge only uses `bnf_parser(raw_formula)[0]`, while the newer branch uses `FormulaParser`.

### 7.2 Older Issue 2: Raw input forecast-start handling can zero valid input values

Branches:

```text
feature/update-omni-vs-python-compare-script
codex/BLOX-2143-pr2951-report-flow-resolved-20260511
```

Reference:

```text
modelAPI/omni-calc/src/engine/exec/steps/input_handler/mod.rs
should_zero_out_input
apply_forecast_start_filter
```

Problem:

Rust can zero an entire input vector when the exact forecast-start period is missing, even if valid post-forecast periods exist.

Example:

```text
Raw input periods: Apr-24, Jun-24, Aug-24
Forecast start: Jul-24

Current Rust:
Apr-24 = 0
Jun-24 = 0
Aug-24 = 0

Expected:
Apr-24 = 0
Jun-24 = 0
Aug-24 = 150
```

Root cause:

Rust checks raw input periods directly. Python has richer normalized dataframe behavior before slicing around forecast start.

Classification:

```text
Shared Rust engine bug in both branches.
```

#### Testing guide

- Relevant branches:
  - `feature/update-omni-vs-python-compare-script`
  - `codex/BLOX-2143-pr2951-report-flow-resolved-20260511`
- Model id: not uniquely provided for this exact generic issue in the report.
- Block id/block name: not uniquely provided for this exact generic issue.
- Exact localhost app route: not safely determined from the provided report.
- Reproduction steps:
  1. Use a model/block with raw input periods where the exact forecast-start period is absent but later forecast periods exist.
  2. Run Python output and Rust omni-calc output for the same block.
  3. Inspect periods after forecast start.
- Wrong behavior:
  - Rust returns zeros for valid post-forecast input-derived values.
- Correct behavior:
  - Rust should zero only pre-forecast values and keep valid post-forecast values.
- Why this maps to the bug:
  - `should_zero_out_input` currently decides at vector level instead of normalizing by model period and slicing row-by-row.

### 7.3 Older Issue 3: Actuals filter zeros all values when time/value lengths differ

Branches:

```text
feature/update-omni-vs-python-compare-script
codex/BLOX-2143-pr2951-report-flow-resolved-20260511
```

Reference:

```text
modelAPI/omni-calc/src/engine/exec/get_source_data/actuals_filter.rs
apply_actuals_filter
```

Problem:

If `time_values.len() != values.len()`, Rust returns an all-zero vector.

Why this is wrong:

A shape mismatch is structural. It should be aligned by key or surfaced as a real error. It should not silently become valid zeros.

Classification:

```text
Shared Rust engine bug in both branches.
```

#### Testing guide

- Relevant branches:
  - `feature/update-omni-vs-python-compare-script`
  - `codex/BLOX-2143-pr2951-report-flow-resolved-20260511`
- Model id: not uniquely provided in the report.
- Block id/block name: not uniquely provided in the report.
- Exact localhost app route: not safely determined from the provided report.
- Reproduction steps:
  1. Use a block that depends on actuals/forecast filtering.
  2. Compare Python vs Rust output for periods around actual/forecast boundary.
  3. Look for series that become all zero in Rust while Python has non-zero values.
- Wrong behavior:
  - Rust turns the whole value array into zeros.
- Correct behavior:
  - Rust should align values by period or fail loudly with a precise shape error.
- Why this maps to the bug:
  - The Rust code treats length mismatch as numeric zero output.

### 7.4 Older Issue 4: Missing filter dimensions/properties return zero vectors

Branches:

```text
feature/update-omni-vs-python-compare-script
codex/BLOX-2143-pr2951-report-flow-resolved-20260511
```

Reference:

```text
modelAPI/omni-calc/src/engine/exec/filter_utils.rs
apply_item_filter
apply_property_filter
```

Problem:

When required filter metadata is missing, Rust can return zeros instead of a hard error.

Example:

```text
values = [100, 200]
filter dimension = Department
Department column missing

Current Rust:
[0, 0]

Expected:
error: required filter dimension missing
```

Classification:

```text
Shared Rust engine bug in both branches.
```

#### Testing guide

- Relevant branches:
  - `feature/update-omni-vs-python-compare-script`
  - `codex/BLOX-2143-pr2951-report-flow-resolved-20260511`
- Model id: not uniquely provided for the shared generic case.
- Block id/block name: not uniquely provided for the shared generic case.
- Exact localhost app route: not safely determined from the report.
- Reproduction steps:
  1. Open a model/block that uses property or dimension filters in formulas.
  2. Compare Python vs Rust output.
  3. Look for indicators where Python has values and Rust returns zeros.
- Wrong behavior:
  - Missing metadata is represented as zero output.
- Correct behavior:
  - Missing required metadata should be a clear Rust calculation error, or the metadata should be reconstructed correctly before filter evaluation.
- Why this maps to the bug:
  - `FilterResult::error(values.len(), msg)` effectively creates zeroed calculation output.

### 7.5 Older Issue 5: Property joins use display names instead of stable ids

Branches:

```text
feature/update-omni-vs-python-compare-script
codex/BLOX-2143-pr2951-report-flow-resolved-20260511
```

Reference:

```text
modelAPI/omni-calc/src/engine/exec/steps/input_handler/mod.rs
join_string_property_to_rows
```

Problem:

Rust joins property values using dimension item names instead of item ids.

Why this is risky:

Display names are not stable. They can duplicate, contain whitespace differences, or change.

Example:

```text
item id 10 name = Sales -> Direct
item id 20 name = Sales -> Indirect

Name-keyed map:
Sales -> one value only
```

Expected:

```text
10 -> Direct
20 -> Indirect
```

Classification:

```text
Shared Rust engine bug in both branches.
```

#### Testing guide

- Relevant branches:
  - `feature/update-omni-vs-python-compare-script`
  - `codex/BLOX-2143-pr2951-report-flow-resolved-20260511`
- Model ids called out in the report:
  - `14205`
  - `14889`
- Block id/block name:
  - Specific block ids are not provided for these model-level examples in the report.
- Exact localhost app route: not safely determined from the report.
- Reproduction steps:
  1. Open `http://localhost:3000`.
  2. Navigate to model `14205` or `14889`.
  3. Inspect department summary / P&L / cashflow style blocks that depend on connected dimensions or item properties.
  4. Compare Python output and Rust output.
- Wrong behavior:
  - Grouped/filtered values can be missing or zero because property joins fail.
- Correct behavior:
  - Property joins should use stable dimension item ids.
- Why this maps to the bug:
  - The Rust join path uses item display names as lookup keys.

### 7.6 Older Issue 6: Rust endpoint fails when requested block dataframe is missing

Branches:

```text
feature/update-omni-vs-python-compare-script
codex/BLOX-2143-pr2951-report-flow-resolved-20260511
```

Reference:

```text
modelAPI/resources/block_kpi_v4_rust.py
BlockKPINewV4Rust.post
```

Problem:

Rust may complete execution but not include the requested block dataframe.

Example:

```text
Request:
/block/36932/outputs/test-omni

Expected:
calc_result["dataframes"]["b36932"]

Current:
Block b36932 not found in results. Available: []
```

Classification:

```text
Shared Rust/API materialization bug.
```

#### Testing guide

- Relevant branches:
  - `feature/update-omni-vs-python-compare-script`
  - `codex/BLOX-2143-pr2951-report-flow-resolved-20260511`
- Model id: not provided in the report for block `36932`.
- Block id: `36932`
- Exact localhost app route: not safely determined from the report.
- Reproduction steps:
  1. Run the local backend with Rust omni-calc enabled.
  2. Call the Rust output endpoint for block `36932`.
  3. Compare against the Python endpoint for the same block/scenario.
- Wrong behavior:
  - Rust endpoint returns an error because `b36932` is missing from `dataframes`.
- Correct behavior:
  - Rust should materialize the requested block dataframe or return a precise internal materialization error with dependency details.
- Why this maps to the bug:
  - The API assumes the requested block is present in `calc_result["dataframes"]`, but Rust does not guarantee that for every block path.

## 8. Newer Rust Implementation Issues

### 8.1 Newer Issue 1: Preload snapshot can miss metadata needed at runtime

Branch:

```text
codex/BLOX-2143-pr2951-report-flow-resolved-20260511
```

Reference:

```text
modelAPI/omni-calc/src/python.rs
modelAPI/omni-calc/src/engine/exec/preload.rs
preload_metadata
collect_metadata_needs
```

Problem:

The newer branch preloads metadata before execution, but the dependency collector is too narrow.

What the code does:

```text
collect_metadata_needs scans plan.property_specs
preload_metadata loads the discovered needs
runtime executes using preloaded metadata
```

Why this can fail:

Runtime can need metadata that is not directly listed in `property_specs`, such as:

- filter specs
- connected dimension links
- formula dependencies
- source-data dimensions
- linked dimension item maps

Wrong output pattern:

```text
missing map -> empty connected dimension/property column -> filter zeros
```

Classification:

```text
Newer Rust regression / preload bug.
```

Branch comparison evidence:

The report says the newer branch has 52 new value-diff blocks that were not value-diff in the older branch. This preload behavior is a main suspect.

#### Testing guide

- Relevant branch: `codex/BLOX-2143-pr2951-report-flow-resolved-20260511`
- Model ids/block areas from report:
  - grouped P&L
  - cashflow
  - department summaries
  - revenue forecast summaries
  - VAT workings
- Exact model/block ids: not fully enumerated for this issue in the report.
- Exact localhost app route: not safely determined from the report.
- Reproduction steps:
  1. Run the newer branch with Rust omni-calc enabled.
  2. Open affected model areas such as grouped P&L, cashflow, department summaries, or VAT workings.
  3. Compare Python output against Rust output for the same block/scenario.
  4. Inspect whether connected/property-filtered values become zero or disappear.
- Wrong behavior:
  - Rust output is zero or missing where Python has values.
- Correct behavior:
  - Preload should include all metadata needed by formulas, filters, connected dimensions, and source data before execution starts.
- Why this maps to the bug:
  - Runtime now depends on preloaded metadata. If preload does not collect a needed dependency, the engine can evaluate with empty or missing maps.

### 8.2 Newer Issue 2: Dimension-type property filter changed behavior and introduced new mismatches

Branch:

```text
codex/BLOX-2143-pr2951-report-flow-resolved-20260511
```

Reference:

```text
modelAPI/omni-calc/src/engine/exec/filter_utils.rs
```

Problem:

The newer branch changes dimension-type property filter behavior. In many planner cases, `matching_item_names` are source dimension item names, but the newer filter path compares them against connected linked-dimension values.

Example from report:

```text
source rows: ["Loan Facility A", "Equity Round A"]
linked type values: ["Loan", "Equity"]
matching_item_names: ["Loan Facility A"]

Older result:
[300000, 0]

Newer result:
[0, 0]
```

Classification:

```text
Newer Rust regression.
```

Affected examples from report:

- model `12966`, block `25393`
- model `13131`, block `26523`
- model `13318`, block `27870`
- model `14720`, block `36488`

#### Testing guide

- Relevant branch: `codex/BLOX-2143-pr2951-report-flow-resolved-20260511`
- Model/block examples:
  - model `12966`, block `25393`
  - model `13131`, block `26523`
  - model `13318`, block `27870`
  - model `14720`, block `36488`
- Exact localhost app route: not safely determined from the provided report.
- Reproduction steps:
  1. Run the newer branch locally.
  2. Open `http://localhost:3000`.
  3. Navigate to one of the listed models.
  4. Open the listed block and inspect output values for financing/cashflow/property-filtered indicators.
  5. Compare Python output vs Rust output.
- Value/behavior to check:
  - Look for indicators such as funding, loans, other fundraising, or cashflow-related rows where Python has non-zero values.
- Wrong behavior:
  - Rust can return zero because source item names are compared against linked dimension values.
- Correct behavior:
  - If `matching_item_names` refer to source dimension items, Rust must match against source dimension item names first.
- Why this maps to the bug:
  - The report directly ties these new-only diffs to the changed dimension-type filter semantics.

### 8.3 Newer Issue 3: Bulk property preload drops missing/null values and changes fallback behavior

Branch:

```text
codex/BLOX-2143-pr2951-report-flow-resolved-20260511
```

Reference:

```text
modelAPI/services/model_metadata_cache_v4.py
get_item_property_values_bulk

modelAPI/omni-calc/src/engine/exec/preload.rs
```

Problem:

Bulk property preload skips cache entries with `None` and Rust removes empty property maps.

Why this matters:

The engine loses the distinction between:

- metadata was not loaded
- item has a null property value
- property exists with a real value

Classification:

```text
Newer preload bug.
```

#### Testing guide

- Relevant branch: `codex/BLOX-2143-pr2951-report-flow-resolved-20260511`
- Model/block ids: not uniquely provided for this issue in the report.
- Exact localhost app route: not safely determined from the report.
- Reproduction steps:
  1. Use a model/block with item properties where some rows have blank/null property values.
  2. Compare Python vs Rust output.
  3. Check connected dimension/property-driven columns and filtered indicators.
- Wrong behavior:
  - Rust may treat null/missing property rows as unloaded metadata and produce empty columns or zeros.
- Correct behavior:
  - Rust should preserve `MissingKey`, `NullValue`, and `StringValue` separately.
- Why this maps to the bug:
  - The bulk API drops null rows, and Rust deletes empty maps, collapsing important state.

### 8.4 Newer Issue 4: Connected dimension joins still use names, so preload magnifies id/name bugs

Branch:

```text
codex/BLOX-2143-pr2951-report-flow-resolved-20260511
```

Reference:

```text
modelAPI/omni-calc/src/engine/exec/steps/input_handler/mod.rs
join_string_property_to_rows
```

Problem:

The underlying bug is shared: joins use item names. The newer preload path uses this path more aggressively, so the issue becomes more visible.

Affected examples from report:

- model `14205` department summary
- model `14889` department/P&L/cashflow chain

#### Testing guide

- Relevant branch: `codex/BLOX-2143-pr2951-report-flow-resolved-20260511`
- Model examples:
  - `14205`
  - `14889`
- Exact block ids: not provided in the report.
- Exact localhost app route: not safely determined from the report.
- Reproduction steps:
  1. Open `http://localhost:3000`.
  2. Navigate to model `14205` or `14889`.
  3. Open department summary, P&L, or cashflow-related blocks.
  4. Compare Python and Rust values for grouped/property-joined rows.
- Wrong behavior:
  - Rust output can show missing/zero values in grouped rows.
- Correct behavior:
  - Connected dimension/property joins should align by item id.
- Why this maps to the bug:
  - The join uses display names instead of stable item ids.

### 8.5 Newer Issue 5: Inherited forecast-start zeroing remains

This is the same shared Rust engine bug described in section 7.2. The newer branch does not fix it.

### 8.6 Newer Issue 6: Inherited actuals length mismatch zeroing remains

This is the same shared Rust engine bug described in section 7.3. The newer branch does not fix it.

### 8.7 Newer Issue 7: Formula parsing improved, but does not resolve all correctness issues

Branch:

```text
codex/BLOX-2143-pr2951-report-flow-resolved-20260511
```

Reference:

```text
modelAPI/calc_engine/rust_bridge.py
_build_block_dict
```

New behavior:

```python
from calc_engine.formula_parsing import FormulaParser

fp = FormulaParser(
    metadata_cache.model_id,
    raw_formula,
    block_id,
    ind_data["id"],
)
parsed_formula = fp.parsed_formula
```

This is an improvement and should be kept.

Important nuance:

The improvement can turn an `ERROR` into a calculable `DIFF`, but it does not prove the Rust output is now correct.

Evidence:

```text
model 14894, block 38360:
older branch: ERROR
newer branch: DIFF
```

Classification:

```text
Fixed older blocker, but correctness still incomplete.
```

### 8.8 Newer Issue 8: Target block dataframe missing error remains

This is the same shared materialization issue described in section 7.6.

## 9. Root Cause Summary

### 9.1 Compare-Script Root Causes

The script is still too JSON-shape-oriented. It should become calculation-output-key-oriented.

Main root causes:

- DeepDiff paths are used as final explanation.
- Numeric tolerance is implemented by truncating floats.
- Missing rows are not reliably classified as correctness failures.
- Speedup is aggregated even when output is wrong.
- Report summary separates endpoint failure from correctness failure poorly.
- Runtime/build provenance is not captured.

### 9.2 Shared Rust Engine Root Causes

The shared Rust issues are mostly caused by using fallback zeroes for structural problems.

Main root causes:

- Raw input forecast filtering happens before full model-period normalization.
- Actuals filter handles shape mismatch by returning zeros.
- Missing filter metadata becomes zeros.
- Property joins use display names instead of item ids.
- Requested block materialization is not guaranteed.

### 9.3 Newer Branch Root Causes

The newer branch adds preload behavior and changes connected/filter semantics.

Main root causes:

- Preload dependency discovery is incomplete.
- Bulk property preload collapses missing/null/unloaded states.
- Dimension-type filter matching changes from source item names to linked values in cases where source names are still expected.
- Name-based joins become more visible because preload relies more heavily on prebuilt maps.

## 10. Issue Classification

### 10.1 Compare-Script / Reporting Only

These do not prove Rust is wrong by themselves:

- list/order normalization
- float truncation
- missing row classification
- first-iteration-only comparison
- empty response classification
- no correct-side classification
- speedup metric labeling
- JSON summary undercounting failures
- Excel fallback behavior
- v2/v32 label naming
- missing branch/build provenance

### 10.2 Real Rust Engine / App-Visible Output Issues

These can produce wrong app-visible data:

- raw input forecast-start zeroing
- actuals length mismatch zeroing
- missing filter metadata returning zeros
- property joins by display name
- target block dataframe missing
- preload metadata coverage gaps
- changed dimension-type property filter behavior
- bulk property null/missing semantics collapse

### 10.3 Older-Only Improvement Area

- raw formula parsing fallback is improved in the newer branch through `FormulaParser`

### 10.4 Newer-Only Regression Areas

- preload metadata coverage
- dimension-type property filter semantic change
- bulk property null/missing behavior

## 11. Recommended Technical Direction

### 11.1 Keep The Newer FormulaParser Change

Do not revert the newer formula parsing. It removes a real execution blocker.

### 11.2 Fix Compare Script Before Trusting Summary Counts

The compare script should be improved so the summary and mismatch rows are reliable.

Minimum required changes:

- flatten output by business key
- compare values with explicit tolerance
- classify missing/extra rows as data diffs when appropriate
- separate endpoint errors from correctness failures
- exclude incorrect blocks from correctness-passed speedup
- preserve branch/build provenance

### 11.3 Fix Newer Rust Regressions Before Promoting Preload Path

The preload path should not be treated as safe until:

- metadata dependency graph includes filters, formulas, connected dimensions, source data, and linked dimension maps
- bulk property preload preserves null/missing/unloaded state
- dimension-type filter matching is made explicit

### 11.4 Fix Shared Rust Zero Fallbacks

Rust should stop using zeros as a fallback for structural errors. Missing metadata, impossible shape mismatch, or missing requested dataframes should be hard errors with clear messages.

## 12. Practical Priority Order

1. Improve compare-script mismatch reporting so future runs are easier to trust.
2. Fix newer dimension-type property filter semantics because it maps to clear new-only diffs.
3. Expand preload dependency discovery.
4. Preserve null/missing property state in bulk preload.
5. Replace zero fallbacks with keyed alignment or hard errors.
6. Move property joins from display names to stable item ids.
7. Add materialization guarantees for requested block output.

## 13. Final Assessment

The newer branch has a good direction because it improves formula parsing and introduces a more scalable preload/runtime approach. However, the current comparison report shows that the newer branch is not yet correctness-safe.

The best path is not a full redesign. The practical path is:

- keep the current branch flow
- improve compare-script diagnostics
- fix the newer preload/filter regressions
- then address shared Rust correctness bugs that existed in both branches

The compare script should become the debugging tool that points directly to the likely cause of each mismatch, instead of only proving that a mismatch exists.
