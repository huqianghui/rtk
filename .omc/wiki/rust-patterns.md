---
title: Rust Patterns & Conventions
tags: [rust, patterns, conventions, error-handling, regex, testing, dependencies]
category: pattern
created: 2026-04-14
updated: 2026-04-14
---

# Rust Patterns & Conventions

Recurring patterns observed across RTK's 55K-line codebase.

## Error Handling

### anyhow::Result + .context() (120+ uses)

Universal error handling. Import: `use anyhow::{Context, Result};`

```rust
fs::read_to_string(path)
    .with_context(|| format!("Failed to read config: {}", path.display()))?;
```

- `.context("static string")` for static messages
- `.with_context(|| format!(...))` for dynamic messages
- Context strings follow "Failed to ..." pattern

### Fallback Pattern (mandatory for all filters)

```rust
let filtered = filter_output(&output.stdout)
    .unwrap_or_else(|e| {
        eprintln!("rtk: filter warning: {}", e);
        output.stdout.clone()  // Passthrough on failure
    });
```

### Exit Code Propagation

Every `run()` returns `Result<i32>`. Main entry:
```rust
fn main() {
    let code = match run_cli() { Ok(code) => code, Err(e) => { eprintln!(...); 1 } };
    std::process::exit(code);
}
```

Helpers: `exit_code_from_output()`, `exit_code_from_status()` handle Unix signals (128+sig).

Direct `process::exit()` only in: `main()`, security-critical `integrity.rs`, hook `rewrite_cmd.rs` (semantic exit codes 1/2/3).

## Regex Patterns

### lazy_static! (25 blocks, primary pattern)

```rust
lazy_static! {
    static ref ERROR_RE: Regex = Regex::new(r"^error\[").unwrap();
}
```

Module-level for shared patterns; function-scoped for localized ones. `.unwrap()` is acceptable here -- bad regex literals are programming errors.

### OnceLock (newer alternative)

```rust
static RE: OnceLock<Regex> = OnceLock::new();
let re = RE.get_or_init(|| Regex::new(r"...").unwrap());
```

Used in `cargo_cmd.rs` (lines 324, 331, 653) and `telemetry.rs`.

### Known violations (Regex::new in functions)

- `summary.rs`, `grep_cmd.rs` -- dynamic patterns (format-based, legitimate)
- `deps.rs` (lines 79-80, 167) -- static patterns that could be cached
- `local_llm.rs` (lines 142, 181, 208, 222) -- repeated static patterns

### Common regex categories

1. **Error detection:** `r"(?i)^.*error[\s:\[].*$"`, `r"^error\[E\d+\]:.*$"`
2. **Section parsing:** `r"^(.+?)\((\d+),(\d+)\):\s+(error|warning)\s+(TS\d+):\s+(.+)$"`
3. **ANSI stripping:** `r"\x1b\[[0-9;]*[a-zA-Z]"` (core/utils.rs)
4. **Log normalization:** timestamps, UUIDs, hex, large numbers, paths
5. **Build output:** warning/error counts, test results, summary lines

## Testing Patterns

### Test module convention (72 files)

```rust
#[cfg(test)]
mod tests {
    use super::*;
    // tests here
}
```

### count_tokens helper

Canonical version in `core/utils.rs:258`. Also duplicated locally in: gh_cmd, gt_cmd, git, psql_cmd.

```rust
fn count_tokens(s: &str) -> usize { s.split_whitespace().count() }
```

### No insta/snapshot testing

Despite CLAUDE.md recommending it, zero `assert_snapshot!` occurrences in the codebase. Tests use standard assertions.

### Test fixtures

6 files in `tests/fixtures/` (mostly dotnet + golangci). Most tests use inline string literals.

### Integration tests

6 `#[ignore]` tests in git.rs and read.rs -- require git repo or built binary.

## Ownership & Borrowing

### Filter function signatures (100% consistent)

```rust
fn filter_output(input: &str) -> String
```

Enforced by `runner::run_filtered` which takes `F: Fn(&str) -> String`.

### run() signatures

```rust
pub fn run(args: &[String], verbose: u8) -> Result<i32>
```

38+ modules follow this pattern.

### Clone usage (conservative)

- `args.to_vec()` for forwarding argument slices
- `entry.file.clone()` for building HashMaps
- `.to_string()` / `.into_owned()` for Cow/&str conversions

### Iterator chains (idiomatic)

- `lines.iter().filter(...).map(...).collect()`
- `args.iter().any(|a| ...)`
- `std::iter::once(...).chain(iter).collect()`
- `.take(N)` for limiting output

## Module Structure Convention

Every `*_cmd.rs` follows:
1. Module doc comment (`//! ...`)
2. Imports (crate internal, then external)
3. Types/enums (ParseState, CargoCommand, etc.)
4. `lazy_static!` block
5. `pub fn run(...)` -- public entry point
6. Private `fn filter_*()` functions
7. `#[cfg(test)] mod tests { ... }`

### automod for module discovery

```rust
// src/cmds/js/mod.rs
automod::dir!(pub "src/cmds/js");
```

All ecosystem `mod.rs` files use this. Top-level `src/cmds/mod.rs` uses explicit `pub mod`.

## Configuration Patterns

### Clap derive

Single `#[derive(Parser)]` struct `Cli`. Subcommands via `#[derive(Subcommand)]`. Key patterns:
- `#[arg(trailing_var_arg = true, allow_hyphen_values = true)]` -- arg forwarding
- `#[arg(action = clap::ArgAction::Count, global = true)]` -- verbosity
- `#[command(external_subcommand)]` -- catch unknown subcommands

### Config loading

```rust
Config::load().map(|c| c.limits).unwrap_or_default()
```

All sections derive `Default` with sensible values. Partial TOML files are valid via `#[serde(default)]`.

## Dependencies

| Crate | Version | Purpose |
|-------|---------|---------|
| `clap` | 4 (derive) | CLI argument parsing |
| `anyhow` | 1.0 | Error handling |
| `regex` | 1 | Pattern matching |
| `lazy_static` | 1.4 | Compile-once regex |
| `serde` / `serde_json` | 1 (derive, preserve_order) | Serialization |
| `toml` | 0.8 | Config parsing |
| `rusqlite` | 0.31 (bundled) | SQLite tracking |
| `chrono` | 0.4 | Date/time |
| `colored` | 2 | Terminal colors |
| `dirs` | 5 | Platform directories |
| `automod` | 1 | Module auto-discovery |
| `sha2` | 0.10 | Hook integrity |
| `ureq` | 2 | HTTP (telemetry) |
| `which` | 8 | Binary resolution |
| `quick-xml` | 0.37 | XML parsing (dotnet trx) |

**Release profile:** `opt-level = 3`, `lto = true`, `codegen-units = 1`, `panic = "abort"`, `strip = true`

## Related Pages

- [[system-architecture]] -- Where these patterns are applied
- [[filter-patterns]] -- Filter-specific implementation details
- [[toml-filter-dsl]] -- Declarative alternative to Rust filters
