# Full Issue 2 - Simple Explanation Of All Source Issues

Branch analyzed:

```text
BLOX-2143-add-omni-calc-runtime-performance-tracing-and-benchmark-baseline
```

Purpose of this file:

```text
Explain every source issue in very simple English.
Use real project code patterns from Omni-Calc.
Show what the current data structure looks like.
Show what output the current code gives.
Explain why the output can be correct but still expensive.
Show what the improved data structure looks like.
Explain exactly why the fix helps in Rust.
```

## High Level Picture

The Rust omni-calc engine is already much faster than Python in many places, but some internal data structures still behave like a first version of the engine:

```text
1. Columns are stored in Vec lists.
2. Column lookup sometimes scans the Vec from the beginning.
3. Large Vec columns are cloned to pass them around.
4. Formula evaluation sometimes clones the whole evaluator context.
5. Hot IDs are stored as strings like "ind123" and parsed repeatedly.
6. Repeated strings are compared and hashed again and again.
```

These issues are not all about wrong output. Most of them are about overhead:

```text
CPU overhead     = repeated scans, parsing, hashing, sorting, string comparison.
Memory overhead  = repeated Vec clones and repeated String allocations.
Cache overhead   = large copied vectors push useful data out of CPU cache.
Future risk      = parallel execution becomes harder if everything mutates or copies shared state.
```

## Quick Summary

| Source Issue | Simple Problem | Simple Fix | Real Impact |
| --- | --- | --- | --- |
| Source Issue 7 | Columns are stored in lists, so lookup scans again and again. | Add `ColumnStore` with ordered columns plus fast index. | Fast column lookup without losing output order. |
| Source Issue 8 | Large column vectors are cloned in hot paths. | Share immutable column data where possible. | Less memory copying and better cache behavior. |
| Source Issue 11 | Snapshots need copied column data. | Use `Arc` shared column storage. | Cheap execution snapshots for future scheduler work. |
| Source Issue 13 | Same hot strings are hashed and compared repeatedly. | Intern strings into compact IDs. | Integer comparison instead of string comparison. |
| Source Issue 19 | Child formula evaluators clone full input context. | Split shared context from evaluator-local state. | Blox functions can create child evaluators cheaply. |
| Source Issue 21 | IDs are hidden inside strings like `"ind123"`. | Use typed IDs like `IndicatorId(123)`. | Less parsing, fewer silent ID mistakes, clearer code. |

---

## Source Issue 7 - Add ColumnStore And Indexed Lookup

### Simple Meaning

Right now, calculation object state stores columns in simple lists.

Real project code:

```rust
// File:
// modelAPI/omni-calc/src/engine/exec/state.rs

pub struct CalcObjectState {
    pub dim_columns: Vec<(String, Vec<String>)>,
    pub number_columns: Vec<(String, Vec<f64>)>,
    pub string_columns: Vec<(String, Vec<String>)>,
    pub connected_dim_columns: Vec<(String, Vec<String>)>,
}
```

That means a block may internally look like this:

```text
CalcObjectState
  number_columns:
    0 -> ("ind100", [10, 20, 30])
    1 -> ("ind200", [40, 50, 60])
    2 -> ("ind300", [70, 80, 90])
```

If Rust wants `"ind300"`, it checks `"ind100"`, then `"ind200"`, then `"ind300"`.

### Current Project Code Pattern

Real project code:

```rust
// File:
// modelAPI/omni-calc/src/engine/exec/state.rs

pub fn get_number_column(&self, name: &str) -> Option<&Vec<f64>> {
    self.number_columns
        .iter()
        .find(|(n, _)| n == name)
        .map(|(_, v)| v)
}
```

Similar lookup is also visible in:

```rust
// File:
// modelAPI/omni-calc/src/engine/exec/steps/calculation.rs

let existing_values = existing_columns
    .iter()
    .find(|(name, _)| name == node_id)
    .map(|(_, vals)| vals.clone());
```

### Current Code Example With Output Comments

```rust
fn get_number_column(columns: &[(String, Vec<f64>)], name: &str) -> Option<Vec<f64>> {
    columns
        .iter()                                  // output: starts scanning at index 0
        .find(|(n, _)| n == name)                // output: compares names one by one
        .map(|(_, values)| values.clone())       // output: clones matched column values
}

fn main() {
    let columns = vec![
        ("ind100".to_string(), vec![10.0, 20.0]), // output: index 0
        ("ind200".to_string(), vec![30.0, 40.0]), // output: index 1
        ("ind300".to_string(), vec![50.0, 60.0]), // output: index 2
    ];

    let result = get_number_column(&columns, "ind300"); // output: checks ind100, ind200, then ind300
    println!("{:?}", result);                           // output: Some([50.0, 60.0])
}
```

### Whole Data Structure Output Before Fix

```text
columns Vec
[
  0: name="ind100", values=[10.0, 20.0]
  1: name="ind200", values=[30.0, 40.0]
  2: name="ind300", values=[50.0, 60.0]
]

lookup("ind300")
  compare "ind100" == "ind300" -> false
  compare "ind200" == "ind300" -> false
  compare "ind300" == "ind300" -> true
  return [50.0, 60.0]
```

### What Is Wrong

The output is correct. The problem is the route to get the output.

For a small block this is fine. For a large model:

```text
500 columns
100 formulas
many formulas reading many previous columns
```

the engine may repeatedly scan the same list.

Simple overhead explanation:

```text
Vec lookup by scan = O(number_of_columns)
HashMap index lookup = usually O(1)
```

In Rust terms:

```text
Vec scan:
  reads each tuple
  compares string bytes
  branches on match or no match
  keeps walking memory until found

Indexed store:
  hashes/looks up the name once
  jumps to the column position
  returns a reference
```

### Better Code After Fix

```rust
use std::collections::HashMap;

struct ColumnStore {
    ordered: Vec<(String, Vec<f64>)>,
    index: HashMap<String, usize>,
}

impl ColumnStore {
    fn new() -> Self {
        Self {
            ordered: Vec::new(),         // output: keeps output order
            index: HashMap::new(),       // output: maps column name -> Vec position
        }
    }

    fn insert(&mut self, name: String, values: Vec<f64>) {
        let pos = self.ordered.len();    // output: next position
        self.index.insert(name.clone(), pos); // output: "ind300" -> 2
        self.ordered.push((name, values));    // output: append column once
    }

    fn get(&self, name: &str) -> Option<&Vec<f64>> {
        let pos = self.index.get(name)?; // output: Some(2) for "ind300"
        Some(&self.ordered[*pos].1)      // output: Some([50.0, 60.0])
    }
}

fn main() {
    let mut store = ColumnStore::new();
    store.insert("ind100".to_string(), vec![10.0, 20.0]); // output: index["ind100"] = 0
    store.insert("ind200".to_string(), vec![30.0, 40.0]); // output: index["ind200"] = 1
    store.insert("ind300".to_string(), vec![50.0, 60.0]); // output: index["ind300"] = 2

    println!("{:?}", store.get("ind300")); // output: Some([50.0, 60.0])
}
```

### Whole Data Structure Output After Fix

```text
ColumnStore
  ordered:
    0 -> ("ind100", [10.0, 20.0])
    1 -> ("ind200", [30.0, 40.0])
    2 -> ("ind300", [50.0, 60.0])

  index:
    "ind100" -> 0
    "ind200" -> 1
    "ind300" -> 2

lookup("ind300")
  index lookup -> 2
  ordered[2] -> [50.0, 60.0]
```

### Impact Of Improvement

This fix helps because:

```text
Before:
  Every lookup can walk many columns.
  Repeated formulas repeat the same walk.

After:
  Column lookup jumps directly to the right position.
  Output order is still stable because ordered Vec remains.
```

Deep Rust impact:

```text
1. Fewer string comparisons in hot loops.
2. Fewer branch checks during repeated lookup.
3. Less chance of cloning a column just to use it.
4. Cleaner foundation for shared/Arc column storage in later issues.
```

---

## Source Issue 8 - Reduce Clone-Heavy Execution Paths

### Simple Meaning

Rust `Vec<T>::clone()` is a deep copy.

For `Vec<f64>`, cloning means:

```text
allocate a new buffer
copy every f64 into the new buffer
return a new Vec pointing to copied data
```

Real project code has several clone-heavy paths.

### Current Project Code Pattern

Real project code:

```rust
// File:
// modelAPI/omni-calc/src/engine/exec/context.rs

arrays.push(Arc::new(Float64Array::from(values.clone())));
```

Real project code:

```rust
// File:
// modelAPI/omni-calc/src/engine/exec/steps/calculation.rs

for (name, values) in existing_columns {
    evaluator.add_column(name.clone(), values.clone());
}
```

Real project code:

```rust
// File:
// modelAPI/omni-calc/src/engine/exec/context.rs

let resolver = CrossObjectResolver::new(
    plan.request.node_maps.clone(),
    plan.request.variable_filters.clone(),
);
```

### Current Code Example With Output Comments

```rust
fn main() {
    let values = vec![10.0, 20.0, 30.0];       // output: one heap allocation for 3 f64 values
    let copy1 = values.clone();                // output: second heap allocation, copies 3 f64 values
    let copy2 = values.clone();                // output: third heap allocation, copies 3 f64 values

    println!("{}", values.len());              // output: 3
    println!("{}", copy1.len());               // output: 3
    println!("{}", copy2.len());               // output: 3
    println!("{}", values.as_ptr() == copy1.as_ptr()); // output: false
}
```

### Whole Data Structure Output Before Fix

```text
values
  pointer: 0xAAA
  data: [10.0, 20.0, 30.0]

copy1
  pointer: 0xBBB
  data: [10.0, 20.0, 30.0]

copy2
  pointer: 0xCCC
  data: [10.0, 20.0, 30.0]

same numbers, but three different memory buffers
```

### What Is Wrong

Again, the output is correct. The overhead is the problem.

For real model data:

```text
3 values     -> cheap
30,000 rows  -> expensive
500 columns  -> very expensive if repeated
```

If a column has `30,000` rows:

```text
30,000 f64 values * 8 bytes = 240,000 bytes per clone
```

If this happens across many columns and steps, Rust spends time moving memory instead of calculating.

Deep Rust explanation:

```text
Vec<T> owns its buffer.
clone() creates another owned buffer.
That buffer copy is a real memory copy.
Large copies pressure allocator, memory bandwidth, and CPU cache.
```

### Better Code After Fix

Use shared immutable data where possible.

```rust
use std::sync::Arc;

fn main() {
    let values: Arc<[f64]> = Arc::from([10.0, 20.0, 30.0]); // output: one allocation
    let shared1 = Arc::clone(&values);                      // output: increments ref count only
    let shared2 = Arc::clone(&values);                      // output: increments ref count only

    println!("{}", values.len());                           // output: 3
    println!("{}", shared1.len());                          // output: 3
    println!("{}", shared2.len());                          // output: 3
    println!("{}", Arc::ptr_eq(&values, &shared1));          // output: true
}
```

### Whole Data Structure Output After Fix

```text
Arc<[f64]>
  pointer: 0xAAA
  data: [10.0, 20.0, 30.0]
  strong_count: 3

values  -> pointer 0xAAA
shared1 -> pointer 0xAAA
shared2 -> pointer 0xAAA

same numbers, same memory buffer
```

### Impact Of Improvement

This fix helps because:

```text
Before:
  Passing a column often means copying all row values.

After:
  Passing a column can mean copying only a small Arc pointer.
```

Deep Rust impact:

```text
1. Reduces heap allocations.
2. Reduces memcpy of large Vec buffers.
3. Keeps hot column data in fewer memory locations.
4. Makes CPU cache behavior better because readers share the same buffer.
5. Makes future parallel read-only execution easier.
```

---

## Source Issue 11 - Snapshot-Friendly Shared Arc Column Storage

### Simple Meaning

A future scheduler needs snapshots of execution state.

A snapshot is a stable read-only view of current columns, so a worker can calculate safely.

If snapshots copy all column data, snapshots become expensive.

### Current Project Code Pattern

Current code stores owned vectors directly:

```rust
// File:
// modelAPI/omni-calc/src/engine/exec/state.rs

pub number_columns: Vec<(String, Vec<f64>)>,
pub string_columns: Vec<(String, Vec<String>)>,
pub connected_dim_columns: Vec<(String, Vec<String>)>,
```

Building final outputs also clones values:

```rust
// File:
// modelAPI/omni-calc/src/engine/exec/context.rs

arrays.push(Arc::new(Float64Array::from(values.clone())));
```

### Current Snapshot Example With Output Comments

```rust
#[derive(Clone)]
struct Snapshot {
    columns: Vec<(String, Vec<f64>)>,
}

fn main() {
    let snapshot1 = Snapshot {
        columns: vec![
            ("ind100".to_string(), vec![1.0, 2.0, 3.0]), // output: owned Vec buffer
            ("ind200".to_string(), vec![4.0, 5.0, 6.0]), // output: owned Vec buffer
        ],
    };

    let snapshot2 = snapshot1.clone(); // output: clones the Vec and every nested Vec<f64>

    println!("{}", snapshot1.columns.len()); // output: 2
    println!("{}", snapshot2.columns.len()); // output: 2
    println!("{}", snapshot1.columns[0].1.as_ptr() == snapshot2.columns[0].1.as_ptr()); // output: false
}
```

### Whole Data Structure Output Before Fix

```text
Snapshot 1
  ind100 -> ptr 0xAAA -> [1.0, 2.0, 3.0]
  ind200 -> ptr 0xBBB -> [4.0, 5.0, 6.0]

Snapshot 2
  ind100 -> ptr 0xCCC -> [1.0, 2.0, 3.0]
  ind200 -> ptr 0xDDD -> [4.0, 5.0, 6.0]

snapshot clone copied the column buffers
```

### What Is Wrong

For a scheduler, snapshot creation may happen many times:

```text
ready layer 1 -> snapshot
ready layer 2 -> snapshot
ready layer 3 -> snapshot
```

If each snapshot copies all columns, then parallel execution loses much of its benefit.

Simple overhead:

```text
snapshot clone with Vec columns = copy all data
snapshot clone with Arc columns = copy small pointers
```

### Better Code After Fix

```rust
use std::sync::Arc;

#[derive(Clone)]
struct Snapshot {
    columns: Vec<(String, Arc<[f64]>)>,
}

fn main() {
    let snapshot1 = Snapshot {
        columns: vec![
            ("ind100".to_string(), Arc::from([1.0, 2.0, 3.0])), // output: owned once
            ("ind200".to_string(), Arc::from([4.0, 5.0, 6.0])), // output: owned once
        ],
    };

    let snapshot2 = snapshot1.clone(); // output: clones names and Arc pointers, not f64 buffers

    println!("{}", snapshot1.columns.len()); // output: 2
    println!("{}", snapshot2.columns.len()); // output: 2
    println!("{}", Arc::ptr_eq(&snapshot1.columns[0].1, &snapshot2.columns[0].1)); // output: true
}
```

### Whole Data Structure Output After Fix

```text
Shared column buffers
  ind100 -> ptr 0xAAA -> [1.0, 2.0, 3.0]
  ind200 -> ptr 0xBBB -> [4.0, 5.0, 6.0]

Snapshot 1
  ind100 -> Arc ptr 0xAAA
  ind200 -> Arc ptr 0xBBB

Snapshot 2
  ind100 -> Arc ptr 0xAAA
  ind200 -> Arc ptr 0xBBB

snapshot clone shares buffers
```

### Impact Of Improvement

This fix helps because:

```text
Before:
  Snapshot creation copies large column data.

After:
  Snapshot creation shares stable columns.
```

Deep Rust impact:

```text
1. Arc gives shared ownership with safe lifetime tracking.
2. Workers can read columns without owning/copying them.
3. Immutable shared data prevents accidental mutation during parallel reads.
4. Scheduler can create cheap read snapshots between execution layers.
5. It prepares the engine for safer Rayon execution later.
```

---

## Source Issue 13 - String Interning For Hot-Path Keys

### Simple Meaning

The engine uses many repeated strings:

```text
"ind219049"
"ind219050"
"_33542"
"dim33548___prop71132"
"Lodge rental revenue"
"Product"
"Time"
```

Strings are useful at API boundaries, but inside hot Rust loops they are expensive.

### Current Project Code Pattern

Real project code stores formula columns using strings:

```rust
// File:
// modelAPI/omni-calc/src/engine/exec/formula_eval.rs

columns: HashMap<String, Vec<f64>>,
dim_string_columns: HashMap<String, Vec<String>>,
integer_columns: HashSet<String>,
```

Real project code builds string keys:

```rust
// File:
// modelAPI/omni-calc/src/engine/exec/steps/calculation.rs

key_parts.push(format!("{}={}", dim_id, values[row_idx]));
key_parts.sort();
key_parts.join("|");
```

Real project code clones string path keys:

```rust
// File:
// modelAPI/omni-calc/src/engine/exec/node_alignment/lookup.rs

lookup.insert(path.clone(), value);
```

### Current Code Example With Output Comments

```rust
fn main() {
    let dim_id = 33542;                               // output: numeric ID
    let dim_value = "Jul-25";                         // output: dimension item text
    let key = format!("{}={}", dim_id, dim_value);    // output: allocates String "33542=Jul-25"

    println!("{}", key);                              // output: 33542=Jul-25
    println!("{}", key == "33542=Jul-25");            // output: true, but compares bytes
}
```

### Whole Data Structure Output Before Fix

```text
HashMap<String, Vec<f64>>
  "ind219049" -> [34224.0, 37448.0, 31710.0]
  "ind219050" -> [0.0, 0.0, 0.0]
  "ind219051" -> [100.0, 200.0, 300.0]

Row key strings
  "33542=Jul-25|33548=Site A"
  "33542=Aug-25|33548=Site A"
  "33542=Sep-25|33548=Site A"

Each key is heap allocated text.
Each lookup hashes text.
Each comparison may inspect bytes.
```

### What Is Wrong

The output is correct, but repeated strings cost CPU and memory.

Simple overhead:

```text
String creation = allocate + copy bytes
String hash     = read bytes to compute hash
String compare  = compare bytes until difference or end
```

If a loop builds row keys for thousands of rows and many indicators, the same string work repeats.

### Better Code After Fix

Use string interning.

Interning means:

```text
Store each unique string once.
Give it a compact numeric ID.
Use the numeric ID in hot paths.
```

```rust
use std::collections::HashMap;

#[derive(Copy, Clone, Debug, PartialEq, Eq, Hash)]
struct Symbol(u32);

struct Interner {
    map: HashMap<String, Symbol>,
    values: Vec<String>,
}

impl Interner {
    fn new() -> Self {
        Self { map: HashMap::new(), values: Vec::new() } // output: empty intern table
    }

    fn intern(&mut self, text: &str) -> Symbol {
        if let Some(symbol) = self.map.get(text) {
            return *symbol;                              // output: reuse existing ID
        }

        let symbol = Symbol(self.values.len() as u32);   // output: next compact ID
        self.values.push(text.to_string());              // output: store text once
        self.map.insert(text.to_string(), symbol);        // output: "ind219049" -> Symbol(0)
        symbol
    }
}

fn main() {
    let mut interner = Interner::new();
    let a = interner.intern("ind219049");                // output: Symbol(0)
    let b = interner.intern("ind219049");                // output: Symbol(0), no new logical symbol
    let c = interner.intern("ind219050");                // output: Symbol(1)

    println!("{:?}", a);                                 // output: Symbol(0)
    println!("{}", a == b);                              // output: true
    println!("{}", a == c);                              // output: false
}
```

### Whole Data Structure Output After Fix

```text
Interner
  values:
    0 -> "ind219049"
    1 -> "ind219050"
    2 -> "33542=Jul-25"

Hot HashMap
  Symbol(0) -> [34224.0, 37448.0, 31710.0]
  Symbol(1) -> [0.0, 0.0, 0.0]

Hot row keys
  [Symbol(2), Symbol(10)]
  [Symbol(3), Symbol(10)]
  [Symbol(4), Symbol(10)]
```

### Impact Of Improvement

This fix helps because:

```text
Before:
  Compare/hash/copy long repeated strings.

After:
  Compare/hash/copy small numeric IDs.
```

Deep Rust impact:

```text
1. `u32`/`u64` IDs are Copy types, so passing them is cheap.
2. Equality is one integer compare instead of byte-by-byte string compare.
3. Hashing an integer is cheaper than hashing string bytes.
4. Less heap allocation means less allocator pressure.
5. Interning also supports typed IDs in Source Issue 21.
```

---

## Source Issue 19 - FormulaEvaluator Shared Context

### Simple Meaning

Formula evaluation uses a context that contains many columns and dimension strings.

Real project code:

```rust
// File:
// modelAPI/omni-calc/src/engine/exec/formula_eval.rs

struct EvalContext {
    columns: HashMap<String, Vec<f64>>,
    dim_string_columns: HashMap<String, Vec<String>>,
    item_date_ranges: HashMap<(i64, String), ItemDateRange>,
    time_values: Vec<String>,
    integer_columns: HashSet<String>,
}
```

The issue is that child evaluators can clone the full context.

### Current Project Code Pattern

Real project code:

```rust
// File:
// modelAPI/omni-calc/src/engine/exec/formula_eval.rs

fn with_raw_properties(&mut self) -> Self {
    self.raw_property_context_clone_count += 1;
    Self {
        ctx: self.ctx.clone(),
        warnings: Vec::new(),
        actuals_context: self.actuals_context.clone(),
        prior_called: false,
        property_filter_context: PropertyFilterContext::without_filtering(),
        last_result_is_integer: false,
        raw_property_context_clone_count: 0,
    }
}
```

This happens when a function needs a child evaluator, for example inside Blox functions where property filtering must behave differently.

### Current Code Example With Output Comments

```rust
use std::collections::HashMap;

#[derive(Clone)]
struct EvalContext {
    columns: HashMap<String, Vec<f64>>,
}

fn main() {
    let mut ctx = EvalContext { columns: HashMap::new() };       // output: empty context
    ctx.columns.insert("ind100".to_string(), vec![1.0, 2.0]);    // output: one column
    ctx.columns.insert("ind200".to_string(), vec![3.0, 4.0]);    // output: two columns

    let child_ctx = ctx.clone();                                 // output: clones HashMap and Vec values

    println!("{}", ctx.columns.len());                           // output: 2
    println!("{}", child_ctx.columns.len());                     // output: 2
    println!("{}", ctx.columns["ind100"].as_ptr() == child_ctx.columns["ind100"].as_ptr()); // output: false
}
```

### Whole Data Structure Output Before Fix

```text
Parent EvalContext
  columns:
    "ind100" -> ptr 0xAAA -> [1.0, 2.0]
    "ind200" -> ptr 0xBBB -> [3.0, 4.0]
  warnings:
    parent warnings

Child EvalContext
  columns:
    "ind100" -> ptr 0xCCC -> [1.0, 2.0]
    "ind200" -> ptr 0xDDD -> [3.0, 4.0]
  warnings:
    empty child warnings

Child needs different local warning/filter behavior, but copied shared input data too.
```

### What Is Wrong

The child evaluator needs its own local state:

```text
warnings
prior_called
property_filter_context
last_result_is_integer
```

But it does not need a deep copy of stable input columns.

Simple overhead:

```text
child evaluator creation = full context clone
full context clone = clone HashMap + clone Vec columns + clone strings
```

Deep Rust explanation:

```text
HashMap<String, Vec<f64>>::clone()
  clones the hash table
  clones every String key
  clones every Vec<f64> value
  allocates new buffers for those Vec values
```

### Better Code After Fix

Split shared read-only data from local evaluator state.

```rust
use std::collections::HashMap;
use std::sync::Arc;

struct EvalSharedContext {
    columns: HashMap<String, Arc<[f64]>>,
}

struct FormulaEvaluator {
    shared: Arc<EvalSharedContext>,
    warnings: Vec<String>,
    property_filtering_enabled: bool,
}

impl FormulaEvaluator {
    fn with_raw_properties(&self) -> Self {
        Self {
            shared: Arc::clone(&self.shared),          // output: cheap pointer clone
            warnings: Vec::new(),                      // output: child gets own warnings
            property_filtering_enabled: false,         // output: child gets own filter behavior
        }
    }
}

fn main() {
    let mut columns = HashMap::new();
    columns.insert("ind100".to_string(), Arc::from([1.0, 2.0])); // output: one shared column

    let parent = FormulaEvaluator {
        shared: Arc::new(EvalSharedContext { columns }),         // output: shared context strong_count=1
        warnings: vec![],
        property_filtering_enabled: true,
    };

    let child = parent.with_raw_properties();                    // output: shared context strong_count=2

    println!("{}", parent.shared.columns.len());                 // output: 1
    println!("{}", child.shared.columns.len());                  // output: 1
    println!("{}", Arc::ptr_eq(&parent.shared, &child.shared));  // output: true
}
```

### Whole Data Structure Output After Fix

```text
Shared Eval Context
  columns:
    "ind100" -> Arc ptr 0xAAA -> [1.0, 2.0]
    "ind200" -> Arc ptr 0xBBB -> [3.0, 4.0]

Parent FormulaEvaluator
  shared -> EvalSharedContext ptr 0x111
  warnings -> parent local Vec
  property_filtering_enabled -> true

Child FormulaEvaluator
  shared -> EvalSharedContext ptr 0x111
  warnings -> child local Vec
  property_filtering_enabled -> false
```

### Impact Of Improvement

This fix helps because:

```text
Before:
  Child evaluator copies large shared input data.

After:
  Child evaluator shares input data and only owns its local mutable fields.
```

Deep Rust impact:

```text
1. Reduces deep clones inside formula evaluation.
2. Makes child evaluators cheap to create.
3. Keeps warning collection independent and correct.
4. Keeps property filtering behavior independent and correct.
5. Allows future formula execution to borrow/share more data safely.
```

---

## Source Issue 21 - Structured Typed IDs And Gradual Migration

### Simple Meaning

The engine hides real IDs inside strings.

Examples:

```text
"ind219049" means indicator id 219049
"b36420" means block id 36420
"_33542" means dimension id 33542
"dim33548___prop71132" means dimension 33548 property 71132
```

This is easy to read, but expensive and risky inside the engine.

### Current Project Code Pattern

Real project code:

```rust
// File:
// modelAPI/omni-calc/src/engine/exec/steps/mod.rs

pub fn get_indicator_for_node<'a>(
    node_id: &str,
    block_spec: &'a BlockSpec,
) -> Option<&'a IndicatorSpec> {
    let id_str = node_id.strip_prefix("ind")?;
    let id: i64 = id_str.parse().ok()?;
    block_spec.indicators.iter().find(|i| i.id == id)
}
```

Real project code:

```rust
// File:
// modelAPI/omni-calc/src/engine/exec/executor.rs

let ind_id: i64 = node_id
    .strip_prefix("ind")
    .and_then(|s| s.parse().ok())
    .unwrap_or(0);
```

Real project code:

```rust
// File:
// modelAPI/omni-calc/src/engine/exec/function_impl/lookup.rs

let dim_part = parts[0].trim_start_matches("dim");
let prop_part = parts[1].trim_start_matches("prop");
let dim_id = dim_part.parse::<i64>().ok()?;
let prop_id = prop_part.parse::<i64>().ok()?;
```

### Current Code Example With Output Comments

```rust
fn parse_indicator_id(node_id: &str) -> i64 {
    node_id
        .strip_prefix("ind")                 // output: Some("219049") for "ind219049"
        .and_then(|s| s.parse::<i64>().ok()) // output: Some(219049)
        .unwrap_or(0)                        // output: 219049, or 0 if parse fails
}

fn main() {
    let good = parse_indicator_id("ind219049");      // output: 219049
    let bad = parse_indicator_id("indicator219049"); // output: 0

    println!("{}", good);                            // output: 219049
    println!("{}", bad);                             // output: 0
}
```

### Whole Data Structure Output Before Fix

```text
Current node IDs
[
  "ind219049",
  "ind219050",
  "b36420",
  "dim33548___prop71132"
]

Every time code needs the numeric ID:
  "ind219049"
    strip_prefix("ind") -> "219049"
    parse::<i64>() -> 219049

Bad string:
  "indicator219049"
    strip_prefix("ind") -> Some("icator219049")
    parse::<i64>() -> None
    unwrap_or(0) -> 0
```

### What Is Wrong

There are two problems.

Performance problem:

```text
The same strings are parsed repeatedly in hot paths.
```

Correctness and debugging problem:

```text
Invalid IDs can silently become 0 when unwrap_or(0) is used.
Then logs or warnings may point to id 0 instead of the real bad value.
```

Deep Rust explanation:

```text
String ID path:
  validate prefix
  slice string
  parse ASCII digits
  handle Option/Result
  maybe fallback to 0

Typed ID path:
  carry numeric ID directly
  compare numeric ID directly
  compiler prevents mixing block ID and indicator ID if types are separate
```

### Better Code After Fix

```rust
#[derive(Copy, Clone, Debug, PartialEq, Eq, Hash)]
struct IndicatorId(i64);

#[derive(Copy, Clone, Debug, PartialEq, Eq, Hash)]
struct BlockId(i64);

#[derive(Copy, Clone, Debug, PartialEq, Eq, Hash)]
struct DimensionId(i64);

#[derive(Copy, Clone, Debug, PartialEq, Eq, Hash)]
struct PropertyId(i64);

#[derive(Copy, Clone, Debug, PartialEq, Eq, Hash)]
enum NodeId {
    Indicator(IndicatorId),
    Block(BlockId),
    DimensionProperty(DimensionId, PropertyId),
}

fn main() {
    let node = NodeId::Indicator(IndicatorId(219049)); // output: typed indicator node

    match node {
        NodeId::Indicator(id) => println!("{:?}", id), // output: IndicatorId(219049)
        NodeId::Block(id) => println!("{:?}", id),     // output: not executed
        NodeId::DimensionProperty(dim, prop) => println!("{:?} {:?}", dim, prop), // output: not executed
    }
}
```

### Whole Data Structure Output After Fix

```text
Typed node IDs
[
  NodeId::Indicator(IndicatorId(219049)),
  NodeId::Indicator(IndicatorId(219050)),
  NodeId::Block(BlockId(36420)),
  NodeId::DimensionProperty(DimensionId(33548), PropertyId(71132))
]

Lookup:
  IndicatorId(219049) == IndicatorId(219050) -> false
  IndicatorId(219049) == IndicatorId(219049) -> true

No prefix stripping.
No string parsing.
No accidental fallback to 0.
```

### Impact Of Improvement

This fix helps because:

```text
Before:
  IDs are text.
  Code repeatedly parses text into numbers.
  Bad text can become 0.

After:
  IDs are numbers with meaning.
  Code compares IDs directly.
  Invalid strings are rejected at the boundary.
```

Deep Rust impact:

```text
1. `IndicatorId(i64)` is small and Copy.
2. Equality is integer equality.
3. Hashing is numeric hashing.
4. Pattern matching separates block IDs, indicator IDs, dimension IDs, and property IDs.
5. The compiler helps prevent using a BlockId where an IndicatorId is expected.
6. Parsing happens once at the API/payload boundary instead of repeatedly in hot execution loops.
```

---

## Source Issue 7 + 8 + 11 + 13 + 19 + 21 Together

These issues are connected.

The clean final direction is:

```text
Typed IDs
  -> make IDs numeric and safe.

String interning
  -> reduce repeated string work for names/keys that still need text identity.

ColumnStore
  -> make column lookup direct and preserve output order.

Arc column storage
  -> make columns cheap to pass and snapshot.

Shared evaluator context
  -> make formula child evaluators cheap.

Snapshot-friendly state
  -> prepare for safe future scheduler/parallel execution.
```

## Final Simple Team Explanation

The current engine often does work like this:

```text
Find string column by scanning a Vec.
Clone the column Vec.
Parse IDs from strings.
Build new string keys.
Clone evaluator context for child formula evaluation.
Repeat this for many indicators and rows.
```

The improved engine should do work like this:

```text
Use typed numeric IDs.
Use indexed column storage.
Share immutable column buffers with Arc.
Use interned symbols for repeated hot strings.
Share formula input context and keep only local evaluator state separate.
Create cheap snapshots for future scheduling.
```

In real performance terms:

```text
Less repeated scanning.
Less repeated parsing.
Less repeated string allocation.
Less repeated Vec copying.
Less memory bandwidth pressure.
Better CPU cache behavior.
Cleaner foundation for safe parallel execution.
```

## Presentation Closing

The fixes do not change the business meaning of model calculations. The intended output stays the same.

The improvement is in how the Rust engine reaches that output:

```text
Current path:
  correct output, but too much repeated internal work

Improved path:
  same output, less CPU, less memory copying, safer future scheduler design
```
