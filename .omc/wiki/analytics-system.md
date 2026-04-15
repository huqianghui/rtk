---
title: Analytics & Reporting System
tags: [analytics, gain, economics, discover, learn, session]
category: architecture
created: 2026-04-14
updated: 2026-04-14
---

# Analytics & Reporting System

RTK provides comprehensive analytics across token savings, economic impact, missed opportunities, and CLI learning.

## Gain Command (`analytics/gain.rs`)

`rtk gain` -- the main token savings dashboard.

**Views:**
- **Default:** KPI summary (total commands, tokens saved, avg savings %, exec time, efficiency meter)
- **By-command:** Ranked table with count, saved tokens, avg savings, impact bar
- **Time breakdowns:** `--daily`, `--weekly`, `--monthly`, `--all`
- **Project scope:** `--project` filters to current working directory
- **Graph:** `--graph` ASCII bar chart of daily savings (last 30 days)
- **History:** `--history` last 10 commands with tier indicators
- **Quota:** `--quota --tier pro|5x|20x` estimates subscription quota preservation
- **Export:** `--format json|csv`
- **Failures:** `--failures` parse failure summary with recovery rate

**Health warnings:** Warns if hook missing/outdated, or if `RTK_DISABLED=1` used >10% of time.

## Claude Code Economics (`analytics/cc_economics.rs`)

`rtk cost` -- combines ccusage spending with RTK savings data.

**Weighted Input CPT formula:**
```
weighted_units = input + 5*output + 1.25*cache_create + 0.1*cache_read
input_cpt = total_cost / weighted_units
rtk_savings_usd = saved_tokens * input_cpt
```

RTK-saved tokens are valued at the derived input cost-per-token since they are input tokens that never entered context.

**Merge logic:** Joins ccusage period data with RTK tracking by time key (handles week alignment differences).

## ccusage Integration (`analytics/ccusage.rs`)

Runs `ccusage` npm package for Claude Code spending data:
1. Check if `ccusage` binary in PATH
2. Fallback to `npx --yes ccusage`
3. Run `ccusage daily|weekly|monthly --json --since 20250101`
4. Parse JSON into `CcusagePeriod` structs

Returns `Ok(None)` when unavailable -- economics module falls back to RTK-only data.

## Session Adoption (`analytics/session_cmd.rs`)

`rtk session` -- measures RTK adoption across Claude Code sessions.

**Flow:**
1. Discover last 10 Claude Code sessions
2. Extract all Bash commands from JSONL transcripts
3. Classify each: already `rtk` prefixed vs. would be rewritten
4. Split chained commands so each part is classified independently
5. Report adoption % per session + overall average

## Discover System (`src/discover/`)

`rtk discover` -- finds missed RTK opportunities in Claude Code history.

**Components:**
- **Shell Lexer** (`lexer.rs`) -- hand-written tokenizer (Arg, Operator, Pipe, Redirect, Shellism)
- **Session Provider** (`provider.rs`) -- reads `~/.claude/projects/` JSONL transcripts
- **Command Registry** (`registry.rs`) -- 53 rules with `classify_command()` and `rewrite_command()`
- **Rules Database** (`rules.rs`) -- regex patterns, RTK equivalents, category, savings estimates
- **Report** (`report.rs`) -- text/JSON output with missed savings table, top unhandled commands

**Pre-classification normalization:** Strips env var prefixes, absolute paths, git global options. Detects redirect operators. 46 ignored command prefixes + 12 exact matches.

## Learn System (`src/learn/`)

`rtk learn` -- detects repeated CLI mistakes and suggests corrections.

**Correction detection algorithm:**
1. Find commands where `is_error=true` and output contains error keywords
2. Skip TDD-cycle errors (compilation failures, test failures)
3. Look ahead within 3-command window
4. Calculate `command_similarity()` via Jaccard on arguments
5. Boost confidence +0.2 if correction succeeded
6. Accept if confidence >= 0.6

**ErrorType enum:** UnknownFlag, CommandNotFound, WrongSyntax, WrongPath, MissingArg, PermissionDenied, Other

**Output:** Console report with wrong->right pairs, or markdown rules file for `.claude/rules/cli-corrections.md` (via `--write-rules`)

## Parser Infrastructure (`src/parser/`)

Unified parsing with three-tier graceful degradation:

| Tier | When | Output |
|------|------|--------|
| 1 (Full) | JSON parsed completely | Compact summary |
| 2 (Degraded) | JSON failed, regex works | Summary with warning marker |
| 3 (Passthrough) | All parsing failed | Truncated raw + `[RTK:PASSTHROUGH]` |

**Canonical types:** `TestResult` (total, passed, failed, skipped, failures), `DependencyState` (packages, outdated)

**TokenFormatter trait:** Compact (default), Verbose, Ultra (`[ok]28 [x]1 [skip]0`)

**JSON extraction helper:** `extract_json_object()` finds JSON in messy output (pnpm banners, dotenv messages) via brace-balancing.

## Related Pages

- [[system-architecture]] -- Data flow through tracking system
- [[core-infrastructure]] -- SQLite tracking, tee recovery
- [[hooks-system]] -- Hook audit and session analysis
