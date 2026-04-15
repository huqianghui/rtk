---
title: System Architecture
tags: [architecture, core, routing, data-flow]
category: architecture
created: 2026-04-14
updated: 2026-04-14
---

# RTK System Architecture

RTK (Rust Token Killer) is a CLI proxy that minimizes LLM token consumption by filtering and compressing command outputs. It achieves 60-90% token savings through smart filtering, grouping, truncation, and deduplication.

## High-Level Architecture

```
CLI input ("rtk git log -10")
  -> Clap parsing (Commands enum, 50+ variants)
  -> Command routing (match on enum variant)
  -> Filter module execution (src/cmds/*/)
  -> Token tracking (SQLite via tracking.rs)
  -> Filtered output to stdout
  -> Exit code propagation
```

## Entry Point (`src/main.rs`, 2299 lines)

### Cli Struct (line 48)

```rust
struct Cli {
    command: Commands,        // Subcommand enum
    verbose: u8,              // -v/-vv/-vvv (global, count action)
    ultra_compact: bool,      // --ultra-compact (global)
    skip_env: bool,           // --skip-env (global)
}
```

### Startup Flow (`run_cli()`, line 1273)

1. `telemetry::maybe_ping()` -- fire-and-forget background ping (1/day, GDPR-gated)
2. `Cli::try_parse()` -- Clap parsing; on failure -> [[fallback-system]]
3. `hook_check::maybe_warn()` -- daily hook staleness check (rate-limited)
4. `integrity::runtime_check()` -- SHA-256 hook verification (operational commands only)
5. `match cli.command { ... }` -- dispatch to handler module
6. `std::process::exit(code)` -- propagate underlying command's exit code

### Commands Enum (line 72, 50+ variants)

Every recognized command maps to a handler module. Key groupings:

| Ecosystem | Variants | Handler Modules |
|-----------|----------|-----------------|
| Git | Git, Gh, Gt, Diff | `cmds::git::*` |
| Rust | Cargo, Err, Test | `cmds::rust::*`, `core::runner` |
| JavaScript | Npm, Npx, Pnpm, Vitest, Tsc, Next, Lint, Prettier, Playwright, Prisma | `cmds::js::*` |
| Python | Ruff, Pytest, Mypy, Pip | `cmds::python::*` |
| Go | Go, GolangciLint | `cmds::go::*` |
| .NET | Dotnet | `cmds::dotnet::*` |
| Cloud | Aws, Docker, Kubectl, Curl, Wget, Psql | `cmds::cloud::*` |
| System | Ls, Tree, Read, Grep, Find, Wc, Env, Json, Log, Deps, Summary, Format, Smart | `cmds::system::*` |
| Ruby | Rake, Rspec, Rubocop | `cmds::ruby::*` |
| Meta | Gain, CcEconomics, Config, Discover, Learn, Session, Init, Verify, Trust, Untrust, Proxy, Telemetry | `analytics::*`, `hooks::*`, `core::*` |

### Nested Subcommand Enums

Complex commands have nested enum routing:
- `GitCommands`: Diff, Log, Status, Show, Add, Commit, Push, Pull, Branch, Fetch, Stash, Worktree, Other
- `CargoCommands`: Build, Test, Clippy, Check, Install, Nextest, Other
- `DockerCommands` -> `ComposeCommands`, `KubectlCommands`
- `GoCommands`, `PnpmCommands`, `PrismaCommands` -> `PrismaMigrateCommands`
- `GtCommands`, `DotnetCommands`, `VitestCommands`

All nested enums with `Other(Vec<OsString>)` use `#[command(external_subcommand)]` for passthrough.

### AgentTarget Enum (line 32)

```rust
pub enum AgentTarget { Claude, Cursor, Windsurf, Cline, Kilocode, Antigravity }
```

Used by `rtk init --agent <target>` for AI tool integration.

## Data Flow Diagrams

### Native Command (`rtk cargo test`)

```
CLI -> Clap succeeds -> telemetry ping -> hook check -> integrity check
  -> cargo_cmd::run(CargoCommand::Test, &args, verbose)
    -> TimedExecution::start()
    -> Command::new("cargo").args(["test", ...]).output()
    -> filter_cargo_test(stdout+stderr) -> filtered string
    -> println!(filtered)
    -> tee_and_hint(raw, "cargo-test", exit_code) if failure
    -> timer.track("cargo test", "rtk cargo test", raw, filtered)
      -> estimate_tokens(raw), estimate_tokens(filtered)
      -> Tracker::new().record(...) -> SQLite INSERT
    -> return exit_code
  -> std::process::exit(code)
```

### TOML-Matched Fallback (`rtk make build`)

```
CLI -> Clap FAILS (no "make" variant)
  -> run_fallback():
    -> not in RTK_META_COMMANDS
    -> toml_filter::find_matching_filter("make build") -> Some(CompiledFilter)
    -> resolved_command("make").args(["build"]).output()
    -> toml_filter::apply_filter(filter, raw) -> 8-stage pipeline
    -> println!(filtered)
    -> timer.track(cmd, "rtk:toml make build", raw, filtered)
```

### Pure Passthrough (`rtk unknown-tool --flag`)

```
CLI -> Clap FAILS
  -> run_fallback():
    -> toml_filter::find_matching_filter() -> None
    -> resolved_command("unknown-tool").stdin/stdout/stderr(Stdio::inherit()).status()
    -> timer.track_passthrough(cmd, "rtk fallback") // 0 tokens, timing only
```

### Proxy Mode (`rtk proxy head -50 file.txt`)

```
CLI -> Clap succeeds (Commands::Proxy)
  -> Spawn child with Stdio::piped()
  -> Two reader threads: stream + capture (up to 1MB each)
  -> timer.track(cmd, "rtk proxy", output, output) // input=output, no filtering
```

## Key Design Decisions

1. **No async** -- Zero `tokio`/`async-std`. Single-threaded. Only threads: telemetry ping + proxy streaming
2. **Fallback-first** -- RTK never blocks a command. Unknown commands pass through transparently
3. **Exit code fidelity** -- Every handler returns `Result<i32>`, propagated via `std::process::exit()`
4. **Fail-open security** -- `is_operational_command()` whitelist means new commands are NOT integrity-checked until explicitly added
5. **ChildGuard RAII** -- `Drop` impl kills+waits on child process, preventing zombies in proxy mode

## Related Pages

- [[core-infrastructure]] -- Shared modules (config, tracking, tee, utils, filter, toml_filter)
- [[filter-patterns]] -- Filter implementation strategies across cmds/
- [[hooks-system]] -- AI agent integration and security
- [[analytics-system]] -- Token savings reporting and economics
