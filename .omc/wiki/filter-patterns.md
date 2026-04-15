---
title: Filter Implementation Patterns
tags: [filters, patterns, cmds, strategies, token-savings]
category: pattern
created: 2026-04-14
updated: 2026-04-14
---

# Filter Implementation Patterns

RTK has 58 Rust source files across 10 ecosystem directories (~29,771 lines) plus 59 TOML declarative filters.

## Module Inventory

| Ecosystem | Files | Key Modules |
|-----------|-------|-------------|
| `git/` | 4 (~5,243 lines) | git.rs (2540), gh_cmd.rs (1461), gt_cmd.rs (781), diff_cmd.rs |
| `rust/` | 3 (~1,862+) | cargo_cmd.rs (1862), runner.rs |
| `js/` | 9 | npm, pnpm (565), vitest, lint (697), tsc, next, prettier, playwright (486), prisma (497) |
| `python/` | 4 | ruff, pytest, mypy, pip |
| `go/` | 2 (~1,777) | go_cmd.rs (1058), golangci_cmd.rs (719) |
| `dotnet/` | 4 (~4,559+) | dotnet_cmd.rs (2313), binlog.rs (1651), dotnet_trx.rs (595) |
| `cloud/` | 5 | aws_cmd.rs (2751), container.rs (765), curl, wget, psql |
| `system/` | 14 | ls, tree, read, grep, find (620), wc, env, json, log, deps, summary, format, local_llm |
| `ruby/` | 3 | rake (527), rspec (1014), rubocop (628) |

## The Canonical `run()` Signature

```rust
pub fn run(args: &[String], verbose: u8) -> Result<i32>
```

Variations:
- **Enum-dispatched:** `run(cmd: CargoCommand, args: &[String], verbose: u8)` (cargo, container, prisma, pnpm, vitest)
- **Subcommand-dispatched:** `run(subcommand: &str, args: &[String], verbose: u8)` (aws, gh)
- **Extra flags:** npm adds `skip_env`, gh adds `ultra_compact`, git adds `max_lines` + `global_args`

Always returns `Result<i32>` -- the underlying command's exit code. Modules never call `process::exit()` directly.

## The `run_filtered()` Skeleton

Used by 20+ modules with 40+ call sites (from `core::runner`):

```rust
pub fn run_filtered<F>(cmd: Command, tool_name: &str, args_display: &str,
    filter_fn: F, opts: RunOptions<'_>) -> Result<i32>
where F: Fn(&str) -> String
```

Phases: Execute -> Filter -> Print+tee -> Stderr passthrough -> Track -> Return exit code

## Filter Strategies (A-J)

### A: Line-by-Line Regex Filtering (30-70% savings)
**Used by:** npm, tree, cargo build/install, many TOML filters

Iterate lines, skip those matching noise patterns:
```rust
for line in output.lines() {
    if line.starts_with('>') && line.contains('@') { continue; }
    if line.trim_start().starts_with("npm WARN") { continue; }
    result.push(line.to_string());
}
```

### B: State Machine Parsing (70-95% savings)
**Used by:** pytest, rake, rspec (text fallback), cargo test

Enum-based phase tracking:
```rust
enum ParseState { Header, TestProgress, Failures, Summary }
// Transitions on marker lines like "=== FAILURES ==="
```

### C: JSON Injection + Structured Parsing (60-90% savings)
**Used by:** ruff, golangci-lint, rspec, vitest, playwright, aws, kubectl

Inject `--output-format=json` or `--format json`, deserialize via serde, emit compact summary. 17 files use `serde_json::from_str`.

### D: NDJSON Streaming (80-95% savings)
**Used by:** go test

Inject `-json` flag, parse each line as `GoTestEvent`, aggregate by package into `PackageResult` structs.

### E: Section/Block-Based Filtering (70-95% savings)
**Used by:** cargo build/test/clippy, git diff

Collect multi-line error/warning blocks, strip noise between them, emit blocks + summary. Limited to first 15 error blocks.

### F: Multi-Command Composition (50-80% savings)
**Used by:** git diff, git show, dotnet build/test

Run multiple sub-commands and combine:
1. `git diff --stat` (file change summary)
2. `git diff` (full diff)
3. `compact_diff()` (truncate hunks, per-file tracking)

### G: Deduplication (80-95% savings)
**Used by:** log_cmd, container (kubectl logs)

Normalize lines (replace timestamps/UUIDs/hex/numbers with placeholders), count occurrences, show unique patterns with repeat counts.

### H: Summary Generation with Grouping (60-90% savings)
**Used by:** tsc, mypy, golangci-lint, rubocop, ruff, grep

Parse structured diagnostics, group by file or rule, emit compact grouped summary.

### I: Format Template Injection (40-60% savings)
**Used by:** docker ps, docker images

Inject custom `--format` template to get exactly the fields needed:
```rust
.args(["ps", "--format", "{{.ID}}\t{{.Names}}\t{{.Status}}\t{{.Image}}\t{{.Ports}}"])
```

### J: TOML DSL Pipeline (variable savings)
**Used by:** 59 built-in TOML filters

8-stage declarative pipeline. See [[toml-filter-dsl]].

## Token Savings Strategies

| Strategy | Savings | Mechanism |
|----------|---------|-----------|
| Noise line stripping | 30-70% | Remove progress bars, compilation lines, blank lines |
| Success short-circuiting | 90-99% | Single summary line on clean success |
| JSON injection + schema compression | 60-90% | Parse structured data, emit only essential fields |
| Diff compaction | 50-80% | Limit hunks to 100 lines, stat summary |
| Error/failure focus | 70-95% | Strip all passing tests, show only failures |
| Log deduplication | 80-95% | Normalize + count unique patterns |
| Format template injection | 40-60% | Custom `--format` gets exact fields |
| Truncation guards | Safety net | `truncate()`, `max_lines`, `head/tail_lines` |

## Three-Tier Graceful Degradation (`src/parser/`)

```
ParseResult<T>:
  Tier 1 (Full)       -- JSON parsed, compact summary
  Tier 2 (Degraded)   -- JSON failed, regex extraction, warning marker
  Tier 3 (Passthrough) -- All parsing failed, truncated raw + [RTK:PASSTHROUGH]
```

Used by vitest and playwright via `OutputParser` trait. `TokenFormatter` trait provides Compact/Verbose/Ultra modes.

## Cross-Cutting Patterns

- **Enum-based subcommand routing:** git, cargo, container, dotnet, go, pnpm, vitest, prisma
- **Flag-aware filtering:** Detect `--nocapture`, `--format`, `--json`, `--stat` to skip filtering when user wants verbose output
- **Tool existence fallback:** `tool_exists("tsc")` -> fallback to `npx tsc`
- **Cross-command routing:** `lint_cmd` routes to `ruff_cmd` or `mypy_cmd` based on project language
- **Noise directory constants:** `NOISE_DIRS` (26 patterns) shared by ls.rs and tree.rs
- **Ruby bundle exec detection:** `ruby_exec("rspec")` auto-detects `bundle exec` vs plain

## Related Pages

- [[system-architecture]] -- Overall routing and data flow
- [[toml-filter-dsl]] -- TOML declarative filter details
- [[rust-patterns]] -- Code conventions in filter modules
