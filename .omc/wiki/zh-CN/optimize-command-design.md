---
title: rtk optimize 命令设计方案
tags: [optimize, 个性化, 会话分析, TOML生成, 配置优化]
category: design
created: 2026-04-14
updated: 2026-04-14
---

# rtk optimize 命令设计方案

基于用户会话行为分析，提供个性化的命令过滤器和配置优化建议。

## 设计目标

RTK 当前提供 **通用优化** (59 个 TOML 过滤器 + 38 个 Rust 模块)。`rtk optimize` 补充 **个性化优化**：

| 维度 | 通用 RTK | rtk optimize 个性化 |
|------|---------|-------------------|
| 过滤器 | 预定义的常见命令 | 根据用户高频未覆盖命令自动生成 TOML |
| 参数 | 保守默认值 | 根据实际使用模式调整 head/tail/max_lines |
| 错误修正 | 无 | 基于历史纠错生成 .claude/rules |
| 覆盖检测 | rtk discover (被动报告) | 主动建议 + 可选自动应用 |

## CLI 接口

```
rtk optimize [OPTIONS]

OPTIONS:
    -p, --project <PATH>      限定项目范围 (默认: 当前目录)
    -a, --all                 分析所有项目
    -s, --since <DAYS>        分析最近 N 天 (默认: 30)
        --sessions <N>        最多分析 N 个会话 (默认: 50)
    -f, --format <FMT>        输出格式: text|json (默认: text)
        --apply               自动应用建议 (写入配置/过滤器/规则)
        --dry-run             显示将要应用的变更但不执行
        --min-frequency <N>   命令最少出现次数才生成建议 (默认: 5)
        --min-savings <PCT>   预估最低节省率才生成 TOML (默认: 30)
    -v, --verbose             显示详细分析过程
```

### Clap 定义

```rust
/// Analyze session history and generate personalized optimization suggestions
Optimize {
    #[arg(short, long)]
    project: Option<String>,

    #[arg(short, long)]
    all: bool,

    #[arg(short, long, default_value = "30")]
    since: u64,

    #[arg(long, default_value = "50")]
    sessions: usize,

    #[arg(short, long, default_value = "text")]
    format: String,

    #[arg(long)]
    apply: bool,

    #[arg(long)]
    dry_run: bool,

    #[arg(long, default_value = "5")]
    min_frequency: usize,

    #[arg(long, default_value = "30")]
    min_savings: f64,
},
```

**注册要求：**
- 添加 `"optimize"` 到 `RTK_META_COMMANDS` 数组
- 不加入 `is_operational_command()` 白名单 (元命令不需要钩子完整性检查)

## 模块结构

```
src/
├── optimize/                        ← 新模块
│   ├── mod.rs                       ← 公共 run() 入口，编排 4 个分析器
│   ├── uncovered.rs                 ← 分析器 1: 未覆盖命令检测
│   ├── config_tuner.rs              ← 分析器 2: 配置参数优化
│   ├── toml_generator.rs            ← 分析器 3: TOML 过滤器自动生成
│   ├── corrections.rs               ← 分析器 4: 错误模式规则生成
│   ├── suggestions.rs               ← Suggestion 类型定义 + 优先级排序
│   ├── applier.rs                   ← --apply 执行引擎
│   └── report.rs                    ← 文本/JSON 输出格式化
├── discover/                        ← 复用 (不修改)
│   ├── provider.rs                  ← ClaudeProvider, ExtractedCommand
│   ├── registry.rs                  ← classify_command(), 69 条规则
│   └── ...
├── learn/                           ← 复用 (不修改)
│   ├── detector.rs                  ← find_corrections(), CommandExecution
│   └── ...
├── analytics/                       ← 复用 (不修改)
│   └── session_cmd.rs               ← count_rtk_commands()
├── core/                            ← 复用 (不修改)
│   ├── tracking.rs                  ← Tracker SQLite 查询
│   ├── toml_filter.rs               ← CompiledFilter, TomlFilterDef
│   └── config.rs                    ← Config, LimitsConfig
└── main.rs                          ← 添加 Commands::Optimize + 路由
```

**新增文件:** 7 个 (~1200 行估算)
**修改文件:** 1 个 (main.rs: +20 行)
**复用不修改:** discover, learn, analytics, core 模块

## 数据流

```
┌─────────────────────────────────────────────────────────┐
│                     rtk optimize                         │
│                      (mod.rs)                            │
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
    │discover/│  │ core/  │  │claude │  │  learn/  │
    │registry │  │tracking│  │session│  │ detector │
    │classify │  │Tracker │  │JSONL  │  │find_corr │
    │_command()│  │queries │  │output │  │ections() │
    └─────────┘  └────────┘  └───────┘  └──────────┘

         │            │          │             │
         v            v          v             v
    ┌─────────────────────────────────────────────────┐
    │              Vec<Suggestion>                      │
    │            (suggestions.rs)                       │
    │  优先级排序: impact_score = frequency * savings   │
    └──────────────────────┬──────────────────────────┘
                           │
              ┌────────────┼────────────┐
              v            v            v
         ┌────────┐  ┌─────────┐  ┌─────────┐
         │report  │  │ --apply │  │--dry-run│
         │ .rs    │  │applier  │  │ applier │
         │text/json│  │  .rs   │  │  .rs    │
         └────────┘  └─────────┘  └─────────┘
```

## 核心类型定义 (suggestions.rs)

```rust
use serde::Serialize;

/// 建议类别
#[derive(Debug, Clone, Serialize)]
pub enum SuggestionKind {
    /// 为未覆盖的高频命令生成 TOML 过滤器
    GenerateTomlFilter {
        filter_name: String,
        match_command: String,
        toml_content: String,
    },
    /// 调整现有过滤器/配置参数
    TuneConfig {
        config_section: String,
        field: String,
        current_value: String,
        suggested_value: String,
        reason: String,
    },
    /// 生成错误纠正规则
    WriteCorrection {
        base_command: String,
        wrong_pattern: String,
        right_pattern: String,
        occurrences: usize,
    },
    /// 将命令加入 hooks.exclude_commands
    ExcludeCommand {
        command: String,
        reason: String,
    },
}

/// 单条优化建议
#[derive(Debug, Clone, Serialize)]
pub struct Suggestion {
    pub kind: SuggestionKind,
    pub category: String,           // "filter", "config", "correction", "exclusion"
    pub impact_score: f64,          // frequency * estimated_savings
    pub estimated_tokens_saved: u64,// 每月预估节省
    pub confidence: f64,            // 0.0 - 1.0
    pub description: String,        // 人类可读描述
}

/// 完整分析报告
#[derive(Debug, Serialize)]
pub struct OptimizeReport {
    pub sessions_analyzed: usize,
    pub commands_analyzed: usize,
    pub days_covered: u64,
    pub suggestions: Vec<Suggestion>,
    pub total_estimated_monthly_savings: u64,
    pub current_coverage_pct: f64,   // 当前 RTK 覆盖率
    pub projected_coverage_pct: f64, // 应用建议后预估覆盖率
}
```

## 四个分析器详细设计

### 分析器 1: 未覆盖命令检测 (uncovered.rs)

**复用:** `discover::registry::classify_command()`, `discover::provider::ClaudeProvider`

**输入:** 会话中所有命令 + 分类结果
**输出:** `Vec<Suggestion>` (GenerateTomlFilter 类型)

```rust
pub fn analyze_uncovered(
    commands: &[ExtractedCommand],
    min_frequency: usize,
    min_savings: f64,
) -> Vec<Suggestion>
```

**算法:**

1. 对每个命令调用 `classify_command()`:
   - `Supported` → 跳过 (已有过滤器)
   - `Ignored` → 跳过 (shell 内建等)
   - `Unsupported` → 累计到 `HashMap<String, UncoveredStats>`

2. `UncoveredStats` 结构:
   ```rust
   struct UncoveredStats {
       base_command: String,
       count: usize,
       total_output_chars: usize,
       sample_outputs: Vec<String>,  // 保留最多 5 个输出样本
       example_full_command: String,
   }
   ```

3. 过滤: `count >= min_frequency`

4. 对每个未覆盖命令，预估可节省的令牌:
   ```rust
   let avg_output_tokens = stats.total_output_chars / stats.count / 4;
   let estimated_savings = avg_output_tokens as f64 * (min_savings / 100.0);
   let monthly_savings = estimated_savings * stats.count as f64 * 30 / days_covered;
   ```

5. 生成 TOML 过滤器建议 (详见分析器 3)

6. 按 `impact_score = monthly_savings` 降序排列

### 分析器 2: 配置参数优化 (config_tuner.rs)

**复用:** `core::tracking::Tracker` 的 SQLite 查询方法

**输入:** 追踪数据库中的历史记录
**输出:** `Vec<Suggestion>` (TuneConfig 类型)

```rust
pub fn analyze_config(
    tracker: &Tracker,
    config: &Config,
) -> Result<Vec<Suggestion>>
```

**分析维度:**

#### 2a. 低节省率命令检测
```rust
// 复用 Tracker::low_savings_commands() (tracking.rs:989)
// 返回 avg savings < 30% 的命令
let low_savers = tracker.low_savings_commands()?;
```
对于低节省率命令，建议:
- 检查是否应加入 `hooks.exclude_commands` (对于 `gh --json` 类结构化输出)
- 检查是否可调整 TOML 过滤器参数增加过滤强度

#### 2b. 输出长度分布分析
```sql
-- 新增查询方法: output_length_percentiles()
SELECT rtk_cmd,
       COUNT(*) as cnt,
       AVG(output_tokens) as avg_out,
       -- 计算 P50, P90, P95
FROM commands
WHERE timestamp > datetime('now', '-30 days')
GROUP BY rtk_cmd
HAVING cnt >= 5
ORDER BY avg_out DESC
```
如果某命令 P50 输出 < 20 行但当前 `head_lines` 设为 50，建议降低。

#### 2c. LimitsConfig 优化
```rust
// 基于实际使用分析 limits 配置
// 例: 如果 grep_max_results=200 但用户平均只看 50 个结果
fn suggest_limits_tuning(
    tracker: &Tracker,
    limits: &LimitsConfig,
) -> Vec<Suggestion>
```

### 分析器 3: TOML 过滤器自动生成 (toml_generator.rs)

**核心创新:** 根据命令输出样本，自动推断过滤规则。

```rust
pub fn generate_toml_filter(
    command: &str,
    sample_outputs: &[String],
) -> Option<String>
```

**生成策略:**

#### 3a. 通用行模式检测
```rust
lazy_static! {
    // 可以安全删除的行模式
    static ref NOISE_PATTERNS: Vec<(&'static str, Regex)> = vec![
        ("empty_lines", Regex::new(r"^\s*$").unwrap()),
        ("timestamp_lines", Regex::new(r"^\d{4}-\d{2}-\d{2}[T ]").unwrap()),
        ("progress_bars", Regex::new(r"[\|/\-\\]\s*\d+%").unwrap()),
        ("separator_lines", Regex::new(r"^[-=_]{3,}$").unwrap()),
        ("ansi_only", Regex::new(r"^(\x1b\[[0-9;]*[a-zA-Z])+\s*$").unwrap()),
    ];
}
```

对输出样本统计每种噪声模式的命中率。如果某模式命中 > 60% 的行，加入 `strip_lines_matching`。

#### 3b. 输出长度启发式
```rust
fn infer_truncation(sample_outputs: &[String]) -> (Option<usize>, Option<usize>, Option<usize>) {
    let line_counts: Vec<usize> = sample_outputs.iter()
        .map(|s| s.lines().count())
        .collect();

    let median = percentile(&line_counts, 50);
    let p95 = percentile(&line_counts, 95);

    let head_lines = if median > 30 { Some(30) } else { None };
    let max_lines = if p95 > 80 { Some(80) } else { None };
    let truncate_at = if max_line_length(sample_outputs) > 200 { Some(200) } else { None };

    (head_lines, max_lines, truncate_at)
}
```

#### 3c. 成功/失败短路检测
```rust
// 如果所有"成功"输出包含相同模式，生成 match_output 规则
fn detect_success_patterns(sample_outputs: &[String]) -> Vec<MatchOutputRule> {
    // 查找在 >80% 的短输出中出现的通用短语
    // 例: "Successfully installed", "Build succeeded", "0 errors"
}
```

#### 3d. 生成 TOML

```rust
fn render_toml(
    name: &str,
    command_regex: &str,
    strip_ansi: bool,
    strip_lines: &[String],
    head_lines: Option<usize>,
    max_lines: Option<usize>,
    truncate_at: Option<usize>,
    on_empty: Option<String>,
    match_output: Vec<MatchOutputRule>,
) -> String {
    // 生成完整的 TOML 过滤器定义 + 内联测试
    format!(r#"[filters.{name}]
description = "Auto-generated filter for {command}"
match_command = "{command_regex}"
strip_ansi = {strip_ansi}
{strip_lines_section}
{head_lines_section}
{max_lines_section}
{truncate_section}
{on_empty_section}
{match_output_section}

[[tests.{name}]]
name = "auto-generated smoke test"
input = """{sample_input}"""
expected = """{sample_expected}"""
"#)
}
```

**输出位置:** `~/.config/rtk/filters.toml` (用户全局) 或 `.rtk/filters.toml` (项目本地)

### 分析器 4: 错误模式规则 (corrections.rs)

**完全复用:** `learn::detector::find_corrections()` + `learn::detector::deduplicate_corrections()`

```rust
pub fn analyze_corrections(
    commands: &[CommandExecution],
    min_confidence: f64,
    min_occurrences: usize,
) -> Vec<Suggestion>
```

**流程:**
1. 调用 `find_corrections(commands)` → `Vec<CorrectionPair>`
2. 过滤 `confidence >= min_confidence`
3. 调用 `deduplicate_corrections()` → `Vec<CorrectionRule>`
4. 过滤 `occurrences >= min_occurrences`
5. 转换为 `Suggestion` (WriteCorrection 类型)

**与 `rtk learn` 的关系:** 完全相同的检测逻辑，但输出为 `Suggestion` 而非直接写文件。当 `--apply` 时才写入 `.claude/rules/cli-corrections.md`。

## 编排流程 (mod.rs)

```rust
pub fn run(
    project: Option<String>,
    all: bool,
    since: u64,
    sessions: usize,
    format: String,
    apply: bool,
    dry_run: bool,
    min_frequency: usize,
    min_savings: f64,
    verbose: u8,
) -> Result<i32> {
    // 1. 收集会话数据
    let provider = ClaudeProvider;
    let project_filter = resolve_project_filter(&project, all)?;
    let session_paths = provider.discover_sessions(
        project_filter.as_deref(), Some(since)
    )?;
    let session_paths = &session_paths[..session_paths.len().min(sessions)];

    // 2. 提取命令和输出
    let (extracted, command_executions) = extract_all_commands(
        &provider, session_paths
    )?;

    // 3. 并行运行 4 个分析器 (实际为顺序，RTK 无 async)
    let mut suggestions = Vec::new();

    // 分析器 1: 未覆盖命令
    suggestions.extend(uncovered::analyze_uncovered(
        &extracted, min_frequency, min_savings
    ));

    // 分析器 2: 配置优化
    if let Ok(tracker) = tracking::Tracker::new() {
        suggestions.extend(config_tuner::analyze_config(
            &tracker, &Config::load().unwrap_or_default()
        )?);
    }

    // 分析器 3: TOML 生成 (基于分析器 1 的结果)
    for suggestion in &mut suggestions {
        if let SuggestionKind::GenerateTomlFilter { .. } = &suggestion.kind {
            // 已在分析器 1 中生成
        }
    }

    // 分析器 4: 错误纠正
    suggestions.extend(corrections::analyze_corrections(
        &command_executions, 0.6, 1
    ));

    // 4. 排序: 按 impact_score 降序
    suggestions.sort_by(|a, b| b.impact_score.partial_cmp(&a.impact_score)
        .unwrap_or(std::cmp::Ordering::Equal));

    // 5. 构建报告
    let report = build_report(
        session_paths.len(), extracted.len(), since, &suggestions
    );

    // 6. 输出
    match format.as_str() {
        "json" => println!("{}", serde_json::to_string_pretty(&report)?),
        _ => {
            println!("{}", report::format_text(&report));
            if dry_run {
                println!("\n{}", applier::format_dry_run(&suggestions)?);
            } else if apply {
                let applied = applier::apply_all(&suggestions)?;
                println!("\n{}", applier::format_applied(&applied));
            }
        }
    }

    Ok(0)
}
```

## 输出示例

### 文本格式

```
═══════════════════════════════════════════════════════════
  RTK Optimize — 个性化优化建议
═══════════════════════════════════════════════════════════

  分析范围: 38 个会话, 2,147 条命令, 最近 30 天
  当前覆盖率: 67.3% → 应用后预估: 89.1%
  预估每月额外节省: ~42,500 tokens ($0.85)

───────────────────────────────────────────────────────────
  TOML 过滤器建议 (3 条)
───────────────────────────────────────────────────────────

  #1  terraform apply  (42 次, ~15,200 tokens/月)
      → 生成 TOML: strip timestamps + progress + empty lines
        head_lines=40, max_lines=80, strip_ansi=true
      → 目标: ~/.config/rtk/filters.toml

  #2  kubectl logs  (38 次, ~12,100 tokens/月)
      → 生成 TOML: strip timestamps + keep error/warning lines
        keep_lines_matching=["error|warn|fatal|panic"]
      → 目标: ~/.config/rtk/filters.toml

  #3  yarn install  (21 次, ~5,600 tokens/月)
      → 生成 TOML: strip progress bars + resolution lines
        strip_lines_matching=["^Resolving:", "^\\[\\d+/\\d+\\]"]
      → 目标: ~/.config/rtk/filters.toml

───────────────────────────────────────────────────────────
  配置优化 (2 条)
───────────────────────────────────────────────────────────

  #4  cargo test: head_lines 50 → 25
      原因: 95% 的输出 < 25 行, 当前设置浪费令牌

  #5  hooks.exclude_commands += ["gh api"]
      原因: 12 次 gh api --jq 调用, 结构化输出不应被过滤

───────────────────────────────────────────────────────────
  错误纠正规则 (1 条)
───────────────────────────────────────────────────────────

  #6  git commit --ammend → git commit --amend  (3 次)
      → 写入: .claude/rules/cli-corrections.md

───────────────────────────────────────────────────────────

  应用建议? 运行: rtk optimize --apply
  预览变更: rtk optimize --dry-run
```

### JSON 格式

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
          "filter_name": "terraform-apply",
          "match_command": "^terraform\\s+apply",
          "toml_content": "[filters.terraform-apply]\n..."
        }
      },
      "category": "filter",
      "impact_score": 15200,
      "estimated_tokens_saved": 15200,
      "confidence": 0.82,
      "description": "Generate TOML filter for terraform apply (42 occurrences)"
    }
  ]
}
```

## --apply 执行引擎 (applier.rs)

```rust
pub fn apply_all(suggestions: &[Suggestion]) -> Result<Vec<ApplyResult>> {
    let mut results = Vec::new();

    for suggestion in suggestions {
        let result = match &suggestion.kind {
            SuggestionKind::GenerateTomlFilter { toml_content, .. } => {
                // 追加到 ~/.config/rtk/filters.toml
                append_toml_filter(toml_content)?
            }
            SuggestionKind::TuneConfig { config_section, field, suggested_value, .. } => {
                // 修改 ~/.config/rtk/config.toml
                update_config_field(config_section, field, suggested_value)?
            }
            SuggestionKind::WriteCorrection { .. } => {
                // 写入 .claude/rules/cli-corrections.md
                // 复用 learn::report::write_rules_file() 的逻辑
                write_correction_rule(suggestion)?
            }
            SuggestionKind::ExcludeCommand { command, .. } => {
                // 添加到 config.hooks.exclude_commands
                add_exclude_command(command)?
            }
        };
        results.push(result);
    }

    Ok(results)
}

#[derive(Debug)]
pub struct ApplyResult {
    pub suggestion_index: usize,
    pub target_file: PathBuf,
    pub action: String,        // "created", "appended", "modified"
    pub success: bool,
    pub error: Option<String>,
}
```

**安全约束:**
- 写入前备份目标文件 (`.bak` 后缀)
- TOML 过滤器写入后自动运行 `rtk verify` 验证
- 配置变更保持向后兼容 (只追加不删除)
- `--dry-run` 只输出 diff，不执行任何文件操作

## 与现有模块的集成关系

```
┌──────────────────────────────────────────────────────────────┐
│                    现有模块 (不修改)                            │
│                                                              │
│  discover/provider.rs ──→ ClaudeProvider                     │
│    - discover_sessions()   会话发现                            │
│    - extract_commands()    命令提取 + 输出内容                  │
│                                                              │
│  discover/registry.rs ──→ classify_command()                  │
│    - 69 条规则分类                                             │
│    - Supported/Unsupported/Ignored                           │
│                                                              │
│  learn/detector.rs ──→ find_corrections()                    │
│    - 滑动窗口检测错误→修正对                                    │
│    - Jaccard 相似度 + TDD 过滤                                │
│    - deduplicate_corrections()                               │
│                                                              │
│  core/tracking.rs ──→ Tracker                                │
│    - low_savings_commands()    低节省率检测                     │
│    - avg_savings_per_command() 每命令平均节省率                  │
│    - get_summary_filtered()    聚合统计                        │
│                                                              │
│  core/config.rs ──→ Config                                   │
│    - Config::load()  读取当前配置                               │
│    - Config::save()  保存修改后配置                              │
│                                                              │
│  core/toml_filter.rs ──→ TomlFilterDef                       │
│    - 了解 TOML 过滤器 schema                                   │
│    - find_matching_filter() 检查是否已有过滤器                   │
│                                                              │
│  analytics/session_cmd.rs ──→ count_rtk_commands()            │
│    - 计算覆盖率百分比                                           │
└──────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────┐
│                    新模块 (src/optimize/)                      │
│                                                              │
│  mod.rs          编排: 收集数据 → 4 分析器 → 排序 → 输出        │
│  uncovered.rs    调用 classify_command() 找未覆盖命令           │
│  config_tuner.rs 调用 Tracker 查询找低效配置                    │
│  toml_generator.rs 基于输出样本推断过滤规则                      │
│  corrections.rs  调用 find_corrections() 找纠错模式             │
│  suggestions.rs  Suggestion 类型 + 排序逻辑                    │
│  applier.rs      文件写入引擎 (TOML/config/rules)              │
│  report.rs       文本/JSON 格式化                              │
└──────────────────────────────────────────────────────────────┘
```

## 关键数据转换

### ExtractedCommand → 两种消费路径

```
ExtractedCommand (from discover/provider.rs)
├── command: String
├── output_content: Option<String>   ← 前 1000 字符
├── output_len: Option<usize>
├── is_error: bool
└── sequence_index: usize

路径 A (分析器 1+3):
  → classify_command(cmd) → Unsupported?
  → 累计 output_content 样本
  → toml_generator::generate_toml_filter()

路径 B (分析器 4):
  → 转换为 CommandExecution { command, is_error, output }
  → find_corrections() → CorrectionPair
  → deduplicate → CorrectionRule
  → 转换为 Suggestion::WriteCorrection
```

### 输出内容限制

`ClaudeProvider::extract_commands()` 仅保留前 ~1000 字符的输出内容。对于 TOML 生成器来说这已足够检测行模式，但可能不足以精确推断 `head_lines` 参数。

**解决方案:** 使用 `output_len` (完整输出长度) 来推断截断参数，使用 `output_content` (前 1000 字符) 来检测行模式。

## 执行模式

### 模式 1: 用户触发 (首选)

```bash
rtk optimize                    # 查看建议
rtk optimize --dry-run          # 预览变更
rtk optimize --apply            # 应用建议
```

**已具备完整基础设施，可立即实现。**

### 模式 2: 定期执行 (简单)

```bash
# 用户自行配置 cron
0 9 * * 1 rtk optimize --format json > ~/.rtk-optimize-report.json

# 或通过 rtk 提供便捷配置
rtk optimize --cron weekly      # 输出 crontab 条目
```

**不需要 RTK 内置调度，外部 cron 即可。** 可选加一个 `--cron` flag 输出 crontab 建议。

### 模式 3: 近实时 (长期目标)

```bash
rtk optimize --watch            # 监听会话目录变化
```

需要 `notify` crate (fsnotify) 监听 `~/.claude/projects/` 目录。当检测到新的会话文件关闭时触发分析。

**复杂度高，建议作为 v2 功能。** 核心价值已被模式 1 覆盖。

## 实现优先级

| 阶段 | 组件 | 复杂度 | 依赖 |
|------|------|--------|------|
| **P0** | suggestions.rs (类型定义) | 低 | 无 |
| **P0** | uncovered.rs (未覆盖检测) | 低 | discover/registry |
| **P0** | corrections.rs (错误纠正) | 低 | learn/detector |
| **P0** | mod.rs (编排 + CLI) | 中 | P0 组件 |
| **P0** | report.rs (文本输出) | 低 | suggestions |
| **P1** | config_tuner.rs (配置优化) | 中 | core/tracking |
| **P1** | toml_generator.rs (TOML 生成) | 中-高 | 输出样本分析 |
| **P1** | applier.rs (--apply 引擎) | 中 | 文件写入 + 备份 |
| **P2** | JSON 输出格式 | 低 | serde |
| **P2** | --cron 提示 | 低 | 无 |
| **P3** | --watch 模式 | 高 | notify crate |

**P0 实现估算:** ~500 行新代码 + 20 行 main.rs 修改
**P0+P1 完整估算:** ~1200 行新代码

## 新增依赖

**P0-P1 阶段: 零新增依赖**
- 所有需要的 crate 已存在: `regex`, `lazy_static`, `serde`/`serde_json`, `anyhow`, `toml`, `chrono`

**P3 阶段 (--watch):
- `notify` crate (~800KB, 跨平台 fsnotify)

## 测试策略

```rust
#[cfg(test)]
mod tests {
    use super::*;

    // 1. TOML 生成器测试: 给定样本输出，验证生成的 TOML 语法正确
    #[test]
    fn test_generate_toml_from_samples() {
        let samples = vec![
            "2024-01-01T12:00:00Z Starting...\nSuccess\n".to_string(),
            "2024-01-02T12:00:00Z Starting...\nSuccess\n".to_string(),
        ];
        let toml = toml_generator::generate_toml_filter("mycmd", &samples);
        assert!(toml.is_some());
        let toml_str = toml.unwrap();
        assert!(toml_str.contains("strip_lines_matching"));
        // 验证生成的 TOML 可被解析
        assert!(toml::from_str::<toml::Value>(&toml_str).is_ok());
    }

    // 2. 建议排序测试: 高影响建议排在前面
    #[test]
    fn test_suggestions_sorted_by_impact() { ... }

    // 3. 集成测试: 完整管道从假会话数据到报告
    #[test]
    fn test_full_pipeline_with_fixtures() { ... }

    // 4. applier 安全测试: 备份 + 回滚
    #[test]
    fn test_apply_creates_backup() { ... }
}
```

## 与现有命令的关系

```
rtk discover  ← "被动发现: 你错过了什么"
    ↓ 数据复用
rtk optimize  ← "主动建议: 你应该怎么做" (新)
    ↓ 应用
rtk verify    ← "验证: 生成的过滤器是否正确"
    ↓ 追踪
rtk gain      ← "度量: 应用后节省了多少"
```

`rtk optimize` 填补了 discover (发现问题) 和 verify (验证方案) 之间的 **方案生成** 环节。

## 相关页面

- [[system-architecture]] -- 命令路由，新命令如何接入
- [[core-infrastructure]] -- 追踪系统，配置加载
- [[filter-patterns]] -- 过滤器实现模式
- [[toml-filter-dsl]] -- TOML 过滤器 DSL 规范
- [[analytics-system]] -- discover, learn, session 模块
- [[fallback-system]] -- TOML 过滤器匹配机制
