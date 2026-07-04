# State & Recovery

Durable run-state, crash recovery, the canonical retry machine, and timeout salvage for the
coordinator. **Loaded by SKILL.md §8-PERSIST / §2-DELEGATE / §9-PAUSE / §ERROR / Startup** when the
coordinator must persist run-state, restart mid-run, decide a retry, or salvage a timed-out agent.

This file owns the **authoritative retry FSM** (`#retry-fsm`). Other references link here instead of
re-specifying retry counts. It never weakens a guardrail: all reconciliation is **read-only** (§0),
state lives in the DB (never in a repo file — §0 protects the project; see the terminal fallback
below for DB-less platforms), and resume is
**crash-recovery only — NOT a periodic cadence** (§9 keeps no pause cadence).

---

## Storage model (with graceful fallback)

Persist structured run-state in two custom tables. The runtime DB may be ephemeral or read-only, so
**always `CREATE TABLE IF NOT EXISTS`** and **degrade gracefully**: if table creation or write fails,
fall back to stuffing the same fields as a structured block inside the existing `todos.description`
(the v1.x approach) and continue. Never abort a run because the custom tables are unavailable.

**Terminal fallback — no session DB or todos tool at all:** keep the approved task list and
per-task state in a `coordinator-state.json` in a scratch/temp location **OUTSIDE the repository**
(never in the working tree — §0 protects the project, not the coordinator's own scratch space),
and tell the user ONCE that crash-resume durability is reduced. Never write state into repo files.

```sql
-- SP-3 / RR-2: structured ledger — one row per task attempt-lifecycle.
CREATE TABLE IF NOT EXISTS coordinator_state (
  todo_id            TEXT PRIMARY KEY,   -- FK to todos.id
  branch             TEXT,               -- expected branch name (persisted at SPAWN time)
  agent_id           TEXT,               -- in-flight implementer/merge/revert agent id (at SPAWN)
  model_tier         TEXT,               -- 'top' | 'standard' | 'light' (implementer tier, SKILL.md §2)
  attempt            INTEGER DEFAULT 1,  -- implementation-attempt counter (see #retry-fsm)
  sentinel_cycle     INTEGER DEFAULT 0,  -- cumulative Sentinel-REJECTED counter (see #retry-fsm)
  worktree_path      TEXT,               -- isolated env to reconcile/clean on resume
  commit_sha         TEXT,               -- branch HEAD reported by implementer
  merge_sha          TEXT,               -- merge SHA from the merge agent's completion report;
                                         -- persisted on receipt, confirmed at §6-VERIFY check 1
  sentinel_report_id TEXT,               -- Sentinel Report ID bound to the merged SHA
  reviewer_model     TEXT,               -- model used for the Sentinel review (MR-5 / retro)
  spawned_at         TEXT,               -- ISO-8601, set at spawn
  completed_at       TEXT                -- ISO-8601, set when task reaches a terminal state
);

-- SP-4: durable running log — replaces the evictable §8-PERSIST conversation log.
-- 'routing' covers tier_resolution entries (Startup TOP resolution) and routing notes.
CREATE TABLE IF NOT EXISTS coordinator_log (
  id        INTEGER PRIMARY KEY AUTOINCREMENT,
  timestamp TEXT NOT NULL,               -- ISO-8601
  category  TEXT NOT NULL CHECK (category IN ('progress','concern','question','suggestion','routing')),
  entry     TEXT NOT NULL
);

-- MR-2: session-scoped live-routing entries (see references/model-routing.md).
CREATE TABLE IF NOT EXISTS coordinator_routing (
  id          INTEGER PRIMARY KEY AUTOINCREMENT,
  module      TEXT,                      -- path/glob the failing task touched
  pattern     TEXT,                      -- concern that broke (e.g. 'TOCTOU', 'migration')
  failed_tier TEXT,
  routed_tier TEXT,                      -- always 'TOP'
  reason      TEXT,                      -- one line + originating task id
  created_at  TEXT                       -- ISO-8601
);
```

**`todos` stays the source of truth for status**; `coordinator_state` holds the structured
narrative fields; `todos.description` remains human-readable prose. The §8-PERSIST "Progress /
Concerns / Questions / Suggestions" log MUST be written to `coordinator_log`, not to conversation
(evictable) and not to a file (§0). Query both tables before any spawn.

### Persist at spawn time (SP-2 / GW-2)

The single largest resumability gap is losing in-flight handles. **Before** each spawn, write the
row so a context-evicted coordinator can find and reconcile the worker deterministically:

```sql
INSERT INTO coordinator_state (todo_id, branch, agent_id, model_tier, attempt, worktree_path, spawned_at)
VALUES (:id, :branch, :agent_id, :tier, :attempt, :worktree, :now)
ON CONFLICT(todo_id) DO UPDATE SET
  branch=:branch, agent_id=:agent_id, model_tier=:tier, attempt=:attempt,
  worktree_path=:worktree, spawned_at=:now;
```

---

## Status-transition guards (SP-6)

`todos.status` is a small state machine. Only these transitions are legal:

| From          | To            | Trigger |
|---------------|---------------|---------|
| `pending`     | `in_progress` | spawn implementer |
| `in_progress` | `done`        | all §6-VERIFY checks pass |
| `in_progress` | `blocked`     | §9 pause / §ERROR escalation to user |
| `blocked`     | `in_progress` | user unblocks / retry resumes |
| `done`        | `in_progress` | **only** after a delegated `git revert` of the merge (§6/§ERROR) |

Everything else is illegal (e.g. `pending`→`done`, `done`→`blocked`). **Make every write
idempotent: assert the expected `from` state in the `WHERE` clause and confirm one row changed.**

```sql
-- Example: mark done only if currently in_progress; rows-affected must equal 1.
UPDATE todos SET status='done', updated_at=:now WHERE id=:id AND status='in_progress';
-- 0 rows ⇒ illegal/duplicate transition: STOP, reconcile via §8a-RESUME, do NOT force the write.
```

This makes replays safe: a re-run that already applied a transition is a no-op, not a corruption.

---

## §8a-RESUME — crash recovery & reconciliation (SP-1 / OA-4 / SP-5)

Run this **on startup whenever an approved task list already exists in the DB** (or after the
coordinator's context is evicted mid-run). It does NOT re-request approval — the list was already
approved. It resumes from the first unfinished task. **All commands here are read-only (§0-safe).**

**1. Load state.** Query `todos` + `coordinator_state` for every task not `done`. If the custom
tables are absent, parse the structured block from `todos.description` (fallback mode).

**2. Reconcile each non-`done` task against git reality** (read-only):

| Persisted state | Git probe (read-only) | Reconciled action |
|-----------------|-----------------------|-------------------|
| `in_progress`, `merge_sha` set | `git log origin/main --grep=<report_id>` shows the merge | Treat as merged → **re-run §6-VERIFY** on `merge_sha`; pass ⇒ `done`, fail ⇒ delegate revert (§6) |
| `in_progress`, branch merged but `merge_sha` unset | `git branch --merged origin/main` lists `branch` | Capture the real merge SHA, then re-verify as above |
| `in_progress`, branch exists, unmerged | `git branch --list <branch>` + `git log <branch>` | Worker likely died → **timeout-salvage** (`#timeout-salvage`) |
| `in_progress`, no branch | `git branch --list <branch>` empty | Nothing landed → reset to `pending`, clean any worktree, respawn |
| `blocked` | — | Leave blocked; surface in resume summary |

Probe commands (never mutate): `git fetch --prune` (read-only sync), `git branch --list <branch>`,
`git branch --merged origin/main`, `git log origin/main --oneline`, `git log <branch> --oneline`,
`git worktree list`.

**Squash/rebase-merge caveat:** squash- or rebase-merged branches NEVER appear in
`git branch --merged origin/main`. Before concluding a worker died (row 3), also probe
`git log origin/main --grep='<branch or report_id>'` and/or `gh pr view <branch> --json state,mergedAt`;
a squash-merged task reconciles as merged → capture the squash commit SHA and re-verify as in row 1.

**3. Finish or roll back an interrupted merge (RR-2).** If a merge was mid-flight (branch merged but
verification never recorded), either confirm it via §6-VERIFY or delegate a revert agent — the
coordinator never merges/reverts itself (§0). Record the outcome in `coordinator_state`.

**4. Clean orphans.** For any reconciled-away worktree/branch this session created, follow the
§On-Completion sweep (`git worktree prune`, then `git worktree remove <path>`; ignore "not found").
A4 exempts session-created environments.

**5. Resume.** Apply a legal status transition per the guards above, then continue the
already-approved list from the first non-`done` task. **No new approval, no cadence** — just resume.

---

## #retry-fsm — canonical retry state machine (RR-3)

The authoritative reconciliation of the §2-DELEGATE retry table, §9 T2 ("2+ attempts"), and the
§ERROR counts. Two **orthogonal** counters tracked per task in `coordinator_state`:

- **`attempt`** — implementation-failure counter (wrong output, off-scope, broke-main-after-revert,
  discarded timeout). Range **1→3**.
- **`sentinel_cycle`** — Sentinel REJECTED fix-and-re-review counter. Range **1→5**, INDEPENDENT of
  `attempt`. A rejection ladder cycle is NOT an implementation attempt and does not increment
  `attempt`. An implementation failure discards the pending rejection report for the new attempt,
  but **`sentinel_cycle` is cumulative per task and never resets** (max 5 total, per §2/§ERROR).

**Implementation-failure ladder (the `attempt` counter):**

| `attempt` | Tier | On failure → next |
|-----------|------|-------------------|
| 1 | as assigned | retry once, same tier, improved prompt → `attempt=2` |
| 2 | same tier | escalate one tier (STANDARD→TOP; stay if already TOP) → `attempt=3` |
| 3 | TOP | **terminal:** STOP → `blocked`, escalate to user |

**Never attempt 4. Never retry twice at a tier that already failed.** Each retry spawns a NEW agent
(never reuse a failed one) and forwards what went wrong (and, for a reverted merge, GW-4's
`## Prior Attempt — REVERTED` SHA + reason).

**Sentinel REJECTED ladder (the `sentinel_cycle` counter)** coexists with the above:
respawn the implementer at **TOP** (trivial mechanical 🔴s → same tier; →
`references/model-routing.md` MR-3), fix **🔴 only** (never 🟡/🟢), re-review
with the prior Report ID + fix delta (`git diff <prev-SHA>..HEAD`). **Max 5 cycles → `blocked`, user.**
This ladder bypasses the `attempt` ladder (§2-DELEGATE "Sentinel rejection override").

**Counter resets / terminal states:**
- Task reaches `done` (§6-VERIFY all-pass) → both counters retired; set `completed_at`.
- `attempt` hits 3, or `sentinel_cycle` hits 5, or a §9/§FORBIDDEN trigger fires → `blocked` (user).
- A delegated revert of a merged task that then must be redone starts a fresh `attempt` (the
  `done`→`in_progress` transition), capped by the same 1→3 ladder.

Specialized per-failure escalations from §ERROR map onto this FSM: **off-scope** and **timeout**
each carry their own "2 → user" sub-limit (still inside the `attempt`≤3 ceiling); **broke-main**
caps at "reverted twice → user".

---

## #timeout-salvage — deterministic timeout salvage (RR-4)

Replaces the undefined "substantial progress" wording with a deterministic rule. On agent timeout,
inspect the branch **read-only** (`git branch --list <branch>`, `git log <branch> --oneline`,
`git status` in the worktree):

| Branch condition | Salvage decision |
|------------------|------------------|
| **Clean committed progress** — commits exist AND working tree clean AND TDD commits present | **ONE takeover** at the **same tier**: spawn a fresh agent to finish from the committed state, then **full re-verification** (Sentinel + §6-VERIFY, no trust inherited). Counts as the current `attempt`. |
| **Dirty or unknown** — uncommitted/partial work, detached state, or branch state unreadable | **Discard**: delegate cleanup of branch + worktree, then respawn fresh from `main` HEAD. Increment `attempt`. |

**Bounded to ONE takeover.** If the single takeover also times out (or a 2nd timeout occurs on the
task), stop salvaging and fall back to the normal `#retry-fsm` escalation — and the §ERROR "2
timeouts → user" limit applies. The coordinator performs no commits or merges during salvage;
branch deletion and worktree removal for session-created environments may be done directly per
§0/GW-5, or delegated.
