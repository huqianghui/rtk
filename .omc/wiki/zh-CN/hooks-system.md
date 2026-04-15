---
title: 钩子系统
tags: [钩子, 安全, 集成, claude-code, gemini, copilot, 权限]
category: architecture
created: 2026-04-14
updated: 2026-04-14
---

# 钩子系统 (`src/hooks/`)

钩子系统是 RTK 与 AI 编程代理的集成层。它拦截 shell 命令, 将其改写为使用 RTK, 并执行安全策略。

## 支持的 AI 代理

Claude Code, Cursor, Codex, Gemini CLI, OpenCode, Windsurf, Cline, Kilocode, Antigravity

## 钩子生命周期

```
rtk init -g
  -> 写入 ~/.claude/hooks/rtk-rewrite.sh (嵌入二进制文件)
  -> 存储 SHA-256 哈希到 ~/.claude/hooks/.rtk-hook.sha256
  -> 修补 ~/.claude/settings.json, 添加 PreToolUse 钩子注册
  -> 写入 rtk-awareness.md 说明文件

AI 代理运行命令 (如 "git status")
  -> Claude Code 触发 PreToolUse 钩子
  -> rtk-rewrite.sh 读取 JSON, 调用 `rtk rewrite "git status"`
  -> permissions.rs 检查 deny/ask/allow 规则
  -> registry.rs 改写为 "rtk git status"
  -> 返回 JSON 包含改写后的命令 + 权限决定
  -> Claude Code 执行 "rtk git status"

启动时:
  -> integrity.rs 验证钩子 SHA-256
  -> hook_check.rs 警告钩子缺失/过期 (限频 1 次/天)
```

## Init 命令 (`init.rs`, 约 700 行)

`rtk init` 安装:
1. **钩子脚本** (`rtk-rewrite.sh`) -- 用于 PreToolUse 事件的 shell 脚本
2. **Settings.json 修补** -- 钩子注册条目
3. **RTK 感知 markdown** -- 注入到 CLAUDE.md/AGENTS.md 的精简文件
4. **项目本地 `.rtk/filters.toml` 模板**
5. **全局过滤器模板** 位于 `~/.config/rtk/filters.toml`
6. **完整性基线** -- 用于篡改检测的 SHA-256 哈希

**修补模式:** `Ask` (提示用户), `Auto` (CI), `Skip` (手动说明)

所有资源通过 `include_str!()` 嵌入。

## 改写命令 (`rewrite_cmd.rs`)

`rtk rewrite <cmd>` -- 钩子脚本使用的核心命令。

**退出码协议:**

| 退出码 | 含义 |
|--------|------|
| 0 | 改写允许 -- 钩子可自动允许 |
| 1 | 无 RTK 等效命令 -- 原样透传 |
| 2 | 匹配拒绝规则 -- 钩子延迟到原生拒绝 |
| 3 | 匹配询问规则 -- 改写但提示用户 |

**安全不变量:** `PermissionVerdict::Default` 映射到退出码 3 (询问), 而非退出码 0 (允许)。未识别的命令永远不会被自动允许。

## 钩子命令处理器 (`hook_cmd.rs`)

三种 AI 代理 JSON 格式:

| 代理 | 格式 | 支持询问 |
|------|------|----------|
| Claude Code | `tool_name` + `tool_input.command` | 是 (`updatedInput`) |
| Copilot CLI | `toolName` + `toolArgs` (驼峰式) | 仅拒绝+建议 |
| Gemini CLI | `tool_name` = `"run_shell_command"` | 否 (仅 allow/deny) |

**安全性:** 包含 `<<` (heredoc) 的命令永远不会被改写。

## 权限系统 (`permissions.rs`)

从 settings.json 读取 Claude Code 权限规则并评估命令。

**判定优先级:** Deny > Ask > Allow > Default (ask)

**配置文件 (合并):**
1. `$PROJECT_ROOT/.claude/settings.json`
2. `$PROJECT_ROOT/.claude/settings.local.json`
3. `~/.claude/settings.json`
4. `~/.claude/settings.local.json`

**模式匹配:** 精确匹配、带词边界的前缀匹配、尾部通配符 (`git push*`)、前导通配符 (`* --force`)、中间通配符 (`git * main`)、全局 (`*`)、冒号语法 (`sudo:*`)

**复合命令安全 (issue #1213):** 用 `&&`, `||`, `|`, `;` 链接的命令被拆分。每个非空段都必须独立匹配 allow 规则才能获得 `Allow` 判定。

## 完整性系统 (`integrity.rs`)

SHA-256 钩子篡改检测。参考: SA-2025-RTK-001。

**IntegrityStatus:** `Verified`, `Tampered`, `NoBaseline`, `NotInstalled`, `OrphanedHash`

**流程:**
1. 安装时 `store_hash()` -- 写入 `.rtk-hook.sha256` (只读 0o444)
2. 启动时 `runtime_check()` -- 比较存储的哈希与当前值
3. `Tampered` -> 阻止执行并报错, exit(1)
4. 无环境变量绕过 -- 合法修改需重新运行 `rtk init -g --auto-patch`

## 信任系统 (`trust.rs`)

控制项目本地 `.rtk/filters.toml` 的加载。这是安全边界, 因为过滤器可以改写输出。

**模型:** 加载前信任。未信任的过滤器被静默跳过。

**信任存储:** `~/.local/share/rtk/trusted_filters.json` -- 以规范路径为键, 存储 SHA-256 + 时间戳

**命令:** `rtk trust` (显示内容 + 风险摘要, 然后存储哈希), `rtk untrust`

**CI 覆盖:** `RTK_TRUST_PROJECT_FILTERS=1` 仅在同时设置 CI 环境变量 (`CI`, `GITHUB_ACTIONS` 等) 时有效。防止 `.envrc` 注入攻击。

**TOCTOU 防护:** 单次读取文件 -> 从缓冲区显示 -> 对同一缓冲区计算哈希。

## 钩子过期检测 (`hook_check.rs`)

通过 `# rtk-hook-version: N` 头部检查钩子版本 (当前版本: 3)。通过标记文件最多每天警告 1 次。交叉检查其他集成 (OpenCode, Cursor, Codex, Gemini) 以避免误报。

## 钩子审计 (`hook_audit_cmd.rs`)

`rtk hook-audit` 解析 `~/.local/share/rtk/hook-audit.log` (通过 `RTK_HOOK_AUDIT=1` 启用)。显示改写/跳过计数、最常改写的命令。

## 相关页面

- [[system-architecture]] -- 整体系统设计
- [[toml-filter-dsl]] -- 需信任的项目过滤器
- [[rust-patterns]] -- 安全模式 (无 unwrap, 复合命令拆分)
