---
title: 过滤器实现模式
tags: [过滤器, 模式, cmds, 策略, 令牌节省]
category: pattern
created: 2026-04-14
updated: 2026-04-14
---

# 过滤器实现模式

RTK 在 10 个生态系统目录中有 58 个 Rust 源文件 (约 29,771 行), 加上 59 个 TOML 声明式过滤器。

## 模块清单

| 生态系统 | 文件数 | 关键模块 |
|----------|--------|----------|
| `git/` | 4 (约 5,243 行) | git.rs (2540), gh_cmd.rs (1461), gt_cmd.rs (781), diff_cmd.rs |
| `rust/` | 3 (约 1,862+ 行) | cargo_cmd.rs (1862), runner.rs |
| `js/` | 9 | npm, pnpm (565), vitest, lint (697), tsc, next, prettier, playwright (486), prisma (497) |
| `python/` | 4 | ruff, pytest, mypy, pip |
| `go/` | 2 (约 1,777 行) | go_cmd.rs (1058), golangci_cmd.rs (719) |
| `dotnet/` | 4 (约 4,559+ 行) | dotnet_cmd.rs (2313), binlog.rs (1651), dotnet_trx.rs (595) |
| `cloud/` | 5 | aws_cmd.rs (2751), container.rs (765), curl, wget, psql |
| `system/` | 14 | ls, tree, read, grep, find (620), wc, env, json, log, deps, summary, format, local_llm |
| `ruby/` | 3 | rake (527), rspec (1014), rubocop (628) |

## 标准 `run()` 函数签名

```rust
pub fn run(args: &[String], verbose: u8) -> Result<i32>
```

变体:
- **枚举分发:** `run(cmd: CargoCommand, args: &[String], verbose: u8)` (cargo, container, prisma, pnpm, vitest)
- **子命令分发:** `run(subcommand: &str, args: &[String], verbose: u8)` (aws, gh)
- **额外标志:** npm 添加 `skip_env`, gh 添加 `ultra_compact`, git 添加 `max_lines` + `global_args`

始终返回 `Result<i32>` -- 底层命令的退出码。模块从不直接调用 `process::exit()`。

## `run_filtered()` 执行骨架

被 20+ 模块的 40+ 调用点使用 (来自 `core::runner`):

```rust
pub fn run_filtered<F>(cmd: Command, tool_name: &str, args_display: &str,
    filter_fn: F, opts: RunOptions<'_>) -> Result<i32>
where F: Fn(&str) -> String
```

阶段: 执行 -> 过滤 -> 打印+tee -> stderr 透传 -> 追踪 -> 返回退出码

## 过滤策略 (A-J)

### A: 逐行正则过滤 (30-70% 节省)
**使用者:** npm, tree, cargo build/install, 多数 TOML 过滤器

遍历各行, 跳过匹配噪声模式的行:
```rust
for line in output.lines() {
    if line.starts_with('>') && line.contains('@') { continue; }
    if line.trim_start().starts_with("npm WARN") { continue; }
    result.push(line.to_string());
}
```

### B: 状态机解析 (70-95% 节省)
**使用者:** pytest, rake, rspec (文本回退), cargo test

基于枚举的阶段跟踪:
```rust
enum ParseState { Header, TestProgress, Failures, Summary }
// 在标记行 (如 "=== FAILURES ===") 处转换
```

### C: JSON 注入 + 结构化解析 (60-90% 节省)
**使用者:** ruff, golangci-lint, rspec, vitest, playwright, aws, kubectl

注入 `--output-format=json` 或 `--format json`, 通过 serde 反序列化, 输出紧凑摘要。17 个文件使用 `serde_json::from_str`。

### D: NDJSON 流式处理 (80-95% 节省)
**使用者:** go test

注入 `-json` 标志, 将每行解析为 `GoTestEvent`, 按包聚合为 `PackageResult` 结构体。

### E: 分段/块过滤 (70-95% 节省)
**使用者:** cargo build/test/clippy, git diff

收集多行错误/警告块, 去除块间噪声, 输出块 + 摘要。限制前 15 个错误块。

### F: 多命令组合 (50-80% 节省)
**使用者:** git diff, git show, dotnet build/test

运行多个子命令并组合:
1. `git diff --stat` (文件变更摘要)
2. `git diff` (完整差异)
3. `compact_diff()` (截断 hunk, 按文件追踪)

### G: 去重 (80-95% 节省)
**使用者:** log_cmd, container (kubectl logs)

规范化行 (用占位符替换时间戳/UUID/十六进制/数字), 计数出现次数, 显示唯一模式及重复计数。

### H: 摘要生成与分组 (60-90% 节省)
**使用者:** tsc, mypy, golangci-lint, rubocop, ruff, grep

解析结构化诊断输出, 按文件或规则分组, 输出紧凑的分组摘要。

### I: 格式模板注入 (40-60% 节省)
**使用者:** docker ps, docker images

注入自定义 `--format` 模板以精确获取所需字段:
```rust
.args(["ps", "--format", "{{.ID}}\t{{.Names}}\t{{.Status}}\t{{.Image}}\t{{.Ports}}"])
```

### J: TOML DSL 管道 (可变节省)
**使用者:** 59 个内置 TOML 过滤器

8 阶段声明式管道。详见 [[toml-filter-dsl]]。

## 令牌节省策略

| 策略 | 节省率 | 机制 |
|------|--------|------|
| 噪声行去除 | 30-70% | 移除进度条、编译行、空行 |
| 成功短路 | 90-99% | 成功时输出单行摘要 |
| JSON 注入 + 模式压缩 | 60-90% | 解析结构化数据, 仅输出关键字段 |
| Diff 压缩 | 50-80% | 限制 hunk 为 100 行, 显示 stat 摘要 |
| 错误/失败聚焦 | 70-95% | 去除所有通过的测试, 仅显示失败 |
| 日志去重 | 80-95% | 规范化 + 计数唯一模式 |
| 格式模板注入 | 40-60% | 自定义 `--format` 获取精确字段 |
| 截断保护 | 安全网 | `truncate()`, `max_lines`, `head/tail_lines` |

## 三层优雅降级 (`src/parser/`)

```
ParseResult<T>:
  第 1 层 (完整)    -- JSON 完整解析, 紧凑摘要
  第 2 层 (降级)    -- JSON 失败, 正则提取, 警告标记
  第 3 层 (透传)    -- 所有解析失败, 截断原始输出 + [RTK:PASSTHROUGH]
```

由 vitest 和 playwright 通过 `OutputParser` 特征使用。`TokenFormatter` 特征提供 Compact/Verbose/Ultra 模式。

## 横切模式

- **枚举子命令路由:** git, cargo, container, dotnet, go, pnpm, vitest, prisma
- **标志感知过滤:** 检测 `--nocapture`, `--format`, `--json`, `--stat` 以在用户需要详细输出时跳过过滤
- **工具存在性回退:** `tool_exists("tsc")` -> 回退到 `npx tsc`
- **跨命令路由:** `lint_cmd` 根据项目语言路由到 `ruff_cmd` 或 `mypy_cmd`
- **噪声目录常量:** `NOISE_DIRS` (26 个模式) 由 ls.rs 和 tree.rs 共享
- **Ruby bundle exec 检测:** `ruby_exec("rspec")` 自动检测 `bundle exec` 还是直接调用

## 相关页面

- [[system-architecture]] -- 整体路由和数据流
- [[toml-filter-dsl]] -- TOML 声明式过滤器详情
- [[rust-patterns]] -- 过滤器模块中的代码规范
