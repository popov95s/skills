# Lean code: Rust

The rules are in [SKILL.md](../SKILL.md). This file shows how to write them in Rust 2021 and
later. Which structure fits which problem: [patterns.md](patterns.md).

## Types

- Kinds are enums. Variants are enums with data: `enum Lookup { Found(Doc), Missing }`.
- A `match` on your own enum has no `_ =>` arm, so a new variant fails to compile. A `_` that
  returns a fallback hides the next kind. List the variants that share an arm with `|`.
- Strings from outside (grammar node kinds, CLI words, config keys) are matched in one adapter
  function that turns them into your enum. `_ =>` is fine there, because the input is open.
  Code past the adapter matches the enum.
- A mode is an enum, never a `bool` parameter. `fn changed_paths(base, staged: bool)` becomes
  `fn changed_paths(base, source: DiffSource)` with `enum DiffSource { WorkingTree, Index }`.
  A `bool` field that states a fact (`is_external`, `is_async`) is data and is fine.
- Several `bool` settings that travel together are a struct with named fields or an enum, not
  a longer parameter list.
- An id is a newtype (`struct FileId(u32)`), built in one place. No `i as u32` at every call
  site.

## Casts and panics

- No narrowing `as` (`usize as u32`, `i64 as u8`): it truncates silently. Use `try_from`, or a
  newtype built in one place. Widening to `usize`, `u64` or `f64` for an index or a ratio is fine.
- No `.unwrap()` outside tests. `.expect("...")` only for a fact the code itself guarantees
  (a built-in glob, a regex literal), with the reason in the message.
- No `panic!` for a case the types can rule out. Make the type rule it out (an exhaustive
  `match`) instead of a runtime "missing" panic.
- `let Some(x) = opt else { return ... };` for guards. No `if let` pyramids.

## Errors

- A module that fails in more than one way returns its own `enum` error, one variant per
  failure mode, with `Display` and `std::error::Error`. Callers `match` on it.
- `anyhow` is for the binary edge: `main`, CLI commands, top-level handlers that only report.
  Library code that a caller must branch on never returns `anyhow::Error`.
- Add context once, where the failure is turned into a report (`.with_context(...)`), not at
  every `?`.
- Use `thiserror` only if the crate already depends on it. Otherwise write the `Display` by
  hand.

## Traits and dispatch

- A trait states one capability, not one type. Several narrow traits combined with a
  supertrait (`trait Language: Parse + Modules + DeadCode`) beat one wide trait.
- Take the trait you use: `&dyn Parse` or `impl Parse`, not the whole combined trait.
- Strategies: one trait, one type per kind in its own file, one picker function with an
  exhaustive `match` that returns `&'static dyn Trait` or `Box<dyn Trait>`. See
  [patterns.md](patterns.md).
- No `HashMap<Kind, Box<dyn Fn>>` and no slice of strategies searched with `find` by kind.
  A static list searched by an outside value (a file extension) is fine; the lookup by your
  own enum is a `match`.
- Feature-gated kinds: gate the enum variant and its picker arm with the same `#[cfg]`, so
  the `match` stays exhaustive in every build.
- No trait with one implementor unless tests inject a fake through it.

## Visibility

- Private by default. `pub(super)` for siblings, `pub(crate)` for the rest of the crate.
- `pub` only for the library's real public API. In a crate whose library only serves its own
  binary, almost everything is `pub(crate)`.

## Files and modules

- `foo/mod.rs` is the index. It holds `mod` lines, `pub use` re-exports and at most a one-line
  `//!`. No types, functions or constants. Logic goes in a file named after it
  (`foo/audit.rs`), re-exported from `mod.rs` so importers keep the same path.
- Pick one module style per repo, `foo/mod.rs` or `foo.rs` next to `foo/`, and don't mix.
- Outside a module, import from the module (`use crate::diff::ChangedFiles`), not its files.
- One responsibility per file, named after what it exports. Under 400 lines.
- `#[path = "..."]` only to wire a test file. Never to pull one module's file into another.

## Tests

- Unit tests live in a sibling file: `audit_tests.rs`, wired from `audit.rs` with
  `#[cfg(test)] #[path = "audit_tests.rs"] mod tests;`. No inline `mod tests { ... }` blocks.
- Integration tests that run the binary or the public API go in `tests/`.
- Use `tempfile` for files on disk, never a fixed path.
- Table cases are an array of `(input, expected)` literals in one loop. See [../testing.md](../testing.md).
- Test-only helpers live in a `#[cfg(test)]` module, not in source with `#[allow(dead_code)]`.

## Suppressions

- No `#[allow(...)]`. `clippy::too_many_arguments` means the parameters form a concept: give
  it a struct. `dead_code` means delete it, or gate it with `#[cfg(...)]`.
- If a lint is a real false positive, ask first, then use `#[expect(lint, reason = "...")]`
  so it fails once the lint stops firing.

## Comments and docs

- `//!` at the top of a file: one line, or nothing. Never a design essay.
- `///` one line, only when the name doesn't say it. No reasoning, no history, no "we tried".
- No `// ---` banners, no `// TODO`, no commented-out code. Tests: the test name carries the
  intent.

## Checks

```
cargo fmt --check
cargo clippy --all-features --all-targets -- -D warnings
cargo test --all-features
```

When you touch `#[cfg(feature = ...)]` code, also build `--no-default-features` and each
feature you gated on its own.
