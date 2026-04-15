---
title: Core Infrastructure
tags: [core, config, tracking, tee, utils, filter, toml-filter, runner]
category: architecture
created: 2026-04-14
updated: 2026-04-14
---

# Core Infrastructure (`src/core/`)

Shared modules used across all RTK command handlers.

## Module Map

| Module | Lines | Purpose |
|--------|-------|---------|
| `config.rs` | 252 | TOML configuration system |
| `tracking.rs` | 1356 | SQLite token metrics |
| `tee.rs` | 506 | Raw output recovery on failure |
| `utils.rs` | 400+ | Shared helpers |
| `filter.rs` | 527 | Language-aware code filtering |
| `toml_filter.rs` | 1400+ | TOML DSL filter engine |
| `runner.rs` | 142 | Shared command execution skeleton |
| `display_helpers.rs` | 200+ | Terminal formatting |
| `telemetry.rs` | 200+ | Usage analytics ping |
| `telemetry_cmd.rs` | 184 | Telemetry management commands |
| `constants.rs` | 7 | Shared constants |

## config.rs -- Configuration System

**Config path:** `~/.config/rtk/config.toml` (via `dirs::config_dir()`)

```rust
pub struct Config {
    pub tracking: TrackingConfig,    // enabled, history_days, database_path
    pub display: DisplayConfig,      // colors, emoji, max_width
    pub filters: FilterConfig,       // ignore_dirs, ignore_files
    pub tee: TeeConfig,              // enabled, mode, max_files, max_file_size, directory
    pub telemetry: TelemetryConfig,  // enabled, consent_given, consent_date
    pub hooks: HooksConfig,          // exclude_commands: Vec<String>
    pub limits: LimitsConfig,        // grep_max_results, status_max_files, etc.
}
```

**Key defaults:**
- `TrackingConfig`: enabled=true, history_days=90
- `DisplayConfig`: colors=true, emoji=true, max_width=120
- `FilterConfig`: ignore_dirs=[".git", "node_modules", "target", "__pycache__", ".venv", "vendor"]
- `LimitsConfig`: grep_max_results=200, grep_max_per_file=25, status_max_files=15, passthrough_max_chars=2000

All sections derive `Default` + `#[serde(default)]`, so partial TOML files are valid.

**API:** `Config::load()`, `Config::save()`, `Config::create_default()`, `limits()` (convenience with fallback)

## tracking.rs -- SQLite Token Metrics

**Database:** `~/.local/share/rtk/history.db`

**Schema:**
```sql
CREATE TABLE commands (
    id INTEGER PRIMARY KEY,
    timestamp TEXT NOT NULL,
    original_cmd TEXT NOT NULL,
    rtk_cmd TEXT NOT NULL,
    input_tokens INTEGER NOT NULL,
    output_tokens INTEGER NOT NULL,
    saved_tokens INTEGER NOT NULL,
    savings_pct REAL NOT NULL,
    exec_time_ms INTEGER DEFAULT 0,
    project_path TEXT DEFAULT ''
);
```

**Key design choices:**
- **WAL mode** + 5s busy timeout for concurrent access from multiple AI instances
- **90-day automatic retention** cleanup on every insert
- **Token estimation:** `ceil(text.len() / 4.0)` -- ~4 chars/token heuristic
- **Schema migrations:** Idempotent `ALTER TABLE ADD COLUMN` wrapped in `let _ =`
- **Project-scoped queries:** SQL `GLOB` (not `LIKE`) to avoid wildcard chars in paths

**Primary tracking API:**
```rust
pub struct TimedExecution { start: Instant }
impl TimedExecution {
    pub fn start() -> Self;
    pub fn track(&self, original_cmd, rtk_cmd, input: &str, output: &str);
    pub fn track_passthrough(&self, original_cmd, rtk_cmd); // 0 tokens
}
```

**Data types:** `CommandRecord`, `GainSummary`, `DayStats`, `WeekStats`, `MonthStats`, `ParseFailureSummary`

## tee.rs -- Raw Output Recovery

When a filtered command fails (non-zero exit), raw unfiltered output is saved so the LLM can re-read it.

**File format:** `{unix_epoch}_{sanitized_slug}.log` in `~/.local/share/rtk/tee/`

**TeeConfig:**
```rust
pub struct TeeConfig {
    pub enabled: bool,              // default: true
    pub mode: TeeMode,             // Failures (default), Always, Never
    pub max_files: usize,          // default: 20 (rotation)
    pub max_file_size: usize,      // default: 1MB
    pub directory: Option<PathBuf>,
}
```

**Directory priority:** `RTK_TEE_DIR` env > `config.tee.directory` > `dirs::data_local_dir()/rtk/tee/`

**API:**
- `tee_raw(raw, slug, exit_code) -> Option<PathBuf>` -- main entry
- `tee_and_hint(raw, slug, exit_code) -> Option<String>` -- tee + `[full output: ~/...]` hint
- `force_tee_hint(raw, slug) -> Option<String>` -- ignores exit code/mode (for truncated AWS)

**Safety:** UTF-8 safe truncation (finds char boundary). `RTK_TEE=0` disables entirely.

## runner.rs -- Shared Execution Skeleton

**`run_filtered()`** -- The canonical filter execution pattern used by 20+ modules:

```rust
pub fn run_filtered<F>(
    cmd: Command, tool_name: &str, args_display: &str,
    filter_fn: F, opts: RunOptions<'_>,
) -> Result<i32>
where F: Fn(&str) -> String
```

**Six phases:** Execute -> Filter -> Print (with tee hint) -> Stderr passthrough -> Track -> Return exit code

**RunOptions builder:**
- `RunOptions::default()` -- combined stdout+stderr to filter
- `RunOptions::stdout_only()` -- filter stdout, pass stderr through
- `.tee("label")` -- enable tee recovery
- `.early_exit_on_failure()` -- skip filter if command fails
- `.no_trailing_newline()` -- suppress trailing newline

**`run_passthrough()`** -- For unrecognized subcommands. `Stdio::inherit()` streaming, tracks timing only.

## utils.rs -- Shared Helpers

| Function | Purpose |
|----------|---------|
| `strip_ansi(text)` | Remove ANSI escapes (lazy_static regex) |
| `truncate(s, max_len)` | Truncate with `...` suffix |
| `format_tokens(n)` | K/M suffix formatting |
| `format_usd(amount)` | Adaptive precision USD |
| `exit_code_from_output(output, label)` | Extract exit code, handle Unix signals (128+sig) |
| `exit_code_from_status(status, label)` | Same for ExitStatus |
| `fallback_tail(output, label, n)` | Last N lines on parse failure |
| `ruby_exec(tool)` | Auto-detect `bundle exec` when Gemfile exists |
| `detect_package_manager()` | Check lockfiles: pnpm > yarn > npm |
| `package_manager_exec(tool)` | Use detected PM's exec mechanism |
| `resolve_binary(name)` | PATH+PATHEXT resolution via `which` crate |
| `resolved_command(name)` | Drop-in for `Command::new()` with PATHEXT |
| `tool_exists(name)` | `which::which(name).is_ok()` |
| `shorten_arn(arn)` | Extract short name from AWS ARNs |
| `human_bytes(bytes)` | KB/MB/GB/TB formatting |
| `count_tokens(text)` | Whitespace-based token count (for tests) |

## filter.rs -- Language-Aware Code Filtering

Used by `rtk read` to strip comments/boilerplate from source files.

**FilterLevel:** `None`, `Minimal`, `Aggressive`

**Language detection:** `Language::from_extension(ext)` -- Rust, Python, JavaScript, TypeScript, Go, C, Cpp, Java, Ruby, Shell, Data, Unknown

**Three implementations of `FilterStrategy` trait:**
1. **NoFilter** -- returns unchanged
2. **MinimalFilter** -- strips comments (keeps doc comments like `///`), removes block comments, normalizes blank lines
3. **AggressiveFilter** -- MinimalFilter + keeps only imports/signatures/declarations. Data formats fall back to MinimalFilter

**`smart_truncate(content, max_lines, lang)`** -- prioritizes function signatures, imports, structural elements

## toml_filter.rs -- TOML DSL Filter Engine

Declarative filter pipeline for commands without native Rust handlers. See [[toml-filter-dsl]] for full details.

**8-stage pipeline:** strip_ansi -> replace -> match_output -> strip/keep_lines -> truncate_lines_at -> head/tail_lines -> max_lines -> on_empty

**Lookup priority:** `.rtk/filters.toml` (trust-gated) > `~/.config/rtk/filters.toml` > built-in (compiled in via build.rs)

**59 built-in TOML filters** covering terraform, make, gcc, brew, ansible, helm, etc.

## display_helpers.rs -- Terminal Formatting

**`PeriodStats` trait** -- Abstract interface for time-period statistics (DayStats, WeekStats, MonthStats). Provides `print_period_table<T>()` generic table printer with header, rows, totals.

**`format_duration(ms)`** -- ms/s/m adaptive formatting

## telemetry.rs -- Usage Analytics

Optional, GDPR-compliant usage ping. At most once per 23 hours.

**Flow:** Check compiled URL -> check `RTK_TELEMETRY_DISABLED=1` -> require `consent_given` -> check marker age -> spawn background thread -> fire-and-forget via `ureq` (2s timeout)

**Device identity:** SHA-256 of persistent salt in `~/.local/share/rtk/telemetry_salt` (0o600 permissions)

**GDPR Art. 17:** `rtk telemetry forget` deletes salt, marker, tracking DB, sends server-side erasure request

## Environment Variables

| Variable | Module | Purpose |
|----------|--------|---------|
| `RTK_NO_TOML=1` | main.rs, toml_filter.rs | Bypass TOML filter engine |
| `RTK_TOML_DEBUG=1` | toml_filter.rs | Debug output for TOML matching |
| `RTK_DB_PATH` | tracking.rs | Override database path |
| `RTK_TEE_DIR` | tee.rs | Override tee output directory |
| `RTK_TEE=0` | tee.rs | Disable tee entirely |
| `RTK_TELEMETRY_DISABLED=1` | telemetry.rs | Disable telemetry |
| `RTK_TRUST_PROJECT_FILTERS=1` | trust.rs | Auto-trust project filters (CI only) |
| `RTK_AUDIT_DIR` | hook_audit_cmd.rs | Override audit log directory |

## Related Pages

- [[system-architecture]] -- Overall system design and routing
- [[toml-filter-dsl]] -- TOML filter pipeline details
- [[rust-patterns]] -- Code conventions used in core/
