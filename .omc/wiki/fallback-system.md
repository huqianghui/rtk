---
title: Fallback & Proxy System
tags: [fallback, proxy, passthrough, extensibility]
category: architecture
created: 2026-04-14
updated: 2026-04-14
---

# Fallback & Proxy System

RTK's most important design principle: **never block a command from running**. Unknown commands always pass through transparently.

## Three-Tier Fallback (main.rs lines 1062-1187)

When `Cli::try_parse()` fails (unrecognized command):

### Tier 1: RTK Meta-Command Error
If `args[0]` is in `RTK_META_COMMANDS` (gain, discover, learn, init, config, proxy, hook-audit, cc-economics, verify, trust, untrust, session, rewrite), show Clap's error message. These are RTK's own commands and a parse failure means bad flags/syntax.

### Tier 2: TOML Filter Match
Look up command in TOML filter registry:
```rust
toml_filter::find_matching_filter(&lookup_cmd)
```
Uses basename of args[0] so `/usr/bin/make` matches `^make\b`. If matched:
- Capture stdout (and stderr if `filter.filter_stderr`)
- Apply 8-stage TOML pipeline
- Print filtered result
- Track in SQLite with "rtk:toml" prefix

### Tier 3: Pure Passthrough
No TOML match. Stream directly:
```rust
resolved_command(args[0])
    .stdin(Stdio::inherit())
    .stdout(Stdio::inherit())
    .stderr(Stdio::inherit())
    .status()
```
Track as passthrough (0 tokens, timing only). Record parse failure for analytics.

## Proxy Mode (`rtk proxy <command>`)

Explicit passthrough with tracking. Used to:
- **Bypass RTK filtering** when filters are buggy or full output needed
- **Track usage metrics** for commands RTK doesn't filter
- **Guarantee compatibility** -- always works

```bash
rtk proxy git log --oneline -20    # Full output, tracked
rtk proxy npm install express      # Raw output, tracked
```

Implementation:
- Spawns child with `Stdio::piped()` for stdout+stderr
- Two reader threads stream to terminal AND capture up to 1MB each
- Registers SIGINT/SIGTERM handlers for child PID
- `ChildGuard` RAII struct ensures cleanup on early exit
- Tracks as input=output (0% savings) in SQLite

## Safety Properties

1. **Any command works** -- `rtk <anything>` is a safe prefix
2. **Exit codes preserved** -- Unix signals (128+sig) handled correctly
3. **No silent failures** -- parse failures recorded via `record_parse_failure_silent()`
4. **Streaming for unknown** -- passthrough uses `Stdio::inherit()`, no buffering
5. **Metrics for all** -- even passthrough records timing for `rtk gain --history`

## Related Pages

- [[system-architecture]] -- Overall routing that leads to fallback
- [[toml-filter-dsl]] -- Tier 2 TOML matching
- [[core-infrastructure]] -- Tracking system
