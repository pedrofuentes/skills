# Sub-Agent Prompt & Handoff Contracts

Operational templates for assembling implementer prompts, chaining context forward,
and validating what comes back. **Loaded by SKILL.md §2-DELEGATE / §5-LIFECYCLE / §7-CHAIN
/ §Sub-Agent Prompt Requirements** when you build an implementer prompt or receive its report.

Scope reminder (Option A choreography): the implementer does TDD → opens a PR → **STOPS**
(reports PR URL + HEAD SHA). It NEVER invokes Sentinel and NEVER merges. The coordinator
invokes Sentinel and spawns a merge agent. Nothing here changes that.

These contracts apply to implementers AND to retry / Sentinel-fix respawns. Merge and revert
agents get a scoped single-operation prompt instead (not the template below).

---

## Implementer prompt template (PE-1) — instantiated, not "verbatim + count"

Build every implementer prompt by **substituting** the bracketed placeholders below from the
current `AGENTS.md` and project config. After substitution the prompt is the literal text the
agent receives — there is no "paraphrase vs. verbatim" judgment call and no bullet counting.

```
## Task — T-[TASK_ID]: [ONE_SENTENCE_GOAL]

## Model Tier — [OPUS|SONNET]: [JUSTIFICATION].
If this proves more complex than scoped, STOP and report to the coordinator.

## Project Constraints Extract   (see "Project Constraints Extract" below — copy, don't summarize)
- Isolation: [ISOLATION_METHOD]   e.g. git worktree add ../wt-[TASK_ID] -b [BRANCH_NAME]
- Branch from current main HEAD: [BASE_SHA]  (git rev-parse origin/main)
- TDD choreography: [TDD_RULE]   (failing-test commit FIRST, then implementation commit)
- Open a PR, then STOP and report PR URL + HEAD SHA. Do NOT invoke Sentinel. Do NOT merge.
- Commit trailer (exact): [COMMIT_TRAILER]
- ASK-FIRST triggers (verbatim from AGENTS.md): [ASK_FIRST_TRIGGERS]
- Test: [TEST_CMD]   Build: [BUILD_CMD]   Lint: [LINT_CMD]   Typecheck: [TYPECHECK_CMD]

## Depends On — [PRED_TASK_IDS]; see Successor Handoff Contract below (empty if none).

## Context — [BACKGROUND, current state of main, relevant interfaces/schemas].

## Requirements — [TESTABLE_ACCEPTANCE_CRITERIA].

## Files to Create/Modify — [EXPLICIT_LIST_WITH_PURPOSE].
Authorized adjacency rules apply (see below); list & justify any extra file in your report.

## Out of Scope — [EXCLUSIONS_AND_FILE_RESTRICTIONS].

## Docs-Impact — [none | README | API docs | changelog | runbook | migration guide].
If non-none and no separate docs task exists, satisfy the docs acceptance criterion in THIS PR.

## Verification — run [TEST_CMD] / [BUILD_CMD] / [LINT_CMD] / [TYPECHECK_CMD]; expected: [RESULTS].

## §FORBIDDEN — paste the §AGENTS.md Rules block (the §FORBIDDEN bullets are a safety block:
   reproduce them in full — never trim, summarize, or substitute placeholders into them).

## Completion Report — return the schema in "Completion / failure report" below. Reports
   missing any required field are rejected and the agent is re-prompted for the missing fields.
```

### Placeholder-completeness check (replaces line-counting)

Before spawning, confirm **every** `[PLACEHOLDER]` is resolved from the current `AGENTS.md` /
project config. If any placeholder cannot be filled from authoritative sources → **STOP** (do
not guess, do not spawn). A missing isolation method, trailer, or ASK-FIRST list is an
AGENTS.md-incompleteness stop per SKILL.md Startup, not a prompt to improvise.

The §FORBIDDEN block is exempt from substitution: it is propagated in full, unchanged.

---

## Project Constraints Extract (PE-2) — mandatory in every implementer prompt

A dedicated, **copied-not-summarized** section so the agent works from exact project rules.
Pull each value verbatim from `AGENTS.md` / config; never paraphrase a command or a trigger.

| Field | Source | Note |
|-------|--------|------|
| ASK-FIRST triggers | AGENTS.md | Full list, verbatim. The agent STOPs on any hit. |
| Isolation command(s) | AGENTS.md | Exact `worktree`/clone command + branch naming rule. |
| Review choreography | AGENTS.md / §5-LIFECYCLE | "Open PR → STOP → report; no self-review/merge." |
| Commit trailer | AGENTS.md | Exact `Co-authored-by:` line. |
| Test / Build / Lint / Typecheck | AGENTS.md, package.json, pyproject.toml, Makefile, etc. | Exact commands; mark "not configured" where absent so the agent skips cleanly. |

If a field is absent from AGENTS.md/config, write `not configured` (the agent skips it) — but
isolation, trailer, and ASK-FIRST are mandatory; their absence is a Startup STOP, not a skip.

---

## Successor Handoff Contract (PE-3) — the §7-CHAIN payload

When a successor depends on a merged predecessor, put a compact handoff schema in the
successor's `## Context` / `## Depends On`. It is a short schema, not a narrative — pass only
the **subset relevant** to the successor (omit empty fields).

| Field | Content |
|-------|---------|
| Changed | What the predecessor changed, in 1–2 lines. |
| Why | The intent, so the successor doesn't re-litigate the design. |
| Invariants | Behavioral invariants the successor must preserve. |
| Interfaces | New/changed interfaces, types, exports — **with exact signatures**. |
| Example call-site | One short snippet showing correct use. |
| Tests added | Test files/cases added, so the successor extends rather than duplicates. |
| Config/env impact | New env vars, flags, schema/config changes. |
| Pitfalls | Known landmines / ordering constraints discovered while building it. |
| Do-not-touch | Files the successor must leave unchanged (predecessor owns them). |

Sources: the predecessor's completion report (below) plus `git diff <merge-SHA>^ <merge-SHA>`.
Bind the contract to the predecessor's merge SHA and have the successor branch from current
`main` HEAD (`git rev-parse origin/main`), never a stale ref.

---

## Completion / failure report schema (PE-4) — required return

Every implementer (and retry/fix respawn) MUST return this. The coordinator **rejects
incomplete reports** and re-prompts for the missing fields before reviewing or merging.

| Field | Required | Content |
|-------|----------|---------|
| Status | ✅ | `complete` \| `blocked` \| `failed` (+ one-line reason if not complete). |
| Branch | ✅ | Branch name. |
| Worktree path | ✅ | Isolated env path (for §6-VERIFY cleanup), or `n/a`. |
| Commit SHAs | ✅ | Ordered list incl. the failing-test commit and HEAD SHA. |
| Changed files | ✅ | Every path touched, with the in-scope vs. adjacency tag (below). |
| Test / Build / Lint / Typecheck | ✅ | Pass/fail per gate; `not configured` if the command is absent. |
| PR URL + HEAD SHA | ✅ | The PR to review; HEAD SHA binds the Sentinel review. |
| Adjacency justifications | ⬚ | Required only if extra files were touched (see PE-5). |
| Unresolved risks | ✅ | Open risks/assumptions, or `none`. |
| Successor notes | ⬚ | Fields for the Handoff Contract if a successor is queued. |

The implementer reports a PR — it never reports a Sentinel verdict or a merge (those are the
coordinator's via §5-LIFECYCLE). Feed accepted reports into §8-PERSIST and resume state
(see `references/state-and-recovery.md`).

---

## Authorized adjacency (PE-5) — narrow allowance, not a loophole

Replaces the absolute "if another file MUST change, STOP." The agent MAY touch a small,
pre-authorized set **without stopping**, but MUST list and justify each extra file in its
completion report (Adjacency justifications). Anything outside this set → STOP and report.

**Authorized (justify each):**
- The paired test file(s) for the code under change.
- A directly-coupled type/schema/interface file the change cannot compile/run without.
- Required export/barrel wiring to surface a new symbol (still subject to §4 sequential-only).
- Minimal docs strictly tied to this task (the PE-2 / LC-3 Docs-Impact requirement).

**NOT authorized (STOP):** unrelated refactors, new dependencies (§FORBIDDEN C2), CI/`.github`/
secrets (§FORBIDDEN B2), `AGENTS.md`/`docs/SENTINEL.md` (§9-T4), migrations/destructive ops, or
any file outside the task's concern. Adjacency never overrides §FORBIDDEN or sequential-only
locks. Unjustified extra files = off-scope → §ERROR (NEVER merge; respawn tighter).

---

## Context-budget policy (PE-6) — what to include vs. trim

Prompts must fit the agent's context without dropping safety. Priority order when trimming:

| Block | Policy |
|-------|--------|
| §FORBIDDEN, AGENTS.md rules, Project Constraints Extract | **NEVER trim.** Always full. |
| Predecessor handoff summary | Cap to a few bullets — only the subset the successor uses. |
| Interfaces / code snippets | Include only signatures/snippets directly relevant to this task. |
| Full Sentinel rejection report | Include in full **only** on a rejection-retry prompt (§ERROR), not on first attempts. |

Trimming only ever applies to non-safety predecessor/reference context; safety-critical blocks
are exempt. This complements §6-VERIFY, not it.

---

## Docs-Impact (LC-3) — docs as a tracked deliverable

Every task carries a **Docs-Impact** field: `none | README | API docs | changelog | runbook |
migration guide`. Set it when building the task list and restate it in the prompt (template
above).

If Docs-Impact is non-`none`, the work is not mergeable until docs are handled:
- Either add a **separate docs task** (own scope/PR), or
- Add a **docs acceptance criterion** to this task's Requirements (satisfied in this PR via the
  authorized-adjacency "minimal task-tied docs" allowance).

The coordinator checks the chosen path is satisfied before the merge agent runs. Docs work is
reversible — never gate it behind §9-T4; the irreversible publish/deploy step still is.
