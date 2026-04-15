---
title: TOML 过滤器 DSL
tags: [toml, 过滤器, dsl, 声明式, 管道]
category: pattern
created: 2026-04-14
updated: 2026-04-14
---

# TOML 过滤器 DSL

RTK 的声明式过滤系统允许在不编写 Rust 代码的情况下添加工具支持。

## 概述

59 个内置 TOML 过滤器处理没有专用 Rust 模块的命令 (terraform, make, gcc, brew, ansible, helm 等)。用户可以添加项目本地或全局 TOML 过滤器。

## 查找优先级 (首次匹配)

1. `.rtk/filters.toml` -- 项目本地 (需通过 [[hooks-system]] 信任)
2. `~/.config/rtk/filters.toml` -- 用户全局
3. 内置 -- `src/filters/*.toml` 由 `build.rs` 拼接, 通过 `include_str!` 嵌入
4. 透传 -- 无匹配, 由调用者直接流式传输

## 过滤器模式

```toml
[filters.terraform-plan]
description = "Terraform plan 输出过滤器"
match_command = "^terraform\\s+plan"
strip_ansi = true
filter_stderr = false

# 正则替换 (按顺序链式执行)
replace = [
    { pattern = "\\d{4}-\\d{2}-\\d{2}T[\\d:]+Z", replacement = "<timestamp>" },
]

# 短路: 如果输出匹配, 立即返回消息
match_output = [
    { pattern = "No changes", message = "terraform plan: 无变更" },
    { pattern = "0 Warning\\(s\\)\\n\\s+0 Error\\(s\\)", message = "ok", unless = "error" },
]

# 行过滤 (互斥)
strip_lines_matching = [
    "^Refreshing state",
    "^\\s*#.*unchanged",
    "^\\s*$",
]
# 或
keep_lines_matching = ["^error", "^warning"]

truncate_lines_at = 200        # 每行最大字符数
head_lines = 50                # 保留前 N 行
tail_lines = 10                # 保留后 N 行
max_lines = 80                 # 绝对行数上限
on_empty = "terraform plan: ok"  # 结果为空时的消息
```

## 8 阶段管道

由 `core/toml_filter.rs` 中的 `apply_filter()` 按顺序执行:

| 阶段 | 字段 | 操作 |
|------|------|------|
| 1 | `strip_ansi` | 移除 ANSI 转义码 |
| 2 | `replace` | 逐行正则替换 (规则链式执行) |
| 3 | `match_output` | 检查完整文本; 如匹配 (且无 `unless`), 返回消息 (短路) |
| 4 | `strip/keep_lines` | 通过 RegexSet 过滤行 (互斥) |
| 5 | `truncate_lines_at` | 截断每行到 N 个字符 |
| 6 | `head/tail_lines` | 保留前 N 行和/或后 N 行 |
| 7 | `max_lines` | 绝对行数上限 |
| 8 | `on_empty` | 结果为空时的替换消息 |

## 内联测试 DSL

每个过滤器可以包含由 `rtk verify` 执行的测试:

```toml
[[tests.gcc]]
name = "去除 include 链, 保留错误和警告"
input = """
In file included from /usr/include/stdio.h:42:
main.c:10:5: error: use of undeclared identifier 'foo'
"""
expected = "main.c:10:5: error: use of undeclared identifier 'foo'"
```

`rtk verify --require-all` 确保每个过滤器至少有一个测试 (CI 强制)。

## 内置过滤器分类 (59 个文件)

| 类别 | 工具 |
|------|------|
| 构建 | gcc, make, gradle, mvn-build, dotnet-build, swift-build, xcodebuild, trunk-build, pio-run, spring-boot, quarto-render |
| 代码检查 | biome, oxlint, shellcheck, hadolint, markdownlint, yamllint, basedpyright, ty, tofu-fmt, mix-format |
| 基础设施 | terraform-plan, tofu-init/plan/validate, helm, gcloud, ansible-playbook, systemctl-status, iptables, fail2ban-client, sops, liquibase |
| 包管理器 | brew-install, bundle-install, composer-install, poetry-install, uv-sync |
| 任务运行器 | just, task, turbo, nx, make, mise, pre-commit |
| 系统 | df, du, ps, stat, ping, ssh, rsync |
| 版本控制 | jj, yadm |
| 其他 | ollama, jira, jq, shopify-theme, skopeo |

## 构建时拼接

`build.rs` 读取所有 `src/filters/*.toml` 文件, 将它们拼接成单个字符串, 并通过 `include_str!` 嵌入。这意味着添加新的 TOML 过滤器只需创建文件 -- 无需 Rust 代码。

## 安全: 信任系统

项目本地 `.rtk/filters.toml` 文件受加载前信任机制约束:
- `rtk trust` 显示内容 + 风险摘要 (replace 规则, match_output, 全匹配), 存储 SHA-256
- 未信任的过滤器被静默跳过
- CI 覆盖: `RTK_TRUST_PROJECT_FILTERS=1` 仅在设置 CI 环境变量时有效

详见 [[hooks-system]] 的信任详情。

## 环境变量

- `RTK_NO_TOML=1` -- 完全绕过 TOML 过滤引擎
- `RTK_TOML_DEBUG=1` -- 将匹配诊断打印到 stderr

## 实现细节

**CompiledFilter** -- 所有正则在首次访问时预编译。注册表是 `lazy_static` 全局变量。

**API:**
- `find_matching_filter(command: &str) -> Option<&'static CompiledFilter>`
- `apply_filter(filter: &CompiledFilter, stdout: &str) -> String`
- `run_filter_tests() -> Vec<TestResult>` (用于 `rtk verify`)

## 相关页面

- [[filter-patterns]] -- 基于 Rust 的过滤策略
- [[hooks-system]] -- 信任系统详情
- [[core-infrastructure]] -- toml_filter.rs 引擎
