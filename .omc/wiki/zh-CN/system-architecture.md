---
title: 系统架构
tags: [架构, 核心, 路由, 数据流]
category: architecture
created: 2026-04-14
updated: 2026-04-14
---

# RTK 系统架构

RTK（Rust Token Killer）是一个 CLI 代理工具，通过过滤和压缩命令输出来最大限度地减少 LLM 令牌消耗。它通过智能过滤、分组、截断和去重实现 60-90% 的令牌节省。

## 高层架构

```
CLI 输入 ("rtk git log -10")
  -> Clap 解析 (Commands 枚举, 50+ 变体)
  -> 命令路由 (匹配枚举变体)
  -> 过滤模块执行 (src/cmds/*/)
  -> 令牌追踪 (SQLite, tracking.rs)
  -> 过滤后输出到 stdout
  -> 退出码传播
```

## 入口点 (`src/main.rs`, 2299 行)

### Cli 结构体 (第 48 行)

```rust
struct Cli {
    command: Commands,        // 子命令枚举
    verbose: u8,              // -v/-vv/-vvv (全局计数)
    ultra_compact: bool,      // --ultra-compact (全局)
    skip_env: bool,           // --skip-env (全局)
}
```

### 启动流程 (`run_cli()`, 第 1273 行)

1. `telemetry::maybe_ping()` -- 后台异步遥测 (每天一次, GDPR 门控)
2. `Cli::try_parse()` -- Clap 解析; 失败则进入 [[fallback-system]]
3. `hook_check::maybe_warn()` -- 每日钩子过期检查 (限频)
4. `integrity::runtime_check()` -- SHA-256 钩子验证 (仅操作命令)
5. `match cli.command { ... }` -- 分发到处理模块
6. `std::process::exit(code)` -- 传播底层命令的退出码

### Commands 枚举 (第 72 行, 50+ 变体)

每个已识别的命令映射到一个处理模块。主要分组：

| 生态系统 | 变体 | 处理模块 |
|----------|------|----------|
| Git | Git, Gh, Gt, Diff | `cmds::git::*` |
| Rust | Cargo, Err, Test | `cmds::rust::*`, `core::runner` |
| JavaScript | Npm, Npx, Pnpm, Vitest, Tsc, Next, Lint, Prettier, Playwright, Prisma | `cmds::js::*` |
| Python | Ruff, Pytest, Mypy, Pip | `cmds::python::*` |
| Go | Go, GolangciLint | `cmds::go::*` |
| .NET | Dotnet | `cmds::dotnet::*` |
| 云服务 | Aws, Docker, Kubectl, Curl, Wget, Psql | `cmds::cloud::*` |
| 系统 | Ls, Tree, Read, Grep, Find, Wc, Env, Json, Log, Deps, Summary, Format, Smart | `cmds::system::*` |
| Ruby | Rake, Rspec, Rubocop | `cmds::ruby::*` |
| 元命令 | Gain, CcEconomics, Config, Discover, Learn, Session, Init, Verify, Trust, Untrust, Proxy, Telemetry | `analytics::*`, `hooks::*`, `core::*` |

### 嵌套子命令枚举

复杂命令具有嵌套枚举路由：
- `GitCommands`: Diff, Log, Status, Show, Add, Commit, Push, Pull, Branch, Fetch, Stash, Worktree, Other
- `CargoCommands`: Build, Test, Clippy, Check, Install, Nextest, Other
- `DockerCommands` -> `ComposeCommands`, `KubectlCommands`
- `GoCommands`, `PnpmCommands`, `PrismaCommands` -> `PrismaMigrateCommands`
- `GtCommands`, `DotnetCommands`, `VitestCommands`

所有带 `Other(Vec<OsString>)` 的嵌套枚举使用 `#[command(external_subcommand)]` 进行透传。

### AgentTarget 枚举 (第 32 行)

```rust
pub enum AgentTarget { Claude, Cursor, Windsurf, Cline, Kilocode, Antigravity }
```

用于 `rtk init --agent <target>` 进行 AI 工具集成。

## 数据流图

### 原生命令 (`rtk cargo test`)

```
CLI -> Clap 成功 -> 遥测 -> 钩子检查 -> 完整性检查
  -> cargo_cmd::run(CargoCommand::Test, &args, verbose)
    -> TimedExecution::start()
    -> Command::new("cargo").args(["test", ...]).output()
    -> filter_cargo_test(stdout+stderr) -> 过滤后字符串
    -> println!(filtered)
    -> tee_and_hint(raw, "cargo-test", exit_code) 如果失败
    -> timer.track("cargo test", "rtk cargo test", raw, filtered)
      -> estimate_tokens(raw), estimate_tokens(filtered)
      -> Tracker::new().record(...) -> SQLite INSERT
    -> 返回 exit_code
  -> std::process::exit(code)
```

### TOML 匹配回退 (`rtk make build`)

```
CLI -> Clap 失败 (无 "make" 变体)
  -> run_fallback():
    -> 不在 RTK_META_COMMANDS 中
    -> toml_filter::find_matching_filter("make build") -> Some(CompiledFilter)
    -> resolved_command("make").args(["build"]).output()
    -> toml_filter::apply_filter(filter, raw) -> 8 阶段管道
    -> println!(filtered)
    -> timer.track(cmd, "rtk:toml make build", raw, filtered)
```

### 纯透传 (`rtk unknown-tool --flag`)

```
CLI -> Clap 失败
  -> run_fallback():
    -> toml_filter::find_matching_filter() -> None
    -> resolved_command("unknown-tool").stdin/stdout/stderr(Stdio::inherit()).status()
    -> timer.track_passthrough(cmd, "rtk fallback") // 0 令牌, 仅计时
```

### 代理模式 (`rtk proxy head -50 file.txt`)

```
CLI -> Clap 成功 (Commands::Proxy)
  -> 以 Stdio::piped() 派生子进程
  -> 两个读取线程: 流式传输 + 捕获 (每个最多 1MB)
  -> timer.track(cmd, "rtk proxy", output, output) // 输入=输出, 无过滤
```

## 关键设计决策

1. **无异步** -- 零 `tokio`/`async-std`。单线程。仅有的线程: 遥测 ping + 代理流式传输
2. **回退优先** -- RTK 永远不会阻止命令执行。未知命令透明透传
3. **退出码保真** -- 每个处理函数返回 `Result<i32>`, 通过 `std::process::exit()` 传播
4. **默认开放安全** -- `is_operational_command()` 白名单意味着新命令在显式添加前不会进行完整性检查
5. **ChildGuard RAII** -- `Drop` 实现杀死并等待子进程, 防止代理模式中的僵尸进程

## 相关页面

- [[core-infrastructure]] -- 共享模块 (config, tracking, tee, utils, filter, toml_filter)
- [[filter-patterns]] -- cmds/ 中的过滤实现策略
- [[hooks-system]] -- AI 代理集成和安全
- [[analytics-system]] -- 令牌节省报告和经济分析
