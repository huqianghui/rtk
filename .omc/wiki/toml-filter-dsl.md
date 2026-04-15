---
title: TOML Filter DSL
tags: [toml, filters, dsl, declarative, pipeline]
category: pattern
created: 2026-04-14
updated: 2026-04-14
---

# TOML Filter DSL

RTK's declarative filter system allows adding tool support without writing Rust code.

## Overview

59 built-in TOML filters handle commands without dedicated Rust modules (terraform, make, gcc, brew, ansible, helm, etc.). Users can add project-local or global TOML filters.

## Lookup Priority (first match wins)

1. `.rtk/filters.toml` -- project-local (trust-gated via [[hooks-system]])
2. `~/.config/rtk/filters.toml` -- user-global
3. Built-in -- `src/filters/*.toml` concatenated by `build.rs`, embedded via `include_str!`
4. Passthrough -- no match, caller streams directly

## Filter Schema

```toml
[filters.terraform-plan]
description = "Terraform plan output filter"
match_command = "^terraform\\s+plan"
strip_ansi = true
filter_stderr = false

# Regex substitutions (chained sequentially)
replace = [
    { pattern = "\\d{4}-\\d{2}-\\d{2}T[\\d:]+Z", replacement = "<timestamp>" },
]

# Short-circuit: if output matches, return message immediately
match_output = [
    { pattern = "No changes", message = "terraform plan: no changes detected" },
    { pattern = "0 Warning\\(s\\)\\n\\s+0 Error\\(s\\)", message = "ok", unless = "error" },
]

# Line filtering (mutually exclusive)
strip_lines_matching = [
    "^Refreshing state",
    "^\\s*#.*unchanged",
    "^\\s*$",
]
# OR
keep_lines_matching = ["^error", "^warning"]

truncate_lines_at = 200        # Max chars per line
head_lines = 50                # Keep first N lines
tail_lines = 10                # Keep last N lines
max_lines = 80                 # Absolute line cap
on_empty = "terraform plan: ok"  # Message if result is empty
```

## 8-Stage Pipeline

Applied in order by `apply_filter()` in `core/toml_filter.rs`:

| Stage | Field | Action |
|-------|-------|--------|
| 1 | `strip_ansi` | Remove ANSI escape codes |
| 2 | `replace` | Line-by-line regex substitutions (rules chained) |
| 3 | `match_output` | Check full blob; if match (and no `unless`), return message (short-circuit) |
| 4 | `strip/keep_lines` | Filter lines by RegexSet (mutually exclusive) |
| 5 | `truncate_lines_at` | Truncate each line to N chars |
| 6 | `head/tail_lines` | Keep first N and/or last N lines |
| 7 | `max_lines` | Absolute line cap |
| 8 | `on_empty` | Replacement message if result is empty |

## Inline Test DSL

Each filter can have tests executed by `rtk verify`:

```toml
[[tests.gcc]]
name = "strips include chain, keeps errors and warnings"
input = """
In file included from /usr/include/stdio.h:42:
main.c:10:5: error: use of undeclared identifier 'foo'
"""
expected = "main.c:10:5: error: use of undeclared identifier 'foo'"
```

`rtk verify --require-all` ensures every filter has at least one test (CI enforcement).

## Built-in Filter Categories (59 files)

| Category | Tools |
|----------|-------|
| Build | gcc, make, gradle, mvn-build, dotnet-build, swift-build, xcodebuild, trunk-build, pio-run, spring-boot, quarto-render |
| Linters | biome, oxlint, shellcheck, hadolint, markdownlint, yamllint, basedpyright, ty, tofu-fmt, mix-format |
| Infrastructure | terraform-plan, tofu-init/plan/validate, helm, gcloud, ansible-playbook, systemctl-status, iptables, fail2ban-client, sops, liquibase |
| Package managers | brew-install, bundle-install, composer-install, poetry-install, uv-sync |
| Task runners | just, task, turbo, nx, make, mise, pre-commit |
| System | df, du, ps, stat, ping, ssh, rsync |
| Version control | jj, yadm |
| Other | ollama, jira, jq, shopify-theme, skopeo |

## Build-Time Concatenation

`build.rs` reads all `src/filters/*.toml` files, concatenates them into a single string, and embeds it via `include_str!`. This means adding a new TOML filter is a file-creation-only change -- no Rust code needed.

## Security: Trust System

Project-local `.rtk/filters.toml` files are subject to trust-before-load:
- `rtk trust` displays content + risk summary (replace rules, match_output, catch-alls), stores SHA-256
- Untrusted filters are silently skipped
- CI override: `RTK_TRUST_PROJECT_FILTERS=1` only with CI env vars set

See [[hooks-system]] for full trust details.

## Environment Variables

- `RTK_NO_TOML=1` -- bypass TOML filter engine entirely
- `RTK_TOML_DEBUG=1` -- print match diagnostics to stderr

## Implementation Details

**CompiledFilter** -- all regexes pre-compiled at first access. Registry is a `lazy_static` global.

**API:**
- `find_matching_filter(command: &str) -> Option<&'static CompiledFilter>`
- `apply_filter(filter: &CompiledFilter, stdout: &str) -> String`
- `run_filter_tests() -> Vec<TestResult>` (for `rtk verify`)

## Related Pages

- [[filter-patterns]] -- Rust-based filter strategies
- [[hooks-system]] -- Trust system details
- [[core-infrastructure]] -- toml_filter.rs engine
