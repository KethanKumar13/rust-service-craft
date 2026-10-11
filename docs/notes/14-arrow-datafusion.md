# Stage 14: Arrow and DataFusion

> Part of **rust-service-craft**: notes on building production-style backend services in Rust.
> If you come from JavaScript or TypeScript, the comparisons below map Rust concepts to familiar JS ideas (typed arrays, `apache-arrow` in JS, and in-process SQL engines like DuckDB-WASM).

## Contents

1. [Why columnar?](#1-why-columnar)
2. [Arrow's memory model](#2-arrows-memory-model)
3. [Schemas and `RecordBatch`](#3-schemas-and-recordbatch)
4. [Building arrays](#4-building-arrays)
5. [Reading arrays: downcasting](#5-reading-arrays-downcasting)
6. [Compute kernels](#6-compute-kernels)
7. [Zero-copy: slicing and sharing](#7-zero-copy-slicing-and-sharing)
8. [DataFusion: a query engine inside your process](#8-datafusion-a-query-engine-inside-your-process)
9. [SQL and the DataFrame API](#9-sql-and-the-dataframe-api)
10. [Plans: logical, physical, and `EXPLAIN`](#10-plans-logical-physical-and-explain)
11. [Streaming results: `execute_stream` vs `collect`](#11-streaming-results-execute_stream-vs-collect)
12. [Pipeline breakers and memory limits](#12-pipeline-breakers-and-memory-limits)
13. [User-defined functions](#13-user-defined-functions)
14. [Writing results: NDJSON, Parquet, IPC](#14-writing-results-ndjson-parquet-ipc)
15. [How this looks in a production service](#15-how-this-looks-in-a-production-service)
16. [Putting it all together](#16-putting-it-all-together)
17. [Exercises](#17-exercises)
18. [Further reading](#18-further-reading)

---

## 1. Why columnar?

Until now every record was a Rust struct: one `ProjectRow` per row, fields side by side in memory. That's **row-oriented** storage, and it's the right shape for an API that reads and writes one entity at a time.

A data pipeline (ETL: extract, transform, load) is different. It reads millions of rows, keeps a few columns, filters, transforms, and writes them somewhere else. For that work, row structs cost you:

| Problem | Effect |
|---|---|
| One allocation per `String` field per row | Millions of small heap allocations |
| Reading one column touches every field | Poor CPU cache use |
| Per-row function calls for every transform | No SIMD, lots of branching |
| Every library has its own row type | Converting between them copies all the data |

**Columnar** storage flips the layout: all values of one column sit next to each other in one buffer.

```text
Row-oriented (Vec<Project>)            Columnar (Arrow)
┌────┬────────┬────────┐               id:      [ 1 | 2 | 3 ]
│ 1  │ Alpha  │ 120.0  │               name:    [ Alpha | Beta | Gamma ]
├────┼────────┼────────┤               budget:  [ 120.0 | 80.5 | null ]
│ 2  │ Beta   │  80.5  │
├────┼────────┼────────┤
│ 3  │ Gamma  │  null  │
└────┴────────┴────────┘
```

**Apache Arrow** is the standard in-memory columnar format. Because many tools agree on it (DataFusion, Polars, DuckDB, Parquet readers, Spark, pandas), data can move between them **without being converted**.

> **JS comparison:** `Float64Array` is the same idea, one contiguous typed buffer per column instead of an array of objects. The `apache-arrow` npm package uses the exact same memory format as the Rust crate.

---

## 2. Arrow's memory model

An Arrow **array** is one column of a single type. Internally it is a few plain buffers:

```text
Int64Array [7, null, 42]

validity bitmap:  1 0 1          (1 bit per value: is it non-null?)
values buffer:    7 | ? | 42     (8 bytes per value, contiguous)

StringArray ["ab", null, "xyz"]

validity bitmap:  1 0 1
offsets buffer:   0 | 2 | 2 | 5  (where each string starts and ends)
values buffer:    a b x y z      (all bytes back to back)
```

Key points:

- **Nulls are a bitmap, not `Option<T>` per value.** The values buffer has a slot for a null, but its contents are meaningless.
- **Strings are one big byte buffer plus offsets.** There's no per-string allocation.
- **Arrays are immutable.** "Changing" a column means computing a new array.
- **Buffers are reference-counted.** `ArrayRef` is `Arc<dyn Array>`, so cloning an `ArrayRef` costs one atomic increment, not a copy. This is the Stage 0 rule "clone the `Arc`, not the data", built into the format.

---

## 3. Schemas and `RecordBatch`

A **`Schema`** describes the columns: names, types, and whether nulls are allowed. A **`RecordBatch`** is a set of equal-length arrays plus their schema. It's the unit of work in Arrow, usually a few thousand rows.

```toml
datafusion = "47"
```

DataFusion re-exports the Arrow version it was built with as `datafusion::arrow`. Use that path instead of adding a separate `arrow` dependency: two different Arrow versions in one build produce confusing "expected `RecordBatch`, found `RecordBatch`" errors.

```rust
use std::sync::Arc;
use datafusion::arrow::array::{ArrayRef, Float64Array, Int64Array, StringArray};
use datafusion::arrow::datatypes::{DataType, Field, Schema, SchemaRef};
use datafusion::arrow::record_batch::RecordBatch;

fn projects_schema() -> SchemaRef {
    Arc::new(Schema::new(vec![
        Field::new("id", DataType::Int64, false),     // not nullable
        Field::new("name", DataType::Utf8, false),
        Field::new("budget", DataType::Float64, true), // nullable
    ]))
}

fn sample_batch() -> Result<RecordBatch, datafusion::arrow::error::ArrowError> {
    let columns: Vec<ArrayRef> = vec![
        Arc::new(Int64Array::from(vec![1, 2, 3])),
        Arc::new(StringArray::from(vec!["Alpha", "Beta", "Gamma"])),
        Arc::new(Float64Array::from(vec![Some(120.0), Some(80.5), None])),
    ];
    RecordBatch::try_new(projects_schema(), columns)
}
```

`try_new` checks that the column count, types, lengths, and nullability match the schema. A mismatch is an `Err`, not a panic.

| Arrow type | Rust array | Typical source |
|---|---|---|
| `Int32` / `Int64` | `Int32Array` / `Int64Array` | Integer IDs, counts |
| `Float64` | `Float64Array` | Measurements |
| `Utf8` / `LargeUtf8` / `Utf8View` | `StringArray` / `LargeStringArray` / `StringViewArray` | Text |
| `Boolean` | `BooleanArray` | Flags, filter masks |
| `Date32` / `Timestamp(..)` | `Date32Array` / `TimestampMicrosecondArray` | Dates and times |
| `Decimal128(p, s)` | `Decimal128Array` | Money (never use floats for money) |

`SchemaRef` is `Arc<Schema>`. Every batch from the same source shares one schema allocation.

---

## 4. Building arrays

Use `From<Vec<_>>` for small fixed data. For data that arrives value by value (database rows, parsed lines), use a **builder**:

```rust
use datafusion::arrow::array::{Float64Builder, Int64Builder, StringBuilder};

let mut ids = Int64Builder::with_capacity(1024);
let mut names = StringBuilder::new();
let mut budgets = Float64Builder::new();

for row in rows_from_somewhere {
    ids.append_value(row.id);
    names.append_value(&row.name);
    match row.budget {
        Some(b) => budgets.append_value(b),
        None => budgets.append_null(),
    }
}

let columns: Vec<ArrayRef> = vec![
    Arc::new(ids.finish()),
    Arc::new(names.finish()),
    Arc::new(budgets.finish()),
];
```

`finish()` returns the array and resets the builder, so you can reuse it for the next batch.

**Batch size is your memory bound.** When converting a stream of rows into Arrow, call `finish()` every N rows (DataFusion's default is 8,192) and send that batch on. Don't build one giant batch for the whole input, for the same reason you didn't `collect` in Stage 10.

---

## 5. Reading arrays: downcasting

A `RecordBatch` stores columns as `ArrayRef` (`Arc<dyn Array>`), so the concrete type is hidden. To read values, **downcast**:

```rust
use datafusion::arrow::array::{Array, AsArray};
use datafusion::arrow::datatypes::Int64Type;

let ids = batch
    .column(0)
    .as_primitive_opt::<Int64Type>()
    .ok_or_else(|| AppError::Internal("id column is not Int64".into()))?;

for i in 0..ids.len() {
    if ids.is_valid(i) {
        let id: i64 = ids.value(i);
        // ...
    }
}

// Or iterate as Option<T>:
let total: i64 = ids.iter().flatten().sum();
```

| Method | On wrong type |
|---|---|
| `as_primitive::<T>()`, `as_string::<i32>()` | Panics |
| `as_primitive_opt::<T>()`, `as_string_opt::<i32>()` | Returns `None` |
| `as_any().downcast_ref::<Int64Array>()` | Returns `None` (older style) |

Following the repository conventions (no panics in runtime code), use the `_opt` versions and turn `None` into an error.

> Prefer **column-at-a-time** work over loops that call `value(i)` for every row. Section 6 shows why.

---

## 6. Compute kernels

Arrow ships **kernels**: functions that process a whole array at once. They use tight loops, often SIMD, and handle nulls correctly.

```rust
use datafusion::arrow::array::Int64Array;
use datafusion::arrow::compute::{filter_record_batch, kernels::cmp::gt};

// "WHERE id > 1" without SQL
let ids = batch.column(0);
let mask = gt(ids, &Int64Array::new_scalar(1))?;   // BooleanArray [false, true, true]
let filtered = filter_record_batch(&batch, &mask)?;
```

Common kernels:

| Module | Examples |
|---|---|
| `compute::kernels::cmp` | `eq`, `neq`, `lt`, `gt`, `lt_eq`, `gt_eq` |
| `compute::kernels::numeric` | `add`, `sub`, `mul`, `div` |
| `compute::kernels::boolean` | `and`, `or`, `not`, `is_null` |
| `compute` | `filter`, `filter_record_batch`, `take`, `sort_to_indices`, `concat_batches`, `cast` |
| `compute::kernels::aggregate` | `sum`, `min`, `max` |

`cast` is how type conversions happen (`Utf8` → `Int64`, `Int32` → `Float64`). It returns an error for impossible casts and produces nulls for values that can't be converted, depending on the cast options.

In practice you rarely call kernels directly. DataFusion calls them for you when it runs SQL. Knowing they exist explains why DataFusion is fast, and they're the tool to use inside a UDF (section 13).

---

## 7. Zero-copy: slicing and sharing

Because buffers are reference-counted and immutable, many operations **share** memory instead of copying it:

```rust
let first_two = batch.slice(0, 2);          // no copy: same buffers, new offset and length
let only_names = batch.project(&[1])?;      // no copy: keeps one column's Arc
let same = batch.clone();                   // no copy: clones a few Arcs
```

| Operation | Copies data? |
|---|---|
| `RecordBatch::clone`, `ArrayRef::clone` | No |
| `slice`, `project` | No |
| `filter`, `take`, `sort` | Yes, it builds new arrays from the selected rows |
| `concat_batches` | Yes, everything is copied into one batch |

That last row is the columnar version of `collect`. `concat_batches` over a whole input materializes all of it. Treat it the way Stage 10 treated `collect::<Vec<_>>()`: fine for a small known-size result, a bug in a streaming path.

A sliced array still keeps its **whole parent buffer** alive. Holding a 10-row slice of a 100 MB batch keeps 100 MB resident. If you need to keep a small slice long-term, copy it on purpose (for example with `take`) and write down why.

---

## 8. DataFusion: a query engine inside your process

**DataFusion** is a SQL query engine written in Rust on top of Arrow. It isn't a server; it's a library. You give it data sources, it gives you a planner, an optimizer, and a parallel streaming executor.

```text
 SQL text / DataFrame calls
          │
          ▼
   ┌──────────────┐     ┌───────────────┐     ┌────────────────┐
   │ LogicalPlan  │ ──▶ │   optimizer   │ ──▶ │ ExecutionPlan  │ ──▶ stream of RecordBatch
   └──────────────┘     └───────────────┘     └────────────────┘
     "what"              predicate pushdown,     "how": operators,
                         projection pushdown     partitions, threads
```

The entry point is a `SessionContext`:

```rust
use datafusion::prelude::*;

let ctx = SessionContext::new();

ctx.register_batch("projects", sample_batch()?)?;                          // in-memory
ctx.register_csv("events", "data/events.csv", CsvReadOptions::new()).await?;
ctx.register_parquet("history", "data/history/", ParquetReadOptions::default()).await?;
```

| Source | How |
|---|---|
| A `RecordBatch` already in memory | `register_batch`, `read_batch` |
| CSV / NDJSON / Parquet / Avro files | `register_csv`, `register_json`, `register_parquet`, `register_avro` |
| A directory of files | Same functions, pass the directory path |
| A database or custom source | Implement `TableProvider` (exercise 8) |

DataFusion uses Tokio and runs work in parallel across **partitions** (by default one per CPU core). Each partition is an independent stream of batches, so a single query uses every core without you spawning any tasks yourself.

> **JS comparison:** like DuckDB-WASM or `sql.js`, a full SQL engine that runs inside your process on data you hand it, with no database server.

---

## 9. SQL and the DataFrame API

The same query can be written two ways. Both produce the same `LogicalPlan`.

```rust
// SQL
let df = ctx
    .sql("SELECT name, budget FROM projects WHERE budget > 100 ORDER BY budget DESC")
    .await?;

// DataFrame API
let df = ctx
    .table("projects")
    .await?
    .filter(col("budget").gt(lit(100.0)))?
    .select(vec![col("name"), col("budget")])?
    .sort(vec![col("budget").sort(false, true)])?;   // descending, nulls first

df.show().await?;   // prints a table: for debugging only, it collects
```

| | SQL | DataFrame API |
|---|---|---|
| Best for | User-written or config-driven queries | Queries built in code |
| Errors found | At `ctx.sql(...)` (parse and plan time) | At each method call |
| Composing | String building (be careful about injection) | Normal Rust values |

Aggregation:

```rust
let df = ctx
    .table("events")
    .await?
    .aggregate(vec![col("tenant_id")], vec![count(col("id")).alias("events")])?;
```

**Never build SQL from user input with `format!`.** The same injection rules as Stage 6 apply. Use the DataFrame API with `lit(value)`, or SQL with parameter placeholders (`$1`) and `with_param_values`.

---

## 10. Plans: logical, physical, and `EXPLAIN`

`EXPLAIN` shows what DataFusion will actually do:

```rust
ctx.sql("EXPLAIN SELECT name FROM events WHERE tenant_id = 7")
    .await?
    .show()
    .await?;
```

```text
logical_plan  | Projection: events.name
              |   Filter: events.tenant_id = Int64(7)
              |     TableScan: events projection=[tenant_id, name], partial_filters=[events.tenant_id = Int64(7)]
physical_plan | ProjectionExec: expr=[name@1 as name]
              |   CoalesceBatchesExec: target_batch_size=8192
              |     FilterExec: tenant_id@0 = 7
              |       RepartitionExec: partitioning=RoundRobinBatch(8)
              |         DataSourceExec: file_groups=..., projection=[tenant_id, name]
```

What to look for:

| Line | Meaning |
|---|---|
| `projection=[tenant_id, name]` on the scan | **Projection pushdown**: unused columns are never read |
| `partial_filters=[...]` / predicate on the scan | **Predicate pushdown**: Parquet can skip whole row groups |
| `RepartitionExec` | Work is split across partitions (parallelism) |
| `SortExec`, `AggregateExec`, `HashJoinExec` | Operators that may hold a lot of data (section 12) |

`EXPLAIN ANALYZE` runs the query and adds row counts and timings for each operator. Use it the way you'd use `EXPLAIN ANALYZE` in PostgreSQL.

---

## 11. Streaming results: `execute_stream` vs `collect`

A `DataFrame` is lazy. Nothing runs until you ask for results, and **how** you ask matters:

```rust
use futures::StreamExt;

// Materializes everything: Vec<RecordBatch> holding the full result
let all: Vec<RecordBatch> = df.clone().collect().await?;

// Streams: one batch at a time, bounded memory
let mut stream = df.execute_stream().await?;   // SendableRecordBatchStream
while let Some(batch) = stream.next().await {
    let batch = batch?;
    write_somewhere(&batch).await?;
}
```

`SendableRecordBatchStream` is `Pin<Box<dyn RecordBatchStream + Send>>`: a Stage 10 `Stream<Item = Result<RecordBatch, DataFusionError>>` that also knows its schema. Everything from Stage 10 applies:

| Stage 10 rule | Arrow / DataFusion version |
|---|---|
| Don't `collect` large data | Don't call `df.collect()` or `concat_batches` on large results |
| Fixed batch size | `SessionConfig::with_batch_size(8192)` |
| Bounded channel to a producer task | Feed batches into `mpsc::channel(n)` and send them on |
| Stop when the consumer goes away | Dropping the stream cancels the query |
| Prove it streams | First batch before the input ends, bounded resident batches |

`df.show()` and `df.collect()` are fine in tests and for small lookup tables. In a pipeline over user data they're the bug.

To hand your own stream of batches to DataFusion code that expects a `SendableRecordBatchStream`, wrap it:

```rust
use datafusion::physical_plan::stream::RecordBatchStreamAdapter;

let stream: SendableRecordBatchStream =
    Box::pin(RecordBatchStreamAdapter::new(schema.clone(), my_batch_stream));
```

---

## 12. Pipeline breakers and memory limits

Some operators can pass each batch through as soon as it arrives. Others can't produce anything until they've seen **all** of their input. Those are **pipeline breakers**:

| Operator | Streams? | Why |
|---|---|---|
| Scan, filter, projection, `LIMIT`, scalar functions | Yes | Each output batch depends on one input batch |
| `ORDER BY` (full sort) | No | The last input row might sort first |
| `GROUP BY` aggregation | Mostly no | Groups stay open until the input ends |
| Hash join | Build side no, probe side yes | The build side is held in a hash table |
| `ORDER BY ... LIMIT n` | Yes, bounded | Keeps only the top `n` rows |

You can't make a full sort stream, but you **can** bound its memory. DataFusion has a **memory pool**. When an operator would go over its share of the pool, it either **spills** to temp files on disk (sorts, some aggregations) or fails with a "resources exhausted" error instead of using all of the machine's RAM.

```rust
use datafusion::execution::runtime_env::RuntimeEnvBuilder;

let config = SessionConfig::new()
    .with_batch_size(8192)
    .with_target_partitions(4);

let runtime = RuntimeEnvBuilder::new()
    .with_memory_limit(512 * 1024 * 1024, 1.0)   // 512 MB pool, 100% usable
    .build_arc()?;

let ctx = SessionContext::new_with_config_rt(config, runtime);
```

Rules for data pipelines:

- Set a **memory limit** for every query run. An unlimited pool means one large query can take down the whole service.
- Check `EXPLAIN` for `SortExec`, `AggregateExec`, and `HashJoinExec` on inputs that can grow without limit.
- Put the **smaller** table on the build side of a join (the optimizer usually does this when it has statistics).
- A "resources exhausted" error is the memory limit doing its job. Treat it as a failed run with a clear message, not something to retry forever.

---

## 13. User-defined functions

When SQL's built-in functions aren't enough (a custom normalization, a domain-specific lookup), register a **scalar UDF**. It receives whole **arrays**, not single values, so write it with kernels or one tight loop:

```rust
use std::sync::Arc;
use datafusion::arrow::array::{Array, AsArray, StringArray};
use datafusion::arrow::datatypes::DataType;
use datafusion::error::{DataFusionError, Result};
use datafusion::logical_expr::{create_udf, ColumnarValue, Volatility};

fn normalize_name(args: &[ColumnarValue]) -> Result<ColumnarValue> {
    let arrays = ColumnarValue::values_to_arrays(args)?;
    let input = arrays
        .first()
        .and_then(|a| a.as_string_opt::<i32>())
        .ok_or_else(|| DataFusionError::Execution("normalize_name expects one Utf8 argument".into()))?;

    // Option<&str> in, Option<String> out: nulls stay null
    let output: StringArray = input
        .iter()
        .map(|v| v.map(|s| s.trim().to_lowercase()))
        .collect();

    Ok(ColumnarValue::Array(Arc::new(output)))
}

let udf = create_udf(
    "normalize_name",
    vec![DataType::Utf8],
    DataType::Utf8,
    Volatility::Immutable,     // same input -> same output, lets the optimizer fold constants
    Arc::new(normalize_name),
);
ctx.register_udf(udf);

ctx.sql("SELECT normalize_name(name) FROM projects").await?;
```

| `Volatility` | Meaning | Example |
|---|---|---|
| `Immutable` | Output depends only on input | `lower`, `normalize_name` |
| `Stable` | Same within one query | `now()` |
| `Volatile` | Can change on every call | `random()` |

Notes:

- A UDF runs on DataFusion's worker threads. **Don't block** inside it (no synchronous HTTP or database calls). Stage 1's `spawn_blocking` rules apply.
- Return an `Err`, don't panic. A panic in a UDF takes down a query thread.
- Aggregate UDFs (`create_udaf`) and window UDFs also exist, for custom `GROUP BY` and `OVER` logic.

---

## 14. Writing results: NDJSON, Parquet, IPC

Once a pipeline produces batches, they go somewhere. Write **batch by batch** from the stream, never after a `collect`.

**NDJSON**, which connects directly to Stage 10's export endpoint:

```rust
use datafusion::arrow::json::LineDelimitedWriter;

fn batch_to_ndjson(batch: &RecordBatch) -> Result<Bytes, ArrowError> {
    let mut writer = LineDelimitedWriter::new(Vec::new());
    writer.write(batch)?;
    writer.finish()?;
    Ok(Bytes::from(writer.into_inner()))
}
```

**Parquet**, the standard columnar *file* format (compressed, with statistics for predicate pushdown):

```rust
use datafusion::parquet::arrow::ArrowWriter;

let file = std::fs::File::create("out.parquet")?;
let mut writer = ArrowWriter::try_new(file, stream.schema(), None)?;
while let Some(batch) = stream.next().await {
    writer.write(&batch?)?;   // buffers up to one row group, then flushes it
}
writer.close()?;              // writes the footer; without it the file is unreadable
```

Or let DataFusion do it: `df.write_parquet("out/", DataFrameWriteOptions::new(), None).await?`.

| Format | Use for |
|---|---|
| NDJSON / CSV | APIs, humans, spreadsheets |
| Parquet | Storage, data lakes, re-reading with filters |
| Arrow IPC (`arrow::ipc::writer::StreamWriter`) | Sending batches between processes with no conversion |

---

## 15. How this looks in a production service

One way to lay out a data-pipeline crate built on these pieces:

```text
crates/
├── pipeline/src/
│   ├── context.rs          # build SessionContext: batch size, partitions, memory limit
│   ├── sources/            # TableProviders: database readers that produce fixed-size batches
│   ├── transforms/         # UDFs and DataFrame steps; each one streams batch by batch
│   ├── sinks/              # Parquet / NDJSON / database writers that consume streams
│   └── run.rs              # wires source -> transforms -> sink with bounded channels
└── pipeline/tests/
    └── streaming.rs        # early output, bounded resident batches, cancellation
```

Conventions:

- **Arrow from `datafusion::arrow`**, so there's only one Arrow version in the build.
- **Fixed batch size** everywhere. Sources call `finish()` every N rows.
- **`execute_stream`, never `collect`**, for anything whose size depends on input.
- **Memory limit per query run**, with spilling enabled for sorts and aggregations.
- **UDFs are column-at-a-time**, non-blocking, and return errors instead of panicking.
- **Zero-copy by default.** Copies (`filter`, `take`, `concat_batches`) are deliberate and documented.
- **Tenant scoping and validation happen before the query starts**, the same as before the first byte in Stage 10.
- **Tests prove streaming**: the first batch arrives before the source ends, and resident batches stay bounded as input grows.

---

## 16. Putting it all together

A complete program that:

1. builds a `RecordBatch` by hand and filters it with a kernel,
2. writes a 200,000-row CSV file,
3. registers it in a `SessionContext` with a memory limit,
4. registers a UDF and runs SQL with `EXPLAIN`,
5. streams the result batch by batch, encodes each batch as NDJSON, and shows that memory per batch stays fixed.

```toml
[dependencies]
datafusion = "47"
futures = "0.3"
tokio = { version = "1", features = ["full"] }
```

```rust
use std::io::{BufWriter, Write};
use std::sync::Arc;

use datafusion::arrow::array::{Array, ArrayRef, AsArray, Float64Array, Int64Array, StringArray};
use datafusion::arrow::compute::{filter_record_batch, kernels::cmp::gt};
use datafusion::arrow::datatypes::{DataType, Field, Schema};
use datafusion::arrow::json::LineDelimitedWriter;
use datafusion::arrow::record_batch::RecordBatch;
use datafusion::error::{DataFusionError, Result};
use datafusion::execution::runtime_env::RuntimeEnvBuilder;
use datafusion::logical_expr::{create_udf, ColumnarValue, Volatility};
use datafusion::prelude::*;
use futures::StreamExt;

const ROWS: u64 = 200_000;
const BATCH_SIZE: usize = 8192;

// ================= 1. Arrow by hand =================

fn hand_built_batch() -> Result<RecordBatch> {
    let schema = Arc::new(Schema::new(vec![
        Field::new("id", DataType::Int64, false),
        Field::new("name", DataType::Utf8, false),
        Field::new("budget", DataType::Float64, true),
    ]));
    let columns: Vec<ArrayRef> = vec![
        Arc::new(Int64Array::from(vec![1, 2, 3])),
        Arc::new(StringArray::from(vec!["Alpha", "Beta", "Gamma"])),
        Arc::new(Float64Array::from(vec![Some(120.0), Some(80.5), None])),
    ];
    Ok(RecordBatch::try_new(schema, columns)?)
}

// ================= 2. a large input file =================

fn write_csv(path: &std::path::Path) -> std::io::Result<()> {
    let mut out = BufWriter::new(std::fs::File::create(path)?);
    writeln!(out, "id,tenant_id,name,amount")?;
    for id in 1..=ROWS {
        let tenant = id % 5;
        writeln!(out, "{id},{tenant},  Customer-{id}  ,{}", (id % 1000) as f64 / 10.0)?;
    }
    out.flush()
}

// ================= 3. a UDF =================

fn normalize_name(args: &[ColumnarValue]) -> Result<ColumnarValue> {
    let arrays = ColumnarValue::values_to_arrays(args)?;
    let input = arrays
        .first()
        .and_then(|a| a.as_string_opt::<i32>())
        .ok_or_else(|| DataFusionError::Execution("normalize_name expects one Utf8 argument".into()))?;
    let output: StringArray = input.iter().map(|v| v.map(|s| s.trim().to_lowercase())).collect();
    Ok(ColumnarValue::Array(Arc::new(output)))
}

fn batch_to_ndjson(batch: &RecordBatch) -> Result<Vec<u8>> {
    let mut writer = LineDelimitedWriter::new(Vec::new());
    writer.write(batch)?;
    writer.finish()?;
    Ok(writer.into_inner())
}

#[tokio::main]
async fn main() -> Result<()> {
    // ---- 1. kernels on a hand-built batch ----
    let batch = hand_built_batch()?;
    let mask = gt(batch.column(0), &Int64Array::new_scalar(1))?;
    let filtered = filter_record_batch(&batch, &mask)?;
    let slice = batch.slice(0, 2); // zero-copy
    println!(
        "hand-built: {} rows, filtered (id > 1): {} rows, slice: {} rows",
        batch.num_rows(),
        filtered.num_rows(),
        slice.num_rows()
    );

    // ---- 2. input file ----
    let path = std::env::temp_dir().join("rsc_stage14_events.csv");
    write_csv(&path).map_err(DataFusionError::from)?;

    // ---- 3. bounded context ----
    let config = SessionConfig::new()
        .with_batch_size(BATCH_SIZE)
        .with_target_partitions(4);
    let runtime = RuntimeEnvBuilder::new()
        .with_memory_limit(256 * 1024 * 1024, 1.0)
        .build_arc()?;
    let ctx = SessionContext::new_with_config_rt(config, runtime);

    let path_str = path.to_string_lossy().to_string();
    ctx.register_csv("events", &path_str, CsvReadOptions::new()).await?;
    ctx.register_udf(create_udf(
        "normalize_name",
        vec![DataType::Utf8],
        DataType::Utf8,
        Volatility::Immutable,
        Arc::new(normalize_name),
    ));

    // ---- 4. plan ----
    let sql = "SELECT id, normalize_name(name) AS name, amount
               FROM events
               WHERE tenant_id = 3 AND amount > 50";
    ctx.sql(&format!("EXPLAIN {sql}")).await?.show().await?;

    // ---- 5. stream, batch by batch ----
    let mut stream = ctx.sql(sql).await?.execute_stream().await?;

    let mut batches = 0usize;
    let mut rows = 0usize;
    let mut bytes_out = 0usize;
    let mut max_batch_rows = 0usize;
    let mut max_batch_mem = 0usize;
    let mut first_line = String::new();

    while let Some(batch) = stream.next().await {
        let batch = batch?;
        let ndjson = batch_to_ndjson(&batch)?;

        if first_line.is_empty() {
            first_line = String::from_utf8_lossy(&ndjson)
                .lines()
                .next()
                .unwrap_or_default()
                .to_owned();
        }

        batches += 1;
        rows += batch.num_rows();
        bytes_out += ndjson.len();
        max_batch_rows = max_batch_rows.max(batch.num_rows());
        max_batch_mem = max_batch_mem.max(batch.get_array_memory_size());
        // `batch` and `ndjson` are dropped here: only one batch is resident at a time
    }

    println!("first line: {first_line}");
    println!(
        "streamed {rows} rows in {batches} batches, {bytes_out} NDJSON bytes, \
         max rows/batch = {max_batch_rows} (limit {BATCH_SIZE}), max batch memory = {} KB",
        max_batch_mem / 1024
    );

    let _ = std::fs::remove_file(&path);
    Ok(())
}
```

Expected output, shortened. The plan details, batch counts, and memory sizes depend on your DataFusion version and CPU count:

```text
hand-built: 3 rows, filtered (id > 1): 2 rows, slice: 2 rows
+---------------+--------------------------------------------------------------------+
| plan_type     | plan                                                               |
+---------------+--------------------------------------------------------------------+
| logical_plan  | Projection: events.id, normalize_name(events.name) AS name, ...    |
|               |   Filter: events.tenant_id = Int64(3) AND events.amount > ...      |
|               |     TableScan: events projection=[id, tenant_id, name, amount] ... |
| physical_plan | ProjectionExec: ...                                                |
|               |   CoalesceBatchesExec: target_batch_size=8192                      |
|               |     FilterExec: tenant_id@1 = 3 AND amount@3 > 50                  |
|               |       RepartitionExec: partitioning=RoundRobinBatch(4) ...         |
|               |         DataSourceExec: file_groups=..., file_type=csv ...         |
+---------------+--------------------------------------------------------------------+
first line: {"id":53,"name":"customer-53","amount":53.0}
streamed 19960 rows in 4 batches, ... NDJSON bytes, max rows/batch = 8192 (limit 8192), max batch memory = ... KB
```

What each part shows:

| Part | Concept |
|---|---|
| `filter_record_batch` with `gt` | **Kernels**: column-at-a-time compute on Arrow arrays |
| `slice` | **Zero-copy** views over shared buffers |
| `EXPLAIN` | **Projection and predicate pushdown**, **partitions** (parallelism) |
| `normalize_name` | A **UDF** working on whole arrays, with nulls preserved |
| `max rows/batch` ≤ 8192 for any input size | **Fixed batch size**: memory per batch doesn't grow with the file |
| `execute_stream` + per-batch NDJSON | **Streaming output**: the same pattern as Stage 10, with batches instead of rows |

Try breaking it on purpose: replace `execute_stream` with `collect().await?` and print the total memory of all batches. Then increase `ROWS` tenfold and watch that number grow, while the streaming version's `max batch memory` stays the same. Add `ORDER BY name` to the query and look for `SortExec` in the plan: that's a pipeline breaker. Lower the memory limit to 1 MB and see the "resources exhausted" error.

---

## 17. Exercises

- [ ] **1. Arrays by hand:** build a `RecordBatch` with `id: Int64`, `email: Utf8` (nullable), and `active: Boolean` using builders. Include at least two nulls. Print it with `datafusion::arrow::util::pretty::print_batches`.
- [ ] **2. Downcast safely:** write `fn sum_column(batch: &RecordBatch, name: &str) -> Result<i64, AppError>` that finds a column by name, downcasts with `as_primitive_opt::<Int64Type>()`, skips nulls, and returns an error for a missing or wrong-typed column. No panics.
- [ ] **3. Kernels only:** without DataFusion, compute `WHERE active AND id > 10` on your exercise 1 batch with `cmp::gt`, `boolean::and`, and `filter_record_batch`.
- [ ] **4. SQL vs DataFrame:** write the section 16 query a second time with the DataFrame API (`filter`, `select`, `col`, `lit`, and the UDF via `ctx.udf("normalize_name")?.call(...)`). Check that both versions return the same row count and that their `EXPLAIN` outputs match.
- [ ] **5. Read a plan:** run `EXPLAIN ANALYZE` on a `GROUP BY tenant_id` over the section 16 file. Find the aggregate operator, how many rows each operator produced, and where the time went. In two sentences, explain why this query can't stream its first result before the input ends.
- [ ] **6. Memory limit:** add `ORDER BY name` to the section 16 query and run it with memory limits of 256 MB, 4 MB, and 1 MB. Record which runs succeed, spill, or fail, and what the error says.
- [ ] **7. Parquet round trip:** stream the section 16 result into a Parquet file with `ArrowWriter`, batch by batch. Register that file with `register_parquet`, query `WHERE tenant_id = 3`, and use `EXPLAIN` to find the pushed-down predicate.
- [ ] **8. Rows to batches:** write a function that takes a Stage 6 SQLx `fetch` stream of `ProjectRow` and returns a stream of `RecordBatch` of at most 1,024 rows, using builders and `finish()`. Wrap it in `RecordBatchStreamAdapter` and pass it through a DataFusion query.
- [ ] **9. Prove it streams:** for exercise 8, write a check that (a) the first batch is received before the source has produced its last row, and (b) the number of batches produced but not consumed never goes above a fixed number, whether the source has 10,000 or 1,000,000 rows (the Stage 10 section 13 technique).
- [ ] **10. Explain it:** in four or five sentences, explain the columnar memory layout, why cloning an `ArrayRef` is cheap, which operators are pipeline breakers, and why `df.collect()` and `concat_batches` are dangerous in a streaming pipeline.

> When you can solve exercises 1–10 without looking anything up, you're ready for **Stage 15: Stream Pipelines**.

---

## 18. Further reading

- [Apache Arrow columnar format spec](https://arrow.apache.org/docs/format/Columnar.html): validity bitmaps, offsets, and buffers in detail
- [`arrow` crate docs](https://docs.rs/arrow): arrays, builders, `RecordBatch`, and the `compute` kernels
- [DataFusion user guide](https://datafusion.apache.org/user-guide/introduction.html): SQL reference, configuration, and `EXPLAIN`
- [DataFusion library guide](https://datafusion.apache.org/library-user-guide/index.html): `SessionContext`, DataFrames, UDFs, custom `TableProvider`s
- [`datafusion-examples`](https://github.com/apache/datafusion/tree/main/datafusion-examples): runnable examples for UDFs, custom sources, memory limits, and Parquet
- [Andrew Lamb: "DataFusion: A Fast, Embeddable, Modular Analytic Query Engine"](https://dl.acm.org/doi/10.1145/3626246.3653368): the design paper, including streaming execution and spilling
- [`parquet` crate: `ArrowWriter`](https://docs.rs/parquet/latest/parquet/arrow/arrow_writer/struct.ArrowWriter.html): writing Parquet from batches
