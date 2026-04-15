---
title: Hooks System
tags: [hooks, security, integration, claude-code, gemini, copilot, permissions]
category: architecture
created: 2026-04-14
updated: 2026-04-14
---

# Hooks System (`src/hooks/`)

The hooks system is RTK's integration layer with AI coding agents. It intercepts shell commands, rewrites them to use RTK, and enforces security policies.

## Supported AI Agents

Claude Code, Cursor, Codex, Gemini CLI, OpenCode, Windsurf, Cline, Kilocode, Antigravity

## Hook Lifecycle

```
rtk init -g
  -> Writes ~/.claude/hooks/rtk-rewrite.sh (embedded in binary)
  -> Stores SHA-256 hash at ~/.claude/hooks/.rtk-hook.sha256
  -> Patches ~/.claude/settings.json with PreToolUse hook registration
  -> Writes rtk-awareness.md instructions

AI agent runs command (e.g., "git status")
  -> Claude Code fires PreToolUse hook
  -> rtk-rewrite.sh reads JSON, calls `rtk rewrite "git status"`
  -> permissions.rs checks deny/ask/allow rules
  -> registry.rs rewrites to "rtk git status"
  -> Returns JSON with rewritten command + permission decision
  -> Claude Code executes "rtk git status"

At startup:
  -> integrity.rs verifies hook SHA-256
  -> hook_check.rs warns if hook missing/outdated (rate-limited 1/day)
```

## Init Command (`init.rs`, ~700 lines)

`rtk init` installs:
1. **Hook script** (`rtk-rewrite.sh`) -- shell script for PreToolUse events
2. **Settings.json patching** -- hook registration entry
3. **RTK awareness markdown** -- slim file injected into CLAUDE.md/AGENTS.md
4. **Project-local `.rtk/filters.toml` template**
5. **Global filter template** at `~/.config/rtk/filters.toml`
6. **Integrity baseline** -- SHA-256 hash for tamper detection

**Patch modes:** `Ask` (prompt user), `Auto` (CI), `Skip` (manual instructions)

All assets embedded via `include_str!()`.

## Rewrite Command (`rewrite_cmd.rs`)

`rtk rewrite <cmd>` -- core command used by the hook script.

**Exit code protocol:**

| Exit | Meaning |
|------|---------|
| 0 | Rewrite allowed -- hook may auto-allow |
| 1 | No RTK equivalent -- pass through unchanged |
| 2 | Deny rule matched -- hook defers to native deny |
| 3 | Ask rule matched -- rewrite but prompt user |

**Security invariant:** `PermissionVerdict::Default` maps to exit 3 (ask), NOT exit 0 (allow). Unrecognized commands are never auto-allowed.

## Hook Command Processors (`hook_cmd.rs`)

Three AI agent JSON formats:

| Agent | Format | Ask Support |
|-------|--------|-------------|
| Claude Code | `tool_name` + `tool_input.command` | Yes (`updatedInput`) |
| Copilot CLI | `toolName` + `toolArgs` (camelCase) | Deny-with-suggestion only |
| Gemini CLI | `tool_name` = `"run_shell_command"` | No (allow/deny only) |

**Safety:** Commands containing `<<` (heredocs) are never rewritten.

## Permissions System (`permissions.rs`)

Reads Claude Code permission rules from settings.json and evaluates commands.

**Verdict precedence:** Deny > Ask > Allow > Default (ask)

**Settings files (merged):**
1. `$PROJECT_ROOT/.claude/settings.json`
2. `$PROJECT_ROOT/.claude/settings.local.json`
3. `~/.claude/settings.json`
4. `~/.claude/settings.local.json`

**Pattern matching:** Exact, prefix with word boundary, trailing wildcard (`git push*`), leading wildcard (`* --force`), middle wildcard (`git * main`), global (`*`), colon syntax (`sudo:*`)

**Compound command security (issue #1213):** Commands chained with `&&`, `||`, `|`, `;` are split. EVERY segment must independently match allow for `Allow` verdict.

## Integrity System (`integrity.rs`)

SHA-256 hook tamper detection. Reference: SA-2025-RTK-001.

**IntegrityStatus:** `Verified`, `Tampered`, `NoBaseline`, `NotInstalled`, `OrphanedHash`

**Flow:**
1. `store_hash()` at install time -- writes `.rtk-hook.sha256` (read-only 0o444)
2. `runtime_check()` at startup -- compares stored vs current
3. `Tampered` -> block execution with error, exit(1)
4. No env-var bypass -- re-run `rtk init -g --auto-patch` for legitimate changes

## Trust System (`trust.rs`)

Controls project-local `.rtk/filters.toml` loading. Security boundary because filters can rewrite output.

**Model:** Trust-before-load. Untrusted filters are silently skipped.

**Trust store:** `~/.local/share/rtk/trusted_filters.json` -- keyed by canonical path, stores SHA-256 + timestamp

**Commands:** `rtk trust` (displays content + risk summary, then stores hash), `rtk untrust`

**CI override:** `RTK_TRUST_PROJECT_FILTERS=1` only works when a CI env var is also set (`CI`, `GITHUB_ACTIONS`, etc.). Prevents `.envrc` injection attacks.

**TOCTOU prevention:** Single read of file -> display from buffer -> hash same buffer.

## Hook Staleness Detection (`hook_check.rs`)

Checks hook version via `# rtk-hook-version: N` header (current version: 3). Warns at most 1/day via marker file. Cross-checks other integrations (OpenCode, Cursor, Codex, Gemini) to avoid false warnings.

## Hook Audit (`hook_audit_cmd.rs`)

`rtk hook-audit` parses `~/.local/share/rtk/hook-audit.log` (enabled via `RTK_HOOK_AUDIT=1`). Shows rewrite/skip counts, top rewritten commands.

## Related Pages

- [[system-architecture]] -- Overall system design
- [[toml-filter-dsl]] -- Trust-gated project filters
- [[rust-patterns]] -- Security patterns (no unwrap, compound command splitting)
