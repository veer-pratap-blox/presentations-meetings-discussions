# PR 2951 - Omni Calc Correctness Report Improvements

PR reviewed:

- PR: https://github.com/BloxSoftware/Blox-Dev/pull/2951
- Base branch: `development`
- Head branch: `feature/update-omni-vs-python-compare-script`
- PR title: `worked on enhancing script and adding test endpoints`
- Local current branch checked: `BLOX-2143-add-omni-calc-runtime-performance-tracing-and-benchmark-baseline`

## 1. Short Answer

PR #2951 is a useful start, but the comparison script is not fully correct yet.

The PR fixes one important issue:

```text
Old script:
  /block/<block_id>/outputs/v2
  /block/<block_id>/outputs/v3.2

Problem:
  /outputs/v2 can route to Rust if the model running version is configured as Rust.
  So some checks could accidentally compare Rust vs Rust.

PR #2951:
  /block/<block_id>/outputs/test-python
  /block/<block_id>/outputs/test-omni

Improvement:
  The script can now call fixed Python and fixed Rust endpoints.
```

But the PR does not fully fix the correctness report.

It does not fix:

- Rust input/forecast zeroing.
- Rust not matching Python input dataframe normalization.
- Rust result extraction failure when no dataframe is produced.
- Over-permissive numeric comparison.
- Order-insensitive diffing that can hide time-series mistakes.
- JSON summary undercounting incorrect blocks.
- JSON export missing HTTP status and endpoint identity.
- Clear separation between value mismatches, warning-only mismatches, metadata mismatches, empty responses, and endpoint errors.

## 2. What PR #2951 Changes

Files changed in the PR:

```text
modelAPI/resources/block_kpi_router.py
modelAPI/scripts/compare_v2_vs_rust_pro.py
modelAPI/services/routes.py
modelAPI/omni-calc/python/omni_calc/omni_calc.cpython-311-darwin.so
modelAPI/omni-calc/target/.rustc_info.json
```

Important code added in `modelAPI/resources/block_kpi_router.py`:

```python
class BlockKPIOutputsTestPython(Resource):
    """
    QA / regression only: Python calc (BlockKPINew), ignoring model running version.
    """

    @token_required
    def post(self, current_user, block_id):
        block = BlocksModel.find_by_id(block_id)
        if not block:
            return {"message": "Block not found"}, 404

        if not check_model_permission(user=current_user, model_id=block.model.id, action="is_read"):
            return unauthorized()

        handler = BlockKPINew()
        return handler.post.__wrapped__(handler, current_user, block_id)


class BlockKPIOutputsTestOmni(Resource):
    """
    QA / regression only: omni-calc Rust path (BlockKPINewV4Rust), ignoring model running version.
    """

    @token_required
    def post(self, current_user, block_id):
        block = BlocksModel.find_by_id(block_id)
        if not block:
            return {"message": "Block not found"}, 404

        if not check_model_permission(user=current_user, model_id=block.model.id, action="is_read"):
            return unauthorized()

        handler = BlockKPINewV4Rust()
        return handler.post.__wrapped__(handler, current_user, block_id)
```

Routes added in `modelAPI/services/routes.py`:

```python
api.add_resource(BlockKPIOutputsTestPython, '/block/<int:block_id>/outputs/test-python')
api.add_resource(BlockKPIOutputsTestOmni, '/block/<int:block_id>/outputs/test-omni')
```

Script default endpoints changed in `modelAPI/scripts/compare_v2_vs_rust_pro.py`:

```python
parser.add_argument('--python-endpoint', default='test-python')
parser.add_argument('--omni-endpoint', default='test-omni')
```

This is good because it removes the biggest endpoint-routing ambiguity.

## 3. Root Cause Coverage In PR #2951

| Root Cause | Fixed in PR #2951? | Explanation |
|---|---:|---|
| Root cause 1: Rust input/forecast filtering can zero valid input-derived values | No | PR does not modify `input_handler/mod.rs` or `input_loaders.rs`. |
| Root cause 2: Rust does not fully mirror Python normalized input dataframe path | No | PR does not modify `calculate_input_indicator.py`, `data_inputs.py`, or `rust_bridge.py`. |
| Root cause 3: Rust response extraction fails when no target block dataframe is produced | No | PR does not modify `block_kpi_v4_rust.py`. |
| Root cause 4: Script is not always comparing Python vs Rust | Partially yes | PR adds fixed test endpoints and uses them by default. This fixes the major routing mistake. |
| Root cause 5: Comparator summary and tolerance are too permissive | No | PR keeps integer truncation, `ignore_order=True`, and incomplete JSON summary counting. |

## 4. Evidence From Current JSON Report

Report checked:

```text
/Users/veerpratapsingh/Desktop/correctness_comparison.json
```

Current JSON summary says:

```json
{
  "total_models": 116,
  "total_blocks": 1103,
  "matched_blocks": 1069,
  "failed_blocks": 4
}
```

But grouping blocks by `match_status` shows:

```json
[
  {
    "status": "COMPLETE_MATCH",
    "count": 1063
  },
  {
    "status": "DATA_MATCH_WITH_WARNINGS_DIFF",
    "count": 6
  },
  {
    "status": "DIFF",
    "count": 23
  },
  {
    "status": "ERROR",
    "count": 11
  }
]
```

The real high-level correctness picture is:

```json
{
  "total": 1103,
  "complete_match": 1063,
  "warning_only": 6,
  "value_diff": 23,
  "error": 11,
  "non_complete": 40
}
```

So the current report is misleading:

```text
Current summary says:
  failed_blocks = 4

Actual non-complete blocks:
  40 blocks

Actual incorrect value blocks:
  23 blocks

Actual error-status blocks:
  11 blocks
```

This is the biggest report-quality issue still remaining after PR #2951.

## 5. Example: Rust Zeroed Valid Python Values

Example from the report:

```text
Model: 14712
Block: 36420 - Site Revenue
Status: DIFF
Diff count: 192
```

Report sample:

```json
{
  "model_id": 14712,
  "block_id": 36420,
  "block_name": "Site Revenue",
  "match_status": "DIFF",
  "diff_count": 192,
  "diff_details_summary": "192 value changes in: Average glamping rate per night, Average lodge rate per night, Glamping rental revenue, Lodge rental revenue, Total Site Revenue",
  "first_value_changes": [
    {
      "path": "root[1]['total']",
      "python_value": 5607334,
      "rust_value": 0,
      "indicator": "Lodge rental revenue"
    },
    {
      "path": "root[2]['total_values'][41]['value']",
      "python_value": 36735,
      "rust_value": 0,
      "indicator": "Glamping rental revenue",
      "period": "May-28"
    }
  ]
}
```

Simple explanation:

```text
Python has real revenue values.
Rust returns 0 for the same indicator and period.
That zero then flows into revenue, P&L, cashflow, and balance sheet blocks.
```

This is not fixed by PR #2951 because PR #2951 only changes comparison endpoints and script labels. It does not change Rust input loading or forecast filtering.

## 6. Root Cause 1 Is Not Fixed: Rust Forecast Filtering Can Zero Valid Data

Relevant current Rust path:

```text
modelAPI/omni-calc/src/engine/exec/steps/input_handler/mod.rs
```

Current code pattern:

```rust
// input_handler/mod.rs

if self.should_zero_out_input(ind_spec.id, block_spec) {
    vec![0.0; row_count]
} else {
    self.apply_forecast_start_filter(
        &input_values,
        time_col.as_deref(),
        &dim_columns,
        row_count,
        timings.as_deref_mut(),
    )
}
```

Simple example:

```rust
let raw_input_times = vec!["Apr-25", "Oct-25"];
let forecast_start = "Jul-25";

// Current Rust logic:
let exact_forecast_date_exists = raw_input_times.contains(&forecast_start);
// output: false

let has_pre_forecast_input = raw_input_times.iter().any(|t| t < &forecast_start);
// output: true, because "Apr-25" is before "Jul-25"

let should_zero = !exact_forecast_date_exists && has_pre_forecast_input;
// output: true

let result = vec![0.0; 6];
// output: [0.0, 0.0, 0.0, 0.0, 0.0, 0.0]
```

What is wrong:

```text
The data has valid forecast-side values after the forecast start.
But Rust zeros the entire vector because the exact forecast start period is missing.
```

What Python does differently:

```python
# calculate_input_indicator.py

input_helper = DataInputExtrapolation(input, ind, block)
input_df, actual_df = input_helper.create_extrapolated_data(input_adjustments, target_scenario)

df, index_flag, start_index_of_forecast, time_item_list, date_format = find_forecast_index(
    input_df,
    time_id,
    granularity,
    time_granularity,
    scenario,
)
```

Python first builds a normalized dataframe, sorts time, applies extrapolation, and then slices/joins. Rust currently works from serialized JSON and does a simpler exact-period check.

Needed report improvement:

```text
When Python has non-zero and Rust has zero for the same indicator/period,
the report should flag it as:

  rust_zeroed_nonzero_python = true
  suspected_area = input_forecast_filtering
```

Example improved report row:

```json
{
  "model_id": 14712,
  "block_id": 36420,
  "indicator": "Glamping rental revenue",
  "period": "May-28",
  "python_value": 36735,
  "rust_value": 0,
  "classification": "RUST_ZEROED_VALID_VALUE",
  "suspected_root_cause": "input_forecast_filtering"
}
```

## 7. Root Cause 2 Is Not Fixed: Rust Does Not Fully Mirror Python Input Dataframe Normalization

Relevant Python files:

```text
modelAPI/calc_engine/calculate_input_indicator.py
modelAPI/calc_engine/data_inputs.py
modelAPI/calc_engine/util.py
```

Python path:

```python
input_helper = DataInputExtrapolation(input, ind, block)
input_df, actual_df = input_helper.create_extrapolated_data(input_adjustments, target_scenario)

input_df = input_df.rename({'value': col_name}).filter(pl.col(col_name).is_not_null())
input_df_cols = sorted([col for col in input_df.columns if col != col_name])
```

Rust bridge path:

```python
# rust_bridge.py

block["input_data"].append({
    "indicator_id": ind_data["id"],
    "input_type": getattr(di, "type", "constant"),
    "data_values": data_values_json,
    "dimensions": input_dimensions,
})
```

Simple difference:

```text
Python receives:
  a normalized Polars dataframe with sorted time, extrapolated values, actual split, and join-ready columns.

Rust receives:
  JSON input values and tries to reproduce equivalent behavior inside Rust.
```

Example:

```text
Raw input JSON:
  [
    {"time": "Apr-25", "site": "A", "value": 100},
    {"time": "Oct-25", "site": "A", "value": 200}
  ]

Python normalized dataframe can become:
  time    site  value
  Apr-25  A     100
  May-25  A     100
  Jun-25  A     100
  Jul-25  A     200
  Aug-25  A     200

Rust simplified loading may see:
  exact Jul-25 missing in raw JSON
  before-forecast time exists
  return zeros
```

Needed report improvement:

```text
The comparison report should include enough input-shape context to explain the mismatch:

  input_type
  has_actuals
  forecast_start_date
  raw_input_time_count
  exact_forecast_date_present
  first_python_nonzero_period
  first_rust_zero_period
```

That makes the report useful for debugging, not just pass/fail.

## 8. Root Cause 3 Is Not Fixed: Rust Result Extraction Can Fail

Relevant file:

```text
modelAPI/resources/block_kpi_v4_rust.py
```

Current code:

```python
block_key = f"b{block_id}"
dataframes = calc_result.get("dataframes", {})

if block_key not in dataframes:
    available_keys = list(dataframes.keys())
    raise ValueError(f"Block {block_key} not found in results. Available: {available_keys}")
```

Example from the JSON report:

```json
{
  "model_id": 14754,
  "block_id": 36932,
  "block_name": "NRE (Projects) Revenue",
  "match_status": "ERROR",
  "diff_summary": "v3.2 endpoint failed",
  "v32_error": "Calculation failed: Result extraction failed: Block b36932 not found in results. Available: []"
}
```

Simple explanation:

```text
Python endpoint returned a response.
Rust endpoint did not produce dataframe key b36932.
The Rust API failed before comparison could happen.
```

This is not fixed in PR #2951 because `block_kpi_v4_rust.py` is unchanged.

Needed report improvement:

```json
{
  "classification": "RUST_OUTPUT_MATERIALIZATION_ERROR",
  "python_status": 200,
  "rust_status": 500,
  "missing_dataframe_key": "b36932",
  "available_dataframe_keys": []
}
```

## 9. Root Cause 4 Is Partially Fixed: Endpoint Identity

Before PR #2951:

```python
v2_results.append(self.call_endpoint("v2", block_id, scenario_id))
v32_results.append(self.call_endpoint("v3.2", block_id, scenario_id))
```

Problem:

```text
/outputs/v2 is routed by model running version.
If running_version = 5, /outputs/v2 goes to Rust.
The script labels it Python anyway.
```

PR #2951:

```python
endpoint = self.python_endpoint if engine == "python" else self.omni_endpoint
url = f"{self.base_url}/block/{block_id}/outputs/{endpoint}"
```

This is better because defaults are:

```text
python_endpoint = test-python
omni_endpoint = test-omni
```

Remaining issue:

```python
endpoint = self.python_endpoint if engine == "python" else self.omni_endpoint
```

If there is a typo, every unknown engine becomes Omni:

```python
engine = "pyhton"
endpoint = self.python_endpoint if engine == "python" else self.omni_endpoint
# output: "test-omni"
```

Better:

```python
if engine == "python":
    endpoint = self.python_endpoint
elif engine == "omni":
    endpoint = self.omni_endpoint
else:
    raise ValueError(f"Unknown engine: {engine}")
```

Also the response should record which handler actually ran.

Example improved response metadata:

```json
{
  "engine_requested": "python",
  "endpoint": "/block/36420/outputs/test-python",
  "handler": "BlockKPINew",
  "status_code": 200
}
```

## 10. Root Cause 5 Is Not Fixed: Comparator Is Too Permissive

Current PR code still does this:

```python
def _truncate_floats(obj):
    if isinstance(obj, float):
        if obj != obj or obj == float("inf") or obj == float("-inf"):
            return 0
        return int(round(obj, 2))

diff = DeepDiff(
    self._truncate_floats(data1),
    self._truncate_floats(data2),
    ignore_order=True,
    ignore_numeric_type_changes=True,
)
```

### Issue 5.1: Integer truncation hides real differences

Current behavior:

```python
python_value = 100.49
rust_value = 100.01

current_python = int(round(python_value, 2))
# output: 100

current_rust = int(round(rust_value, 2))
# output: 100

current_python == current_rust
# output: True
```

Why this is wrong:

```text
The engines differ by 0.48.
For finance outputs, that can be real, especially when repeated across many rows.
The current script marks it as equal.
```

Better:

```python
from math import isclose

python_value = 100.49
rust_value = 100.01

isclose(python_value, rust_value, rel_tol=1e-9, abs_tol=1e-6)
# output: False
```

Report improvement:

```json
{
  "python_value": 100.49,
  "rust_value": 100.01,
  "abs_delta": 0.48,
  "rel_delta": 0.004776,
  "within_tolerance": false
}
```

### Issue 5.2: `ignore_order=True` can hide time-series ordering bugs

Current behavior:

```python
python_values = [
    {"name": "Jan-26", "value": 10},
    {"name": "Feb-26", "value": 20}
]

rust_values = [
    {"name": "Feb-26", "value": 20},
    {"name": "Jan-26", "value": 10}
]

DeepDiff(python_values, rust_values, ignore_order=True)
# output: {}
```

Why this is wrong:

```text
The UI and downstream calculations often depend on time-series order.
If Rust returns the same values in the wrong period/order, that is still a correctness issue.
```

Better:

```python
def compare_total_values_by_period(python_values, rust_values):
    for index, (py, rs) in enumerate(zip(python_values, rust_values)):
        if py["name"] != rs["name"]:
            return {
                "type": "ORDER_OR_PERIOD_MISMATCH",
                "index": index,
                "python_period": py["name"],
                "rust_period": rs["name"],
            }
    return None

compare_total_values_by_period(python_values, rust_values)
# output:
# {
#   "type": "ORDER_OR_PERIOD_MISMATCH",
#   "index": 0,
#   "python_period": "Jan-26",
#   "rust_period": "Feb-26"
# }
```

### Issue 5.3: Warning-only differences are counted as matched

Current script:

```python
values_match = (
    match_status in [
        "COMPLETE_MATCH",
        "DATA_MATCH_WITH_WARNINGS_DIFF",
        "METADATA_CHANGES",
    ]
)
```

Current report:

```text
COMPLETE_MATCH: 1063
DATA_MATCH_WITH_WARNINGS_DIFF: 6
matched_blocks summary: 1069
```

Why this is confusing:

```text
Data may match, but warning output changed.
That should be visible as a separate category, not merged into normal matched count.
```

Better summary:

```json
{
  "complete_match": 1063,
  "data_match_warning_diff": 6,
  "metadata_only_diff": 0,
  "value_diff": 23,
  "endpoint_error": 11,
  "total_non_complete": 40
}
```

### Issue 5.4: Failed block count is wrong

Current script:

```python
failed = sum(1 for r in block_results if r.v2_error or r.v32_error)
```

Why this undercounts:

```text
Some blocks have match_status = ERROR but do not have v2_error or v32_error.
Some blocks have value diffs, but failed_blocks ignores them.
The JSON summary says failed_blocks = 4, but match_status shows 11 ERROR blocks.
```

Better:

```python
status_counts = Counter(block.match_status for block in all_blocks)

summary = {
    "complete_match": status_counts["COMPLETE_MATCH"],
    "warning_only": status_counts["DATA_MATCH_WITH_WARNINGS_DIFF"],
    "metadata_only": status_counts["METADATA_CHANGES"],
    "value_diff": status_counts["DIFF"],
    "error": status_counts["ERROR"],
    "non_complete": total_blocks - status_counts["COMPLETE_MATCH"],
}
```

Output:

```json
{
  "complete_match": 1063,
  "warning_only": 6,
  "metadata_only": 0,
  "value_diff": 23,
  "error": 11,
  "non_complete": 40
}
```

### Issue 5.5: JSON export still uses old names and misses status fields

PR #2951 updates Excel headers to say Python and Omni-calc.

But JSON export still uses:

```json
{
  "v2_avg_ms": 123.4,
  "v32_avg_ms": 56.7,
  "v2_response": [],
  "v32_response": []
}
```

It also does not export:

```text
python_status
omni_status
python_endpoint
omni_endpoint
python_label
omni_label
engine_requested
handler_used
```

Better JSON:

```json
{
  "python": {
    "endpoint": "/block/36420/outputs/test-python",
    "status_code": 200,
    "elapsed_ms": 1200.5,
    "handler": "BlockKPINew"
  },
  "omni": {
    "endpoint": "/block/36420/outputs/test-omni",
    "status_code": 200,
    "elapsed_ms": 240.2,
    "handler": "BlockKPINewV4Rust"
  }
}
```

### Issue 5.6: JSON output path is not handled cleanly

PR #2951 keeps `export_to_json()`, but the main flow always calls:

```python
export_to_excel(model_results, output_file)
```

So if someone passes:

```bash
--output /Users/veerpratapsingh/Desktop/results.json
```

the script can still write an Excel workbook to a `.json` filename if `openpyxl` is installed.

Better:

```python
if output_file.endswith(".json"):
    export_to_json(model_results, output_file)
elif output_file.endswith(".xlsx"):
    export_to_excel(model_results, output_file)
else:
    export_to_json(model_results, output_file + ".json")
    export_to_excel(model_results, output_file + ".xlsx")
```

## 11. Concrete Improvements We Should Make In This PR

### Improvement 1: Keep PR endpoint fix, but make engine selection strict

Current PR:

```python
endpoint = self.python_endpoint if engine == "python" else self.omni_endpoint
```

Better:

```python
def resolve_endpoint(self, engine: str) -> str:
    if engine == "python":
        return self.python_endpoint
    if engine == "omni":
        return self.omni_endpoint
    raise ValueError(f"Unknown engine: {engine}")
```

Impact:

```text
Prevents typo-based false comparisons.
Makes the script fail loudly if engine identity is invalid.
```

### Improvement 2: Add engine identity to each result

Current result:

```json
{
  "v2_avg_ms": 120,
  "v32_avg_ms": 50
}
```

Better result:

```json
{
  "python": {
    "endpoint": "test-python",
    "status_code": 200,
    "elapsed_ms": 120,
    "response_kind": "list",
    "handler_expected": "BlockKPINew"
  },
  "omni": {
    "endpoint": "test-omni",
    "status_code": 200,
    "elapsed_ms": 50,
    "response_kind": "list",
    "handler_expected": "BlockKPINewV4Rust"
  }
}
```

Impact:

```text
When someone reads the report later, they can prove which endpoint was called.
This prevents confusion between /outputs/v2 routing and fixed engine testing.
```

### Improvement 3: Replace integer truncation with tolerance-based numeric diff

Current:

```python
return int(round(obj, 2))
```

Better:

```python
from math import isclose

def numeric_diff(py, rs, abs_tol=1e-6, rel_tol=1e-9):
    if isclose(py, rs, abs_tol=abs_tol, rel_tol=rel_tol):
        return None

    return {
        "python_value": py,
        "rust_value": rs,
        "abs_delta": abs(py - rs),
        "rel_delta": abs(py - rs) / max(abs(py), abs(rs), 1.0),
    }
```

Impact:

```text
Small floating noise is ignored.
Real value drift is reported.
No legitimate sub-unit differences are silently deleted.
```

### Improvement 4: Compare response by semantic keys, not full DeepDiff first

Current:

```python
DeepDiff(data1, data2, ignore_order=True)
```

Better:

```python
def index_indicators(response):
    return {item["id"]: item for item in response if isinstance(item, dict) and "id" in item}


def compare_engine_outputs(python_response, rust_response):
    py_by_id = index_indicators(python_response)
    rs_by_id = index_indicators(rust_response)

    missing_in_rust = sorted(set(py_by_id) - set(rs_by_id))
    extra_in_rust = sorted(set(rs_by_id) - set(py_by_id))

    value_diffs = []
    for indicator_id in sorted(set(py_by_id) & set(rs_by_id)):
        value_diffs.extend(compare_indicator(indicator_id, py_by_id[indicator_id], rs_by_id[indicator_id]))

    return {
        "missing_in_rust": missing_in_rust,
        "extra_in_rust": extra_in_rust,
        "value_diffs": value_diffs,
    }
```

Impact:

```text
The report says exactly which indicator is missing or wrong.
It does not depend on array position unless order itself is part of the expected output.
```

### Improvement 5: Compare time-series values by period and index

Better:

```python
def compare_total_values(indicator_id, py_values, rs_values):
    diffs = []

    if len(py_values) != len(rs_values):
        diffs.append({
            "type": "LENGTH_MISMATCH",
            "indicator_id": indicator_id,
            "python_len": len(py_values),
            "rust_len": len(rs_values),
        })

    for index, (py, rs) in enumerate(zip(py_values, rs_values)):
        py_period = py.get("name")
        rs_period = rs.get("name")

        if py_period != rs_period:
            diffs.append({
                "type": "PERIOD_MISMATCH",
                "indicator_id": indicator_id,
                "index": index,
                "python_period": py_period,
                "rust_period": rs_period,
            })
            continue

        diff = numeric_diff(float(py.get("value", 0)), float(rs.get("value", 0)))
        if diff:
            diff.update({
                "type": "VALUE_MISMATCH",
                "indicator_id": indicator_id,
                "period": py_period,
                "index": index,
            })
            diffs.append(diff)

    return diffs
```

Impact:

```text
This catches both wrong numbers and wrong periods.
It also gives the exact row to debug.
```

### Improvement 6: Classify known Rust failure patterns

For zeroing:

```python
def classify_value_diff(py_value, rust_value):
    if py_value not in (0, None) and rust_value == 0:
        return "RUST_ZEROED_VALID_VALUE"
    if py_value == 0 and rust_value not in (0, None):
        return "RUST_ADDED_VALUE"
    return "VALUE_MISMATCH"
```

For missing dataframes:

```python
def classify_error(error_text):
    if "not found in results. Available: []" in error_text:
        return "RUST_OUTPUT_MATERIALIZATION_ERROR"
    if "Calculation failed" in error_text:
        return "CALCULATION_ERROR"
    return "ENDPOINT_ERROR"
```

Impact:

```text
The report becomes actionable.
Instead of only saying DIFF, it says which failure family is most likely.
```

### Improvement 7: Fix summary counts

Better summary schema:

```json
{
  "total_models": 116,
  "total_blocks": 1103,
  "complete_match": 1063,
  "data_match_warning_diff": 6,
  "metadata_only_diff": 0,
  "value_diff": 23,
  "python_error": 0,
  "omni_error": 3,
  "both_error": 8,
  "empty_or_non_meaningful_response": 7,
  "total_non_complete": 40,
  "correctness_pass_rate": "96.37%"
}
```

Impact:

```text
Company/team discussion becomes much clearer.
We can say exactly how many blocks are true value mismatches versus endpoint failures.
```

### Improvement 8: Always write machine-readable JSON

Recommended output behavior:

```text
Always write:
  correctness_comparison.json

Optionally also write:
  correctness_comparison.xlsx
```

Why:

```text
Excel is good for humans.
JSON is required for CI, historical baselines, trend tracking, and automated regression checks.
```

### Improvement 9: Add a correctness baseline file

Example:

```json
{
  "baseline_name": "omni-calc-python-rust-correctness",
  "created_from_branch": "BLOX-2143-add-omni-calc-runtime-performance-tracing-and-benchmark-baseline",
  "models": [14712, 14754, 14789, 14870, 14879, 14907],
  "known_failures": [
    {
      "model_id": 14712,
      "block_id": 36420,
      "classification": "RUST_ZEROED_VALID_VALUE"
    },
    {
      "model_id": 14754,
      "block_id": 36932,
      "classification": "RUST_OUTPUT_MATERIALIZATION_ERROR"
    }
  ]
}
```

Impact:

```text
The team can separate known failures from new regressions.
This is important for long-running Rust correctness migration work.
```

## 12. Manual Test Commands For PR #2951

Start API locally on the PR branch, then run:

```bash
cd /Users/veerpratapsingh/Desktop/blox/Blox-Dev/modelAPI
source .venv/bin/activate
export API_TOKEN='paste-token-here'

python scripts/compare_v2_vs_rust_pro.py \
  --base-url http://127.0.0.1:5000 \
  --block-ids 36420,36932,37089,37968,38089 \
  --python-endpoint test-python \
  --omni-endpoint test-omni \
  --python-label python \
  --omni-label omni-calc \
  --include-indicator-summary \
  --output /Users/veerpratapsingh/Desktop/pr2951-smoke.xlsx
```

Also test endpoints directly:

```bash
curl -sS -X POST "http://127.0.0.1:5000/block/36420/outputs/test-python" \
  -H "Authorization: $API_TOKEN"

curl -sS -X POST "http://127.0.0.1:5000/block/36420/outputs/test-omni" \
  -H "Authorization: $API_TOKEN"
```

Expected:

```text
test-python should always call BlockKPINew.
test-omni should always call BlockKPINewV4Rust.
For affected blocks, the report should show Rust value diffs or Rust extraction errors.
```

## 13. PR Hygiene Issues

PR #2951 includes generated/binary artifacts:

```text
modelAPI/omni-calc/python/omni_calc/omni_calc.cpython-311-darwin.so
modelAPI/omni-calc/target/.rustc_info.json
```

These should not be part of this script/report PR unless the team intentionally wants to commit compiled local artifacts.

Why this matters:

```text
They are machine-specific.
They make the PR harder to review.
They can cause merge noise unrelated to correctness comparison.
```

## 14. Final Recommendation

PR #2951 should be treated as a base implementation only.

It correctly starts fixing the endpoint-routing problem by adding:

```text
/outputs/test-python
/outputs/test-omni
```

But to fully implement Rust vs Python correctness comparison, we should still add:

```text
1. Strict engine endpoint selection.
2. Engine identity in every result row.
3. Tolerance-based numeric comparison instead of integer truncation.
4. Period-aware time-series comparison instead of global ignore_order=True.
5. Semantic indicator-by-id comparison.
6. Separate summary counts for complete, warning-only, metadata-only, value-diff, and error.
7. Error classification for Rust zeroing and missing dataframe output.
8. Always-on JSON report plus optional Excel report.
9. Correct JSON field names: python_* and omni_* instead of v2/v32.
10. Regression baseline support for known failing models/blocks.
```

Most important point for team discussion:

```text
PR #2951 fixes "are we calling the right endpoints?"
It does not yet fix "are we comparing the outputs accurately and reporting failures honestly?"
```

