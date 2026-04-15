---
title: Rust 模式与规范
tags: [rust, 模式, 规范, 错误处理, 正则, 测试, 依赖]
category: pattern
created: 2026-04-14
updated: 2026-04-14
---

# Rust 模式与规范

RTK 55K 行代码库中观察到的重复模式。

## 错误处理

### anyhow::Result + .context() (120+ 处使用)

通用错误处理。导入: `use anyhow::{Context, Result};`

```rust
fs::read_to_string(path)
    .with_context(|| format!("Failed to read config: {}", path.display()))?;
```

- `.context("静态字符串")` 用于静态消息
- `.with_context(|| format!(...))` 用于动态消息
- 上下文字符串遵循 "Failed to ..." 模式

### 回退模式 (所有过滤器必须)

```rust
let filtered = filter_output(&output.stdout)
    .unwrap_or_else(|e| {
        eprintln!("rtk: filter warning: {}", e);
        output.stdout.clone()  // 失败时透传
    });
```

### 退出码传播

每个 `run()` 返回 `Result<i32>`。主入口:
```rust
fn main() {
    let code = match run_cli() { Ok(code) => code, Err(e) => { eprintln!(...); 1 } };
    std::process::exit(code);
}
```

辅助函数: `exit_code_from_output()`, `exit_code_from_status()` 处理 Unix 信号 (128+sig)。

直接 `process::exit()` 仅出现在: `main()`, 安全关键的 `integrity.rs`, 钩子 `rewrite_cmd.rs` (语义退出码 1/2/3)。

## 正则模式

### lazy_static! (25 个块, 主要模式)

```rust
lazy_static! {
    static ref ERROR_RE: Regex = Regex::new(r"^error\[").unwrap();
}
```

模块级用于共享模式; 函数作用域用于局部模式。此处 `.unwrap()` 是可接受的 -- 错误的正则字面量是编程错误。

### OnceLock (较新的替代方案)

```rust
static RE: OnceLock<Regex> = OnceLock::new();
let re = RE.get_or_init(|| Regex::new(r"...").unwrap());
```

用于 `cargo_cmd.rs` (第 324, 331, 653 行) 和 `telemetry.rs`。

### 已知违规 (函数内 Regex::new)

- `summary.rs`, `grep_cmd.rs` -- 动态模式 (基于 format, 合理)
- `deps.rs` (第 79-80, 167 行) -- 可缓存的静态模式
- `local_llm.rs` (第 142, 181, 208, 222 行) -- 重复的静态模式

### 常见正则类别

1. **错误检测:** `r"(?i)^.*error[\s:\[].*$"`, `r"^error\[E\d+\]:.*$"`
2. **段落解析:** `r"^(.+?)\((\d+),(\d+)\):\s+(error|warning)\s+(TS\d+):\s+(.+)$"`
3. **ANSI 去除:** `r"\x1b\[[0-9;]*[a-zA-Z]"` (core/utils.rs)
4. **日志规范化:** 时间戳, UUID, 十六进制, 大数字, 路径
5. **构建输出:** 警告/错误计数, 测试结果, 摘要行

## 测试模式

### 测试模块约定 (72 个文件)

```rust
#[cfg(test)]
mod tests {
    use super::*;
    // 测试代码
}
```

### count_tokens 辅助函数

规范版本在 `core/utils.rs:258`。也在以下模块中有本地副本: gh_cmd, gt_cmd, git, psql_cmd。

```rust
fn count_tokens(s: &str) -> usize { s.split_whitespace().count() }
```

### 无 insta/快照测试

尽管 CLAUDE.md 推荐使用, 但代码库中 `assert_snapshot!` 出现次数为零。测试使用标准断言。

### 测试固件

`tests/fixtures/` 中 6 个文件 (主要是 dotnet + golangci)。大多数测试使用内联字符串字面量。

### 集成测试

git.rs 和 read.rs 中 6 个 `#[ignore]` 测试 -- 需要 git 仓库或已构建的二进制文件。

## 所有权与借用

### 过滤函数签名 (100% 一致)

```rust
fn filter_output(input: &str) -> String
```

由 `runner::run_filtered` 强制执行, 其接受 `F: Fn(&str) -> String`。

### run() 签名

```rust
pub fn run(args: &[String], verbose: u8) -> Result<i32>
```

38+ 模块遵循此模式。

### Clone 使用 (保守)

- `args.to_vec()` 用于转发参数切片
- `entry.file.clone()` 用于构建 HashMap
- `.to_string()` / `.into_owned()` 用于 Cow/&str 转换

### 迭代器链 (惯用)

- `lines.iter().filter(...).map(...).collect()`
- `args.iter().any(|a| ...)`
- `std::iter::once(...).chain(iter).collect()`
- `.take(N)` 用于限制输出

## 模块结构约定

每个 `*_cmd.rs` 遵循:
1. 模块文档注释 (`//! ...`)
2. 导入 (先 crate 内部, 后外部)
3. 类型/枚举 (ParseState, CargoCommand 等)
4. `lazy_static!` 块
5. `pub fn run(...)` -- 公共入口点
6. 私有 `fn filter_*()` 函数
7. `#[cfg(test)] mod tests { ... }`

### automod 模块发现

```rust
// src/cmds/js/mod.rs
automod::dir!(pub "src/cmds/js");
```

所有生态系统 `mod.rs` 文件使用此方式。顶层 `src/cmds/mod.rs` 使用显式 `pub mod`。

## 配置模式

### Clap derive

单一 `#[derive(Parser)]` 结构体 `Cli`。通过 `#[derive(Subcommand)]` 定义子命令。关键模式:
- `#[arg(trailing_var_arg = true, allow_hyphen_values = true)]` -- 参数转发
- `#[arg(action = clap::ArgAction::Count, global = true)]` -- 详细度
- `#[command(external_subcommand)]` -- 捕获未知子命令

### 配置加载

```rust
Config::load().map(|c| c.limits).unwrap_or_default()
```

所有部分派生 `Default` 并具有合理的默认值。部分 TOML 文件通过 `#[serde(default)]` 有效。

## 依赖

| 包 | 版本 | 用途 |
|----|------|------|
| `clap` | 4 (derive) | CLI 参数解析 |
| `anyhow` | 1.0 | 错误处理 |
| `regex` | 1 | 模式匹配 |
| `lazy_static` | 1.4 | 单次编译正则 |
| `serde` / `serde_json` | 1 (derive, preserve_order) | 序列化 |
| `toml` | 0.8 | 配置解析 |
| `rusqlite` | 0.31 (bundled) | SQLite 追踪 |
| `chrono` | 0.4 | 日期/时间 |
| `colored` | 2 | 终端颜色 |
| `dirs` | 5 | 平台目录 |
| `automod` | 1 | 模块自动发现 |
| `sha2` | 0.10 | 钩子完整性 |
| `ureq` | 2 | HTTP (遥测) |
| `which` | 8 | 二进制文件解析 |
| `quick-xml` | 0.37 | XML 解析 (dotnet trx) |

**发布配置:** `opt-level = 3`, `lto = true`, `codegen-units = 1`, `panic = "abort"`, `strip = true`

## 相关页面

- [[system-architecture]] -- 这些模式的应用场景
- [[filter-patterns]] -- 过滤器特定的实现细节
- [[toml-filter-dsl]] -- Rust 过滤器的声明式替代方案
