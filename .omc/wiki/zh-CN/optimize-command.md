---
title: "rtk optimize — 个性化优化引擎"
tags: [optimize, 个性化, 会话分析, TOML生成, 配置调优, 纠错规则]
category: architecture
created: 2026-04-15
updated: 2026-04-15
---

# rtk optimize — 个性化优化引擎

分析 Claude Code 会话历史，生成个性化优化建议：未覆盖命令检测、自动生成 TOML 过滤器、配置参数调优、CLI 错误纠正规则。

## 动机与背景

### 通用优化与个性化优化之间的差距

RTK 自带 **通用优化** — 59 个 TOML 过滤器 + 38 个 Rust 过滤模块，覆盖常见命令（git、cargo、npm、docker 等），可实现 60-90% 的令牌节省。

但每个开发者的工作流都是独特的：

- **未覆盖命令**：Terraform 用户每天运行 `terraform plan` 40 次，但 RTK 没有 Terraform 过滤器，所有输出未经压缩直接透传。
- **配置不够优化**：RTK 默认 `head_lines=50` 适合大多数用户，但如果你的 `cargo test` 输出始终少于 25 行，这个限制本身就在浪费令牌。
- **重复错误**：开发者输入 `git commit --ammend` 三次才纠正为 `--amend`，AI 助手应该学习这种模式。
- **结构化输出浪费**：运行 `gh pr list --json` 产生机器可读输出，RTK 的文本过滤器无法有效压缩，只增加延迟而无收益。

`rtk optimize` 通过分析 **实际会话行为** 并生成 **个性化建议** 来弥合这一差距。

### 与现有命令的关系

```
rtk discover  <- "被动发现：你错过了什么？"
    | 数据复用
    v
rtk optimize  <- "主动建议：你应该怎么做？"（新）
    | 应用
    v
rtk verify    <- "验证：生成的过滤器是否正确？"
    | 追踪
    v
rtk gain      <- "度量：你节省了多少？"
```

`rtk optimize` 填补了 `discover`（问题识别）和 `verify`（方案验证）之间的 **方案生成** 环节。

## 使用场景

### 场景 1：新团队接入

团队采用 RTK，但使用了默认过滤器集之外的工具（Bazel、Terraform、Pulumi 等）。

```bash
# 使用一周后
rtk optimize --since 7

# 输出："为 `bazel build` 生成 TOML 过滤器（87 次使用，每月可节省约 15,200 令牌）"
# 输出："为 `pulumi up` 生成 TOML 过滤器（23 次使用，每月可节省约 8,400 令牌）"

rtk optimize --apply  # 自动生成并安装 TOML 过滤器
```

### 场景 2：长期使用后的配置调优

使用 RTK 一个月后，追踪数据揭示了优化机会。

```bash
rtk optimize --since 30

# 输出："将 `gh api --jq` 排除在过滤之外（结构化 JSON，仅 3% 节省率）"
# 输出："降低 cargo test 的 head_lines（P95 输出 < 25 行）"
```

### 场景 3：CLI 纠错规则

开发者反复输入错误命令，浪费令牌在错误输出和重试上。

```bash
rtk optimize

# 输出："纠正：git commit --ammend -> git commit --amend（3 次出现）"
# 应用到：.claude/rules/cli-corrections.md
```

### 场景 4：CI/CD 集成

导出优化报告为 JSON 格式，用于仪表板或自动化管道。

```bash
rtk optimize --format json --since 7 > optimization-report.json

# 通过 cron 定期检查
0 9 * * 1 rtk optimize --apply --min-frequency 10
```

## CLI 接口

```
rtk optimize [OPTIONS]

OPTIONS:
    -p, --project <PATH>      限定项目范围（默认：当前目录）
    -a, --all                 分析所有项目
    -s, --since <DAYS>        分析最近 N 天（默认：30）
        --sessions <N>        最多分析 N 个会话（默认：50）
    -f, --format <FMT>        输出格式：text|json（默认：text）
        --apply               自动应用所有建议（写入配置/过滤器/规则）
        --dry-run             预览变更但不执行
        --min-frequency <N>   命令最少出现次数才生成建议（默认：5）
        --min-savings <PCT>   预估最低节省率才生成 TOML（默认：30）
    -v, --verbose             显示详细分析过程
```

### 示例

```bash
rtk optimize                          # 分析最近 30 天，文本报告
rtk optimize --since 7 --format json  # 最近 7 天，JSON 输出
rtk optimize --dry-run                # 预览 --apply 会做什么
rtk optimize --apply                  # 应用所有建议（带备份）
rtk optimize --min-frequency 3        # 降低检测阈值
rtk optimize --all --since 14         # 所有项目，最近 2 周
```

## 架构

### 模块结构

```
src/optimize/
├── mod.rs              <- 管道编排器：收集数据 -> 4 个分析器 -> 排序 -> 输出
├── suggestions.rs      <- 类型定义（SuggestionKind、Suggestion、OptimizeReport）
├── uncovered.rs        <- 分析器 1：检测高频未覆盖命令
├── toml_generator.rs   <- 分析器 2：自动生成 TOML 过滤器定义
├── config_tuner.rs     <- 分析器 3：建议配置参数调优
├── corrections.rs      <- 分析器 4：提取 CLI 错误纠正规则
├── report.rs           <- 文本和 JSON 报告格式化
└── applier.rs          <- --apply 执行引擎（带备份的文件写入）
```

**共计：8 个新文件约 1800 行新代码，加上现有文件约 80 行修改。**

### 数据流

```
┌─────────────────────────────────────────────────────────┐
│                     rtk optimize                         │
│                      (mod.rs)                            │
│                                                          │
│  1. 解析项目过滤器                                        │
│  2. ClaudeProvider.discover_sessions()                   │
│  3. provider.extract_commands() 逐会话提取                │
│  4. 运行 4 个分析器                                       │
│  5. 按 impact_score 降序排列                              │
│  6. 计算覆盖率（当前 vs 预估）                             │
│  7. 输出报告 / 应用 / 试运行                               │
└──────────┬──────────┬──────────┬──────────┬─────────────┘
           │          │          │          │
           v          v          v          v
    ┌──────────┐ ┌─────────┐ ┌────────┐ ┌───────────┐
    │uncovered │ │ config  │ │  toml  │ │corrections│
    │  .rs     │ │tuner.rs │ │gen.rs  │ │   .rs     │
    └────┬─────┘ └────┬────┘ └───┬────┘ └─────┬─────┘
         │            │          │             │
         v            v          v             v
    ┌─────────┐  ┌────────┐  ┌───────┐  ┌──────────┐
    │discover/│  │ core/  │  │样本   │  │  learn/  │
    │registry │  │tracking│  │输出   │  │ detector │
    │classify │  │Tracker │  │分析   │  │find_corr │
    │_command()│  │查询    │  │       │  │ections() │
    └─────────┘  └────────┘  └───────┘  └──────────┘
         │            │          │             │
         v            v          v             v
    ┌─────────────────────────────────────────────────┐
    │              Vec<Suggestion>                      │
    │  按 impact_score 降序排列                         │
    └──────────────────────┬──────────────────────────┘
                           │
              ┌────────────┼────────────┐
              v            v            v
         ┌────────┐  ┌─────────┐  ┌─────────┐
         │report  │  │ --apply │  │--dry-run│
         │文本/JSON│  │applier  │  │ applier │
         └────────┘  └─────────┘  └─────────┘
```

### 依赖（零新增 crate）

所有实现复用现有依赖：

| Crate | 在 optimize 中的用途 |
|-------|---------------------|
| `regex` + `lazy_static` | toml_generator 中的噪声模式检测 |
| `serde` + `serde_json` | Suggestion/Report 序列化 |
| `toml` | TOML 验证 + 生成 |
| `anyhow` | 带上下文的错误处理 |
| `dirs` | 全局过滤器路径解析 |

### 复用模块（无修改）

| 模块 | optimize 使用的功能 |
|------|-------------------|
| `discover::registry` | `classify_command()` — 69 条规则检测 Supported/Unsupported/Ignored |
| `discover::registry` | `split_command_chain()` — 分割 `&&`、`\|\|`、`;` 复合命令 |
| `discover::provider` | `ClaudeProvider` — 读取 `~/.claude/projects/` JSONL 会话 |
| `discover::provider` | `ExtractedCommand` — 命令 + output_content + output_len |
| `learn::detector` | `find_corrections()` — 滑动窗口错误->修复检测 |
| `learn::detector` | `deduplicate_corrections()` — 合并相似纠正 |
| `core::config` | `Config::load()` / `Config::save()` — 读写 config.toml |
| `core::tracking` | `Tracker` — SQLite 查询节省数据 |

### 修改的模块

| 模块 | 变更 |
|------|------|
| `src/main.rs` | +`Commands::Optimize` 变体、路由、RTK_META_COMMANDS 中添加 `"optimize"` |
| `src/core/tracking.rs` | +`output_percentiles_by_command()` — 为配置调优器提供的 GROUP BY 查询 |

## 四个分析器 — 详细设计

### 分析器 1：未覆盖命令检测（`uncovered.rs`）

**目的：** 查找没有 RTK 过滤器的高频命令，自动生成 TOML 过滤器建议。

**算法：**

1. 对每个 `ExtractedCommand`，调用 `split_command_chain()` 然后 `classify_command()`
2. 将 `Classification::Unsupported` 结果累积到 `HashMap<base_command, UncoveredStats>`
   - 追踪：count、total_output_chars、sample_outputs（最多 5 个）
3. 按 `count >= min_frequency` 过滤
4. 估算每月令牌节省：
   ```
   avg_tokens = avg_output_chars / 4
   monthly_count = count * 30 / days_covered
   estimated_savings = avg_tokens * (min_savings_pct / 100) * monthly_count
   ```
5. 对每个命令调用 `toml_generator::generate_toml_filter()`
6. 返回 `Vec<Suggestion>`，类型为 `SuggestionKind::GenerateTomlFilter`

**影响分数：** `sqrt(count * avg_output_chars / 1000)`，上限 100。

### 分析器 2：TOML 过滤器自动生成（`toml_generator.rs`）

**目的：** 根据命令名和输出样本，推断过滤规则并生成有效的 TOML 过滤器定义。

**噪声模式检测** 通过 `lazy_static!` 正则表达式：

| 模式 | 正则表达式 | 典型命中率 |
|------|-----------|-----------|
| 空行 | `^\s*$` | 10-30% |
| 时间戳 | `^\s*\d{4}-\d{2}-\d{2}[T ]\d{2}:\d{2}:\d{2}` | 0-80%（日志输出） |
| 进度条 | `\d+%\|[bars]\|\.{4,}\|[=+>]` | 0-50%（安装/构建） |
| 分隔线 | `^[\s\-=\*_]{3,}\s*$` | 5-15% |

**算法：**

1. 收集所有样本输出的所有行
2. 对每个噪声模式计算命中率；如果 >60% 则包含在 `strip_lines_matching` 中
3. 始终包含空行过滤作为基准
4. 计算行数统计用于截断提示：
   - 如果 max_line_count > 100：添加 `head_lines`、`tail_lines`、`max_lines`
5. 检测成功短路：在 >80% 的短输出中出现的短语 -> `match_output` 规则
   - 候选词："success"、"ok"、"done"、"complete"、"passed"、"up to date"、"0 errors"
6. 渲染完整 TOML：`match_command`、`strip_ansi`、`strip_lines_matching`、截断、`on_empty`
7. 从第一个样本添加内联测试部分
8. **验证** 生成的 TOML：通过 `toml::from_str::<toml::Value>()` 解析；解析失败则回退到最小过滤器

**返回 `None`** 的条件：样本太稀疏（未检测到噪声模式且总行数 < 20）。

### 分析器 3：配置参数调优（`config_tuner.rs`）

**目的：** 分析追踪数据和当前配置，建议参数优化。

**子分析：**

**3a. 低节省率检测：**
- 调用 `tracker.low_savings_commands(20)` 查找平均节省率 <30% 的命令
- 对结构化输出（包含 `--json`、`--format json`、`-o json`）：建议 `ExcludeCommand`
- 对其他低节省率命令：建议 `TuneConfig`（检查过滤规则）

**3b. 输出百分位分析：**
- 调用 `tracker.output_percentiles_by_command()`（新 SQL 查询）
- 返回 (命令, 计数, 平均输出令牌, 最大输出令牌)，条件：计数 >= 5
- 如果 avg_tokens < 50 且 max_tokens < 200 但 passthrough_max_chars > 500：建议降低限制
- 如果 avg_tokens > 2000 且 max_tokens > 5000：建议添加 head/tail 截断，附带估算节省

### 分析器 4：CLI 错误纠正（`corrections.rs`）

**目的：** 提取重复的 错误->修正 模式，生成纠正规则到 `.claude/rules/`。

**100% 复用 learn 模块** — 无新检测逻辑：

1. 调用 `learn::detector::find_corrections(commands)` — 滑动窗口 + Jaccard 相似度
2. 按 `confidence >= 0.6` 过滤
3. 调用 `learn::detector::deduplicate_corrections()` — 合并相似纠正
4. 按 `occurrences >= min_occurrences` 过滤
5. 将每个 `CorrectionRule` 转换为 `Suggestion::WriteCorrection`

### 建议优先级与评分

所有建议按 `impact_score` 降序排列：

| 建议类型 | 评分公式 | 典型范围 |
|---------|---------|---------|
| GenerateTomlFilter | `sqrt(count * avg_output / 1000)` | 10-100 |
| ExcludeCommand | 固定 30 | 30 |
| TuneConfig（检查） | 固定 15-20 | 15-20 |
| WriteCorrection | `occurrences * 10` | 10-50 |

## 类型定义（`suggestions.rs`）

```rust
#[derive(Debug, Clone, Serialize)]
pub enum SuggestionKind {
    GenerateTomlFilter { toml_content: String },
    TuneConfig { field: String, current: String, suggested: String },
    WriteCorrection { wrong: String, right: String, error_type: String },
    ExcludeCommand { command: String, reason: String },
}

#[derive(Debug, Clone, Serialize)]
pub struct Suggestion {
    pub kind: SuggestionKind,
    pub category: String,           // "TOML Filter", "Config", "Correction", "Exclusion"
    pub impact_score: u32,          // 0-100
    pub estimated_tokens_saved: u64,// 每月估算
    pub confidence: f64,            // 0.0-1.0
    pub description: String,        // 人类可读
}

#[derive(Debug, Serialize)]
pub struct OptimizeReport {
    pub sessions_analyzed: usize,
    pub commands_analyzed: usize,
    pub days_covered: u64,
    pub suggestions: Vec<Suggestion>,
    pub total_estimated_monthly_savings: u64,
    pub current_coverage_pct: f64,
    pub projected_coverage_pct: f64,
}
```

## 应用引擎（`applier.rs`）

`--apply` 标志将建议写入磁盘。`--dry-run` 标志预览变更。

### 每种建议类型的操作

| 类型 | 目标 | 操作 |
|------|------|------|
| `GenerateTomlFilter` | `~/.config/rtk/filters.toml` | 追加 TOML 过滤器定义 |
| `TuneConfig` | `config.toml` | 加载、修改字段、保存 |
| `WriteCorrection` | `.claude/rules/cli-corrections.md` | 追加纠正规则 |
| `ExcludeCommand` | `config.toml` → `hooks.exclude_commands` | 添加到排除列表 |

### 安全保证

- **写入前备份**：所有目标文件使用 `.bak` 扩展名备份
- **创建父目录**：缺失的目录自动创建
- **TOML 仅追加**：永远不覆盖现有过滤器定义
- **配置向后兼容**：只添加字段，不删除

## 输出示例

### 文本报告

```
===============================================================
  RTK Optimize Report
===============================================================

  Sessions: 38 | Commands: 2,147 | Period: 30 days
  Coverage: 67.3% -> 89.1% (projected)
  Est. monthly savings: ~42,500 tokens

---------------------------------------------------------------
  TOML Filter Suggestions
---------------------------------------------------------------

  1. [HIGH] Generate TOML filter for `terraform apply` (42 uses, ~15,200 tokens/month)
  2. [MED]  Generate TOML filter for `kubectl logs` (38 uses, ~12,100 tokens/month)

---------------------------------------------------------------
  Config Tuning
---------------------------------------------------------------

  3. [MED]  Exclude `gh api --jq` from filtering (3% savings, structured output)
  4. [LOW]  Review filter for `cargo test` — only 12% avg savings

---------------------------------------------------------------
  CLI Corrections
---------------------------------------------------------------

  5. [MED]  git commit --ammend -> git commit --amend (3 occurrences)

---------------------------------------------------------------
  Apply: rtk optimize --apply
  Preview: rtk optimize --dry-run
```

### JSON 报告

```json
{
  "sessions_analyzed": 38,
  "commands_analyzed": 2147,
  "days_covered": 30,
  "current_coverage_pct": 67.3,
  "projected_coverage_pct": 89.1,
  "total_estimated_monthly_savings": 42500,
  "suggestions": [
    {
      "kind": {
        "GenerateTomlFilter": {
          "toml_content": "[filters.terraform-apply]\n..."
        }
      },
      "category": "TOML Filter",
      "impact_score": 85,
      "estimated_tokens_saved": 15200,
      "confidence": 0.7,
      "description": "Generate TOML filter for `terraform apply` (42 uses, ~15200 tokens/month saved)"
    }
  ]
}
```

## 覆盖率计算

报告包含当前和预估的 RTK 过滤器覆盖率：

```rust
fn compute_coverage(commands: &[ExtractedCommand], suggestions: &[Suggestion]) -> (f64, f64) {
    // 对每个命令：分割链式命令，对每部分分类
    // current = supported_count / total_count * 100
    // projected = (supported + new_toml_filters) / total * 100
}
```

使用与 `rtk discover` 和 `rtk session` 相同的 `discover::registry` 中的 `classify_command()`。

## 测试

### 单元测试（各模块内）

| 模块 | 测试内容 |
|------|---------|
| `suggestions.rs` | 序列化往返、所有 SuggestionKind 变体 |
| `uncovered.rs` | 低频过滤、高频命令检测、已支持命令排除 |
| `toml_generator.rs` | 基本过滤器生成、空输入、时间戳检测、TOML 有效性 |
| `config_tuner.rs` | 结构化输出检测、建议分类 |
| `corrections.rs` | 纠正检测、去重、置信度过滤 |
| `report.rs` | 文本格式章节、JSON 往返、影响格式化、令牌格式化 |
| `applier.rs` | 试运行格式化、应用结果格式化、空建议 |
| `mod.rs` | 覆盖率计算（空、全支持、混合） |

### 构建验证

```bash
cargo fmt --all && cargo clippy --all-targets && cargo test --all
# 结果：1473 个测试通过，0 个失败，0 个新警告
```

### 手动测试

```bash
rtk optimize --help              # 验证 CLI 参数
rtk optimize --since 7           # 分析真实会话
rtk optimize --format json       # 验证 JSON 结构
rtk optimize --dry-run           # 预览但不写入
```

## 实现时间线

| 阶段 | 组件 | 状态 |
|------|------|------|
| P0 | suggestions.rs、uncovered.rs、corrections.rs、mod.rs、report.rs | 已完成 |
| P1 | config_tuner.rs、toml_generator.rs、applier.rs | 已完成 |
| P2 | JSON 输出、--dry-run 预览 | 已完成 |
| P3 | --watch 模式（fsnotify，未来） | 未计划 |

**总计实现：8 个新文件约 1800 行 + 2 个已修改文件约 80 行。零新增 crate 依赖。**

## 相关页面

- [[system-architecture]] — 命令路由，新命令如何接入
- [[core-infrastructure]] — 追踪系统，配置加载
- [[filter-patterns]] — 过滤器实现模式
- [[toml-filter-dsl]] — TOML 过滤器 DSL 规范
- [[analytics-system]] — discover、learn、session 模块
- [[fallback-system]] — TOML 过滤器匹配机制
