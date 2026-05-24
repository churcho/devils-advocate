<p align="center">
  <img src="banner.png" alt="Devil's Advocate" width="100%">
</p>

# devils-advocate-elixir

A [Claude Code](https://docs.anthropic.com/en/docs/claude-code) plugin that runs binary pass/fail critiques on Elixir / Phoenix / LiveView / Ecto / OTP / Oban code and plans. Every criterion either PASSes or FAILs. Every FAIL carries a `file:line` and a `Fix:`. No percentage scores, no calibration anchors, no hand-waving.

## Why

Claude writes Elixir confidently. Too confidently. Left unchecked it'll tell you the code looks great right up until production catches fire. This plugin is the skeptical colleague who says "yeah, but what about..." instead of "LGTM," inspired by [Confidently Wrong](https://brandon.cc/confidently-wrong).

## Install

```
/plugin marketplace add churcho/devils-advocate-elixir
/plugin install devils-advocate-elixir@devils-advocate-elixir
```

<details>
<summary>Manual install</summary>

```bash
git clone https://github.com/churcho/devils-advocate-elixir.git ~/.claude/plugins/devils-advocate-elixir
```

Add to `~/.claude/settings.json`:

```json
{
  "plugins": [
    "~/.claude/plugins/devils-advocate-elixir"
  ]
}
```

Or single session: `claude --plugin-dir ~/.claude/plugins/devils-advocate-elixir`

</details>

## Commands

| Slash command | What you get |
|---|---|
| `/devils-advocate:critique` | Binary PASS/FAIL critique of the most recent code change or plan document Claude wrote (or any path you point it at) |
| `/devils-advocate:critique --consensus` | Same critique, but Claude's findings are independently checked by Codex and reconciled in a third Claude pass |
| `/devils-advocate:log` | Session history: every check, its PASS count, the git SHA it was run against |

### `/devils-advocate:critique`

Auto-detects whether the target is code or a plan and runs the matching criteria set. When Claude wrote the target in the same conversation, it dispatches an independent subagent so the reviewer never sees the author's reasoning, only the artifact and the codebase.

Things it catches: hand-rolling auth instead of using `Phoenix.Token`, calling `String.to_atom/1` on user input, a `GenServer` wrapping a pure function, a `with` chain hiding errors behind a tangled `else`, reinventing `Ecto.Changeset` validation, missing authorization middleware in Absinthe resolvers, duplicating a helper that already lives in a context module, plans where Oban job ordering is wrong, N+1 queries in `LiveView.mount/3`, schema changes shipped without a rollback story, non-assertive map access where the key is required, `raw/1` on user data in HEEx, `@doc` on private functions, grouped aliases.

**Code critique — 24 criteria across 8 dimensions:**

- **Correctness**: `mix test` passes against the changed surface, function bodies match their `@doc`/`@spec`, edge cases (nil, empty, oversized) handled.
- **Security**: no hardcoded secrets, external input validated via `Ecto.Changeset` / `NimbleOptions`, no SQL / atom / command injection, Plug or Absinthe middleware enforces auth on protected operations.
- **Quality**: no unused aliases or `defp`, no `TODO` / `raise "not implemented"`, every `{:error, _}` propagated or logged, no god-modules, primitive obsession, or boolean obsession.
- **Performance**: no N+1 Ecto queries, no `Repo.all` returning unbounded rows in request paths, no synchronous DB or HTTP in `LiveView.mount/3` or `handle_event/3`.
- **Consistency**: `@spec` on every public function, schema field types match migration column types, naming follows project namespace, predicates end in `?`, and code follows whatever pattern 5+ examples elsewhere use.
- **Idiomatic Elixir**: assertive map access for required keys, clean `with` chains (no complex `else`), no `String.to_atom/1` on dynamic input, exhaustive case/with on multi-shape returns.
- **Integration**: aliases resolve, no grouped `alias M.{A, B}`, new public functions have a `DataCase` test, whole-suite `mix test` still green.
- **Architecture**: contexts own Repo access, schemas don't bypass changesets, no `try/rescue` hiding validation skips, no `GenServer` wrapping a pure function.

**Plan critique — 26 criteria across 11 dimensions:**

- **Completeness**: requirements covered, no placeholders, edge cases addressed, migration file / rollback strategy / backfill worker named when schema changes.
- **Correctness**: Hex APIs verified against hexdocs.pm for the version pinned in `mix.lock`; Ecto / Phoenix / Oban patterns match the corresponding `*-thinking` skill.
- **Testability**: each task names a test file and assertion shape, `mix test <path>` or `LiveViewTest` strategy stated, tests use the project's `DataCase`.
- **Security**: secrets via config providers, every external input has a named `Ecto.Changeset` / `NimbleOptions` / pattern-match boundary, Plug or Absinthe middleware named for protected operations.
- **Consistency**: `@spec` named for each new public function, module names follow project namespace.
- **Simplicity**: no `GenServer` where a function suffices, no behaviour where one implementation exists, established libraries (Ecto, Oban, Cachex, FunWithFlags, Mox) used for solved problems.
- **Dependencies**: tasks ordered correctly, Hex packages and version constraints verified to exist.
- **Resilience**: schema migrations have a `down` story, Oban backfills idempotent, query cost (indexes) and LiveView re-render scope accounted for.
- **Idiomatic Elixir**: background work names the Oban worker module, queue, uniqueness constraints, retry policy, and idempotency strategy.
- **Integration**: import paths reference real modules, plan follows the project's context / controller / LiveView / resolver layering.
- **Architecture**: no bypassing context modules to hit Repo directly, no band-aid `try/rescue`, new FunWithFlags flags cite `docs/feature-flags.md`.

Every FAIL ships with a `Fix:` line. Example output for a code critique:

```
DEVIL'S ADVOCATE CRITIQUE (Binary Eval — Elixir)
═══════════════════════════════════════════════

Target: code changes for BackendAPI.Deals.WebhookHandler

  Correctness:
    tests-pass ............. PASS — mix test test/backend_api/deals/webhook_handler_test.exs (12 tests, 0 failures)
    logic-correct .......... PASS
    edge-cases ............. FAIL — empty payload not handled at lib/backend_api/deals/webhook_handler.ex:45.
                              Fix: pattern-match `%{} = payload` at the head and short-circuit with {:error, :empty_payload}.

  Security:
    no-secrets ............. PASS
    input-validated ........ FAIL — raw map merged into Repo.insert at lib/backend_api/deals/webhook_handler.ex:78.
                              Fix: route through Deals.create_webhook_event_changeset/1 (Ecto.Changeset.cast + validate_required).
    no-injection ........... FAIL — String.to_atom on payload["type"] at lib/backend_api/deals/webhook_handler.ex:62.
                              Fix: use String.to_existing_atom inside a try/rescue, or map known string → atom explicitly.
    auth-enforced .......... PASS

  Idiomatic Elixir:
    assertive-access ....... FAIL — payload[:event] used at line 31 where the key is required.
                              Fix: pattern-match `%{event: event} = payload` in the function head.

  ...

Result: 19/24 PASS — 5 criteria need fixing

Failing criteria with fixes:
1. edge-cases: short-circuit empty payload at lib/backend_api/deals/webhook_handler.ex:45
2. input-validated: route through changeset at lib/backend_api/deals/webhook_handler.ex:78
3. no-injection: replace String.to_atom at lib/backend_api/deals/webhook_handler.ex:62
4. assertive-access: pattern-match payload at lib/backend_api/deals/webhook_handler.ex:31
5. ...
```

### `/devils-advocate:log`

Reads `.devils-advocate/session.md` and prints the running tally: number of checks, pass-rate trend, worst run, and the git SHA each check was run against. Lists the per-check files saved under `.devils-advocate/logs/`.

## Standards discovery

Before evaluating anything, the critique skill builds context from the project:

- Reads `CLAUDE.md` and `AGENTS.md` for documented conventions. Standards violations cause the relevant criterion to FAIL.
- Globs `docs/adr/`, `docs/decisions/`, `adr/`, `decisions/`, `doc/architecture/decisions/`, and `**/ADR-*.md` for architectural decision records.
- Globs `~/.claude/plugins/**/claude-code-elixir/elixir/*/skills/*-thinking/SKILL.md`. When `elixir-thinking`, `phoenix-thinking`, `ecto-thinking`, `otp-thinking`, or `oban-thinking` is installed, FAIL messages cite the skill's rule name (`elixir-thinking: 'avoid non-assertive map access'`) instead of paraphrasing.
- Prefers `code_search code search / calls-to / calls-from / depends-on` over raw `grep` when a `.code_search/surrealdb.rocksdb` index exists at the project root, then falls back to `grep`.
- Scans for context modules, Plug pipelines, Absinthe middleware, behaviour modules, and supervision trees to map architectural boundaries.
- Greps for how similar operations are done elsewhere. If 5+ instances of an operation use one approach (Repo access via context modules, resolver layering, Oban dispatch pattern), the change is held to that pattern even if no doc says so.

## Consensus mode

Opt in to a Claude → Codex → Claude reconciliation loop for higher-confidence critiques. Trigger by either:

- Passing `--consensus` (or `consensus`, `cross-model`, `--cross-model`) on the command, or
- Setting `"consensus": true` at the top level of `.devils-advocate/config.json`:

```json
{
  "hooks": {"pre-commit-warning": false, "plan-file-detect": false},
  "consensus": true,
  "consensus_model": "gpt-5-codex"
}
```

`consensus_model` is forwarded to `codex exec -m <model>`. Omit it to use the local `codex` CLI's default. Plan critiques stay single-model unless `--consensus-plans` or `"consensus_plans": true` is set, since plans are short and the extra round triples token spend.

The three rounds:

1. **Claude initial critique** runs Steps 1–4 normally, holds the PASS/FAIL list internally, and does not write the log yet.
2. **Codex adversarial pass** shells out to the local `codex` CLI. Codex independently evaluates every criterion and tags each Round 1 FAIL as `CONFIRM`, `DISPUTE`, or `EXTEND` (better fix). Output is strict JSON.
3. **Claude reconciliation** buckets findings into `CONSENSUS FAIL`, `CONSENSUS PASS`, `DISPUTED`, and `CODEX-ONLY FAIL`. Claude re-reads cited `file:line`s for codex-only fails and either promotes them to consensus or demotes to disputed. Round 3 is authoritative and is the only round that writes to `.devils-advocate/logs/` or appends the session log.

Fallback behavior (the mode never crashes the critique):

- Missing `codex` CLI → single-model with a `WARNING` line.
- `consensus_model` outside the `^[A-Za-z0-9._-]+$` allowlist → refuse to invoke Codex, fall back with a `WARNING` line.
- Malformed Codex JSON → retry once, then fall back; raw response saved to `.devils-advocate/logs/codex-error-<timestamp>.txt`.
- Timeout (120s) → retry once with 180s, then fall back.
- More than 50% of findings disputed → no fallback. Full Codex response written to `.devils-advocate/logs/codex-disputes-<timestamp>.txt` (open in a plain text viewer, do not paste back into Claude). User-visible output shows only the dispute count and a pointer to the file.
- Network error → same as timeout.
- Target unreadable → Context Gate refuses before invoking Codex.

### Security

The Round 2 shell-out passes attacker-influenced material (the critiqued file, Claude's quoted evidence) to an external process and an external API. The mode hardens against both shell injection and prompt injection:

- `consensus_model` is validated against the allowlist regex above before any shell invocation. A config value with shell metacharacters is refused, not escaped.
- The prompt is written to a tempfile and streamed on stdin. The prompt body never appears in argv or in shell heredoc syntax, so a target file containing a literal heredoc terminator cannot break out.
- Attacker-influenced blocks (target description, standards summary, Round 1 findings JSON) are wrapped in `<<<UNTRUSTED_BEGIN>>>` / `<<<UNTRUSTED_END>>>` markers. The Codex prompt's preamble tells the second model to treat content between markers as data, never instructions.
- Round 1 evidence strings are scanned for the Step 3 secret patterns (`sk_`, `ghp_`, `xox`, `Bearer`, `AWS_SECRET`, `password.*=...`) before serialization. Any match is replaced with `[REDACTED]` so a leaked secret in the critiqued file does not get re-sent to OpenAI.
- Round 3 sanitizes every string field in the Codex response before rendering or persisting: lines starting with `Ignore`, `Disregard`, `System:`, `Assistant:`, `User:`, or `<!--` are stripped, triple-backtick fences are collapsed, and each string is capped at 500 characters.
- The session log entry contains criterion-level counts only. Free-form text from Codex stays out of `session.md` and out of the main per-check log.
- The `codex-review` skill delegation, when used, pins to `~/.claude/skills/codex-review/SKILL.md` only. The wildcard path `~/.claude/plugins/**/codex-review/SKILL.md` is not honored, so an unrelated plugin cannot redefine the consensus invocation.

## Session log & hooks

Every check appends an entry to `.devils-advocate/session.md` with the check number, timestamp, git SHA, PASS count, and failing-criteria list. The full critique body is written to `.devils-advocate/logs/check-N-critique-YYYY-MM-DD-HHMM.md`. Add `.devils-advocate/` to your `.gitignore`.

Two hooks ship with the plugin:

- `pre-commit-warning` prints a non-blocking nudge on `git commit` if no `.devils-advocate/.commit-reviewed` marker exists. The commit proceeds either way; running `/devils-advocate:critique` creates the marker, which is then consumed on the next commit.
- `plan-file-detect` suggests running `/devils-advocate:critique` whenever a file with `plan`/`plans` in its name or directory is written.

Both are configurable in `.devils-advocate/config.json`:

```json
{"hooks": {"pre-commit-warning": false, "plan-file-detect": false}}
```

## License

MIT
