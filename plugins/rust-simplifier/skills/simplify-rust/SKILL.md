---
name: simplify-rust
description: >
  Simplifies Rust code toward plain, data-oriented, idiomatic Rust while preserving exact
  behavior. Use after writing or changing Rust, when reviewing a Rust diff or crate, or when
  asked to simplify, clean up, or make Rust more idiomatic.
---

# Rust Simplifier

Refine Rust for clarity **while preserving exact behavior**: the same outputs, returned errors, panics callers rely on, ordering and public API.

## Principles

- **Plain Rust is plenty.** Structs, enums, functions and modules carry most programs. Traits, generics, macros, builders and async are paid for when the code already needs them.
- **The one-sitting test.** A reader opens a module and predicts what it does in one pass.
- **Make illegal states unrepresentable.** Invariants live in types, and the runtime checks they replace get deleted.
- **Data first.** Model the data, then write functions over it.
- **Borrow before you clone.** Ownership is design; fix a borrow error by restructuring the borrow.
- **Validate at the edge, trust inside.** Necessary complexity concentrates at I/O, SQL, FFI and parsing boundaries.
- **Obviously correct, not fewest lines.** Remove accidental complexity; keep the necessary kind visible and in one place.

## Process

1. **Scope** — the Rust written or changed this session (`git diff`). A whole-crate review happens on request.
2. **Standards** — read `CLAUDE.md`, `AGENTS.md`, `sdd/standards/` and `docs/ARCHITECTURE.md`. A rule stated there wins over this skill.
3. **Baseline** — run `cargo test` and `cargo clippy --all-targets` and record existing failures, before the first edit.
4. **Find** — match the code against the index below, and read the `references/patterns.md` section for each match you intend to change.
5. **Gate** — for a whole-crate review, or any change that touches the public API, list the findings grouped by section and let the user pick. Session-scoped refinements continue straight to step 6.
6. **Refine** in small, behavior-preserving steps.
7. **Verify:**
   ```bash
   cargo fmt --all
   cargo clippy --all-targets --all-features -- -D warnings
   cargo test --all-features
   ```
8. **Report** what changed, grouped by section, and what you left alone on purpose.

## Index

Spot the shape on the left, refine toward the target, and read the section in `references/patterns.md` for the reasoning, the cases to leave alone, and an example.

| Spot | Target | § |
|---|---|---|
| `.unwrap()` / `.expect()` that input can reach in library code | `?` with a typed error | 1 |
| `String` or `Box<dyn Error>` in a library's public errors | `thiserror` enum; `anyhow` + context in binaries | 1 |
| `match` arms that only re-wrap an error | `?` | 1 |
| `.clone()` added to satisfy the borrow checker | restructured borrow, `std::mem::take` | 2 |
| `String` / `Vec<T>` / `PathBuf` parameters that are only read | `&str`, `&[T]`, `impl AsRef<Path>` | 2 |
| Boolean flags, `Option`s that are set together | `enum` | 3 |
| Raw integers for ids or units that are easy to swap | newtypes | 3 |
| `_ =>` arm over your own enum | exhaustive `match` | 3 |
| Trait with one implementation | concrete type | 4 |
| `Box<dyn Trait>` over a closed set | `enum` + `match` | 4 |
| `Box<dyn Fn>` parameter | `impl FnMut` | 4 |
| Builder for a handful of required fields | `new(a, b, c)` or a struct literal | 4 |
| `pub` everywhere, `utils.rs`, logic in `main.rs` | `pub(crate)`, functions beside their data, thin binary | 5 |
| `Manager` / `Service` / `Helper` types, `get_` getters | types named for what they are | 5 |
| Nested `if let` / `match` pyramids | `let ... else`, `?`, early return | 6 |
| `collect()` into a `Vec` only to iterate it | the iterator itself | 6 |
| `Arc<Mutex<T>>` passed around, `async` without concurrent I/O | single owner, channels, `rayon`, sync code | 7 |
| `format!` into SQL, panics on malformed bytes, logic in FFI glue | bound parameters, bounds-checked parsing, thin wrappers | 8 |
| `mockall` over your own code | temp files, in-memory SQLite, fixtures | 9 |
| Lints configured per crate or absent | `[workspace.lints]` | 10 |
