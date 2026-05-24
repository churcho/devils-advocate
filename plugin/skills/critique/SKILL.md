---
name: critique
description: Adversarial binary critique for Elixir/Phoenix/LiveView/Ecto/OTP/Oban code or plans.
---

# Devil's Advocate Critique

You are running an **adversarial binary critique**. Every criterion either passes or fails, no percentage scores, no wiggle room. You must be your own harshest critic.

## Target Detection

Determine whether you are critiquing **code** or a **plan** based on conversation context:
- If the user provides a path to a plan/design document, or you just wrote one, **plan critique** (26 criteria)
- If you just wrote code, or the user asks you to critique code changes, **code critique** (24 criteria)
- If the user provides a `mix.exs` path, an `.ex`/`.exs` file, or a Phoenix LiveView (`.heex`) template, treat it as **code critique** with Elixir context active
- If unclear, ask the user

## Scope-Bounded Critique

**Critique ONLY what was requested.** Do not penalize for out-of-scope features. If a criterion doesn't apply to the target (e.g., `no-injection` for a project with no user input), mark it PASS with a note, don't skip it.

## Process

### Step 0: Independence Gate

**If you wrote or contributed to the target being critiqued (same conversation), you MUST dispatch an independent subagent.** Same-context critique has author bias, the author fills in gaps mentally and rationalizes decisions. Independent critique only sees the artifact and codebase.

**Dispatch pattern:** Before dispatching, replace `TARGET_PATH` with the actual file path or description of changes, and expand the criteria placeholder with the appropriate criteria block from Step 4 (code or plan).

```
Agent({
  description: "Independent DA critique",
  model: "opus",
  prompt: `You are a devil's advocate reviewer. Perform an adversarial binary critique of: TARGET_PATH

Read CLAUDE.md and AGENTS.md first for project conventions. Then read the target file. Then verify all claims against the actual codebase using Read, Grep, Glob, and Bash (mix compile --warnings-as-errors, mix credo --strict <file>, mix format --check-formatted <file>, mix test <focused_path>, mix dialyzer when dialyxir is in mix.exs). Use code_search code search, code_search code calls-to, code_search code calls-from, and code_search code depends-on for pattern and impact analysis before forming opinions on architectural criteria; fall back to Grep when code_search is unavailable.

CRITERIA_BLOCK

For each criterion: PASS with brief evidence, or FAIL with file:line and a Fix: suggestion.
Write results to .devils-advocate/logs/check-N-critique-YYYY-MM-DD-HHMM.md
Write session entry to .devils-advocate/session.md
Run: touch .devils-advocate/.commit-reviewed`
})
```

**When to run inline (without subagent):** Only when critiquing code or plans you did NOT write in this conversation (e.g., reviewing someone else's PR, auditing existing code).

**Fallback:** If the Agent tool is unavailable or dispatch fails, proceed with inline critique but prepend a warning: `WARNING: Self-critique, author bias may be present.`

### Step 0c: Cross-Model Consensus Gate

Activate **consensus mode** when any of the following is true:

1. The slash-command argument contains `consensus`, `--consensus`, `cross-model`, or `--cross-model` (case-insensitive). Example: `/devils-advocate:critique --consensus`.
2. `.devils-advocate/config.json` contains `"consensus": true` at the top level (peer of the existing `"hooks"` block).

Otherwise run single-model as normal — this gate is additive.

For plan critiques, consensus mode is gated separately behind `--consensus-plans` on the command line or `"consensus_plans": true` in config. Default plan flow stays single-model since plans are short and the extra round is expensive.

When consensus mode is active, wrap whichever execution path (inline or subagent) the Independence Gate chose with a three-round loop:

**Round 1 — Claude initial critique.** Run Steps 1–4 as normal. Produce the standard binary PASS/FAIL list with `file:line` and `Fix:`. Hold the output internally; do not write to `.devils-advocate/logs/` yet.

**Round 2 — Codex adversarial pass.** Shell out to the local `codex` CLI with the target, the Step 1 standards-discovery summary, the Step 4 criteria block, and Round 1's findings. Codex independently evaluates every criterion AND, for every Round 1 FAIL, declares `CONFIRM` (same finding), `DISPUTE` (it does not see the issue — explain why), or `EXTEND` (the issue is real but the fix is wrong — provide a corrected `Fix:`). Codex must also surface any criterion it would mark FAIL that Round 1 marked PASS.

Use the project's existing `codex-review` skill pattern if it is locatable at `~/.claude/skills/codex-review/SKILL.md` or `~/.claude/plugins/**/codex-review/SKILL.md` — its invocation is known to work on the user's machine. Otherwise run `codex --help` to confirm flags and use a shape similar to:

```bash
codex exec -m "${consensus_model:-}" - <<'PROMPT'
You are running an adversarial binary critique against an Elixir project.

Target: <relative path or description>

Project standards summary (from CLAUDE.md, AGENTS.md, ADRs, code_search dominant patterns):
<<<...condensed by Claude...>>>

Criteria to evaluate (each must return PASS or FAIL with file:line + Fix:):
<<<...full criteria block from Step 4...>>>

Round 1 (Claude) findings:
<<<...JSON of Claude's PASS/FAIL list with evidence and fixes...>>>

Tasks:
1. Independently evaluate every criterion against the target. Return your own PASS/FAIL for each.
2. For every Round 1 FAIL, declare CONFIRM, DISPUTE (with reason), or EXTEND (with corrected Fix:).
3. Surface any criterion you would mark FAIL that Round 1 marked PASS.

Output STRICT JSON:
{
  "codex_findings": [
    {"criterion": "no-injection", "result": "FAIL", "file": "lib/...", "line": 62, "fix": "..."}
  ],
  "round1_responses": [
    {"criterion": "edge-cases", "stance": "CONFIRM"},
    {"criterion": "input-validated", "stance": "DISPUTE", "reason": "..."},
    {"criterion": "no-injection", "stance": "EXTEND", "fix": "..."}
  ],
  "codex_only_fails": [
    {"criterion": "doc-discipline", "file": "lib/...", "line": 12, "fix": "..."}
  ]
}
PROMPT
```

`consensus_model` comes from `.devils-advocate/config.json` (`consensus_model` key) when set. Do not hardcode a model name in the prompt. Do not pipe API keys or env vars into the invocation — `codex` handles its own auth.

**Round 3 — Claude reconciliation.** Read Codex's response. Compare against Round 1. Bucket every disagreement:

- **CONSENSUS FAIL** — both models flagged it (highest confidence, must fix).
- **CONSENSUS PASS** — both models cleared it.
- **DISPUTED** — exactly one model flagged it (surface; do not auto-fix).
- **CODEX-ONLY FAIL** — Round 1 missed it; Claude re-reads the cited `file:line` and judges. If accepted, promote to CONSENSUS FAIL. If rejected, demote to DISPUTED with Claude's counter-reason.

Only Round 3 writes to `.devils-advocate/logs/` and appends the session log entry. Claude's reconciliation pass is authoritative.

**Failure modes (must handle, do not crash the critique):**

| Failure | Behavior |
|---|---|
| `codex` CLI not found in PATH | Skip Round 2 and Round 3. Run single-model. Prepend: `WARNING: Consensus mode requested but Codex CLI not found. Falling back to single-model critique.` |
| Codex returns non-JSON / malformed JSON | Retry once. If still malformed, write the raw response to `.devils-advocate/logs/codex-error-<timestamp>.txt` and fall back with: `WARNING: Codex returned malformed output (logged to ...). Falling back to single-model.` |
| Codex times out (>120 seconds) | Retry once with 180-second timeout. If still times out, fall back as above with a timeout-specific WARNING. |
| Codex disputes >50% of Claude's findings | Do NOT fall back. Surface: `HIGH DISPUTE: Codex disputed N of M Claude findings. Manual review recommended.` Show all disputes in the Disputed section. |
| Network error reaching Codex's backend | Same as timeout: retry once, then fall back. |
| Consensus mode requested but Claude can't read the target (permission, missing file) | Refuse via Step 0b (Context Gate). Do not invoke Codex on missing context. |

Do not silently downgrade. Every fallback path must produce a visible `WARNING` line at the top of the output. Do not let Codex's output overwrite Claude's. Do not cache Codex responses across runs.

When consensus mode is active, use the consensus-mode output format described in **Output Format** below.

### Step 0b: Context Gate

Before critiquing (whether inline or via subagent), verify you have sufficient context:
1. **Have you read the relevant files?** If you haven't used Read/Grep to examine the actual code or plan, STOP.
2. **Do you understand the task?** If the task was vague or you can't restate it precisely, STOP.
3. **Do you know the project structure?** If you haven't explored the repo enough to understand how components connect, STOP.
4. **Is there something to critique?** If the task was conversational with no code or plan output, say so.

If any check fails, output a **CONTEXT INSUFFICIENT** block:
```
CONTEXT INSUFFICIENT
═══════════════════════════════════════
Cannot provide a meaningful critique. Missing:
• [what's missing]
• [what's needed]

Action required:
1. [specific step]
2. [specific step]
```

Do NOT produce results without context. A critique with insufficient context is worse than no critique, it creates false confidence.

### Step 1: Discover project standards

Search for documented standards, architectural decisions, and existing patterns:

1. **Standards files** — Use Read to check for `CLAUDE.md` and `AGENTS.md` in the project root. Note any conventions, required patterns, or constraints they define.
2. **ADR files** — Use Glob to search for architectural decision records: `docs/adr/*.md`, `docs/decisions/*.md`, `adr/*.md`, `decisions/*.md`, `doc/architecture/decisions/*.md`, `**/ADR-*.md`. Read any that exist.
3. **`claude-code-elixir` thinking-skills** — Glob for `~/.claude/plugins/**/claude-code-elixir/elixir/*/skills/*-thinking/SKILL.md` and read the ones matching the diff's surface: `elixir-thinking` always; `phoenix-thinking` for LiveView/Plug/PubSub; `ecto-thinking` for schemas/Repo/migrations; `otp-thinking` for GenServer/Supervisor/Task; `oban-thinking` for Oban workers. If none are installed, name the skills in your output so the user knows what would have been loaded.
4. **`code_search` call graph** — If a `.code_search/surrealdb.rocksdb` exists at the project root, use `code_search code search`, `code_search code calls-to`, `code_search code calls-from`, and `code_search code depends-on` for pattern discovery rather than raw `grep`. Note this as the preferred path, with grep as fallback.
5. **Existing patterns** *(code mode only)* — Use Grep (or `code_search code search`) to find utilities, helpers, or conventions similar to the code being critiqued. Look for patterns the work might be duplicating.
6. **Architectural domain** — Identify what layer, module, or service the change touches (context module, controller, LiveView, resolver, schema, Oban worker, GenServer). Note which boundaries exist around it.
7. **Dominant patterns** — Grep (or `code_search code search`) for how similar operations are done elsewhere in the codebase. Find 3-5 examples of the same category of operation (Repo access via contexts, Absinthe resolvers, LiveView event handling, Oban job dispatch, etc.) and note the dominant pattern. If 5+ instances do it one way, that's the established pattern, violations FAIL even if undocumented.
8. **Boundary markers** — Look for context modules (canonical Elixir boundary), Absinthe schemas, Plug pipelines, behaviour modules, or supervision trees that indicate intentional architectural boundaries.

**Important:** If standards files exist but contain no actionable conventions or constraints, treat it as if no standards were found.

Record what you find, standards violations should cause the relevant criteria to FAIL.

### Step 2: Identify the target

What is being critiqued? State it clearly. Note whether this is a code critique or plan critique.

### Step 3: Gather evidence

Before evaluating, collect concrete evidence:
- Use Read/Grep/Glob (and `code_search` when available) to examine the code or plan thoroughly
- **Code mode:** Search for test files, run focused tests with Bash, search for the Elixir security patterns below, check `mix.exs` / `mix.lock` for dependency context
- **Plan mode:** Verify referenced Hex package APIs exist on hexdocs.pm for the version pinned in `mix.lock`, check task ordering, verify referenced files/modules exist
- Note specific file paths and line numbers for any issues

**Elixir security patterns to grep for (code mode):**
- `String.to_atom/1` invoked on user-supplied or external input (memory-leak / crash risk — atoms are never garbage-collected)
- raw SQL via `Ecto.Adapters.SQL.query!/3` or `query/3` with string interpolation rather than parameterized bindings
- EEx/HEEx `raw/1` or `Phoenix.HTML.raw/1` on non-static input (XSS)
- `Code.eval_string/1` and `Code.eval_quoted/1` (arbitrary code execution)
- unsupervised `spawn/1`, `Task.start/1`, `Task.async/1` outside a `Task.Supervisor` (lost crash signals)
- `System.cmd/2` or `:os.cmd/1` with interpolated input (command injection)
- `Plug.Conn.put_resp_header/3` writes of user-controlled values without sanitization (response splitting)
- `:erlang.binary_to_term/1` on untrusted input (arbitrary atom/term creation)
- hardcoded secrets / tokens: regex for `["']sk_`, `["']ghp_`, `["']xox`, `Bearer `, `AWS_SECRET`, `password.*=.*["']`. Surface candidates, let the reviewer judge.

**Mix commands to run (code mode, where applicable):**
- `mix compile --warnings-as-errors` against the focused module
- `mix credo --strict <file>` on each changed `.ex`/`.exs`
- `mix format --check-formatted <file>`
- focused `mix test <test_path>` on the relevant test module
- `mix dialyzer` only at the verification-gate close (slow), and only if `dialyxir` is in `mix.exs`

### Step 4: Evaluate against criteria

Run every criterion in the appropriate set. For each criterion:
- **PASS** — criterion is satisfied, with brief evidence
- **FAIL** — criterion is violated, with `file:line` evidence and a **Fix:** suggestion

Every FAIL must include a `Fix:` — a fail without a fix is useless.

#### Code Criteria (24 criteria, 8 dimensions)

```
Correctness:
  tests-pass          — `mix test <focused>` exits 0 against the changed surface.
  logic-correct       — Function bodies do what their @doc/@spec claim; no doc/impl drift.
  edge-cases          — nil, empty list/map, zero, negative, oversized inputs handled.

Security:
  no-secrets          — No hardcoded keys, tokens, or DB credentials in source.
  input-validated     — External input validated via Ecto.Changeset, NimbleOptions, or
                        explicit pattern matching at the boundary.
  no-injection        — No SQL string interpolation in Ecto.Adapters.SQL.query, no raw/1
                        on user data in HEEx, no String.to_atom/1 on user data, no
                        System.cmd / :os.cmd with interpolation, no :erlang.binary_to_term
                        on untrusted input.
  auth-enforced       — Plug pipelines and Absinthe middleware enforce authorization on
                        protected operations; resolvers do not bypass.

Quality:
  no-dead-code        — No unused @moduledoc, no commented-out blocks, no unreferenced
                        defp, no unused aliases or imports.
  no-placeholders     — No TODO/FIXME/raise "not implemented" left in.
  error-handling      — Every {:error, _} branch either propagates or logs via
                        Logger.warning/error (per project AppLogger discipline). No
                        silent swallows ({:error, _} -> :ok).
  no-code-smell       — No god-modules, primitive obsession, alternative return types,
                        boolean obsession, unrelated multi-clause functions, scattered
                        process interfaces, namespace trespassing.

Performance:
  no-obvious-perf     — No N+1 Ecto queries, no Enum.map followed by Enum.into where
                        Map.new would do, no Enum on streams that should stay lazy, no
                        Repo.all in a request path returning unbounded rows, no
                        synchronous DB/HTTP in LiveView mount or handle_event.

Consistency:
  types-consistent    — @spec on every public function in changed modules; @spec return
                        shapes match actual returns; struct field types in Ecto schemas
                        match the migration column types.
  naming-matches      — Module names follow project namespace; predicate functions end
                        in `?`; bang variants exist where the non-bang version returns
                        {:ok, _} | {:error, _}; private helpers are defp.
  patterns-followed   — Code follows dominant patterns from Step 1 standards discovery;
                        if 5+ instances of an operation use a particular approach (e.g.,
                        Repo access via context modules, not directly in controllers),
                        the change must too.

Idiomatic Elixir:
  assertive-access    — Required map keys accessed via `map.key` or pattern match, not
                        `map[:key]`. `map[:key]` is reserved for genuinely optional keys.
  with-clauses-clean  — No complex `else` blocks in `with`; errors normalized by helper
                        functions before the chain (per official anti-patterns doc).
  no-dynamic-atoms    — No String.to_atom/1 on dynamic input; use explicit mapping or
                        String.to_existing_atom/1 inside a try/rescue.
  pattern-exhaustive  — Every case/with on a function with multiple return shapes covers
                        each shape; Decimal.parse, Integer.parse, Repo.insert,
                        Repo.update all have exhaustive handlers.

Integration:
  imports-correct     — All `alias`/`import`/`require` resolve to real modules; no
                        grouped aliases `alias M.{A, B}` (project rule); aliases live at
                        module top, never inside functions.
  tests-exist         — New public functions have at least one high-level test using the
                        project's DataCase (e.g., BackendAPI.DataCase), not ExUnit.Case
                        directly.
  no-regressions      — Whole-suite `mix test` (verification-gate phase) passes; no
                        previously-passing test now fails.

Architecture:
  boundaries-respected — Context modules own Repo access; controllers/LiveViews call
                         contexts, not Repo directly. GraphQL resolvers go through
                         contexts. Schemas don't bypass changesets on writes.
  no-hacky-shortcuts   — No String.to_existing_atom hidden in a try/rescue to dodge
                         legitimate validation. No `with` rewritten as nested case just
                         to avoid an `else`. No GenServer/Agent wrapping a pure function
                         (process anti-pattern: code-organization-by-process).
```

#### Plan Criteria (26 criteria, 11 dimensions)

```
Completeness:
  req-coverage        — Plan addresses every requirement it claims to.
  no-placeholders     — No TODO/FIXME/stub/placeholder items.
  edge-cases          — Plan handles error states, empty inputs, boundary conditions.
  migration-plan      — If the plan introduces schema changes, it names the migration
                        file, the rollback strategy, and whether a backfill Oban job is
                        required.

Correctness:
  api-verified        — All Hex package APIs cited (Ecto, Phoenix, Absinthe, Oban,
                        Cachex, etc.) verified against hexdocs.pm for the installed
                        version in mix.lock, not assumed.
  patterns-correct    — Ecto query patterns match `ecto-thinking` guidance; Phoenix
                        patterns match `phoenix-thinking`; Oban patterns match
                        `oban-thinking`.

Testability:
  tests-per-step      — Each task names the test file and at least one assertion shape
                        (no "add tests").
  verification        — `mix test <path>` invocation or LiveViewTest assertion strategy
                        stated for each behavioral change.
  datacase-used       — Tests use the project's DataCase (e.g., BackendAPI.DataCase),
                        not ExUnit.Case directly, per the project rule.

Security:
  no-secrets          — Secrets handled via env vars / config providers, never hardcoded.
  input-validated     — Plan names the Ecto.Changeset, NimbleOptions schema, or explicit
                        pattern-match boundary that validates each external input.
  auth-designed       — Plan names the Plug / Absinthe middleware enforcing authorization
                        for protected operations.

Consistency:
  types-consistent    — Plan names the @spec it expects on each new public function.
  naming-matches      — Module names follow project namespace; predicates end in `?`.

Simplicity:
  no-overengineering  — No GenServer where a function suffices. No behaviour where one
                        implementation exists. No macro where a function fits.
  no-reinvention      — Plan uses established libraries (Ecto, Oban, Cachex,
                        FunWithFlags, Mox, ExUnit) for solved problems.

Dependencies:
  correct-order       — Tasks ordered correctly (no step depends on a later step).
  deps-available      — Hex package and version constraint in mix.exs / mix.lock verified
                        to exist; new dep additions named with rationale.

Resilience:
  rollback-plan       — Schema/data migrations have a `down` story; Oban backfills are
                        idempotent.
  perf-considered     — Plan accounts for query cost (indexes named), background-job
                        throughput, and LiveView re-render scope.

Idiomatic Elixir:
  oban-design         — If the plan introduces background work, it names the Oban worker
                        module, queue, uniqueness constraints, retry policy, and
                        idempotency strategy (per `oban-thinking` guidance).

Integration:
  imports-correct     — Import paths reference modules that exist; no grouped aliases.
  follows-patterns    — Plan follows the project's context / controller / LiveView /
                        resolver layering.

Architecture:
  boundaries-respected — Plan does not bypass context modules to hit Repo directly, does
                         not bypass changesets on writes, does not add Repo calls to
                         controllers or resolvers, does not bypass AppLogger for trace
                         output.
  no-hacky-shortcuts   — No band-aid fixes hidden as `try/rescue` blocks. No
                         renaming-as-refactor where a rewrite is needed.
  feature-flag-noted   — If the plan introduces a new FunWithFlags flag, it cites
                         `docs/feature-flags.md` and commits to updating that registry
                         (per project mandatory rule).
```

### Step 5: Write the session log entry

Read `.devils-advocate/session.md` first (if it exists), then use the Write tool to write the full existing contents plus your new entry appended at the end. Create the directory and file if they don't exist. Before writing, use Bash to run `git rev-parse --short HEAD` to get the current commit SHA. Use this format:

   ```markdown
   ## Check #N — Critique | YYYY-MM-DD HH:MM | <git-sha>
   - **Result:** X/Y PASS
   - **Failing:** [comma-separated list of failing criteria, or "none"]
   - **Summary:** [1-2 sentence summary]
   ```

   Increment the check number based on existing entries in the file.

After writing the session log entry, also write the full formatted critique output (everything from the Output Format section) to `.devils-advocate/logs/check-{N}-critique-{YYYY-MM-DD}-{HHMM}.md` using the same check number and timestamp. Create the `logs/` directory if it doesn't exist.

After writing both log files, run `touch .devils-advocate/.commit-reviewed` to signal that a critique has been performed. This allows the pre-commit hook to permit the next `git commit`.

## Output Format

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
    assertive-access ....... FAIL — payload[:event] used at line 31 where the key is required (function only called with that key present).
                              Fix: pattern-match `%{event: event} = payload` in the function head.
    with-clauses-clean ..... PASS
    no-dynamic-atoms ....... FAIL — see no-injection above (same root cause).
    pattern-exhaustive ..... PASS

  Architecture:
    boundaries-respected ... PASS
    no-hacky-shortcuts ..... PASS

  [... remaining dimensions ...]

Result: 19/24 PASS — 5 criteria need fixing

Failing criteria with fixes:
1. edge-cases: short-circuit empty payload at lib/backend_api/deals/webhook_handler.ex:45
2. input-validated: route through changeset at lib/backend_api/deals/webhook_handler.ex:78
3. no-injection: replace String.to_atom at lib/backend_api/deals/webhook_handler.ex:62
4. assertive-access: pattern-match payload at lib/backend_api/deals/webhook_handler.ex:31
5. no-dynamic-atoms: same root cause as no-injection — replace String.to_atom

Unverified:
• Did not run `mix dialyzer` (PLT build cost; see verification-gate close)
• Did not check whether `webhook_handler_test.exs` covers `{:error, :network_timeout}` path
```

### Output Format — Consensus Mode

When Step 0c activated consensus mode, replace the standard banner with a Models header, show dual PASS/FAIL columns, replace `Result:` with `Consensus Result:`, and add three reconciliation sections:

```
DEVIL'S ADVOCATE CRITIQUE (Binary Eval — Elixir — Consensus Mode)
════════════════════════════════════════════════════════════════

Target: code changes for BackendAPI.Deals.WebhookHandler
Models: Claude (Opus) → Codex (gpt-5-codex) → Claude (reconciliation)

  Correctness:
    tests-pass ............. PASS / PASS  (consensus)
    logic-correct .......... PASS / PASS  (consensus)
    edge-cases ............. FAIL / FAIL  (consensus — see below)

  Security:
    no-secrets ............. PASS / PASS  (consensus)
    input-validated ........ FAIL / PASS  (DISPUTED — Codex argues changeset is implicit at line 78)
    no-injection ........... FAIL / FAIL  (consensus — Codex extended the fix)

  Idiomatic Elixir:
    assertive-access ....... FAIL / FAIL  (consensus)

  ...

Consensus Result: 18/24 CONSENSUS PASS — 4 consensus FAILs, 1 codex-only accepted, 1 disputed

Consensus FAILs (fix first):
1. edge-cases: short-circuit empty payload at lib/backend_api/deals/webhook_handler.ex:45
2. no-injection: replace String.to_atom at lib/backend_api/deals/webhook_handler.ex:62
   (Codex extended the fix: use String.to_existing_atom inside a try/rescue, fallback to {:error, :unknown_event_type})
3. assertive-access: pattern-match payload at lib/backend_api/deals/webhook_handler.ex:31
4. pattern-exhaustive (codex-only, accepted): add {:error, _} clause to Repo.insert handler at lib/backend_api/deals/webhook_handler.ex:84

Disputed (surface to user, no auto-fix):
1. input-validated: Claude sees raw map at line 78; Codex argues upstream caller already cast.
   Resolution path: read upstream caller and decide. If caller does NOT cast, this is a real FAIL.

Unverified:
• Did not run `mix dialyzer` (PLT build cost)
• Codex round timed out once and was retried; second attempt succeeded
```

## Rules

- Be genuinely critical, not performatively critical
- Every FAIL must cite `file:line` evidence and include a `Fix:` suggestion
- Every PASS should have brief evidence (not just "looks fine")
- Anchor your criticisms in specific, concrete concerns, not vague "could be better"
- If you realize the work has a genuine flaw during assessment, say so clearly
- Never skip the session log write
- The "Unverified" section is MANDATORY, must list at least one thing. If you claim you verified everything, you're lying.
- If the project depends on `claude-code-elixir` thinking-skills (loaded in Step 1), cite the skill's rule name in your FAIL message rather than re-deriving the rule from memory. Example: `Fix: ... (elixir-thinking: 'avoid non-assertive map access')`.
