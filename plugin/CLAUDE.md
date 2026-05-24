# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is the `devils-advocate-elixir` **Claude Code plugin**: adversarial self-critique tuned for Elixir / Phoenix / LiveView / Ecto / OTP / Oban projects. It's a prompt-only plugin, no build step, no dependencies, no compiled code. The entire plugin is Markdown skill files and Node.js hooks.

## Architecture

The plugin follows the Claude Code plugin structure:

- **`plugin/.claude-plugin/plugin.json`** — Plugin metadata (name, version, description). Version here is the source of truth.
- **`.claude-plugin/marketplace.json`** — Marketplace registry entry (lives at repo root, not inside `plugin/`). Source points to `./plugin`. Version must stay in sync with `plugin.json`.
- **`skills/`** — Each subdirectory contains a `SKILL.md` file that defines a slash command:
  - `critique/` → `/devils-advocate:critique` — Binary pass/fail critique of code or plan documents. Auto-detects target type. 24 criteria for code across 8 dimensions, 26 criteria for plans across 11 dimensions. Optional `--consensus` mode runs a Claude → Codex → Claude reconciliation loop.
  - `log/` → `/devils-advocate:log` — Display session history
- **`hooks/HOOKS.md`** — Companion documentation explaining the inline hook logic step-by-step (hooks are `node -e` one-liners due to plugin path constraints).
- **`hooks/hooks.json`** — Registers two hooks:
  - `PreToolUse` hook (Bash, key: `pre-commit-warning`) — prints a non-blocking warning on `git commit` if no `.devils-advocate/.commit-reviewed` marker exists, nudging the user to run critique first. The commit proceeds regardless. The marker is created by the critique skill after writing the session log and consumed (deleted) on the next commit, suppressing the warning when critique has already been performed.
  - `PostToolUse` hook (Write, key: `plan-file-detect`) — detects when a plan file is written (matching paths with `plan`/`plans` in the name or directory) and suggests running `/devils-advocate:critique`
  - All hooks are **configurable** via `.devils-advocate/config.json` in the user's project. All hooks are on by default. To disable a hook, set its key to `false` under `hooks`: `{"hooks":{"pre-commit-warning":false}}`

## Key Conventions

- **Binary evaluation** — All critiques use binary pass/fail per criterion. No percentage scores. Each criterion either PASS or FAIL. Every FAIL must cite `file:line` evidence and include a `Fix:` suggestion.
- **Two criteria sets** — Code critiques use 24 criteria across 8 dimensions (Correctness, Security, Quality, Performance, Consistency, Idiomatic Elixir, Integration, Architecture). Plan critiques use 26 criteria across 11 dimensions (Completeness, Correctness, Testability, Security, Consistency, Simplicity, Dependencies, Resilience, Idiomatic Elixir, Integration, Architecture).
- **Auto-detection** — The critique skill determines whether it's reviewing code or a plan based on conversation context. No explicit mode flag needed. A `mix.exs` / `.ex` / `.exs` / `.heex` path activates Elixir code-critique mode.
- **Elixir thinking-skills and `code_search`** — When critiquing inside an Elixir project, the critique skill globs for `claude-code-elixir` thinking-skills (`elixir-thinking`, `phoenix-thinking`, `ecto-thinking`, `otp-thinking`, `oban-thinking`) so it can cite their rule names in FAIL messages. When a `.code_search/surrealdb.rocksdb` index exists, `code_search code search/calls-to/calls-from/depends-on` is preferred over raw `grep` for pattern and impact analysis. Both fall back gracefully if absent.
- **Cross-model consensus mode** — The critique skill supports a Claude → Codex → Claude reconciliation loop, opt-in via `--consensus` (or `consensus`/`cross-model`/`--cross-model`) on the slash command or `"consensus": true` in `.devils-advocate/config.json`. Round 1 is Claude's normal critique, Round 2 shells out to the local `codex` CLI, Round 3 is Claude's reconciliation into CONSENSUS FAIL / CONSENSUS PASS / DISPUTED / CODEX-ONLY FAIL buckets. Consensus mode falls back to single-model with a `WARNING` line if the `codex` CLI is missing, malformed, or times out — it never crashes the critique. Plan critiques stay single-model unless `--consensus-plans` or `consensus_plans: true` is set, since the extra round roughly triples token spend.
- **Consensus-mode sanitization** — Step 0c hardens the Codex shell-out against prompt injection and shell injection: `consensus_model` must match `^[A-Za-z0-9._-]+$` before invocation, the prompt is written to a tempfile and streamed on stdin (no heredoc-escape vector), attacker-influenced blocks (target description, standards summary, Round 1 findings) are fenced with `<<<UNTRUSTED_BEGIN>>>` / `<<<UNTRUSTED_END>>>` markers, Step 3 secret-pattern matches in Round 1 evidence are replaced with `[REDACTED]` before being sent to OpenAI, Round 3 sanitizes Codex string fields before render, and HIGH DISPUTE output is written to a separate `.devils-advocate/logs/codex-disputes-<timestamp>.txt` file (the user-visible output only shows the dispute count and a pointer to the file).
- **SKILL.md frontmatter** — Each skill has YAML frontmatter with `name` and `description`. The `description` field must be short enough to avoid `ENAMETOOLONG` errors during plugin installation (this was a real bug, see commit `b381119`).
- **Session log** — The critique skill appends entries to `.devils-advocate/session.md` in the user's project (not this repo). Entries include git SHA, timestamp, check number, and pass count. The log skill only reads, never writes.
- **Individual log files** — The critique skill also writes the full formatted output to `.devils-advocate/logs/check-{N}-critique-{YYYY-MM-DD}-{HHMM}.md`. This preserves the complete critique for later reference. The log skill lists available log files.
- **Scope-bounded critique** — The critique skill only evaluates what was requested, never penalizes for out-of-scope features. If a criterion doesn't apply, it's marked PASS with a note.
- **Standards discovery** — The critique skill reads `CLAUDE.md`, `AGENTS.md`, and searches for ADR files. Standards violations cause relevant criteria to FAIL.
- **Existing patterns detection** — In code mode, the critique skill greps (or `code_search code search`es) for existing utilities/helpers/conventions that the critiqued code might be duplicating.
- **Architecture enforcement** — The critique skill checks for architectural boundary violations (context modules, Plug pipelines, Absinthe middleware, behaviour modules, supervision trees) and hacky shortcuts (symptom-fixing, special-case conditionals, bypassing existing systems). Pattern discovery is code-enforced: if 5+ instances in the codebase do something one way, that's the established pattern regardless of documentation.
- **Independence gate** — When critiquing your own work (same conversation), the critique skill dispatches an independent subagent to avoid author bias. Falls back to inline critique with a bias warning if the Agent tool is unavailable.
- **Context gate** — The critique skill refuses to produce results if it lacks sufficient context. This prevents false-confidence scoring.
- **Evidence requirement** — Every FAIL must cite `file:line` references. Results without evidence are invalid.
- **Unverified section** — Mandatory in every critique. Must list at least one thing not checked.
- **Version syncing** — When bumping versions, update both `plugin.json` and `marketplace.json`.

### Consensus mode config

Opt in per-project by adding the following to `.devils-advocate/config.json`:

```json
{
  "hooks": {"pre-commit-warning": false, "plan-file-detect": false},
  "consensus": true,
  "consensus_model": "gpt-5-codex"
}
```

`consensus_model` is optional and is passed to `codex exec -m <model>`. If omitted, the local `codex` CLI's default model is used. `consensus_plans` (boolean) gates consensus mode for plan critiques separately and defaults to `false`.

## Working in This Repo

Changes are validated by:
1. Running `bash plugin/scripts/check-consistency.sh` — automated checks for JSON validity, version sync, binary criteria presence, context gate, unverified section, session log references, evidence requirements, and frontmatter description lengths
2. Running `bash plugin/scripts/test-plugin.sh` — deeper test suite covering plugin metadata, frontmatter validation, binary criteria completeness, session log format, output format structure, hook validation, standards discovery, context gate refusal format, and CLAUDE.md accuracy
3. Reading the skill Markdown for correctness
4. Installing the plugin locally and invoking the slash commands

To test locally: install the plugin via `claude --plugin-dir ./plugin` from the repo root, then invoke commands like `/devils-advocate:critique` in an Elixir project with code changes.

## Historical Context

This plugin was renamed from `confidence-loops` / `confidence-loop` to `devils-advocate`, then renamed again to `devils-advocate-elixir` in v4.0.0 when the scope tightened to Elixir / Phoenix / LiveView / Ecto / OTP / Oban projects. The `.confidence-loop/` directory contains legacy session data from before the rename. The `docs/` directory (gitignored) contains original design and implementation plans.

v3.0.0 consolidated four scoring skills (`critique`, `critique-plan`, `pre`, `second-opinion`) into a single `critique` skill with binary pass/fail evaluation. Percentage scoring was removed entirely.

v4.0.0 swapped JS/TS examples and `npm`/`tsc` verification commands for `mix`-native equivalents, expanded the criteria sets to 24 code (across 8 dimensions including a new Idiomatic Elixir dimension) and 26 plan (across 11 dimensions including a new Idiomatic Elixir dimension), wired Step 1 to load `claude-code-elixir` thinking-skills and prefer `code_search` over raw grep, and added the opt-in cross-model consensus mode (Claude → Codex → Claude).
