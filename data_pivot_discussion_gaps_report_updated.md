# Data Pivot Refactor Discussion

Inspection basis: fetched `origin/development` and inspected commit `34be524947d7056e1d976a991d4edb2847fa9411`. All GitHub links in this report point to the `development` branch only.

Important baseline note: `modelAPI/calc_engine/data_pivot_resolution.py` is not present in `development`. If that file exists locally, it is recent local work and is not treated as current baseline code in this report.

Story: As a backend team, we want all data display resolution logic to live in the data pivot class so that grouping, filtering, and expansion are handled in one place and output APIs are no longer responsible for business logic they should not own.

## 1. Issue Summary

The backend currently has two related ownership problems:

1. Some logic is true `DataPivot` logic but still lives outside the pivot classes. This includes dataframe filtering, pivot output extraction, time expansion, and post-pivot shaping.
2. Some logic is `DataPivot` resolution logic. This is the code that converts request/model metadata into `DataPivot`-ready inputs, such as normalized filters, selected dimensions, connected dimensions, rows, columns, total queries, and export query variants.

Both kinds of logic are currently spread across resource files, utility files, calc controllers, export code, Rust proxy setup, and dashboard consumers. That makes plan-page grouping/filtering and connected-dimension behavior hard to keep consistent.

The goal is not to move every helper into `data_pivot.py`. The goal is to make `DataPivot` the public owner of display resolution behavior. Internal shared implementation can live in a new helper file, but resource/output files should call `DataPivot`, not call lower-level helper classes directly.

## 2. Target Architecture

The desired ownership model is:

- API/resource files validate access, parse request transport shape, load model/block/scenario context, call calc, call `DataPivot`, and return response JSON or files.
- `DataPivot` owns display behavior: filter handling, query interpretation, grouping, aggregation, time display, expansion, and pivoted-data extraction.
- `DataPivot` also exposes public class/static methods for resolution steps needed before construction, for example `DataPivot.normalize_plan_filters(...)`, `DataPivot.resolve_output_dimensions(...)`, and `DataPivot.build_output_queries(...)`.
- A new file such as `modelAPI/calc_engine/data_pivot_resolution.py` can hold shared resolution helpers, DTOs, adapters, and mixin methods used by all three pivot classes.
- Output APIs should not import or call `DataPivotResolutionMixin` directly. They should call `DataPivot`.
- Export code should keep Excel/file/presentation concerns, but data filtering and query mutation should become pivot-owned.
- `blox-bi` consumer reshaping should be a follow-up unless the story explicitly includes consumer-service changes.

### Why A New `data_pivot_resolution.py` File Is Needed

`development` currently has three pivot implementations:

- `modelAPI/calc_engine/data_pivot.py`
- `modelAPI/calc_engine/data_pivot_v3.py`
- `modelAPI/calc_engine/data_pivot_v4.py`

Putting new shared methods separately avoids copying the same methods into all three classes. A new `data_pivot_resolution.py` file would be an internal implementation module that can be mixed into every `DataPivot` version.

The public call should still be:

```python
DataPivot.normalize_plan_filters(...)
DataPivot.resolve_output_dimensions(...)
DataPivot.build_output_queries(...)
```

The public call should not be:

```python
DataPivotResolutionMixin.normalize_plan_filters(...)
```

### How `data_pivot_resolution.py` Differs From `data_pivot.py`, `data_pivot_v3.py`, And `data_pivot_v4.py`

The existing `data_pivot*.py` files are pivot execution classes. They take raw dataframes, dimensions, indicators, `query_params`, and fiscal-year settings, then parse query params, apply filters, aggregate, pivot, and format output.

Current examples:

- `modelAPI/calc_engine/data_pivot.py` starts `DataPivot` and parses query params during construction: [development lines 13-38](https://github.com/BloxSoftware/Blox-Dev/blob/development/modelAPI/calc_engine/data_pivot.py#L13-L38).
- `modelAPI/calc_engine/data_pivot_v3.py` defines the v3 pivot implementation: [development lines 18-46](https://github.com/BloxSoftware/Blox-Dev/blob/development/modelAPI/calc_engine/data_pivot_v3.py#L18-L46).
- `modelAPI/calc_engine/data_pivot_v4.py` defines the v4 pivot implementation: [development lines 17-42](https://github.com/BloxSoftware/Blox-Dev/blob/development/modelAPI/calc_engine/data_pivot_v4.py#L17-L42).

The proposed `data_pivot_resolution.py` should not replace those classes. It should hold shared pre-pivot and post-pivot resolution helpers that every `DataPivot` version can expose. Examples:

- Normalize plan-page filters into the filter list consumed by `DataPivot`.
- Resolve selected output dimensions and connected dimensions.
- Build standard output query params for main and total pivots.
- Build model-data total query params.
- Help with time expansion and export result filtering when those behaviors must be shared.
- Provide small adapters for dict-backed, SQLAlchemy-backed, and proxy-backed dimensions.

In short:

- `data_pivot*.py`: executes the pivot.
- `data_pivot_resolution.py`: shares DataPivot-owned resolution helpers across pivot versions.
- Resource files: call `DataPivot` and stay thin.

## 3. Gap-By-Gap Analysis

### Gap 1: Multiple Pivot Classes Can Drift

Label: `DataPivot logic`

Problem summary: There are three `DataPivot` classes in development. Each owns its own execution path for parsing query params, applying filters, aggregating, and formatting output.

Current files and lines:

- `modelAPI/calc_engine/data_pivot.py`: [`DataPivot` setup and query parsing entry](https://github.com/BloxSoftware/Blox-Dev/blob/development/modelAPI/calc_engine/data_pivot.py#L13-L38).
- `modelAPI/calc_engine/data_pivot_v3.py`: [`DataPivot` v3 setup](https://github.com/BloxSoftware/Blox-Dev/blob/development/modelAPI/calc_engine/data_pivot_v3.py#L18-L46).
- `modelAPI/calc_engine/data_pivot_v4.py`: [`DataPivot` v4 setup](https://github.com/BloxSoftware/Blox-Dev/blob/development/modelAPI/calc_engine/data_pivot_v4.py#L17-L42).

What the current code is doing: Each class constructs a pivot from raw calc output and `query_params`.

Why it is a gap: Any shared display behavior added directly to one pivot class can be missed in the others. That weakens the single-source-of-truth goal.

What should change: Do not merge all three implementations immediately. First, add a shared internal helper/mixin and expose it through every `DataPivot` class.

Where the responsibility should go: Shared behavior should live in a helper/resolution module used by `DataPivot`, while actual pivot execution remains in the existing `data_pivot*.py` files.

Why this is better: It gives all output versions the same public resolution API without forcing a risky full pivot rewrite.

### Gap 2: Existing Pivot Classes Already Own Some Runtime Query Parsing, But The Inputs Are Built Elsewhere

Label: `DataPivot logic`

Problem summary: `DataPivot` already parses rows, columns, filters, display levels, group-by values, and connected-dimension maps inside the pivot classes. But output APIs still build those query params before calling pivot.

Current files and lines:

- `modelAPI/calc_engine/data_pivot.py`: [`__parse_query_params` starts](https://github.com/BloxSoftware/Blox-Dev/blob/development/modelAPI/calc_engine/data_pivot.py#L106-L215), and [`__parse_axis_params` reads `display_levels`, `filter`, `group_by`, and `sort`](https://github.com/BloxSoftware/Blox-Dev/blob/development/modelAPI/calc_engine/data_pivot.py#L216-L263).
- `modelAPI/calc_engine/data_pivot_v3.py`: [`__parse_query_params` starts](https://github.com/BloxSoftware/Blox-Dev/blob/development/modelAPI/calc_engine/data_pivot_v3.py#L118-L203), and [`__parse_axis_params` reads display/grouping fields](https://github.com/BloxSoftware/Blox-Dev/blob/development/modelAPI/calc_engine/data_pivot_v3.py#L216-L263).
- `modelAPI/calc_engine/data_pivot_v4.py`: [`__parse_query_params` starts](https://github.com/BloxSoftware/Blox-Dev/blob/development/modelAPI/calc_engine/data_pivot_v4.py#L110-L199), and [`__parse_axis_params` reads display/grouping fields](https://github.com/BloxSoftware/Blox-Dev/blob/development/modelAPI/calc_engine/data_pivot_v4.py#L212-L258).

What the current code is doing: Runtime pivot logic is already inside `DataPivot`, but it assumes the API has already produced correct `query_params`.

Why it is a gap: The pivot layer is only partially authoritative. It executes query params, but output APIs still decide how those query params should be created.

What should change: Keep low-level runtime parsing in `DataPivot`. Move query construction into `DataPivot` public helpers so the same owner creates and executes the query shape.

Where the responsibility should go: Actual parsing and filtering stay directly in the pivot classes. Request-to-query conversion should live in a shared resolution helper exposed through `DataPivot`.

Why this is better: It closes the ownership gap between "who builds the query" and "who executes the query."

### Gap 3: Plan-Page Filter Translation Is Outside Pivot

Label: `DataPivot resolution logic`

Problem summary: Plan-page params such as `dim_filter` and `time_filter` are converted into `DataPivot` filter objects in `util.query_params_helpers`.

Current files and lines:

- `modelAPI/util/query_params_helpers.py`: [`build_filters_from_plan_params`](https://github.com/BloxSoftware/Blox-Dev/blob/development/modelAPI/util/query_params_helpers.py#L43-L80) parses JSON/list filters and appends `{"Time": time_filter}`.
- `modelAPI/resources/new_block_kpi.py`: [v2 output calls this helper](https://github.com/BloxSoftware/Blox-Dev/blob/development/modelAPI/resources/new_block_kpi.py#L239-L241).
- `modelAPI/resources/block_kpi.py`: [legacy output calls this helper](https://github.com/BloxSoftware/Blox-Dev/blob/development/modelAPI/resources/block_kpi.py#L929-L931).
- `modelAPI/resources/block_kpi_v3.py`: [v3 output calls this helper](https://github.com/BloxSoftware/Blox-Dev/blob/development/modelAPI/resources/block_kpi_v3.py#L336-L338).
- `modelAPI/resources/block_kpi_v4.py`: [v4 output calls this helper](https://github.com/BloxSoftware/Blox-Dev/blob/development/modelAPI/resources/block_kpi_v4.py#L379-L381).
- `modelAPI/resources/block_kpi_v4_rust.py`: [Rust output calls this helper](https://github.com/BloxSoftware/Blox-Dev/blob/development/modelAPI/resources/block_kpi_v4_rust.py#L515-L517).
- `modelAPI/resources/scenarios.py`: [scenario comparison repeats filter parsing inline](https://github.com/BloxSoftware/Blox-Dev/blob/development/modelAPI/resources/scenarios.py#L361-L364).

What the current code is doing: It converts plan-page filters into the list format that `DataPivot` consumes.

Why it is a gap: Output code owns the filter-normalization rules, even though the result is not a generic API shape. It is a pivot-specific filter shape.

What should change: Add `DataPivot.normalize_plan_filters(dim_filter, time_filter)` and update callers to use it. Remove `build_filters_from_plan_params` if no external compatibility is needed; otherwise keep a temporary wrapper that delegates to `DataPivot`.

Where the responsibility should go: New shared resolution helper used by `DataPivot`, publicly exposed as `DataPivot.normalize_plan_filters(...)`.

Why this is better: Plan-page filter handling becomes consistent across legacy, v2, v3, v4, Rust, and scenario comparison paths.

### Gap 4: Output APIs Resolve Direct And Connected Dimensions Before Pivot

Label: `DataPivot resolution logic`

Problem summary: Output handlers parse `dim_id`, look up requested dimensions, inspect block dimensions, parse dimension-property `data_format`, and mutate `group_property_id` before calling `DataPivot`.

Current files and lines:

- `modelAPI/resources/new_block_kpi.py`: [v2 output resolves direct and connected dimensions](https://github.com/BloxSoftware/Blox-Dev/blob/development/modelAPI/resources/new_block_kpi.py#L52-L89).
- `modelAPI/resources/block_kpi.py`: [legacy output resolves direct and connected dimensions](https://github.com/BloxSoftware/Blox-Dev/blob/development/modelAPI/resources/block_kpi.py#L616-L650).
- `modelAPI/resources/block_kpi_v3.py`: [v3 output performs cache-backed connected-dimension resolution](https://github.com/BloxSoftware/Blox-Dev/blob/development/modelAPI/resources/block_kpi_v3.py#L95-L149).
- `modelAPI/resources/block_kpi_v4.py`: [v4 output performs proxy-backed connected-dimension resolution](https://github.com/BloxSoftware/Blox-Dev/blob/development/modelAPI/resources/block_kpi_v4.py#L115-L158).
- `modelAPI/resources/block_kpi_v4_rust.py`: [Rust output repeats v4-style connected-dimension resolution](https://github.com/BloxSoftware/Blox-Dev/blob/development/modelAPI/resources/block_kpi_v4_rust.py#L125-L176).
- `modelAPI/resources/scenarios.py`: [scenario comparison repeats connected-dimension lookup](https://github.com/BloxSoftware/Blox-Dev/blob/development/modelAPI/resources/scenarios.py#L267-L299).

What the current code is doing: It decides whether a requested dimension is already on the block or is connected through a dimension-type property. It then prepares a dimension object for pivot rows.

Why it is a gap: Connected-dimension lookup is part of pivot display resolution. The output layer should not know how to inspect dimension properties or mutate `group_property_id`.

What should change: Add `DataPivot.resolve_output_dimensions(block_dimensions, requested_dim_ids, dimension_lookup)`. The method should return resolved dimension wrappers that include the actual pivot dimension and optional `group_by` metadata.

Where the responsibility should go: New shared resolution helper used by `DataPivot`, publicly exposed as `DataPivot.resolve_output_dimensions(...)`.

Why this is better: Direct and connected dimension resolution becomes one rule. It also reduces mutation of shared model/proxy dimension objects inside resource files.

### Gap 5: Output APIs Build Main And Total Pivot Query Params

Label: `DataPivot resolution logic`

Problem summary: Output handlers build `dim_rows`, `dims`, main pivot queries, and total pivot queries manually.

Current files and lines:

- `modelAPI/resources/new_block_kpi.py`: [v2 output builds dimension rows, dims metadata, main query, and total query](https://github.com/BloxSoftware/Blox-Dev/blob/development/modelAPI/resources/new_block_kpi.py#L245-L272).
- `modelAPI/resources/block_kpi.py`: [legacy output builds dimension rows and total query](https://github.com/BloxSoftware/Blox-Dev/blob/development/modelAPI/resources/block_kpi.py#L938-L970).
- `modelAPI/resources/block_kpi_v3.py`: [v3 output builds the same query shapes](https://github.com/BloxSoftware/Blox-Dev/blob/development/modelAPI/resources/block_kpi_v3.py#L348-L388).
- `modelAPI/resources/block_kpi_v4.py`: [v4 output builds the same query shapes](https://github.com/BloxSoftware/Blox-Dev/blob/development/modelAPI/resources/block_kpi_v4.py#L387-L416).
- `modelAPI/resources/block_kpi_v4_rust.py`: [Rust output builds the same query shapes](https://github.com/BloxSoftware/Blox-Dev/blob/development/modelAPI/resources/block_kpi_v4_rust.py#L526-L555).
- `modelAPI/resources/scenarios.py`: [scenario comparison builds query params inline](https://github.com/BloxSoftware/Blox-Dev/blob/development/modelAPI/resources/scenarios.py#L366-L389).

What the current code is doing: It decides the `rows`, `columns`, `filters`, `group_by`, and total query shape before calling `DataPivot`.

Why it is a gap: The API layer is constructing the pivot contract. That is resolution logic for the pivot layer.

What should change: Add `DataPivot.build_output_queries(resolved_dimensions, dim_sort, time_display_levels, filters)`. It should return `dims`, `main_query`, and `total_query`.

Where the responsibility should go: New shared resolution helper used by `DataPivot`, publicly exposed as `DataPivot.build_output_queries(...)`.

Why this is better: All output versions generate identical pivot query shapes and total query shapes.

### Gap 6: Calc Controllers Discover Pivot Dimensions Before Pivot

Label: `DataPivot resolution logic`

Problem summary: Calc controllers build the dimension list used by `DataPivot`, including block dimensions, Time, connected dimensions, and item preloading.

Current files and lines:

- `modelAPI/calc_engine/calc_controller_new.py`: [`__get_output_dimensions_for_block`](https://github.com/BloxSoftware/Blox-Dev/blob/development/modelAPI/calc_engine/calc_controller_new.py#L567-L623) discovers pivot dimensions and time column.
- `modelAPI/calc_engine/calc_controller_v3.py`: [v3 controller repeats the same dimension discovery](https://github.com/BloxSoftware/Blox-Dev/blob/development/modelAPI/calc_engine/calc_controller_v3.py#L457-L513).
- `modelAPI/calc_engine/calc_controller_v4.py`: [v4 controller repeats the same dimension discovery](https://github.com/BloxSoftware/Blox-Dev/blob/development/modelAPI/calc_engine/calc_controller_v4.py#L598-L654).

What the current code is doing: It builds the complete dimension list required by pivot and identifies the Time column name.

Why it is a gap: The comments say this is the list used by `DataPivot`. That makes it pivot resolution behavior, but it is duplicated in controllers.

What should change: Add `DataPivot.build_pivot_dimensions_for_block(blockmodel)`. Controllers can call it, but should not own the rules.

Where the responsibility should go: New shared resolution helper used by `DataPivot`, exposed through `DataPivot`.

Why this is better: Calc controllers stay focused on calculation orchestration. Pivot owns the dimensions needed for display resolution.

### Gap 7: Time Range Filtering For Model Data Values Lives Outside Pivot

Label: `DataPivot logic`

Problem summary: Model data values endpoints filter raw dataframes by start/end time before constructing a pivot.

Current files and lines:

- `modelAPI/resources/utils.py`: [`apply_time_filter`](https://github.com/BloxSoftware/Blox-Dev/blob/development/modelAPI/resources/utils.py#L4-L43) parses `%b-%y` values and filters a dataframe.
- `modelAPI/resources/models.py`: [Python model data values calls `apply_time_filter`](https://github.com/BloxSoftware/Blox-Dev/blob/development/modelAPI/resources/models.py#L1375-L1384).
- `modelAPI/resources/model_data_values_rust.py`: [Rust model data values calls `apply_time_filter`](https://github.com/BloxSoftware/Blox-Dev/blob/development/modelAPI/resources/model_data_values_rust.py#L125-L152).

What the current code is doing: It filters a raw Polars dataframe on a Time column before pivot execution.

Why it is a gap: This is not just request parsing. It is dataframe filtering for pivoted display data, so it is actual `DataPivot` logic.

What should change: Move this behavior behind `DataPivot.apply_time_filter(...)` or an instance-level time filtering path if the team wants it applied as part of pivot construction.

Where the responsibility should go: Directly on `DataPivot` or a shared helper mixed into `DataPivot`. Public access should be through `DataPivot`.

Why this is better: Python and Rust model data value endpoints use the same time filtering behavior.

### Gap 8: Model Data Values Build Total Query Params Outside Pivot

Label: `DataPivot resolution logic`

Problem summary: Model data values endpoints build total query params with Time rows, Indicators columns, and special variance display levels.

Current files and lines:

- `modelAPI/resources/models.py`: [Python model data values builds total query params](https://github.com/BloxSoftware/Blox-Dev/blob/development/modelAPI/resources/models.py#L1400-L1417).
- `modelAPI/resources/model_data_values_rust.py`: [Rust model data values builds the same total query params](https://github.com/BloxSoftware/Blox-Dev/blob/development/modelAPI/resources/model_data_values_rust.py#L172-L198).

What the current code is doing: It creates a standard total pivot query and adds `"T"` display level for variance instruments.

Why it is a gap: The endpoint owns the pivot query shape for model totals. That is query resolution behavior.

What should change: Add `DataPivot.build_model_total_query(current_block_filter, instrument_type)`.

Where the responsibility should go: New shared resolution helper used by `DataPivot`, exposed through `DataPivot`.

Why this is better: Total query behavior is shared by Python and Rust model data values.

### Gap 9: Time Expansion Columns Are Added Outside Pivot

Label: `DataPivot logic`

Problem summary: Model data values endpoints add `Quarter`, `Year`, and `sort` columns after pivoting.

Current files and lines:

- `modelAPI/resources/models.py`: [Python model data values maps `Time` into Quarter/Year/sort columns](https://github.com/BloxSoftware/Blox-Dev/blob/development/modelAPI/resources/models.py#L1421-L1450).
- `modelAPI/resources/model_data_values_rust.py`: [Rust model data values maps `Time` into Quarter/Year/sort columns](https://github.com/BloxSoftware/Blox-Dev/blob/development/modelAPI/resources/model_data_values_rust.py#L200-L219).

What the current code is doing: It enriches pivoted output with display fields derived from the pivot's time granularity.

Why it is a gap: This is expansion of pivot output. It is actual display logic and should not be duplicated in endpoints.

What should change: Add `DataPivot.add_time_expansion_columns(df, time_granularity)`.

Where the responsibility should go: Directly on `DataPivot` or shared helper mixed into `DataPivot`.

Why this is better: Time expansion remains consistent wherever pivoted model data is returned.

### Gap 10: Export Mutates Query Params To Include All Indicator Rows

Label: `DataPivot resolution logic`

Problem summary: Export code removes the indicator row filter before passing query params to calc.

Current files and lines:

- `modelAPI/resources/block_export.py`: [`_include_all_indicator_rows`](https://github.com/BloxSoftware/Blox-Dev/blob/development/modelAPI/resources/block_export.py#L44-L67) clones query params and removes the indicator row filter.
- `modelAPI/resources/block_export.py`: [export parses/cleans query params and calls `_include_all_indicator_rows`](https://github.com/BloxSoftware/Blox-Dev/blob/development/modelAPI/resources/block_export.py#L207-L210).

What the current code is doing: It changes the query shape so export gets all indicator rows.

Why it is a gap: This is a pivot query variant. Export is deciding how to alter `DataPivot` query behavior.

What should change: Add `DataPivot.include_all_indicator_rows(query_params)`.

Where the responsibility should go: New shared resolution helper used by `DataPivot`, exposed through `DataPivot`.

Why this is better: Export can request a pivot-owned query variant without embedding pivot query mutation logic.

### Gap 11: Export Filters Pivot JSON By Time Level

Label: `DataPivot logic`

Problem summary: Export code filters returned JSON data/schema columns based on selected time level `Y`, `Q`, or `M`.

Current file and lines:

- `modelAPI/resources/block_export.py`: [export filters JSON data/schema by selected time level](https://github.com/BloxSoftware/Blox-Dev/blob/development/modelAPI/resources/block_export.py#L251-L291).

What the current code is doing: It keeps metadata fields, keeps yearly/quarterly/monthly time columns based on display level, and rewrites `jsonstr["data"]` and `jsonstr["schema"]["fields"]`.

Why it is a gap: This is filtering of pivot output data. It should be consistent with pivot display-level behavior, not embedded in export formatting code.

What should change: Add `DataPivot.filter_json_by_time_level(jsonstr, time_level)` or make export request the correct resolved output shape directly from pivot.

Where the responsibility should go: Directly on `DataPivot` or shared helper mixed into `DataPivot`.

What should stay outside pivot: Export filename generation, Excel workbook generation, worksheet naming limits, and row sorting for presentation can stay in `block_export.py`.

Why this is better: Export output filtering uses the same display-level rules as other pivot consumers.

### Gap 12: Output `fetch_ind_detail` Extracts Indicator Frames And Dimension Item Data

Label: `DataPivot logic`

Problem summary: Output handlers filter the pivoted dataframe by indicator, clean null/NaN/inf values, and build dimension item payloads.

Current files and lines:

- `modelAPI/resources/new_block_kpi.py`: [v2 output filters indicator frames and builds dimension item data](https://github.com/BloxSoftware/Blox-Dev/blob/development/modelAPI/resources/new_block_kpi.py#L125-L171).
- `modelAPI/resources/block_kpi.py`: [legacy output filters indicator frames and builds dimension item data](https://github.com/BloxSoftware/Blox-Dev/blob/development/modelAPI/resources/block_kpi.py#L701-L755).
- `modelAPI/resources/block_kpi_v3.py`: [v3 output repeats the same pattern](https://github.com/BloxSoftware/Blox-Dev/blob/development/modelAPI/resources/block_kpi_v3.py#L206-L246).
- `modelAPI/resources/block_kpi_v4.py`: [v4 output repeats the same pattern](https://github.com/BloxSoftware/Blox-Dev/blob/development/modelAPI/resources/block_kpi_v4.py#L228-L274).
- `modelAPI/resources/block_kpi_v4_rust.py`: [Rust output repeats the same pattern](https://github.com/BloxSoftware/Blox-Dev/blob/development/modelAPI/resources/block_kpi_v4_rust.py#L241-L295).

What the current code is doing: It derives per-indicator totals and dimension item rows from pivoted dataframes.

Why it is a gap: This is post-pivot display logic. Resource files should assemble the response, but not interpret pivoted dataframe internals repeatedly.

What should change: Add `DataPivot.get_indicator_pivot_frames(...)` and `DataPivot.build_dimension_item_data(...)`.

Where the responsibility should go: Directly on `DataPivot` or shared helper mixed into `DataPivot`.

Why this is better: Indicator output extraction becomes consistent across legacy, v2, v3, v4, and Rust paths.

### Gap 13: Rust Paths Build Incomplete Dimension Proxies

Label: `DataPivot resolution logic`

Problem summary: Rust output/model data values code builds ad hoc dimension objects with empty item properties and no `group_property_id`.

Current files and lines:

- `modelAPI/resources/block_kpi_v4_rust.py`: [Rust output discovers Time and connected dimensions manually](https://github.com/BloxSoftware/Blox-Dev/blob/development/modelAPI/resources/block_kpi_v4_rust.py#L421-L455), then [creates local `ItemProxy` and `DimProxy` with empty `item_properties` and `group_property_id = None`](https://github.com/BloxSoftware/Blox-Dev/blob/development/modelAPI/resources/block_kpi_v4_rust.py#L457-L484).
- `modelAPI/resources/model_data_values_rust.py`: [`_build_dimension_proxies`](https://github.com/BloxSoftware/Blox-Dev/blob/development/modelAPI/resources/model_data_values_rust.py#L298-L329) creates lightweight proxies with empty `item_properties` and `group_property_id = None`.
- `modelAPI/calc_engine/proxy_classes_v4.py`: [`DimensionProxy`](https://github.com/BloxSoftware/Blox-Dev/blob/development/modelAPI/calc_engine/proxy_classes_v4.py#L21-L70) already loads `group_property_id`, properties, items, and item properties.

What the current code is doing: It creates local compatibility objects for pivot, but the objects are missing metadata that connected-dimension and property-based behavior may need.

Why it is a gap: Pivot resolution depends on complete dimension metadata. Incomplete Rust proxies can make Rust behavior diverge from Python behavior.

What should change: Use `calc_engine.proxy_classes_v4.DimensionProxy` instead of local incomplete proxies. Dimension discovery rules should also be shared through `DataPivot`.

Where the responsibility should go: Complete proxy construction should remain in `proxy_classes_v4.py`; pivot dimension discovery should live in `DataPivot` resolution helpers.

Why this is better: Rust and Python paths feed equivalent dimension objects into pivot.

### Gap 14: Dashboard/BI Consumers Still Reshape Returned Data

Label: `DataPivot resolution logic` follow-up

Problem summary: `blox-bi` consumers still group and reshape `raw_data`/`total_data` returned from Model API.

Current files and lines:

- `blox-bi/app/instrument_engine/table.py`: [table instruments build `dim_df` from raw data and group by dimension/time](https://github.com/BloxSoftware/Blox-Dev/blob/development/blox-bi/app/instrument_engine/table.py#L98-L136).
- `blox-bi/app/instrument_engine/line.py`: [line/bar instruments build `dim_df` and group by dimension/time](https://github.com/BloxSoftware/Blox-Dev/blob/development/blox-bi/app/instrument_engine/line.py#L96-L128).
- `blox-bi/app/instrument_engine/pie.py`: [pie instruments apply filters and group by dimension](https://github.com/BloxSoftware/Blox-Dev/blob/development/blox-bi/app/instrument_engine/pie.py#L60-L80).
- `blox-bi/app/instrument_engine/variance.py`: [variance instruments build and sort dimension breakout rows](https://github.com/BloxSoftware/Blox-Dev/blob/development/blox-bi/app/instrument_engine/variance.py#L410-L454).

What the current code is doing: It compensates downstream by reshaping returned data into instrument-specific dimension structures.

Why it is a gap: It shows the same ownership issue downstream. Dashboard consumers should eventually rely on stable pivot-resolved output instead of recreating grouping behavior.

What should change: Treat as a follow-up after Model API centralizes the producer contract. Do not change `blox-bi` in the first Model API refactor unless BLOX-2075 explicitly includes consumer changes.

Where the responsibility should go: First into Model API `DataPivot`; later simplify `blox-bi` consumers where the producer contract supports it.

Why this is better: It avoids changing producer and consumer behavior at the same time.

### Boundary: Request Transport Normalization Should Stay Outside Pivot

Label: not a data pivot gap

Current file and lines:

- `modelAPI/util/block_outputs_request.py`: [`parse_block_outputs_post_data`](https://github.com/BloxSoftware/Blox-Dev/blob/development/modelAPI/util/block_outputs_request.py#L18-L49) merges JSON body params, applies default request values, and coerces `dim_id` to string.

What the current code is doing: It normalizes HTTP/body shape and plan-page defaults.

Why it should stay outside pivot: This is request transport handling, not data display resolution. `DataPivot` should not know about Flask request parser behavior or nested `params` payloads.

## 4. Cross-Cutting Themes

- There is a split between actual `DataPivot` logic and `DataPivot` resolution logic. Both should be pivot-owned, but they do not need to live in the same physical file.
- API/output files currently build pivot-specific query shapes. This makes APIs own business logic they should only pass through.
- Plan-page filter normalization is duplicated or centralized in the wrong layer. It produces `DataPivot` filters, so `DataPivot` should own it.
- Connected-dimension handling is split across output APIs, calc controllers, Rust proxy builders, and pivot internals.
- Time behavior is spread across utilities and endpoints: time range filtering, time display levels, total queries, and Quarter/Year/sort expansion.
- Export mixes data-resolution behavior with presentation/file behavior.
- Rust paths need complete proxy objects so centralized resolution behaves the same in Python and Rust.
- `blox-bi` is a consumer follow-up, not the first Model API refactor target.

## 5. Recommended Refactor Plan

1. Establish the shared public surface on all `DataPivot` classes.
   - Add `modelAPI/calc_engine/data_pivot_resolution.py` as an internal implementation module.
   - Mix shared methods into `data_pivot.py`, `data_pivot_v3.py`, and `data_pivot_v4.py`.
   - Resource files should call `DataPivot`, not `DataPivotResolutionMixin`.

2. Move plan filter normalization first.
   - Add `DataPivot.normalize_plan_filters(dim_filter, time_filter)`.
   - Replace calls to `build_filters_from_plan_params(...)`.
   - Remove the old helper, or keep a temporary wrapper only if external compatibility requires it.

3. Centralize requested dimension and connected-dimension resolution.
   - Add `DataPivot.resolve_output_dimensions(...)`.
   - Replace direct/connected dimension parsing in legacy, v2, v3, v4, Rust, and scenario output paths.
   - Support SQLAlchemy objects, dicts, and metadata-cache proxies.

4. Centralize output query construction.
   - Add `DataPivot.build_output_queries(...)`.
   - Replace inline `dim_rows`, `dims`, main query, and total query construction.
   - This depends on resolved dimensions.

5. Move controller dimension discovery behind `DataPivot`.
   - Add `DataPivot.build_pivot_dimensions_for_block(blockmodel)`.
   - Replace repeated controller methods with delegation.

6. Move actual dataframe/time `DataPivot` logic.
   - Add `DataPivot.apply_time_filter(...)`.
   - Add `DataPivot.add_time_expansion_columns(...)`.
   - Add indicator pivot frame and dimension item helpers.

7. Move model-data total query resolution.
   - Add `DataPivot.build_model_total_query(...)`.
   - Replace duplicate Python and Rust model data values logic.

8. Move export data behavior.
   - Add `DataPivot.include_all_indicator_rows(...)`.
   - Add `DataPivot.filter_json_by_time_level(...)` or adjust export to request the correct resolved shape from pivot.
   - Keep Excel/file/presentation sorting outside pivot.

9. Fix Rust proxy completeness.
   - Replace local ad hoc proxies with `calc_engine.proxy_classes_v4.DimensionProxy`.
   - Make Rust and Python paths provide equivalent metadata to pivot.

10. Add focused regression coverage.
    - Plan filter normalization.
    - Direct and connected output dimension resolution.
    - Main/total output query construction.
    - DataPivot version exposure of shared methods.
    - Model data values time filtering and expansion.
    - Export query mutation and time-level output filtering.
    - Rust proxy completeness for connected dimensions.

11. Track `blox-bi` as a follow-up.
    - Revisit consumer grouping after Model API returns stable pivot-resolved data.

## 6. Meeting Discussion Notes

- The key design decision is ownership: resource files should not decide grouping, filters, expansion, connected dimensions, or total query shape.
- `DataPivot` should be the public owner even if implementation is split into a helper file.
- `data_pivot_resolution.py` is useful because it avoids copying shared methods into three pivot classes.
- `data_pivot_resolution.py` should not become a new public API for resource files. It should be an internal implementation detail.
- Actual `DataPivot` logic means dataframe/pivot behavior: filter application, time filtering, output expansion, indicator frame extraction.
- `DataPivot` resolution logic means turning request/model metadata into pivot-ready inputs: normalized filters, resolved dimensions, query params, total query params, connected dimension maps.
- Export should keep file presentation behavior but give data filtering/query mutation back to `DataPivot`.
- Rust paths should use complete proxy classes so centralized resolution has the same metadata in every path.
- The first refactor should stay inside Model API. `blox-bi` can be aligned after the producer contract is stable.

## 7. Final Conclusion

The clean architecture is: API receives the request, calc produces raw block data, and `DataPivot` owns the rules for turning that data plus request metadata into display-ready results.

Some of the missing behavior is actual pivot logic and should sit directly on `DataPivot`. Some of it is resolution/preparation logic and can live in a new `data_pivot_resolution.py` helper that is mixed into each `DataPivot` class. In both cases, the public entry point should be `DataPivot`.

This approach keeps output APIs thin, removes duplicated query/filter/grouping logic, aligns Python and Rust paths, and gives plan page, model data values, export, and future dashboard consumers one consistent backend resolution model.
# Data Pivot Refactor Discussion

Inspection basis: fetched `origin/development` at commit `34be524947d7056e1d976a991d4edb2847fa9411`. All links below point to the `development` branch.

Story: As a backend team, we want all data display resolution logic to live in the data pivot class so that grouping, filtering, and expansion are handled in one place and output APIs are no longer responsible for business logic they should not own.

## 1. Issue Summary

The backend currently splits data display resolution across `DataPivot`, output APIs, plan-page helpers, export code, model data value endpoints, calc controllers, and Rust proxy setup. This means the API layer is not just passing request state to the pivot layer; it is deciding which dimensions to use, how connected dimensions resolve, how plan filters become pivot filters, how output rows and totals are shaped, and how time/group expansion is returned.

That creates duplicated logic and inconsistent behavior across legacy output, v2, v3, v4, Rust, scenario comparison, export, dashboard consumers, and model data values. The core architectural problem is ownership: display resolution rules should be a pivot capability, not repeated endpoint behavior.

This refactor matters because plan-page filtering, group-by behavior, connected-dimension handling, export behavior, and dashboard output should all reuse the same rules. If each endpoint continues to build its own pivot query or post-process pivot data, future changes will keep drifting.

## 2. Target Architecture

The target ownership model should be:

- Requests enter Model API through resource classes.
- API/resource code validates access, reads request fields, loads the block/scenario/model, and passes request state into `DataPivot`.
- `DataPivot` is the public source of truth for data display resolution: filter normalization, output dimension resolution, connected-dimension resolution, row/column query construction, total query construction, time filtering, time expansion, and post-pivot frame extraction.
- Shared implementation can live in a new internal helper/mixin such as `modelAPI/calc_engine/data_pivot_resolution.py`, but output APIs should call `DataPivot.normalize_plan_filters(...)`, `DataPivot.resolve_output_dimensions(...)`, `DataPivot.build_output_queries(...)`, etc.
- Output/API layers should become thin presentation layers: call calc, call pivot, format the response, return JSON or export files.
- Export-specific formatting, Excel file naming, worksheet constraints, and presentation row sorting can remain in export code because they are not generic pivot resolution.
- Dashboard/BI consumer reshaping should be treated as a follow-up after the Model API producer contract is centralized.

Recommended structure:

- Add an internal `DataPivotResolutionMixin` or similarly named helper in `modelAPI/calc_engine/data_pivot_resolution.py`.
- Mix it into all current `DataPivot` classes.
- Keep `DataPivot` as the public entry point. Resource files should not import or call the mixin directly.
- Centralize shared logic once, while preserving the existing three pivot implementations until the team chooses a larger consolidation.

## 3. Gap-By-Gap Analysis

### Gap 1: There are multiple `DataPivot` classes, so shared resolution behavior can drift

Problem summary: The development branch has three pivot classes. They each parse query params, apply filters, aggregate, and format output in parallel implementations.

Why it is a gap: The story says `DataPivot` should be the single source of truth. With three implementations, new resolution helpers must be shared across all versions or the same bug can stay fixed in one route and broken in another.

Current files:

- `modelAPI/calc_engine/data_pivot.py`: [`DataPivot` class starts here](https://github.com/BloxSoftware/Blox-Dev/blob/development/modelAPI/calc_engine/data_pivot.py#L13-L38). It parses query params and begins pivot setup.
- `modelAPI/calc_engine/data_pivot_v3.py`: [`DataPivot` v3 class starts here](https://github.com/BloxSoftware/Blox-Dev/blob/development/modelAPI/calc_engine/data_pivot_v3.py#L18-L46). It has its own setup path.
- `modelAPI/calc_engine/data_pivot_v4.py`: [`DataPivot` v4 class starts here](https://github.com/BloxSoftware/Blox-Dev/blob/development/modelAPI/calc_engine/data_pivot_v4.py#L17-L42). It has another implementation path.

What the current code is doing: Each file defines a separate pivot class and performs its own query parsing, dimension filtering, aggregation parameter creation, and formatting.

What should change: Add shared resolution behavior through an internal helper/mixin and mix it into all three `DataPivot` classes. This is a move/additive refactor first, not a full class merge.

Target owner: A helper/resolution class used by `DataPivot`, exposed publicly through each `DataPivot` class.

Why this is better: It gives the API a stable public entry point while avoiding a risky full merge of v1/v3/v4 pivot implementations in the first pass.

### Gap 2: Plan-page filter translation is outside pivot

Problem summary: Plan-page params such as `dim_filter` and `time_filter` are converted into `DataPivot` filter objects in a utility module and then called by output APIs.

Why it is a gap: The helper explicitly creates the filter shape consumed by `DataPivot`, but it lives outside the pivot layer. Output APIs are deciding how request filters become pivot filters.

Current files:

- `modelAPI/util/query_params_helpers.py`: [`build_filters_from_plan_params`](https://github.com/BloxSoftware/Blox-Dev/blob/development/modelAPI/util/query_params_helpers.py#L43-L80) parses JSON/list filters and appends `{"Time": time_filter}`.
- `modelAPI/resources/new_block_kpi.py`: [v2 output calls the helper](https://github.com/BloxSoftware/Blox-Dev/blob/development/modelAPI/resources/new_block_kpi.py#L239-L241).
- `modelAPI/resources/block_kpi.py`: [legacy output calls the helper](https://github.com/BloxSoftware/Blox-Dev/blob/development/modelAPI/resources/block_kpi.py#L929-L931).
- `modelAPI/resources/block_kpi_v3.py`: [v3 output calls the helper](https://github.com/BloxSoftware/Blox-Dev/blob/development/modelAPI/resources/block_kpi_v3.py#L336-L338).
- `modelAPI/resources/block_kpi_v4.py`: [v4 output calls the helper](https://github.com/BloxSoftware/Blox-Dev/blob/development/modelAPI/resources/block_kpi_v4.py#L379-L381).
- `modelAPI/resources/block_kpi_v4_rust.py`: [Rust output calls the helper](https://github.com/BloxSoftware/Blox-Dev/blob/development/modelAPI/resources/block_kpi_v4_rust.py#L515-L517).
- `modelAPI/resources/scenarios.py`: [scenario comparison repeats inline filter parsing](https://github.com/BloxSoftware/Blox-Dev/blob/development/modelAPI/resources/scenarios.py#L361-L364).

What the current code is doing: It normalizes plan-page filter input and appends a time filter before `DataPivot` is called.

What should change: Move this behavior into `DataPivot.normalize_plan_filters(dim_filter, time_filter)`. Remove `build_filters_from_plan_params` if no external compatibility is needed. If compatibility is needed temporarily, keep only a thin wrapper that delegates to `DataPivot`.

Target owner: Helper/resolution class used by `DataPivot`, publicly called as `DataPivot.normalize_plan_filters(...)`.

Why this is better: Filter normalization becomes one pivot-owned behavior across legacy, v2, v3, v4, Rust, and scenarios.

### Gap 3: Output APIs resolve requested dimensions and connected dimensions before pivot

Problem summary: Output handlers parse `dim_id`, look up dimensions, inspect block dimensions, parse dimension-property metadata, and mutate `group_property_id` before constructing a pivot query.

Why it is a gap: Connected-dimension resolution is pivot behavior. The API should not know how dimension-type properties link one dimension to another or when `group_property_id` should be set.

Current files:

- `modelAPI/resources/new_block_kpi.py`: [v2 output resolves direct and connected dimensions](https://github.com/BloxSoftware/Blox-Dev/blob/development/modelAPI/resources/new_block_kpi.py#L52-L89).
- `modelAPI/resources/block_kpi.py`: [legacy output resolves direct and connected dimensions](https://github.com/BloxSoftware/Blox-Dev/blob/development/modelAPI/resources/block_kpi.py#L616-L650).
- `modelAPI/resources/block_kpi_v3.py`: [v3 output performs cache-backed connected-dimension resolution](https://github.com/BloxSoftware/Blox-Dev/blob/development/modelAPI/resources/block_kpi_v3.py#L95-L149).
- `modelAPI/resources/block_kpi_v4.py`: [v4 output performs proxy-backed connected-dimension resolution](https://github.com/BloxSoftware/Blox-Dev/blob/development/modelAPI/resources/block_kpi_v4.py#L115-L158).
- `modelAPI/resources/block_kpi_v4_rust.py`: [Rust output repeats v4-style connected-dimension resolution](https://github.com/BloxSoftware/Blox-Dev/blob/development/modelAPI/resources/block_kpi_v4_rust.py#L125-L176).
- `modelAPI/resources/scenarios.py`: [scenario comparison repeats connected-dimension lookup](https://github.com/BloxSoftware/Blox-Dev/blob/development/modelAPI/resources/scenarios.py#L267-L299).

What the current code is doing: It decides whether requested dimensions are direct block dimensions or connected dimensions referenced by a property. It then returns a dimension object that `DataPivot` can use.

What should change: Add `DataPivot.resolve_output_dimensions(block_dimensions, requested_dim_ids, dimension_lookup)`. Resource files should pass block dimensions and a lookup function, then use the resolved dimensions returned by `DataPivot`.

Target owner: Helper/resolution class used by `DataPivot`, publicly called as `DataPivot.resolve_output_dimensions(...)`.

Why this is better: Direct/connected dimension resolution becomes one rule instead of six endpoint-specific copies. It also avoids mutating shared dimension objects in resource code.

### Gap 4: Output APIs build pivot rows, dims metadata, and total pivot queries

Problem summary: Output handlers build `dim_rows`, `dims`, the main pivot query, and the total pivot query outside `DataPivot`.

Why it is a gap: Choosing `rows`, `columns`, `filters`, `group_by`, and the total query shape is generic pivot resolution. Output APIs should not own the pivot query contract.

Current files:

- `modelAPI/resources/new_block_kpi.py`: [v2 output builds dim rows, dims metadata, main query, and total query](https://github.com/BloxSoftware/Blox-Dev/blob/development/modelAPI/resources/new_block_kpi.py#L245-L272).
- `modelAPI/resources/block_kpi.py`: [legacy output builds dim rows and total query](https://github.com/BloxSoftware/Blox-Dev/blob/development/modelAPI/resources/block_kpi.py#L938-L970).
- `modelAPI/resources/block_kpi_v3.py`: [v3 output builds the same query shapes](https://github.com/BloxSoftware/Blox-Dev/blob/development/modelAPI/resources/block_kpi_v3.py#L348-L388).
- `modelAPI/resources/block_kpi_v4.py`: [v4 output builds the same query shapes](https://github.com/BloxSoftware/Blox-Dev/blob/development/modelAPI/resources/block_kpi_v4.py#L387-L416).
- `modelAPI/resources/block_kpi_v4_rust.py`: [Rust output builds the same query shapes](https://github.com/BloxSoftware/Blox-Dev/blob/development/modelAPI/resources/block_kpi_v4_rust.py#L526-L555).
- `modelAPI/resources/scenarios.py`: [scenario comparison builds the output query inline](https://github.com/BloxSoftware/Blox-Dev/blob/development/modelAPI/resources/scenarios.py#L366-L389).

What the current code is doing: It turns selected dimensions into pivot rows, adds `"Indicators"`, creates time columns with display levels, and creates a separate total query.

What should change: Add `DataPivot.build_output_queries(resolved_dimensions, dim_sort, time_display_levels, filters)`. Return `dims`, `main_query`, and `total_query` from one place.

Target owner: Helper/resolution class used by `DataPivot`, publicly called as `DataPivot.build_output_queries(...)`.

Why this is better: Every output path sends the same query shape to the pivot layer. Changes to grouping or total behavior happen once.

### Gap 5: Calc controllers discover output dimensions and connected dimensions before pivot

Problem summary: Calc controllers build the full dimension list used by `DataPivot`, including direct block dimensions, Time, and connected dimensions from dimension-type properties.

Why it is a gap: These comments already describe the list as "used by DataPivot". The resolution rules are pivot-owned behavior but live in three controllers.

Current files:

- `modelAPI/calc_engine/calc_controller_new.py`: [`__get_output_dimensions_for_block`](https://github.com/BloxSoftware/Blox-Dev/blob/development/modelAPI/calc_engine/calc_controller_new.py#L567-L623) builds dimensions, adds Time, parses dimension-type properties, and touches items.
- `modelAPI/calc_engine/calc_controller_v3.py`: [v3 controller repeats the same method](https://github.com/BloxSoftware/Blox-Dev/blob/development/modelAPI/calc_engine/calc_controller_v3.py#L457-L513).
- `modelAPI/calc_engine/calc_controller_v4.py`: [v4 controller repeats the same method](https://github.com/BloxSoftware/Blox-Dev/blob/development/modelAPI/calc_engine/calc_controller_v4.py#L598-L654).

What the current code is doing: It discovers the dimension set needed by pivot and returns `dimensions, time_column_name`.

What should change: Move this rule into `DataPivot.build_pivot_dimensions_for_block(blockmodel)` or the internal resolution helper used by `DataPivot`. Controllers can still call it, but they should not own the rules.

Target owner: Helper/resolution class used by `DataPivot`, publicly called as `DataPivot.build_pivot_dimensions_for_block(...)`.

Why this is better: Controllers stay focused on calculation orchestration, while pivot owns the dimensions it needs for display resolution.

### Gap 6: Model data values endpoints perform time filtering, total-query construction, and time expansion outside pivot

Problem summary: Model data value endpoints filter raw data by start/end time, build total query params, and add `Quarter`, `Year`, and `sort` expansion columns after pivot.

Why it is a gap: Time filtering and time expansion are display-resolution behaviors. They should be consistent with `DataPivot` time behavior and not duplicated in endpoint code.

Current files:

- `modelAPI/resources/utils.py`: [`apply_time_filter`](https://github.com/BloxSoftware/Blox-Dev/blob/development/modelAPI/resources/utils.py#L4-L43) parses `%b-%y` strings and filters a Polars dataframe.
- `modelAPI/resources/models.py`: [model data values calls `apply_time_filter`](https://github.com/BloxSoftware/Blox-Dev/blob/development/modelAPI/resources/models.py#L1375-L1384), [builds a total pivot query](https://github.com/BloxSoftware/Blox-Dev/blob/development/modelAPI/resources/models.py#L1400-L1417), and [adds Quarter/Year/sort columns](https://github.com/BloxSoftware/Blox-Dev/blob/development/modelAPI/resources/models.py#L1421-L1450).
- `modelAPI/resources/model_data_values_rust.py`: [Rust model data values calls `apply_time_filter`](https://github.com/BloxSoftware/Blox-Dev/blob/development/modelAPI/resources/model_data_values_rust.py#L125-L152), [builds the same total query](https://github.com/BloxSoftware/Blox-Dev/blob/development/modelAPI/resources/model_data_values_rust.py#L172-L198), and [adds Quarter/Year/sort columns](https://github.com/BloxSoftware/Blox-Dev/blob/development/modelAPI/resources/model_data_values_rust.py#L200-L219).

What the current code is doing: It handles request time ranges, computes a totals pivot for requested filters, and enriches pivot output with time expansion fields.

What should change: Move generic pieces into `DataPivot`: `apply_time_filter`, `build_model_total_query`, and `add_time_expansion_columns`. Keep model endpoint response assembly outside pivot.

Target owner: Helper/resolution class used by `DataPivot`, publicly called as `DataPivot.apply_time_filter(...)`, `DataPivot.build_model_total_query(...)`, and `DataPivot.add_time_expansion_columns(...)`.

Why this is better: Model data values and Rust model data values will share the same time and total-resolution behavior.

### Gap 7: Export mutates pivot queries and filters pivot JSON by time level

Problem summary: Export code removes the indicator row filter before calling calc and then filters the returned JSON columns by the selected time level.

Why it is a gap: Removing row filters from the query and filtering pivot JSON by time level are resolution behaviors. They are currently mixed into export file generation.

Current files:

- `modelAPI/resources/block_export.py`: [`_include_all_indicator_rows`](https://github.com/BloxSoftware/Blox-Dev/blob/development/modelAPI/resources/block_export.py#L44-L67) clones query params and removes the indicator row filter.
- `modelAPI/resources/block_export.py`: [export parses/cleans query params and calls `_include_all_indicator_rows`](https://github.com/BloxSoftware/Blox-Dev/blob/development/modelAPI/resources/block_export.py#L207-L210).
- `modelAPI/resources/block_export.py`: [export filters returned JSON data/schema by `Y`, `Q`, or `M`](https://github.com/BloxSoftware/Blox-Dev/blob/development/modelAPI/resources/block_export.py#L251-L291).
- `modelAPI/resources/block_export.py`: [export then sorts rows and calls file export](https://github.com/BloxSoftware/Blox-Dev/blob/development/modelAPI/resources/block_export.py#L293-L296).

What the current code is doing: It changes the query passed to calc and trims pivot output columns based on time display level.

What should change: Move query mutation and time-level JSON filtering into `DataPivot`, for example `DataPivot.include_all_indicator_rows(query_params)` and `DataPivot.filter_json_by_time_level(jsonstr, time_level)`. Keep file naming, worksheet-name limits, Excel response generation, and indicator row sorting in export.

Target owner: Helper/resolution class used by `DataPivot` for query/result filtering. Export remains owner of presentation/file concerns.

Why this is better: Export uses the same pivot-owned filtering rules while staying responsible only for export presentation.

### Gap 8: Output `fetch_ind_detail` post-processes pivot frames by indicator and dimension item

Problem summary: Output handlers filter the pivoted frame by indicator, clean null/inf values, convert totals, and build `dim_item_data` by popping dimension columns.

Why it is a gap: The API is interpreting pivoted dataframes and rebuilding dimension item rows. That is display-resolution logic after pivot, not response wiring.

Current files:

- `modelAPI/resources/new_block_kpi.py`: [v2 output filters indicator frames and builds dimension item data](https://github.com/BloxSoftware/Blox-Dev/blob/development/modelAPI/resources/new_block_kpi.py#L125-L171).
- `modelAPI/resources/block_kpi.py`: [legacy output filters indicator frames and builds dimension item data](https://github.com/BloxSoftware/Blox-Dev/blob/development/modelAPI/resources/block_kpi.py#L701-L755).
- `modelAPI/resources/block_kpi_v3.py`: [v3 output repeats the same pattern](https://github.com/BloxSoftware/Blox-Dev/blob/development/modelAPI/resources/block_kpi_v3.py#L206-L246).
- `modelAPI/resources/block_kpi_v4.py`: [v4 output repeats the same pattern](https://github.com/BloxSoftware/Blox-Dev/blob/development/modelAPI/resources/block_kpi_v4.py#L228-L274).
- `modelAPI/resources/block_kpi_v4_rust.py`: [Rust output repeats the same pattern](https://github.com/BloxSoftware/Blox-Dev/blob/development/modelAPI/resources/block_kpi_v4_rust.py#L241-L295).

What the current code is doing: It extracts one indicator's rows from the pivoted dataframe, converts total values, and creates dimension item payloads.

What should change: Move dataframe extraction and dimension item construction into `DataPivot`, for example `DataPivot.get_indicator_pivot_frames(...)` and `DataPivot.build_dimension_item_data(...)`. Keep final response object assembly in resource files.

Target owner: Helper/resolution class used by `DataPivot`, publicly called through `DataPivot`.

Why this is better: Indicator/dimension extraction becomes consistent across legacy, v2, v3, v4, and Rust outputs.

### Gap 9: Rust paths build incomplete dimension proxies for pivot

Problem summary: Rust output paths build ad hoc dimension/item proxy objects that omit properties and set `group_property_id = None`.

Why it is a gap: Pivot and connected-dimension behavior depend on dimension properties, item properties, and group property IDs. Incomplete proxies make Rust behavior fragile and different from non-Rust paths.

Current files:

- `modelAPI/resources/block_kpi_v4_rust.py`: [Rust output discovers Time and connected dimensions manually](https://github.com/BloxSoftware/Blox-Dev/blob/development/modelAPI/resources/block_kpi_v4_rust.py#L421-L455), then [creates local `ItemProxy` and `DimProxy` with empty `item_properties` and `group_property_id = None`](https://github.com/BloxSoftware/Blox-Dev/blob/development/modelAPI/resources/block_kpi_v4_rust.py#L457-L484).
- `modelAPI/resources/model_data_values_rust.py`: [`_build_dimension_proxies`](https://github.com/BloxSoftware/Blox-Dev/blob/development/modelAPI/resources/model_data_values_rust.py#L298-L329) creates lightweight proxies with empty `item_properties` and `group_property_id = None`.
- `modelAPI/calc_engine/proxy_classes_v4.py`: [`DimensionProxy`](https://github.com/BloxSoftware/Blox-Dev/blob/development/modelAPI/calc_engine/proxy_classes_v4.py#L21-L70) already loads `group_property_id`, properties, items, and item properties from metadata cache.

What the current code is doing: It creates minimal proxy objects just to satisfy `DataPivot` shape requirements, but the shape is incomplete for connected-dimension and property-based behavior.

What should change: Use the complete `calc_engine.proxy_classes_v4.DimensionProxy` in Rust output and Rust model data values. Dimension discovery itself should be centralized through `DataPivot` resolution helpers.

Target owner: Proxy construction belongs in proxy classes. Pivot dimension discovery belongs in `DataPivot` or its internal resolution helper.

Why this is better: Rust and Python paths feed equivalent dimension objects into pivot, which reduces branch-specific behavior.

### Gap 10: Dashboard/BI consumers still reshape pivot data

Problem summary: `blox-bi` consumers take `raw_data`/`total_data` from Model API and build grouped dimension structures themselves.

Why it is a gap: It shows the same architectural pressure downstream: consumers compensate for producer-side resolution not being centralized enough. However, this is outside the first Model API refactor scope.

Current files:

- `blox-bi/app/instrument_engine/table.py`: [table instruments build `dim_df` from raw data and group by dimension/time](https://github.com/BloxSoftware/Blox-Dev/blob/development/blox-bi/app/instrument_engine/table.py#L98-L136).
- `blox-bi/app/instrument_engine/line.py`: [line/bar instruments build `dim_df` and group by dimension/time](https://github.com/BloxSoftware/Blox-Dev/blob/development/blox-bi/app/instrument_engine/line.py#L96-L128).
- `blox-bi/app/instrument_engine/pie.py`: [pie instruments apply filters and group by dimension](https://github.com/BloxSoftware/Blox-Dev/blob/development/blox-bi/app/instrument_engine/pie.py#L60-L80).
- `blox-bi/app/instrument_engine/variance.py`: [variance instruments build and sort dimension breakout rows](https://github.com/BloxSoftware/Blox-Dev/blob/development/blox-bi/app/instrument_engine/variance.py#L410-L454).

What the current code is doing: It reshapes returned Model API data into instrument-specific dimension data.

What should change: Keep this as a follow-up after Model API centralizes the producer contract. Do not change `blox-bi` in the first Model API pivot refactor unless BLOX-2075 explicitly includes consumer-service changes.

Target owner: Follow-up consumer alignment after `DataPivot` has a stable resolved output contract.

Why this is better: It avoids changing producer and consumer behavior at the same time.

### Boundary Note: Request-body normalization can stay outside pivot

Problem summary: `parse_block_outputs_post_data` merges JSON body params, applies request defaults, and coerces `dim_id` to string.

Current file:

- `modelAPI/util/block_outputs_request.py`: [`parse_block_outputs_post_data`](https://github.com/BloxSoftware/Blox-Dev/blob/development/modelAPI/util/block_outputs_request.py#L18-L49) normalizes request transport shape and default request values.

What should stay: This should remain outside `DataPivot`. It is request parsing, not data display resolution.

Why this boundary matters: The goal is not to move every utility into pivot. The goal is to move grouping/filtering/expansion/data-resolution behavior into pivot while keeping HTTP/request concerns in the API layer.

## 4. Cross-Cutting Themes

- Duplicated normalization: Plan filters, time filters, output query rows, total query rows, and time expansion are built repeatedly in multiple endpoints.
- API/output layers own business logic: Resource files are deciding dimension ownership, connected-dimension mapping, group-by behavior, and pivot query shape.
- Connected-dimension handling is split: Output endpoints, calc controllers, Rust proxy builders, and `DataPivot` internals each know part of the connected-dimension story.
- Plan-page logic leaks into generic output paths: Plan-page filter params are normalized outside pivot even though the result is a `DataPivot` filter payload.
- Export mixes data resolution with presentation: Export should own workbook/file behavior, but query mutation and time-level JSON filtering are generic pivot behavior.
- Rust and Python paths are not symmetrical: Rust paths sometimes use incomplete local proxies while non-Rust paths use richer model/proxy objects.
- Consumers compensate downstream: `blox-bi` instruments still group and reshape data after Model API, which should be revisited after the Model API contract is cleaned up.

## 5. Recommended Refactor Plan

1. Add an internal pivot resolution module.
   - Create `modelAPI/calc_engine/data_pivot_resolution.py`.
   - Define shared helpers/classes there, such as `DataPivotResolutionMixin`, resolved dimension DTOs, and adapter utilities.
   - Mix it into `data_pivot.py`, `data_pivot_v3.py`, and `data_pivot_v4.py`.
   - Public calls should go through `DataPivot`, not `DataPivotResolutionMixin`.

2. Move plan filter normalization first.
   - Add `DataPivot.normalize_plan_filters(...)`.
   - Replace calls to `build_filters_from_plan_params(...)`.
   - Remove the helper or keep a temporary wrapper only if external compatibility requires it.
   - This is low risk and proves the ownership model.

3. Centralize output dimension resolution.
   - Add `DataPivot.resolve_output_dimensions(...)`.
   - Replace direct/connected dimension parsing in legacy, v2, v3, v4, Rust, and scenario output paths.
   - Make it support both object-backed and dict/proxy-backed dimensions.

4. Centralize output pivot query construction.
   - Add `DataPivot.build_output_queries(...)`.
   - Replace inline `dim_rows`, `dims`, main query, and total query construction.
   - This depends on centralized resolved dimensions.

5. Move calc-controller output dimension discovery.
   - Add `DataPivot.build_pivot_dimensions_for_block(...)`.
   - Replace the repeated controller methods with delegation.
   - This keeps controllers focused on calculation orchestration.

6. Move model data values time behavior.
   - Add pivot-owned helpers for time filtering, model total query construction, and time expansion columns.
   - Replace duplicate code in Python and Rust model data values endpoints.

7. Move export data-resolution behavior.
   - Move indicator-row query mutation and time-level JSON filtering into `DataPivot`.
   - Keep export filename, workbook formatting, worksheet constraints, and row presentation sorting in export.

8. Fix Rust dimension proxy completeness.
   - Replace local proxy construction with `calc_engine.proxy_classes_v4.DimensionProxy`.
   - Align Rust dimension objects with the non-Rust path before relying on centralized connected-dimension resolution.

9. Move indicator post-pivot dataframe extraction.
   - Add helpers for indicator frame extraction and dimension item data construction.
   - Keep final response assembly and input/actual value response fields in resource files.

10. Treat `blox-bi` as a follow-up.
    - Once Model API returns stable pivot-resolved data, simplify dashboard consumers where appropriate.
    - Avoid changing producer and consumer behavior in the same first refactor unless the story scope explicitly requires it.

## 6. Meeting Discussion Notes

- The main problem is not that the current code fails everywhere; it is that resolution ownership is scattered.
- `DataPivot` already consumes rows, columns, filters, display levels, and group-by settings. The code that builds those structures should move closer to `DataPivot`.
- Output APIs should not parse dimension-property `data_format` to discover connected dimensions.
- Output APIs should not decide the `rows` contract for dimension pivots and total pivots.
- Plan-page filter translation should be a `DataPivot` capability because it creates the exact filter shape `DataPivot` consumes.
- Export should keep presentation concerns, but data filtering and query mutation should be pivot-owned.
- Rust paths need complete dimension proxies before connected-dimension behavior can be trusted.
- The team should agree whether the new shared code is a mixin, helper service, or resolver class. The important decision is that resource files call `DataPivot`, not the lower-level implementation class.
- The team should agree whether `build_filters_from_plan_params` can be removed immediately or needs a temporary compatibility wrapper.
- `blox-bi` should be tracked separately unless BLOX-2075 explicitly includes consumer changes.

## 7. Final Conclusion

Centralizing resolution behavior in `DataPivot` is the cleaner design because `DataPivot` is already the layer that understands rows, columns, filters, dimensions, indicators, group-by behavior, time levels, and pivoted frames.

The recommended architecture keeps resource files thin and keeps shared behavior behind the `DataPivot` public entry point. A new internal resolution helper/mixin is useful, but it should be an implementation detail of `DataPivot`, not a new public utility that output APIs call directly.

This reduces duplication, makes connected-dimension behavior consistent, aligns Python and Rust paths, and gives plan page, dashboard, model data values, and export flows one place to reuse display-resolution rules.
