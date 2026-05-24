<p align="center">
  <img src="banner.png" alt="Devil's Advocate" width="100%">
</p>

# devils-advocate-elixir

Claude's harshest critic, tuned for Elixir / Phoenix / LiveView / Ecto / OTP / Oban projects. A [Claude Code](https://docs.anthropic.com/en/docs/claude-code) plugin that critiques Claude's work with binary pass/fail evaluation. Every criterion either passes or fails, no percentage scores, no wiggle room.

## Why

Claude writes Elixir confidently. Too confidently. Left unchecked, it'll tell you everything looks wonderful right up until production catches fire. Inspired by [Confidently Wrong](https://brandon.cc/confidently-wrong).

A [devil's advocate](https://en.wikipedia.org/wiki/Devil%27s_advocate) argues against a position not because they believe the other side, but to surface the holes everyone else missed. This plugin gives Claude that role: the skeptical colleague who says "yeah, but what about..." instead of "LGTM."

Every criterion demands `file:line` evidence and a fix suggestion, no hand-waving, no vibes-based reviews.

## What it catches

It'll flag you for reinventing `Ecto.Changeset` validation, missing authorization middleware in Absinthe resolvers, duplicating a helper that already lives in a context module, plans where Oban job ordering is wrong, N+1 Ecto queries in `LiveView.mount/3`, and deploying schema changes without a migration rollback story. It knows when you're hand-rolling auth instead of using `Phoenix.Token` or a battle-tested library, when you're calling `String.to_atom/1` on user input, when a `GenServer` is wrapping a pure function for no reason, and when a `with` chain hides errors behind a tangled `else` block.

New in v4.0: it catches **idiomatic Elixir violations** end-to-end. Non-assertive map access (`map[:key]` where the key is required), dynamic atom creation, unsupervised tasks, raw SQL via `Ecto.Adapters.SQL.query!`, `raw/1` on user data in HEEx, `@doc` on private functions, and grouped aliases. It loads `claude-code-elixir` thinking-skills when present so it can cite their rule names instead of paraphrasing.

It works on both code and plans, auto-detecting which criteria set to use based on what you're reviewing.

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

| Slash command | Natural language |
|---|---|
| `/devils-advocate:critique` | "critique" or "critique this plan" |
| `/devils-advocate:log` | "show critique log" |

### `/devils-advocate:critique`

Binary pass/fail critique across every dimension that matters. Auto-detects whether you're reviewing code or a plan document.

**Independence gate:** When critiquing work Claude wrote in the same conversation, it automatically dispatches an independent subagent to avoid author bias. The reviewer never sees the author's reasoning, only the artifact and codebase.

**Code critique (24 criteria, 8 dimensions):**

- **Correctness** — Tests pass (`mix test`)? Logic matches `@doc`/`@spec`? Edge cases (nil, empty, oversized) handled?
- **Security** — No hardcoded secrets, input validated via `Ecto.Changeset` / `NimbleOptions`, no SQL/atom/cmd injection, auth enforced in Plug / Absinthe middleware?
- **Quality** — No dead code (unused aliases, commented blocks, unreferenced `defp`), no `TODO`/`raise "not implemented"`, every `{:error, _}` propagated or logged, no god-modules / primitive obsession / boolean obsession?
- **Performance** — No N+1 Ecto queries, no `Repo.all` returning unbounded rows in request paths, no synchronous DB/HTTP in `LiveView.mount/3` or `handle_event/3`?
- **Consistency** — `@spec` on every public function, schema field types match migration column types, naming follows project namespace and predicate-with-`?` conventions, patterns follow what 5+ examples elsewhere in the codebase do?
- **Idiomatic Elixir** — Assertive map access (`map.key` over `map[:key]` for required keys), clean `with` chains (no complex `else`), no `String.to_atom/1` on dynamic input, exhaustive case/with on multi-shape returns?
- **Integration** — Aliases resolve, no grouped `alias M.{A, B}`, new public functions covered by `DataCase` tests, no whole-suite regressions?
- **Architecture** — Contexts own Repo access, schemas don't bypass changesets, no try/rescue hiding validation skips, no `GenServer` wrapping a pure function?

**Plan critique (26 criteria, 11 dimensions):**

- **Completeness** — Requirements covered, no placeholders, edge cases addressed, migration files / rollback strategy / backfill workers named?
- **Correctness** — Hex package APIs verified against hexdocs.pm for the version in `mix.lock`, Ecto/Phoenix/Oban patterns match the corresponding thinking-skill guidance?
- **Testability** — Each task names a test file and assertion shape, `mix test <path>` or `LiveViewTest` strategy stated, tests use the project's `DataCase`?
- **Security** — Secrets handled via config providers, each external input has a named `Ecto.Changeset` / `NimbleOptions` / pattern-match boundary, Plug / Absinthe middleware named for protected operations?
- **Consistency** — `@spec` named for each new public function, module names follow project namespace?
- **Simplicity** — No `GenServer` where a function suffices, no behaviour where one implementation exists, established libraries (Ecto, Oban, Cachex, FunWithFlags, Mox) used for solved problems?
- **Dependencies** — Tasks ordered correctly, Hex packages and version constraints verified to exist?
- **Resilience** — Schema migrations have a `down` story, Oban backfills idempotent, query cost (indexes) and LiveView re-render scope accounted for?
- **Idiomatic Elixir** — Background work names the Oban worker module, queue, uniqueness constraints, retry policy, and idempotency strategy?
- **Integration** — Import paths reference real modules, plan follows the project's context/controller/LiveView/resolver layering?
- **Architecture** — No bypassing context modules to hit Repo directly, no band-aid `try/rescue`, new FunWithFlags flags cite `docs/feature-flags.md`?

Every FAIL comes with a `Fix:` suggestion. Example output:

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

Session history: total checks, pass rate trend, and git SHA linking each check to a specific commit. Individual critiques are saved to `.devils-advocate/logs/` for later reference.

## Standards & Project Awareness

The critique skill automatically discovers your project's documented standards before evaluating:

- **`CLAUDE.md` / `AGENTS.md`** — Your conventions, required patterns, and constraints. Standards violations cause relevant criteria to FAIL.
- **ADR files** — Searched in `docs/adr/`, `docs/decisions/`, `adr/`, `decisions/`, `doc/architecture/decisions/`, and `**/ADR-*.md`.
- **`claude-code-elixir` thinking-skills** — Globbed from `~/.claude/plugins/**/claude-code-elixir/elixir/*/skills/*-thinking/SKILL.md`. When present, the critique cites their rule names (e.g., `elixir-thinking: 'avoid non-assertive map access'`) in FAIL messages instead of paraphrasing.
- **`code_search` call graph** — When `.code_search/surrealdb.rocksdb` exists at the project root, `code_search code search/calls-to/calls-from/depends-on` is preferred over raw `grep` for pattern and impact analysis. Falls back to `grep` if absent.
- **Existing patterns** — Utilities, helpers, and context-module functions already in your codebase that the critiqued code might be duplicating.
- **Architectural boundaries** — Context modules, Plug pipelines, Absinthe middleware, behaviour modules, and supervision trees that indicate intentional boundaries. If 5+ instances in your codebase do something one way, that's the established pattern: violations fail even if undocumented.

## Session Log & Hooks

Every check is logged to `.devils-advocate/session.md` with a git SHA, so you can correlate results with specific commits. Full critique output is saved to individual files in `.devils-advocate/logs/`. Add `.devils-advocate/` to your `.gitignore`.

A pre-commit hook nudges you to run a critique before committing, the commit still proceeds, it's just a reminder. A plan-file hook suggests running `/devils-advocate:critique` when you write a plan file. Both hooks are configurable via `.devils-advocate/config.json`:

```json
{"hooks": {"pre-commit-warning": false, "plan-file-detect": false}}
```

## License

MIT
