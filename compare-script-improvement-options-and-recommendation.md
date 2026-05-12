# Compare Script Improvement Options And Recommendation

## 1. Goal

The current compare script is trying to answer two questions:

1. Does Rust omni-calc return the same output as the Python calculation path?
2. If Rust is faster, is that speedup meaningful and safe to trust?

Right now, the script is useful for broad comparison, but it is not yet strong enough for fast debugging. It can tell us that a block mismatched, but it often does not clearly explain:

- the exact business value that mismatched
- whether Rust or Python is likely wrong
- which code path probably caused the mismatch
- whether the issue is an engine bug, compare-script bug, normalization issue, or reporting issue

The improvement should make the script more actionable without turning it into a full redesign.

## 2. Current Script Limitations

### 2.1 What the current script does well

The script already provides useful coverage:

- Runs Python and Rust endpoints for many models and blocks.
- Captures endpoint success/error status.
- Captures rough API timing for both sides.
- Produces JSON/Excel-style report output.
- Uses DeepDiff to detect response differences.
- Can show whether a block is a complete match, warning diff, value diff, or error.

This is a good base. We should build on it.

### 2.2 What it does not explain clearly enough

The current output is still too low-level and path-based.

Example current style:

```json
{
  "values_changed": {
    "root[2]['total_values'][4]['value']": {
      "old_value": 300000,
      "new_value": 0
    }
  }
}
```

This proves a mismatch exists, but it does not immediately tell the developer:

- which indicator this is
- which time period this is
- which model/block is affected
- whether the Rust value is probably wrong
- what code path probably caused it

### 2.3 Why current mismatch output is not sufficient for fast debugging

The current mismatch output forces engineers to manually inspect raw responses and code paths.

That is slow because a developer has to:

1. Open the JSON report.
2. Decode DeepDiff paths.
3. Find the indicator and period manually.
4. Compare the old branch and newer branch behavior manually.
5. Guess whether the mismatch came from preload, filter logic, formula parsing, input handling, or report normalization.

The script should do more of that work automatically.

## 3. Option 1 Analysis: Benchmarking-Branch-Based Improvement

Option 1 means extending the compare script using the runtime tracing/logging ideas already added in:

```text
BLOX-2143-add-omni-calc-runtime-performance-tracing-and-benchmark-baseline
```

### 3.1 What runtime logs/details are available

The benchmarking branch adds a Rust runtime timing structure with detailed execution phases.

Important timing fields include:

- `total_runtime_ms`
- `preload_runtime_ms`
- `metadata_needs_ms`
- `preload_bulk_ms`
- `preload_fallback_ms`
- `dimension_extract_ms`
- `property_extract_ms`
- `serial_calc_steps_ms`
- `connected_dimension_preload_ms`
- `input_step_ms`
- `property_step_ms`
- `calculation_step_ms`
- `sequential_step_ms`
- `calc_dependency_resolution_ms`
- `cross_object_resolution_ms`
- `dimension_property_collection_ms`
- `connected_dim_collection_ms`
- `connected_dim_property_collection_ms`
- `formula_setup_ms`
- `formula_eval_ms`
- `property_filter_context_ms`
- `actuals_handling_ms`
- `actuals_row_key_ms`
- `forecast_membership_ms`
- `recordbatch_materialization_ms`
- `final_result_build_ms`

It also adds counters like:

- number column clone count
- string column clone count
- RecordBatch clone count
- formula context clone count
- warning clone count
- estimated clone bytes

The reporting docs describe runtime run folders like:

```text
modelAPI/omni-calc/perf-runs/<run-name>/
  manifest.json
  runtime_timings.json
  runtime_metrics.csv
  runtime_report.md
  raw/
    logs/
```

And comparison output like:

```text
modelAPI/omni-calc/perf-comparisons/<baseline>-vs-<candidate>/
  comparison_summary.md
  runtime_comparison.csv
  hotspots_comparison.csv
  regressions.json
```

### 3.2 How these logs could be used in the compare script

The compare script could attach runtime timing to each Python-vs-Rust block comparison.

For example, each block result could include:

```json
{
  "model_id": 12966,
  "block_id": 25393,
  "match_status": "DIFF",
  "python_api_ms": 820.4,
  "rust_api_ms": 230.1,
  "rust_runtime_phases": {
    "preload_runtime_ms": 80.5,
    "property_filter_context_ms": 12.3,
    "formula_eval_ms": 70.2,
    "final_result_build_ms": 20.1
  }
}
```

The script could also compare stage-level timing between two Rust branches:

```text
older Rust preload: n/a
newer Rust preload: 80.5 ms
formula_eval: -20%
property_filter_context: +60%
```

### 3.3 Whether the logs are enough to improve correctness investigation

They help, but they are not enough by themselves.

Runtime timings can answer:

- where Rust spent time
- whether preload got slower
- whether formula evaluation got faster
- whether connected dimension preload ran
- whether actuals/filter code paths were hit

But runtime timings usually cannot answer:

- which exact value became wrong
- which exact indicator/period mismatched
- whether `matching_item_names` were interpreted incorrectly
- whether a missing property was null, missing, or unloaded
- whether a zero came from a legitimate filter result or an error fallback

So Option 1 improves debugging context, but not the core correctness diagnosis.

### 3.4 Does this help only timing or also correctness?

Mostly timing. Some correctness debugging improves indirectly.

Useful indirect correctness clues:

- High `preload_fallback_ms` may suggest metadata was not preloaded correctly.
- Non-zero `connected_dimension_preload_ms` confirms connected dimension path was involved.
- High `actuals_handling_ms` suggests actuals filter path is active.
- Runtime phase changes between branches can point to a newly active path.

But the logs do not directly classify the mismatch cause.

### 3.5 Compare-script changes needed for Option 1

To use benchmarking-branch runtime data effectively, the compare script would need to:

1. Enable runtime tracing flags when running Rust:

```text
OMNI_CALC_PERF_TRACE=1
OMNI_CALC_PERF_COUNTERS=1
OMNI_CALC_TRACE_OUTPUT=json
```

2. Capture Rust timing payload per block.
3. Store timing fields in block result objects.
4. Add phase-level sections to JSON/Markdown/Excel output.
5. Separate API latency from Rust engine runtime.
6. Report speedup in at least two forms:
   - API latency speedup
   - Rust engine runtime stage breakdown
7. Preserve current end-to-end timing so existing speedup analysis is not lost.

### 3.6 Pros

- Builds on work already added in the benchmarking branch.
- Improves performance reporting accuracy.
- Helps identify runtime hotspots.
- Separates total HTTP latency from Rust engine time.
- Useful for performance regression work.
- Helps explain why a branch got slower or faster.

### 3.7 Cons

- Does not directly explain value mismatches.
- Adds more output volume.
- Requires runtime tracing support to be available in the running backend.
- Can be misleading if used as the primary correctness tool.
- Some timing data is Rust-only unless Python has equivalent stage timings.

### 3.8 Implementation complexity

Medium.

Most pieces already exist in the benchmark branch, but the compare script must be changed to collect, store, and display the runtime phase data.

The biggest risk is output complexity: adding timing sections without improving mismatch cause could make the report longer but not much easier to debug.

### 3.9 How much it helps correctness debugging

Low to medium.

It helps narrow down which Rust stage was active or expensive, but it does not directly tell why a specific value is wrong.

## 4. Option 2 Analysis: Direct Mismatch-Cause Reporting Improvement

Option 2 means improving the compare script so the output itself explains mismatch causes in a useful level of detail.

The goal is for the script to say:

```text
This exact indicator/period mismatched.
Rust is likely wrong.
This looks like a dimension-type property filter regression.
Likely code path: filter_utils.rs.
```

### 4.1 How this can be implemented on top of the current script

This can be built incrementally on the current script.

The key change is to stop using raw DeepDiff paths as the final debugging output.

Implementation steps:

1. Keep the existing endpoint calling flow.
2. Normalize and flatten Python/Rust responses into business rows.
3. Compare rows by stable keys.
4. Create structured mismatch records.
5. Add mismatch classification rules.
6. Add branch-comparison context when available.
7. Add suspected code-path fields.

### 4.2 What extra fields should be added to output

Each mismatch should include fields like:

```json
{
  "model_id": 12966,
  "block_id": 25393,
  "block_name": "Cashflow",
  "indicator_id": 126150,
  "indicator_name": "Loans in",
  "period": "May-24",
  "dimension_path": null,
  "python_value": 300000,
  "rust_value": 0,
  "delta": -300000,
  "delta_pct": -100.0,
  "mismatch_type": "value_mismatch",
  "likely_wrong_side": "rust",
  "confidence": "high",
  "likely_cause": "dimension_type_property_filter_semantics_changed",
  "suspected_code_path": "modelAPI/omni-calc/src/engine/exec/filter_utils.rs",
  "branch_pattern": "matched_or_not_diff_in_older_branch_but_diff_in_newer_branch"
}
```

### 4.3 How the script should classify mismatch cause

The script should classify using simple, explainable rules first.

Example rules:

#### Rule A: Newer-only diff

If a block matched or was not a value diff in the older branch, but is a value diff in the newer branch:

```text
classification: newer_rust_regression
likely_wrong_side: rust
confidence: high or medium
```

#### Rule B: Rust zero while Python non-zero

If Python has non-zero and Rust has zero for many rows:

```text
classification: rust_zeroing_or_filter_drop
likely causes:
- forecast_start_zeroing
- actuals_length_mismatch_zeroing
- missing_filter_metadata_zeroing
- preload_missing_metadata
```

#### Rule C: Source item name vs connected linked value pattern

If affected block/model is one of the known newer-only financing/cashflow examples:

```text
classification: dimension_type_property_filter_semantics_changed
suspected_code_path: filter_utils.rs
```

#### Rule D: Missing Rust row

If Python has a canonical row and Rust does not:

```text
classification: missing_rust_output_row
likely_wrong_side: rust
```

#### Rule E: Missing requested block dataframe

If Rust endpoint errors with `Block b{id} not found in results`:

```text
classification: rust_materialization_bug
suspected_code_path: block_kpi_v4_rust.py / Rust result assembly
```

#### Rule F: Both branches diff

If both older and newer branches differ from Python:

```text
classification: shared_rust_engine_bug
confidence: medium
```

### 4.4 Pros

- Directly improves correctness debugging.
- Gives developers the exact row/value that failed.
- Helps decide whether Rust or Python is likely wrong.
- Makes the report actionable without manually opening raw responses.
- Can be implemented incrementally.
- Does not require a full benchmark infrastructure merge.
- Helps reviewers understand if a mismatch is script-only, reporting-only, or engine-side.

### 4.5 Cons

- Cause classification can be wrong if rules are too aggressive.
- Needs careful confidence labels.
- Requires stable flattening logic per response shape.
- Some mismatches will still be `unknown`.
- Needs historical/branch comparison data to classify regressions well.

### 4.6 Implementation complexity

Medium.

The hardest part is output flattening and stable row-key generation. But this is still a focused script improvement, not a full engine redesign.

### 4.7 How much it helps correctness debugging

High.

This is the most useful improvement for the current problem because the biggest pain is not only detecting mismatches. The biggest pain is understanding why the mismatch happened.

## 5. Comparison Of Both Options

| Area | Option 1: Runtime Tracing | Option 2: Direct Mismatch-Cause Reporting |
|---|---|---|
| Main value | Performance stage visibility | Correctness debugging clarity |
| Best for | Timing, hotspots, regressions in runtime phases | Finding exact wrong values and likely causes |
| Helps identify exact mismatched value | No | Yes |
| Helps identify likely wrong side | Indirectly | Yes |
| Helps explain preload/filter bug | Somewhat | Yes, if classification rules are added |
| Implementation complexity | Medium | Medium |
| Requires benchmark branch runtime support | Yes | No |
| Builds on current compare script | Yes | Yes |
| Risk | More output without clear cause | Misclassification if confidence is not handled |
| Fastest debugging value | Medium | High |
| Best long-term value | High for performance work | High for correctness work |

### 5.1 Which gives faster debugging value?

Option 2 gives faster debugging value.

A developer needs to know:

```text
What value is wrong?
Where is it wrong?
Which side is likely wrong?
What code path should I inspect?
```

Option 2 answers those directly.

### 5.2 Which gives more accurate correctness diagnosis?

Option 2 is better for correctness diagnosis.

Option 1 can say that preload or formula evaluation took time, but it does not prove that preload caused a specific zero or mismatch.

### 5.3 Which is easier to implement on top of the current script?

Both are medium complexity.

Option 2 is more directly connected to the existing compare logic. It mainly changes normalization, comparison, and report output.

Option 1 requires runtime tracing data from the running Rust engine and more environment coordination.

### 5.4 Do they complement each other?

Yes.

The best final design combines both:

- Option 2 explains correctness mismatches.
- Option 1 adds runtime-stage context for performance and stage-level clues.

But they should not be implemented at the same priority.

## 6. Recommended Design

Recommendation:

```text
Implement Option 2 first, then add Option 1 as stage-level enrichment.
```

Reason:

The current urgent problem is correctness diagnosis, not only runtime benchmarking. The report already shows many `DIFF` blocks, but the output does not make it easy enough to find the cause.

Option 2 should be the first improvement because it turns the compare script into a practical debugging tool.

Option 1 should come second because it is still valuable, especially after mismatch rows are structured. Runtime phases are much more useful when attached to exact mismatch records.

## 7. Detailed Implementation Idea

### 7.1 Keep current endpoint flow

Do not rewrite the whole script.

Keep:

- model/block discovery
- Python endpoint call
- Rust endpoint call
- timing capture
- JSON/Excel export
- existing CLI options

### 7.2 Add flattened output comparison

Add a function that turns each response into keyed rows.

Example target shape:

```python
{
    (indicator_id, period, dimension_path): {
        "indicator_id": indicator_id,
        "indicator_name": indicator_name,
        "period": period,
        "dimension_path": dimension_path,
        "value": value,
    }
}
```

Then compare:

```python
all_keys = set(python_rows) | set(rust_rows)

for key in all_keys:
    py = python_rows.get(key)
    rs = rust_rows.get(key)

    if py is None:
        report_extra_rust_row(key, rs)
    elif rs is None:
        report_missing_rust_row(key, py)
    elif not values_close(py["value"], rs["value"]):
        report_value_mismatch(key, py, rs)
```

### 7.3 Add real numeric tolerance

Replace float truncation with explicit tolerance:

```python
math.isclose(py_value, rust_value, abs_tol=1e-6, rel_tol=1e-9)
```

Do not turn `NaN` or `inf` into zero.

### 7.4 Add mismatch cause classification

Add a classification layer that receives:

- mismatch row
- block metadata
- old/new branch comparison context, if available
- endpoint error text, if any
- known pattern rules

Example output:

```json
{
  "mismatch_type": "value_mismatch",
  "likely_cause": "dimension_type_property_filter_semantics_changed",
  "likely_wrong_side": "rust",
  "confidence": "high",
  "suspected_code_path": "modelAPI/omni-calc/src/engine/exec/filter_utils.rs"
}
```

### 7.5 Add better summary counts

Report:

- `complete_match_blocks`
- `warning_diff_blocks`
- `value_diff_blocks`
- `error_blocks`
- `script_classification_warnings`
- `newer_only_regressions`
- `shared_rust_mismatches`
- `unknown_mismatches`

### 7.6 Preserve current speedup/runtime comparison behavior

Keep existing timing fields, but rename them accurately:

```json
{
  "python_api_avg_ms": 1000,
  "rust_api_avg_ms": 400,
  "api_latency_speedup": 2.5,
  "speedup_valid_for_correctness": false
}
```

For mismatched blocks, still show timing, but mark it as not counted in correctness-passed speedup.

### 7.7 Add Option 1 later as enrichment

After Option 2 exists, add optional runtime tracing fields:

```json
{
  "rust_runtime_phases": {
    "preload_runtime_ms": 80.5,
    "formula_eval_ms": 70.2,
    "property_filter_context_ms": 12.3
  },
  "runtime_clue": "preload path active; check preload metadata coverage"
}
```

This gives the best combined design:

- structured mismatch explains what failed
- runtime phases explain where Rust spent time
- classification links both to likely code paths

## 8. Example Improved Compare-Script Output

### 8.1 Current output: mismatch detected but no useful cause

```json
{
  "model_id": 12966,
  "block_id": 25393,
  "match_status": "DIFF",
  "diff_summary": "values_changed: 2",
  "diff_details": {
    "values_changed": {
      "root[2]['total_values'][4]['value']": {
        "old_value": 300000,
        "new_value": 0
      }
    }
  }
}
```

Problem:

This does not tell the developer what indicator, period, or code path is involved.

### 8.2 Improved output: likely root cause shown

```json
{
  "model_id": 12966,
  "block_id": 25393,
  "match_status": "DIFF",
  "mismatches": [
    {
      "indicator_id": 126150,
      "indicator_name": "Loans in",
      "period": "May-24",
      "python_value": 300000,
      "rust_value": 0,
      "delta": -300000,
      "mismatch_type": "value_mismatch",
      "likely_wrong_side": "rust",
      "confidence": "high",
      "likely_cause": "dimension_type_property_filter_semantics_changed",
      "suspected_code_path": "modelAPI/omni-calc/src/engine/exec/filter_utils.rs",
      "explanation": "This block is a newer-only diff and matches the pattern where source dimension item names are compared against linked dimension values."
    }
  ]
}
```

Why this is better:

The developer can go directly to the affected indicator and code path.

### 8.3 Improved output: runtime-stage comparison if Option 1 is used

```json
{
  "model_id": 12966,
  "block_id": 25393,
  "match_status": "DIFF",
  "python_api_avg_ms": 820.4,
  "rust_api_avg_ms": 230.1,
  "api_latency_speedup": 3.56,
  "speedup_valid_for_correctness": false,
  "rust_runtime_phases": {
    "total_runtime_ms": 180.2,
    "preload_runtime_ms": 70.5,
    "metadata_needs_ms": 10.2,
    "preload_bulk_ms": 44.8,
    "property_filter_context_ms": 8.1,
    "formula_eval_ms": 50.4,
    "final_result_build_ms": 12.7
  },
  "runtime_note": "Rust is faster at API level, but this speedup is excluded from correctness-passed speedup because output values differ."
}
```

Why this is useful:

It preserves performance information while preventing people from treating a faster wrong answer as a valid win.

## 9. Final Recommendation

Implement both, but in this order:

1. Option 2 first: direct mismatch-cause reporting.
2. Option 1 second: runtime tracing enrichment from the benchmarking branch.

This gives the best practical workflow:

```text
First: make correctness mismatches clear and actionable.
Then: attach runtime phase data to explain performance and stage-level behavior.
```

This approach builds on the current branch flow and avoids a redesign. It makes the compare script a better engineering tool while preserving the existing speedup/runtime comparison behavior.
