---
title: 分析与报告系统
tags: [分析, gain, 经济, discover, learn, 会话]
category: architecture
created: 2026-04-14
updated: 2026-04-14
---

# 分析与报告系统

RTK 提供全面的分析, 涵盖令牌节省、经济影响、错失机会和 CLI 学习。

## Gain 命令 (`analytics/gain.rs`)

`rtk gain` -- 主要的令牌节省仪表板。

**视图:**
- **默认:** KPI 摘要 (总命令数, 节省的令牌, 平均节省率 %, 执行时间, 效率仪表)
- **按命令:** 排名表格, 含计数、节省的令牌、平均节省率、影响条
- **时间维度:** `--daily`, `--weekly`, `--monthly`, `--all`
- **项目范围:** `--project` 过滤到当前工作目录
- **图表:** `--graph` 每日节省的 ASCII 柱状图 (最近 30 天)
- **历史:** `--history` 最近 10 个命令, 带层级指示器
- **配额:** `--quota --tier pro|5x|20x` 估算订阅配额节省
- **导出:** `--format json|csv`
- **失败:** `--failures` 解析失败摘要及恢复率

**健康警告:** 钩子缺失/过期时警告, 或 `RTK_DISABLED=1` 使用超过 10% 时警告。

## Claude Code 经济分析 (`analytics/cc_economics.rs`)

`rtk cost` -- 将 ccusage 消费数据与 RTK 节省数据结合。

**加权输入 CPT 公式:**
```
weighted_units = input + 5*output + 1.25*cache_create + 0.1*cache_read
input_cpt = total_cost / weighted_units
rtk_savings_usd = saved_tokens * input_cpt
```

RTK 节省的令牌按推导的输入每令牌成本计价, 因为它们是从未进入上下文的输入令牌。

**合并逻辑:** 按时间键连接 ccusage 周期数据与 RTK 追踪数据 (处理周对齐差异)。

## ccusage 集成 (`analytics/ccusage.rs`)

运行 `ccusage` npm 包获取 Claude Code 消费数据:
1. 检查 `ccusage` 二进制文件是否在 PATH 中
2. 回退到 `npx --yes ccusage`
3. 运行 `ccusage daily|weekly|monthly --json --since 20250101`
4. 解析 JSON 为 `CcusagePeriod` 结构体

不可用时返回 `Ok(None)` -- 经济模块回退到仅 RTK 数据。

## 会话采用率 (`analytics/session_cmd.rs`)

`rtk session` -- 衡量 Claude Code 会话中的 RTK 采用率。

**流程:**
1. 发现最近 10 个 Claude Code 会话
2. 从 JSONL 转录中提取所有 Bash 命令
3. 分类每个命令: 已使用 `rtk` 前缀 vs. 可被改写
4. 拆分链式命令, 每部分独立分类
5. 报告每个会话的采用率 % + 总体平均值

## 发现系统 (`src/discover/`)

`rtk discover` -- 在 Claude Code 历史中查找错失的 RTK 机会。

**组件:**
- **Shell 词法分析器** (`lexer.rs`) -- 手写分词器 (Arg, Operator, Pipe, Redirect, Shellism)
- **会话提供者** (`provider.rs`) -- 读取 `~/.claude/projects/` JSONL 转录
- **命令注册表** (`registry.rs`) -- 53 条规则, `classify_command()` 和 `rewrite_command()`
- **规则数据库** (`rules.rs`) -- 正则模式、RTK 等效命令、类别、节省估算
- **报告** (`report.rs`) -- 文本/JSON 输出, 含错失节省表、最常用未处理命令

**预分类规范化:** 去除环境变量前缀、绝对路径、git 全局选项。检测重定向操作符。46 个忽略的命令前缀 + 12 个精确匹配。

## 学习系统 (`src/learn/`)

`rtk learn` -- 检测重复的 CLI 错误并建议纠正。

**纠正检测算法:**
1. 查找 `is_error=true` 且输出包含错误关键字的命令
2. 跳过 TDD 循环错误 (编译失败、测试失败)
3. 在 3 个命令的窗口内向前查找
4. 通过参数的 Jaccard 相似度计算 `command_similarity()`
5. 如果纠正成功则置信度 +0.2
6. 置信度 >= 0.6 则接受

**ErrorType 枚举:** UnknownFlag, CommandNotFound, WrongSyntax, WrongPath, MissingArg, PermissionDenied, Other

**输出:** 控制台报告 (错误->正确对), 或用于 `.claude/rules/cli-corrections.md` 的 markdown 规则文件 (通过 `--write-rules`)

## 解析器基础设施 (`src/parser/`)

统一解析, 三层优雅降级:

| 层级 | 触发条件 | 输出 |
|------|----------|------|
| 1 (完整) | JSON 完整解析 | 紧凑摘要 |
| 2 (降级) | JSON 失败, 正则有效 | 带警告标记的摘要 |
| 3 (透传) | 所有解析失败 | 截断的原始输出 + `[RTK:PASSTHROUGH]` |

**标准类型:** `TestResult` (total, passed, failed, skipped, failures), `DependencyState` (packages, outdated)

**TokenFormatter 特征:** Compact (默认), Verbose, Ultra (`[ok]28 [x]1 [skip]0`)

**JSON 提取辅助:** `extract_json_object()` 通过大括号平衡在混乱输出中查找 JSON (pnpm 横幅、dotenv 消息)。

## 相关页面

- [[system-architecture]] -- 通过追踪系统的数据流
- [[core-infrastructure]] -- SQLite 追踪, tee 恢复
- [[hooks-system]] -- 钩子审计和会话分析
