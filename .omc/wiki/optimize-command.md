---
title: "rtk optimize — Personalized Optimization Engine"
tags: [optimize, personalization, session-analysis, toml-generation, config-tuning, corrections]
category: architecture
created: 2026-04-15
updated: 2026-04-15
---

# rtk optimize — Personalized Optimization Engine

Analyzes Claude Code session history and generates personalized optimization suggestions: uncovered command detection, auto-generated TOML filters, config parameter tuning, and CLI error correction rules.

## Motivation & Background

### The Gap Between Generic and Personal

RTK ships with **generic optimizations** — 59 TOML filters + 38 Rust filter modules covering common commands (git, cargo, npm, docker, etc.). These deliver 60-90% token savings for supported commands.

However, every developer's workflow is unique:

- **Uncovered commands**: A Terraform user runs `terraform plan` 40 times/day, but RTK has no Terraform filter. All that output passes through uncompressed.
- **Suboptimal config**: RTK's default `head_lines=50` works for most users, but if your `cargo test` output is consistently <25 lines, you're wasting tokens on the limit itself.
- **Repeated mistakes**: A developer types `git commit --ammend` three times before correcting to `--amend`. An AI assistant should learn this pattern.
- **Structured output waste**: Running `gh pr list --json` produces machine-readable output that RTK's text filters can't meaningfully compress, adding latency without benefit.

`rtk optimize` bridges this gap by analyzing **actual session behavior** and generating **personalized suggestions**.

### Relationship to Existing Commands

```
rtk discover  <- "Passive discovery: what did you miss?"
    | data reuse
    v
rtk optimize  <- "Active suggestions: what should you do?" (NEW)
    | apply
    v
rtk verify    <- "Validation: are generated filters correct?"
    | tracking
    v
rtk gain      <- "Measurement: how much did you save?"
```

`rtk optimize` fills the **solution generation** gap between `discover` (problem identification) and `verify` (solution validation).

## Use Cases

### Use Case 1: New Team Onboarding

A team adopts RTK but uses tools not in the default filter set (Bazel, Terraform, Pulumi, etc.).

```bash
# After 1 week of usage
rtk optimize --since 7

# Output: "Generate TOML filter for `bazel build` (87 uses, ~15,200 tokens/month saved)"
# Output: "Generate TOML filter for `pulumi up` (23 uses, ~8,400 tokens/month saved)"

rtk optimize --apply  # Auto-generates and installs TOML filters
```

### Use Case 2: Config Tuning After Extended Use

After a month of RTK usage, tracking data reveals optimization opportunities.

```bash
rtk optimize --since 30

# Output: "Exclude `gh api --jq` from filtering (structured JSON, only 3% savings)"
# Output: "Reduce head_lines for cargo test (P95 output < 25 lines)"
```

### Use Case 3: CLI Correction Rules

Developers make recurring typos that waste tokens on error output + retry.

```bash
rtk optimize

# Output: "Correction: git commit --ammend -> git commit --amend (3 occurrences)"
# Applied to: .claude/rules/cli-corrections.md
```

### Use Case 4: CI/CD Integration

Export optimization reports as JSON for dashboards or automated pipelines.

```bash
rtk optimize --format json --since 7 > optimization-report.json

# Periodic check via cron
0 9 * * 1 rtk optimize --apply --min-frequency 10
```

## CLI Interface

```
rtk optimize [OPTIONS]

OPTIONS:
    -p, --project <PATH>      Scope to a specific project (default: current directory)
    -a, --all                 Analyze all projects
    -s, --since <DAYS>        Analyze last N days (default: 30)
        --sessions <N>        Maximum sessions to analyze (default: 50)
    -f, --format <FMT>        Output format: text|json (default: text)
        --apply               Auto-apply all suggestions (writes config/filters/rules)
        --dry-run             Preview changes without applying
        --min-frequency <N>   Minimum command occurrences to generate suggestion (default: 5)
        --min-savings <PCT>   Minimum estimated savings % to generate TOML (default: 30)
    -v, --verbose             Show detailed analysis progress
```

### Examples

```bash
rtk optimize                          # Analyze last 30 days, text report
rtk optimize --since 7 --format json  # Last 7 days, JSON output
rtk optimize --dry-run                # Show what --apply would change
rtk optimize --apply                  # Apply all suggestions (with backups)
rtk optimize --min-frequency 3        # Lower detection threshold
rtk optimize --all --since 14         # All projects, last 2 weeks
```

## Architecture

### Module Structure

```
src/optimize/
├── mod.rs              <- Pipeline orchestrator: collect data -> 4 analyzers -> sort -> output
├── suggestions.rs      <- Type definitions (SuggestionKind, Suggestion, OptimizeReport)
├── uncovered.rs        <- Analyzer 1: Detect high-frequency uncovered commands
├── toml_generator.rs   <- Analyzer 2: Auto-generate TOML filter definitions
├── config_tuner.rs     <- Analyzer 3: Suggest config parameter tuning
├── corrections.rs      <- Analyzer 4: Extract CLI error correction rules
├── report.rs           <- Text and JSON report formatting
└── applier.rs          <- --apply execution engine (file writes with backups)
```

**Total: ~1800 lines of new code across 8 files, plus ~80 lines modified in existing files.**

### Data Flow

```
┌─────────────────────────────────────────────────────────┐
│                     rtk optimize                         │
│                      (mod.rs)                            │
│                                                          │
│  1. Resolve project filter                               │
│  2. ClaudeProvider.discover_sessions()                   │
│  3. provider.extract_commands() per session              │
│  4. Run 4 analyzers                                      │
│  5. Sort by impact_score descending                      │
│  6. Compute coverage (current vs projected)              │
│  7. Output report / apply / dry-run                      │
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
    │discover/│  │ core/  │  │sample │  │  learn/  │
    │registry │  │tracking│  │output │  │ detector │
    │classify │  │Tracker │  │analysis│  │find_corr │
    │_command()│  │queries │  │       │  │ections() │
    └─────────┘  └────────┘  └───────┘  └──────────┘
         │            │          │             │
         v            v          v             v
    ┌─────────────────────────────────────────────────┐
    │              Vec<Suggestion>                      │
    │  Sorted by impact_score descending               │
    └──────────────────────┬──────────────────────────┘
                           │
              ┌────────────┼────────────┐
              v            v            v
         ┌────────┐  ┌─────────┐  ┌─────────┐
         │report  │  │ --apply │  │--dry-run│
         │text/json│  │applier  │  │ applier │
         └────────┘  └─────────┘  └─────────┘
```

### Dependencies (Zero New Crates)

All implementation reuses existing dependencies:

| Crate | Usage in optimize |
|-------|-------------------|
| `regex` + `lazy_static` | Noise pattern detection in toml_generator |
| `serde` + `serde_json` | Suggestion/Report serialization |
| `toml` | TOML validation + generation |
| `anyhow` | Error handling with context |
| `dirs` | Global filters path resolution |

### Reused Modules (No Modifications)

| Module | What optimize uses |
|--------|-------------------|
| `discover::registry` | `classify_command()` — 69 rules to detect Supported/Unsupported/Ignored |
| `discover::registry` | `split_command_chain()` — split `&&`, `\|\|`, `;` compound commands |
| `discover::provider` | `ClaudeProvider` — read `~/.claude/projects/` JSONL sessions |
| `discover::provider` | `ExtractedCommand` — command + output_content + output_len |
| `learn::detector` | `find_corrections()` — sliding window error->fix detection |
| `learn::detector` | `deduplicate_corrections()` — merge similar corrections |
| `core::config` | `Config::load()` / `Config::save()` — read/write config.toml |
| `core::tracking` | `Tracker` — SQLite queries for savings data |

### Modified Modules

| Module | Change |
|--------|--------|
| `src/main.rs` | +`Commands::Optimize` variant, routing, `"optimize"` in RTK_META_COMMANDS |
| `src/core/tracking.rs` | +`output_percentiles_by_command()` — GROUP BY query for config tuner |

## Four Analyzers — Detailed Design

### Analyzer 1: Uncovered Command Detection (`uncovered.rs`)

**Purpose:** Find high-frequency commands that have no RTK filter, and auto-generate TOML filter suggestions.

**Algorithm:**

1. For each `ExtractedCommand`, call `split_command_chain()` then `classify_command()`
2. Accumulate `Classification::Unsupported` results into `HashMap<base_command, UncoveredStats>`
   - Track: count, total_output_chars, sample_outputs (max 5)
3. Filter by `count >= min_frequency`
4. Estimate monthly token savings:
   ```
   avg_tokens = avg_output_chars / 4
   monthly_count = count * 30 / days_covered
   estimated_savings = avg_tokens * (min_savings_pct / 100) * monthly_count
   ```
5. Call `toml_generator::generate_toml_filter()` for each command
6. Return as `Vec<Suggestion>` with `SuggestionKind::GenerateTomlFilter`

**Impact score:** `sqrt(count * avg_output_chars / 1000)`, capped at 100.

### Analyzer 2: TOML Filter Auto-Generation (`toml_generator.rs`)

**Purpose:** Given a command name and sample outputs, infer filtering rules and generate a valid TOML filter definition.

**Noise pattern detection** via `lazy_static!` regexes:

| Pattern | Regex | Typical hit rate |
|---------|-------|-----------------|
| Empty lines | `^\s*$` | 10-30% |
| Timestamps | `^\s*\d{4}-\d{2}-\d{2}[T ]\d{2}:\d{2}:\d{2}` | 0-80% (log output) |
| Progress bars | `\d+%\|[bars]\|\.{4,}\|[=+>]` | 0-50% (install/build) |
| Separators | `^[\s\-=\*_]{3,}\s*$` | 5-15% |

**Algorithm:**

1. Collect all lines across all sample outputs
2. For each noise pattern, compute hit rate; include in `strip_lines_matching` if >60%
3. Always include empty line stripping as baseline
4. Compute line count statistics for truncation hints:
   - If max_line_count > 100: add `head_lines`, `tail_lines`, `max_lines`
5. Detect success short-circuits: phrases in >80% of short outputs -> `match_output` rules
   - Candidates: "success", "ok", "done", "complete", "passed", "up to date", "0 errors"
6. Render complete TOML with `match_command`, `strip_ansi`, `strip_lines_matching`, truncation, `on_empty`
7. Add inline test section from first sample
8. **Validate** generated TOML via `toml::from_str::<toml::Value>()`; fall back to minimal filter on parse failure

**Returns `None`** if samples are too sparse (no noise patterns detected AND total lines < 20).

### Analyzer 3: Config Parameter Tuning (`config_tuner.rs`)

**Purpose:** Analyze tracking data and current config to suggest parameter optimizations.

**Sub-analyses:**

**3a. Low-savings detection:**
- Calls `tracker.low_savings_commands(20)` to find commands with avg savings <30%
- For structured output (contains `--json`, `--format json`, `-o json`): suggest `ExcludeCommand`
- For other low-savings commands: suggest `TuneConfig` (review filter rules)

**3b. Output percentile analysis:**
- Calls `tracker.output_percentiles_by_command()` (new SQL query)
- Returns (command, count, avg_output_tokens, max_output_tokens) for commands with count >= 5
- If avg_tokens < 50 and max_tokens < 200 but passthrough_max_chars > 500: suggest reducing limit
- If avg_tokens > 2000 and max_tokens > 5000: suggest adding head/tail truncation with estimated savings

### Analyzer 4: CLI Error Corrections (`corrections.rs`)

**Purpose:** Extract recurring error->fix patterns and generate correction rules for `.claude/rules/`.

**100% reuse of learn module** — no new detection logic:

1. Call `learn::detector::find_corrections(commands)` — sliding window + Jaccard similarity
2. Filter by `confidence >= 0.6`
3. Call `learn::detector::deduplicate_corrections()` — merge similar corrections
4. Filter by `occurrences >= min_occurrences`
5. Convert each `CorrectionRule` to `Suggestion::WriteCorrection`

### Suggestion Priority & Scoring

All suggestions are sorted by `impact_score` descending:

| Suggestion Type | Score Formula | Typical Range |
|----------------|---------------|---------------|
| GenerateTomlFilter | `sqrt(count * avg_output / 1000)` | 10-100 |
| ExcludeCommand | Fixed 30 | 30 |
| TuneConfig (review) | Fixed 15-20 | 15-20 |
| WriteCorrection | `occurrences * 10` | 10-50 |

## Type Definitions (`suggestions.rs`)

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
    pub estimated_tokens_saved: u64,// Monthly estimate
    pub confidence: f64,            // 0.0-1.0
    pub description: String,        // Human-readable
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

## Apply Engine (`applier.rs`)

The `--apply` flag writes suggestions to disk. The `--dry-run` flag previews changes.

### Actions Per Suggestion Kind

| Kind | Target | Action |
|------|--------|--------|
| `GenerateTomlFilter` | `~/.config/rtk/filters.toml` | Append TOML filter definition |
| `TuneConfig` | `config.toml` | Load, modify field, save |
| `WriteCorrection` | `.claude/rules/cli-corrections.md` | Append correction rule |
| `ExcludeCommand` | `config.toml` → `hooks.exclude_commands` | Add to exclusion list |

### Safety Guarantees

- **Backup before write**: All target files backed up with `.bak` extension
- **Create parent dirs**: Missing directories created automatically
- **Append-only for TOML**: Never overwrites existing filter definitions
- **Config backward-compatible**: Only adds fields, never removes

## Output Examples

### Text Report

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

### JSON Report

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

## Coverage Computation

The report includes current and projected RTK filter coverage:

```rust
fn compute_coverage(commands: &[ExtractedCommand], suggestions: &[Suggestion]) -> (f64, f64) {
    // For each command: split chains, classify each part
    // current = supported_count / total_count * 100
    // projected = (supported + new_toml_filters) / total * 100
}
```

This uses the same `classify_command()` from `discover::registry` that powers `rtk discover` and `rtk session`.

## Testing

### Unit Tests (in each module)

| Module | Tests |
|--------|-------|
| `suggestions.rs` | Serialization round-trip, all SuggestionKind variants |
| `uncovered.rs` | Low-frequency filtering, frequent command detection, supported command exclusion |
| `toml_generator.rs` | Basic filter generation, empty input, timestamp detection, TOML validity |
| `config_tuner.rs` | Structured output detection, suggestion categories |
| `corrections.rs` | Correction detection, deduplication, confidence filtering |
| `report.rs` | Text format sections, JSON round-trip, impact formatting, token formatting |
| `applier.rs` | Dry-run formatting, applied results formatting, empty suggestions |
| `mod.rs` | Coverage computation (empty, all-supported, mixed) |

### Build Verification

```bash
cargo fmt --all && cargo clippy --all-targets && cargo test --all
# Result: 1473 tests passed, 0 failed, 0 new warnings
```

### Manual Testing

```bash
rtk optimize --help              # Verify CLI args
rtk optimize --since 7           # Analyze real sessions
rtk optimize --format json       # Verify JSON structure
rtk optimize --dry-run           # Preview without writing
```

## Implementation Timeline

| Phase | Components | Status |
|-------|-----------|--------|
| P0 | suggestions.rs, uncovered.rs, corrections.rs, mod.rs, report.rs | Done |
| P1 | config_tuner.rs, toml_generator.rs, applier.rs | Done |
| P2 | JSON output, --dry-run preview | Done |
| P3 | --watch mode (fsnotify, future) | Not planned |

**Total implementation: ~1800 lines across 8 new files + ~80 lines in 2 modified files. Zero new crate dependencies.**

## Related Pages

- [[system-architecture]] — Command routing, how new commands integrate
- [[core-infrastructure]] — Tracking system, config loading
- [[filter-patterns]] — Filter implementation patterns
- [[toml-filter-dsl]] — TOML filter DSL specification
- [[analytics-system]] — discover, learn, session modules
- [[fallback-system]] — TOML filter matching mechanism
