# Rust patterns

One section per concept: the target, why it holds, when to leave code as it is, and an example. `SKILL.md` indexes these sections by the code shape to spot.

> **Sources.** Distilled from the [Rust API Guidelines](https://rust-lang.github.io/api-guidelines/), [Clippy](https://rust-lang.github.io/rust-clippy/master/) lint rationale, the [Rust Design Patterns](https://rust-unofficial.github.io/patterns/) book, Andrew Gallant's [error handling in Rust](https://blog.burntsushi.net/rust-error-handling/), and Alex Kladov's essays on [large Rust workspaces](https://matklad.github.io/2021/08/22/large-rust-workspaces.html) and [ARCHITECTURE.md](https://matklad.github.io/2021/02/06/ARCHITECTURE.md.html). Paraphrased; read the originals for nuance.

## Contents
1. Errors
2. Ownership and borrowing
3. Types
4. Traits, generics and dispatch
5. Modules, visibility and naming
6. Control flow and iterators
7. Concurrency
8. Boundaries: SQL, binary parsing, FFI
9. Testing
10. Workspace lints and CI

---

## 1. Errors

**Target.** Library crates return a `thiserror` enum, one per crate or per module where domains differ. Binaries and tests use `anyhow` with `.context(...)`. Errors propagate with `?`, with context added where the error would otherwise be ambiguous.

**Why.** A library's errors are part of its API: callers match on them and show a person something actionable. A binary is the final caller, so it only needs context. Panics are for bugs; anything a user, a file or the network can trigger returns `Err`.

**Leave it** when `unwrap()` is in a test, where it gives a clear panic location, or when `expect` guards an invariant proven locally, with a message saying why and a scoped lint allowance.

```rust
// crates/core/src/error.rs
#[derive(Debug, thiserror::Error)]
pub enum StoreError {
    #[error("store is locked by another process (lock file at {0})")]
    Locked(PathBuf),
    #[error("could not open store at {path}")]
    Open { path: PathBuf, #[source] source: std::io::Error },
    #[error("unsupported schema version {0}")]
    UnsupportedVersion(u32),
    #[error("invalid status code {0}")]
    InvalidStatus(i64),
    #[error(transparent)]
    Sqlite(#[from] rusqlite::Error),
}
pub type Result<T> = std::result::Result<T, StoreError>;
```

```rust
// crates/cli/src/main.rs
fn main() -> anyhow::Result<()> {
    let args = Args::parse();
    let store = core::open(&args.path)
        .with_context(|| format!("opening {}", args.path.display()))?;
    println!("{}", serde_json::to_string_pretty(&store.summary())?);
    Ok(())
}
```

A `match` whose arms only re-wrap the error collapses to `?`, since `#[from]` does the conversion:

```rust
let rows = stmt.query_map([], Order::from_row)?;
```

A justified `expect`:

```rust
#[allow(clippy::expect_used, reason = "pattern is a compile-time constant")]
let re = Regex::new(r"^[0-9a-f]{32}$").expect("static regex is valid");
```

## 2. Ownership and borrowing

**Target.** Parameters borrow (`&str`, `&[T]`, `&Path`, `impl AsRef<Path>`) unless the function stores or consumes the value. Functions return owned values when the caller needs them, and iterators or slices when it only reads. A borrow-checker error is fixed by restructuring, in this order:

1. Borrow for a shorter scope.
2. Split the struct so disjoint fields are borrowed separately.
3. Pass the callee what it needs, not the whole struct.
4. Move ownership deliberately (`std::mem::take`, returning values).
5. Clone, knowingly, when the data is small or truly needs two owners.

**Why.** A borrow-checker error usually points at an unclear ownership story. A `.clone()` silences it and keeps the story unclear, at a hidden cost. `Rc<RefCell<_>>`, `Arc<Mutex<_>>` and `unsafe` move the same problem to runtime.

**Leave it** at step 5, and use `Cow<'_, str>` only where both the borrowed and owned paths are common and the difference is measured.

```rust
fn count_matching(names: &[String], prefix: &str) -> usize {
    names.iter().filter(|n| n.starts_with(prefix)).count()
}

pub fn open(path: impl AsRef<Path>) -> Result<Store> { let path = path.as_ref(); /* ... */ }
```

Split the borrow so the callee takes only what it needs:

```rust
fn log(out: &mut Vec<String>, name: &str) { out.push(name.to_owned()); }
log(&mut self.lines, &self.config.name);   // replaces cloning config.name first
```

Take instead of clone-and-clear:

```rust
let batch = std::mem::take(&mut self.pending);
```

Return an iterator when callers only read:

```rust
pub fn overdue(orders: &[Order], today: Date) -> impl Iterator<Item = &Order> {
    orders.iter().filter(move |o| o.due < today)
}
```

## 3. Types

**Target.** Invariants live in types. Boolean flags and `Option`s that are set together become an `enum`. Ids and units that are easy to swap become newtypes. A value that must be validated gets a type whose only constructor validates. A `match` over your own enum is exhaustive. Derives cover what is used, with `Copy` for small value types.

**Why.** Types are the cheapest tests: the compiler checks them on every build, and the defensive checks they replace can be deleted. That is compression: fewer lines, stronger guarantees. A `_ =>` arm silently absorbs a variant added later.

**Leave it** when the enum is foreign and `#[non_exhaustive]`, which requires the wildcard, and when mixing two id types is no real risk.

```rust
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
enum Status { Draft, Submitted, Cancelled }   // replaces `submitted: bool, cancelled: bool`

impl TryFrom<i64> for Status {
    type Error = StoreError;
    fn try_from(v: i64) -> Result<Self> {
        match v {
            0 => Ok(Status::Draft),
            1 => Ok(Status::Submitted),
            2 => Ok(Status::Cancelled),
            other => Err(StoreError::InvalidStatus(other)),
        }
    }
}
```

```rust
#[derive(Debug, Clone, Copy, PartialEq, Eq, Hash)]
pub struct OrderId(pub i64);
#[derive(Debug, Clone, Copy, PartialEq, Eq, Hash)]
pub struct CustomerId(pub i64);
// fn assign(order: OrderId, to: CustomerId) can't be called with the arguments swapped
```

```rust
pub struct Percent(u8);
impl Percent {
    pub fn new(value: u8) -> Option<Self> { (value <= 100).then_some(Self(value)) }
    pub fn get(self) -> u8 { self.0 }
}
```

```rust
enum Location { Unknown, Gps { lat: f64, lon: f64 } }   // replaces has_gps + two Options
```

## 4. Traits, generics and dispatch

**Target.** Concrete types by default. A trait earns its place with two or more real implementations or a published extension point. A closed set of variants is an `enum` + `match`. Callbacks and single-use polymorphism take `impl Trait` or a generic; `dyn` is a deliberate choice for heterogeneous collections or to cut compile-time bloat. A function beats `macro_rules!` unless the macro removes repetition functions can't. A struct literal or `new(a, b, c)` beats a builder for fewer than about five fields with no optional combinations. Lifetimes are elided unless the compiler needs them or a name documents a relationship.

**Why.** Every mechanism has a price every future reader pays:

| Mechanism | Price |
|---|---|
| Trait | A second place to look; dispatch to reason about |
| Generic parameter | Longer signatures, bounds that leak outward, slower compiles |
| `Box<dyn Trait>` | Lost exhaustiveness, allocation, object-safety constraints |
| Macro | Code you can't read at the call site; worse errors |
| Builder | A second type mirroring the first |

Pay it when the code already needs the capability.

**Leave it** when a trait has real implementations or a documented extension point, or a generic removes real duplication. Parameterizing over a trait only so a test can mock it is not real variation; see section 9.

```rust
pub struct Blobs { root: PathBuf }   // replaces `trait BlobStore` with its single `FsBlobStore` impl
impl Blobs { pub fn read(&self, key: &str) -> Result<Option<Vec<u8>>> { /* ... */ } }
```

```rust
#[derive(Debug, Clone, Copy)]
enum SortKey { Name, Created, Size }   // replaces Vec<Box<dyn SortKey>>
impl SortKey {
    fn column(self) -> &'static str {
        match self { SortKey::Name => "name", SortKey::Created => "created_at", SortKey::Size => "size" }
    }
}
```

```rust
pub fn import(rows: &[Row], mut progress: impl FnMut(Stage, f32)) -> Result<()>;   // replaces Box<dyn Fn(Stage, f32)>
```

## 5. Modules, visibility and naming

**Target.**
- One concept per module. Items are private or `pub(crate)`; `pub` is reserved for what `lib.rs` re-exports.
- Functions sit beside the data they work on: methods when the data owns the behavior (`order.is_overdue()`), free functions when behavior spans types (`reconcile(&ledger, &mut orders)`).
- Binaries parse arguments, call the library and print. Logic, SQL and parsing live in the library crate.
- Crates split along real compile-time and API boundaries (a core library, a CLI, an FFI layer). The dependency graph stays shallow and one-directional: core never depends on the CLI or FFI crates.
- Names say what a thing is: `Ledger`, `Invoice`, `open_snapshot`. Getters are bare (`order.total()`). Constructors: `new` (infallible), `try_new` / `open` / `parse` (fallible), `from_*` / `with_*` for alternates. Conversions follow std: `as_*` (cheap borrow), `to_*` (expensive), `into_*` (consumes), and `From` / `TryFrom` impls carry the rest.

**Why.** A small public surface is a small set of promises. Role names like `OrderManager` or `ReportService` and grab-bag modules like `utils.rs` pull behavior away from the data it belongs to.

**Leave it** when a file runs past about 400 lines but still holds one concept. That length is a reason to look, not a rule.

```
crates/core/src/
├── lib.rs        // pub use of the API only
├── error.rs
├── store.rs      // one concept per module
├── import/
│   ├── mod.rs
│   └── csv.rs
└── query.rs
```

```rust
// lib.rs
mod error; mod import; mod query; mod store;
pub use error::{Result, StoreError};
pub use query::{OrderFilter, OrderPage};
pub use store::Store;
```

## 6. Control flow and iterators

**Target.** Flat control flow: `let ... else`, `?` and early returns. Iterator chains when they read in one pass; a plain `for` loop when the chain would need comments, tuple folds or mutable juggling. `Option` / `Result` combinators (`map`, `and_then`, `ok_or_else`, `unwrap_or_default`) where they read more directly than a `match`. Iterators stay lazy until something needs a collection.

```rust
let Some(entry) = cache.get(&key) else { return Ok(None) };
Ok(entry.versions.iter().find(|v| v.size >= min).map(|v| v.bytes.clone()))
```

```rust
for id in orders.iter().map(|o| o.id) { index(id)?; }   // replaces collecting ids into a Vec first
```

A loop with two accumulators reads better than a fold over a tuple:

```rust
let (mut paid, mut open) = (0, 0);
for o in orders { if o.paid { paid += 1 } else { open += 1 } }
```

## 7. Concurrency

**Target.** Escalate in order: sequential, then `rayon` for CPU-bound batches, then threads with channels, then async for real concurrent I/O. Each piece of mutable state has one owner; workers receive owned inputs and send results back. Locks, std or async, are released before an `.await` or long work.

**Why.** `async` costs a runtime, colored functions, `Send` bounds and harder stack traces. `Arc<Mutex<_>>` costs shared mutable state, lock ordering and contention. Both are worth it only for the concurrency they buy.

**Leave it** when a library is sync: it stays sync even if one caller is async.

```rust
let results: Vec<_> = ids.par_iter().map(|id| process(*id)).collect::<Result<_>>()?;
```

## 8. Boundaries: SQL, binary parsing, FFI

Read the subsection for a boundary the code actually has.

**Target.** Validate and convert once, at the edge, into domain types; inside, trust them. Boundary modules stay small, heavily tested and free of business logic. `unsafe` lives only in a module that exists for it, and each block carries a `// SAFETY:` comment.

**Why.** Some complexity is necessary. Contained at the edge it is cheap; leaking inward, it spreads re-checks everywhere.

### SQL

Shown with `rusqlite`; the rules hold for `sqlx` and `diesel`.

```rust
pub fn orders_for(conn: &Connection, customer: CustomerId, limit: u32) -> Result<Vec<Order>> {
    let mut stmt = conn.prepare_cached(
        "SELECT id, total, status FROM orders WHERE customer_id = ?1 ORDER BY created_at, id LIMIT ?2")?;
    let rows = stmt.query_map(params![customer.0, limit], Order::from_row)?;
    Ok(rows.collect::<rusqlite::Result<_>>()?)
}

impl Order {
    fn from_row(r: &rusqlite::Row<'_>) -> rusqlite::Result<Self> {
        Ok(Self { id: OrderId(r.get(0)?), total: r.get(1)?, status: r.get(2)? })
    }
}
```
- Values are bound parameters; dynamic column names come from an enum (`SortKey::column()` in section 4).
- A database the code only reads is opened with `OpenFlags::SQLITE_OPEN_READ_ONLY`.
- Pagination is keyset: `WHERE (sort_key, id) > (?1, ?2) ORDER BY sort_key, id LIMIT ?3`.
- Threads each get a connection from a small pool, and writes go through one writer.

### Binary parsing

```rust
fn read_u32_be(buf: &[u8], at: usize) -> Result<u32> {
    let end = at.checked_add(4).ok_or(ParseError::Truncated { at })?;
    let Some(&[a, b, c, d]) = buf.get(at..end) else {
        return Err(ParseError::Truncated { at });
    };
    Ok(u32::from_be_bytes([a, b, c, d]))
}
```
- Offsets computed from file data use `checked_add`; slicing uses `get`, which returns `None` on truncated input.
- The parser is pure (`&[u8] -> Result<Parsed>`), so fixtures test it and a fuzzer can drive it later.

### FFI (UniFFI or C)

- The FFI crate only wraps: convert types, call the core, map errors.
- Exported types are small, stable data records and enums.
- The core error maps into one FFI error enum with human messages; panics are caught before they cross.
- Long operations run off the caller's thread and report progress through a callback interface.

## 9. Testing

**Target.** Tests use real inputs: temp directories, in-memory SQLite, fixture files built with real tools. One behavior per test, named for the behavior (`paging_returns_every_order_once`). Unit tests sit in `#[cfg(test)] mod tests` beside the code; integration tests in `tests/` use only the public API.

**Why.** A trait created so `mockall` can implement it tests the mock, and the mock drifts from reality.

**Leave it** when a test needs private or slow data: mark it `#[ignore]` and read the data path from an env var.

```rust
#[test]
fn refuses_store_with_live_lock() {
    let dir = tempfile::tempdir().unwrap();
    let path = fixture_store(dir.path());
    std::fs::write(path.with_extension("lock"), b"x").unwrap();
    assert!(matches!(Store::open(&path), Err(StoreError::Locked(_))));
}
```

## 10. Workspace lints and CI

**Target.** Clippy with `-D warnings` is the floor. Stricter lints live at the workspace level, so tools enforce the standard. A lint is silenced locally, with `#[allow(..., reason = "...")]`, never globally to turn CI green.

```toml
# Cargo.toml (workspace)
[workspace.lints.rust]
unsafe_code = "deny"            # a crate that needs it (FFI glue) adds #![allow(unsafe_code)] with a reason

[workspace.lints.clippy]
all = { level = "warn", priority = -1 }
unwrap_used = "warn"            # allowed in tests via #[cfg_attr(test, allow(clippy::unwrap_used))]
expect_used = "warn"
needless_pass_by_value = "warn"
redundant_clone = "warn"        # nursery lint; drop it if it proves noisy
large_enum_variant = "warn"
```
```toml
# each crate's Cargo.toml
[lints]
workspace = true
```
