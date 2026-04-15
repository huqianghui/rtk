---
title: 核心基础设施
tags: [核心, 配置, 追踪, tee, 工具, 过滤器, toml过滤器, 运行器]
category: architecture
created: 2026-04-14
updated: 2026-04-14
---

# 核心基础设施 (`src/core/`)

所有 RTK 命令处理程序共享的模块。

## 模块一览

| 模块 | 行数 | 用途 |
|------|------|------|
| `config.rs` | 252 | TOML 配置系统 |
| `tracking.rs` | 1356 | SQLite 令牌指标 |
| `tee.rs` | 506 | 失败时原始输出恢复 |
| `utils.rs` | 400+ | 共享工具函数 |
| `filter.rs` | 527 | 语言感知代码过滤 |
| `toml_filter.rs` | 1400+ | TOML DSL 过滤引擎 |
| `runner.rs` | 142 | 共享命令执行骨架 |
| `display_helpers.rs` | 200+ | 终端格式化 |
| `telemetry.rs` | 200+ | 使用分析 ping |
| `telemetry_cmd.rs` | 184 | 遥测管理命令 |
| `constants.rs` | 7 | 共享常量 |

## config.rs -- 配置系统

**配置路径:** `~/.config/rtk/config.toml` (通过 `dirs::config_dir()`)

```rust
pub struct Config {
    pub tracking: TrackingConfig,    // enabled, history_days, database_path
    pub display: DisplayConfig,      // colors, emoji, max_width
    pub filters: FilterConfig,       // ignore_dirs, ignore_files
    pub tee: TeeConfig,              // enabled, mode, max_files, max_file_size, directory
    pub telemetry: TelemetryConfig,  // enabled, consent_given, consent_date
    pub hooks: HooksConfig,          // exclude_commands: Vec<String>
    pub limits: LimitsConfig,        // grep_max_results, status_max_files 等
}
```

**关键默认值:**
- `TrackingConfig`: enabled=true, history_days=90
- `DisplayConfig`: colors=true, emoji=true, max_width=120
- `FilterConfig`: ignore_dirs=[".git", "node_modules", "target", "__pycache__", ".venv", "vendor"]
- `LimitsConfig`: grep_max_results=200, grep_max_per_file=25, status_max_files=15, passthrough_max_chars=2000

所有部分派生 `Default` + `#[serde(default)]`, 因此部分 TOML 文件也是有效的。

**API:** `Config::load()`, `Config::save()`, `Config::create_default()`, `limits()` (带回退的便捷函数)

## tracking.rs -- SQLite 令牌指标

**数据库:** `~/.local/share/rtk/history.db`

**表结构:**
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

**关键设计选择:**
- **WAL 模式** + 5 秒忙等待超时, 支持多个 AI 实例并发访问
- **90 天自动保留** 每次插入时清理
- **令牌估算:** `ceil(text.len() / 4.0)` -- 约 4 字符/令牌的启发式算法
- **表迁移:** 幂等 `ALTER TABLE ADD COLUMN` 包装在 `let _ =` 中
- **项目范围查询:** 使用 SQL `GLOB` (非 `LIKE`) 避免路径中的通配符

**主要追踪 API:**
```rust
pub struct TimedExecution { start: Instant }
impl TimedExecution {
    pub fn start() -> Self;
    pub fn track(&self, original_cmd, rtk_cmd, input: &str, output: &str);
    pub fn track_passthrough(&self, original_cmd, rtk_cmd); // 0 令牌
}
```

**数据类型:** `CommandRecord`, `GainSummary`, `DayStats`, `WeekStats`, `MonthStats`, `ParseFailureSummary`

## tee.rs -- 原始输出恢复

当过滤后的命令失败 (非零退出码) 时, 保存未过滤的原始输出, 以便 LLM 可以重新读取。

**文件格式:** `{unix_epoch}_{sanitized_slug}.log` 位于 `~/.local/share/rtk/tee/`

**TeeConfig:**
```rust
pub struct TeeConfig {
    pub enabled: bool,              // 默认: true
    pub mode: TeeMode,             // Failures (默认), Always, Never
    pub max_files: usize,          // 默认: 20 (轮转)
    pub max_file_size: usize,      // 默认: 1MB
    pub directory: Option<PathBuf>,
}
```

**目录优先级:** `RTK_TEE_DIR` 环境变量 > `config.tee.directory` > `dirs::data_local_dir()/rtk/tee/`

**API:**
- `tee_raw(raw, slug, exit_code) -> Option<PathBuf>` -- 主入口
- `tee_and_hint(raw, slug, exit_code) -> Option<String>` -- tee + `[full output: ~/...]` 提示
- `force_tee_hint(raw, slug) -> Option<String>` -- 忽略退出码/模式 (用于 AWS 截断)

**安全性:** UTF-8 安全截断 (查找字符边界)。`RTK_TEE=0` 完全禁用。

## runner.rs -- 共享执行骨架

**`run_filtered()`** -- 20+ 模块使用的标准过滤执行模式:

```rust
pub fn run_filtered<F>(
    cmd: Command, tool_name: &str, args_display: &str,
    filter_fn: F, opts: RunOptions<'_>,
) -> Result<i32>
where F: Fn(&str) -> String
```

**六个阶段:** 执行 -> 过滤 -> 打印 (含 tee 提示) -> stderr 透传 -> 追踪 -> 返回退出码

**RunOptions 构建器:**
- `RunOptions::default()` -- 合并 stdout+stderr 后过滤
- `RunOptions::stdout_only()` -- 仅过滤 stdout, 透传 stderr
- `.tee("label")` -- 启用 tee 恢复
- `.early_exit_on_failure()` -- 命令失败时跳过过滤
- `.no_trailing_newline()` -- 不添加尾部换行

**`run_passthrough()`** -- 用于未识别的子命令。`Stdio::inherit()` 流式传输, 仅追踪计时。

## utils.rs -- 共享工具函数

| 函数 | 用途 |
|------|------|
| `strip_ansi(text)` | 移除 ANSI 转义码 (lazy_static 正则) |
| `truncate(s, max_len)` | 截断并添加 `...` 后缀 |
| `format_tokens(n)` | K/M 后缀格式化 |
| `format_usd(amount)` | 自适应精度美元格式 |
| `exit_code_from_output(output, label)` | 提取退出码, 处理 Unix 信号 (128+sig) |
| `exit_code_from_status(status, label)` | ExitStatus 版本 |
| `fallback_tail(output, label, n)` | 解析失败时取最后 N 行 |
| `ruby_exec(tool)` | 存在 Gemfile 时自动检测 `bundle exec` |
| `detect_package_manager()` | 检查锁文件: pnpm > yarn > npm |
| `package_manager_exec(tool)` | 使用检测到的包管理器的 exec 机制 |
| `resolve_binary(name)` | 通过 `which` 的 PATH+PATHEXT 解析 |
| `resolved_command(name)` | 带 PATHEXT 感知的 `Command::new()` 替代 |
| `tool_exists(name)` | `which::which(name).is_ok()` |
| `shorten_arn(arn)` | 从 AWS ARN 提取短名称 |
| `human_bytes(bytes)` | KB/MB/GB/TB 格式化 |
| `count_tokens(text)` | 基于空格的令牌计数 (用于测试) |

## filter.rs -- 语言感知代码过滤

由 `rtk read` 使用, 从源文件中去除注释/样板代码。

**FilterLevel:** `None`, `Minimal`, `Aggressive`

**语言检测:** `Language::from_extension(ext)` -- Rust, Python, JavaScript, TypeScript, Go, C, Cpp, Java, Ruby, Shell, Data, Unknown

**三种 `FilterStrategy` 特征实现:**
1. **NoFilter** -- 原样返回
2. **MinimalFilter** -- 去除注释 (保留文档注释如 `///`), 移除块注释, 规范化空行
3. **AggressiveFilter** -- MinimalFilter + 仅保留 import/签名/声明。数据格式回退到 MinimalFilter

**`smart_truncate(content, max_lines, lang)`** -- 优先保留函数签名、导入和结构元素

## toml_filter.rs -- TOML DSL 过滤引擎

用于没有原生 Rust 处理程序的命令的声明式过滤管道。详见 [[toml-filter-dsl]]。

**8 阶段管道:** strip_ansi -> replace -> match_output -> strip/keep_lines -> truncate_lines_at -> head/tail_lines -> max_lines -> on_empty

**查找优先级:** `.rtk/filters.toml` (需信任) > `~/.config/rtk/filters.toml` > 内置 (通过 build.rs 编译)

**59 个内置 TOML 过滤器** 覆盖 terraform, make, gcc, brew, ansible, helm 等。

## display_helpers.rs -- 终端格式化

**`PeriodStats` 特征** -- 时间周期统计的抽象接口 (DayStats, WeekStats, MonthStats)。提供 `print_period_table<T>()` 通用表格打印, 包含表头、行和合计。

**`format_duration(ms)`** -- ms/s/m 自适应格式化

## telemetry.rs -- 使用分析

可选的、符合 GDPR 的使用 ping。最多每 23 小时一次。

**流程:** 检查编译的 URL -> 检查 `RTK_TELEMETRY_DISABLED=1` -> 需要 `consent_given` -> 检查标记文件年龄 -> 派生后台线程 -> 通过 `ureq` 发送 (2 秒超时)

**设备身份:** `~/.local/share/rtk/telemetry_salt` 中持久化 salt 的 SHA-256 (Unix 上 0o600 权限)

**GDPR 第 17 条:** `rtk telemetry forget` 删除 salt、标记、追踪数据库, 并发送服务端擦除请求

## 环境变量

| 变量 | 模块 | 用途 |
|------|------|------|
| `RTK_NO_TOML=1` | main.rs, toml_filter.rs | 绕过 TOML 过滤引擎 |
| `RTK_TOML_DEBUG=1` | toml_filter.rs | TOML 匹配调试输出 |
| `RTK_DB_PATH` | tracking.rs | 覆盖数据库路径 |
| `RTK_TEE_DIR` | tee.rs | 覆盖 tee 输出目录 |
| `RTK_TEE=0` | tee.rs | 完全禁用 tee |
| `RTK_TELEMETRY_DISABLED=1` | telemetry.rs | 禁用遥测 |
| `RTK_TRUST_PROJECT_FILTERS=1` | trust.rs | 自动信任项目过滤器 (仅 CI) |
| `RTK_AUDIT_DIR` | hook_audit_cmd.rs | 覆盖审计日志目录 |

## 相关页面

- [[system-architecture]] -- 整体系统设计和路由
- [[toml-filter-dsl]] -- TOML 过滤管道详情
- [[rust-patterns]] -- core/ 中使用的代码规范
