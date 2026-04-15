---
title: 回退与代理系统
tags: [回退, 代理, 透传, 可扩展性]
category: architecture
created: 2026-04-14
updated: 2026-04-14
---

# 回退与代理系统

RTK 最重要的设计原则: **永远不阻止命令执行**。未知命令始终透明透传。

## 三层回退 (main.rs 第 1062-1187 行)

当 `Cli::try_parse()` 失败 (未识别的命令) 时:

### 第 1 层: RTK 元命令错误
如果 `args[0]` 在 `RTK_META_COMMANDS` 中 (gain, discover, learn, init, config, proxy, hook-audit, cc-economics, verify, trust, untrust, session, rewrite), 显示 Clap 的错误消息。这些是 RTK 自身的命令, 解析失败意味着参数/语法错误。

### 第 2 层: TOML 过滤器匹配
在 TOML 过滤器注册表中查找命令:
```rust
toml_filter::find_matching_filter(&lookup_cmd)
```
使用 args[0] 的基本名称, 因此 `/usr/bin/make` 匹配 `^make\b`。如果匹配:
- 捕获 stdout (如果 `filter.filter_stderr` 则也捕获 stderr)
- 应用 8 阶段 TOML 管道
- 打印过滤后的结果
- 在 SQLite 中以 "rtk:toml" 前缀追踪

### 第 3 层: 纯透传
无 TOML 匹配。直接流式传输:
```rust
resolved_command(args[0])
    .stdin(Stdio::inherit())
    .stdout(Stdio::inherit())
    .stderr(Stdio::inherit())
    .status()
```
作为透传追踪 (0 令牌, 仅计时)。记录解析失败用于分析。

## 代理模式 (`rtk proxy <command>`)

带追踪的显式透传。用于:
- **绕过 RTK 过滤** -- 当过滤器有 bug 或需要完整输出时
- **追踪使用指标** -- 对 RTK 未过滤的命令
- **保证兼容性** -- 始终有效

```bash
rtk proxy git log --oneline -20    # 完整输出, 已追踪
rtk proxy npm install express      # 原始输出, 已追踪
```

实现:
- 以 `Stdio::piped()` 派生子进程, 用于 stdout+stderr
- 两个读取线程: 流式传输到终端 + 捕获 (每个最多 1MB)
- 注册 SIGINT/SIGTERM 处理程序用于子进程 PID
- `ChildGuard` RAII 结构体确保提前退出时的清理
- 作为 input=output (0% 节省) 追踪到 SQLite

## 安全属性

1. **任何命令都有效** -- `rtk <任何命令>` 是安全的前缀
2. **退出码保留** -- 正确处理 Unix 信号 (128+sig)
3. **无静默失败** -- 解析失败通过 `record_parse_failure_silent()` 记录
4. **未知命令流式传输** -- 透传使用 `Stdio::inherit()`, 无缓冲
5. **全部有指标** -- 即使透传也记录计时用于 `rtk gain --history`

## 相关页面

- [[system-architecture]] -- 导致回退的整体路由
- [[toml-filter-dsl]] -- 第 2 层 TOML 匹配
- [[core-infrastructure]] -- 追踪系统
