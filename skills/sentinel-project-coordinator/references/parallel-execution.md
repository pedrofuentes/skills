# Parallel Execution — Fleet Safety, Merge Order & Cleanup

Operational playbook for safely running and reaping a parallel fleet. **Loaded by SKILL.md
§4-PARALLEL, §6-VERIFY check 8, §7-CHAIN, §ERROR, and §On Completion** when the coordinator
parallelizes work, predicts/sequences merges, respawns a reverted task, or tears down isolation.

Guardrails preserved verbatim: **HARD CAP 5 implementers (default ceiling)**, **one-at-a-time
merge**, **`git revert` not `git reset` on main**, **A4 only restricts environments outside this
session**. Nothing here weakens those. Rebase tiering → `references/model-routing.md`; retry/attempt
counters → `references/state-and-recovery.md#retry-fsm`.

---

## PC-1: Sequential-only files taxonomy (expanded)

Any task touching a file in this taxonomy → run **sequential, never in a fleet**. These files share
one trait: **git auto-merges them cleanly (no text conflict) yet the result is semantically
corrupt.** A green merge is NOT proof of correctness for them.

| Class | Examples / globs | Why it auto-merges but corrupts |
|-------|------------------|---------------------------------|
| Barrel / index files | `index.ts`, `mod.rs`, `__init__.py` re-exports | Two added export lines never textually collide; duplicate/missing symbols surface only at build. |
| Route / router registration | router config, URL maps, DI containers | Appended registrations merge; duplicate routes or shadowed handlers pass merge, fail at runtime. |
| Config & lockfiles | `package.json`, `*.lock`, `tsconfig`, env config | Adjacent keys merge; semantic conflicts (dup scripts, incompatible versions) survive. |
| Shared types / schemas | shared `*.d.ts`, type modules | Independent additions merge into an inconsistent type surface. |
| **DB migrations / numbered schema** | `migrations/0007_*.sql`, `V12__*.sql`, timestamped/positional files | Two agents pick the **same next ordinal**; both files coexist after merge, but only one applies or they apply out of order. |
| **Snapshot & golden fixtures** | `__snapshots__/`, `*.snap`, `*.golden`, `*.approved.txt` | Regenerated snapshots merge line-by-line into a hybrid that matches neither implementation. |
| **i18n message catalogs** | `messages.*.json`, `*.po`, `locales/*.json` | New keys merge cleanly; collided keys/IDs or ordering drift silently win-last. |
| **Codegen source schemas** | `*.proto`, `schema.graphql`, `openapi.yaml`, Avro/JSON-Schema | Merged source schema regenerates a corrupt client/server stub; the breakage is in generated output, not the merge. |
| **CHANGELOG / release notes** | `CHANGELOG.md`, `RELEASES.md` | Both prepend under the same heading; entries interleave or duplicate the version stanza. |
| **Monorepo workspace manifests** | `pnpm-workspace.yaml`, `turbo.json`, `nx.json`, root `workspaces` | Package/pipeline lists merge; duplicate or conflicting project graph entries pass merge, break the build graph. |
| Generated code | codegen output committed to the repo | Same root cause as its source schema — regenerate, never hand-merge. |

**Rule:** if two in-flight tasks each declare a file in any class above, **serialize them** (§4
fleet → sequential). Treat codegen *source* and its *generated output* as one unit.

---

## PC-2: Pairwise file-overlap pre-flight

Run **before launching a fleet** and **again whenever queuing a new agent into a running fleet**.

1. Collect each candidate task's declared **`Files to Create/Modify`** + **`Out of Scope`** lists
   (§Sub-Agent Prompt Requirements). Expand globs against the worktree.
2. For every unordered pair `(A,B)` of tasks intended to run concurrently, compute the intersection
   of their declared file sets.
3. Disposition:
   - **Empty intersection** → safe to run in parallel.
   - **Non-empty intersection** → **merge the two tasks into one** (preferred when they are really
     one unit of work) **or serialize that pair** (one waits for the other's §6-VERIFY). Never fleet
     a colliding pair on the promise of a later rebase.
   - **Any shared file is a PC-1 sequential-only class** → serialize unconditionally, even if you
     believe the edits are disjoint (auto-merge hides the corruption).
4. Record each non-empty intersection and its disposition in state before spawning.

This is a cheap static check; PC-4 (smoke probe) and PC-5 (merge-tree prediction) catch what static
declarations miss.

---

## PC-3: Concurrency-budget accounting

The cap counts **implementers**, but real host concurrency is higher: every Sentinel review, merge
agent, and rebaser is an additional live worktree/process competing for **ports, the database, and
package caches**. With 5 implementers plus their reviewers/mergers you can reach ~10 concurrent
worktrees.

**Accounting rule:**

| Counts against the cap of 5 | Adds real host load (budget for it, not capped) |
|-----------------------------|-------------------------------------------------|
| In-flight **implementers** building a task | Sentinel reviewers, merge agents, revert agents, rebasers |

- **Keep 5 as the default ceiling.** Never auto-raise it.
- A raise above 5 requires **explicit user approval recorded in state** (e.g.
  `coordinator_state.concurrency_cap` with the approving turn). Absent that record, the ceiling is 5.
- If the PC-4 probe reports port/DB/cache contention, **lower** effective concurrency (more
  serialization), never raise it.
- Do **not** count non-implementer agents toward 5, but do confirm the host can sustain the extra
  reviewer/merge load before saturating all 5 implementer slots.

---

## PC-4: Fleet isolation smoke probe

Before the **first fleet spawn of the session**, empirically verify isolation instead of trusting the
AGENTS.md text check alone.

1. Spawn a **throwaway probe agent** (single-purpose, may run at FAST tier per
   `references/model-routing.md` MR-4): create one worktree **exactly per AGENTS.md's isolation method**,
   run the **project's test command** inside it while main's environment is also live.
2. The probe reports contention against main's env:
   - **Ports** — does the suite bind a fixed port already held by main? (`EADDRINUSE`)
   - **Locks** — shared DB file / single test database / global lockfile contention.
   - **Caches** — shared package/build cache writes that race (e.g. one global cache dir).
3. Disposition:
   - **Clean** → fleet mode allowed (still capped at 5). **Cache the result for the session** (e.g.
     `coordinator_state.isolation_probe = ok`); do not re-probe.
   - **Contention** → **drop to sequential, warn the user** with the specific collision; record the
     probe result so the rest of the run stays sequential unless the user resolves isolation.
4. Tear down the probe worktree via the GW-5 teardown sequence.

---

## PC-5: Conflict prediction + deterministic merge order

Applies when **multiple APPROVED fleet branches** are queued to merge. Predict conflicts read-only
**before** touching main; this does NOT change the one-at-a-time merge invariant — it only chooses
the order.

1. `git fetch origin` so `origin/main` is current.
2. For each approved branch `B`, predict its conflict surface against main **without writing**:
   ```
   git merge-tree --write-tree origin/main <B>        # modern; conflicts listed in output
   # fallback: git merge-tree $(git merge-base origin/main B) origin/main B
   ```
   Count conflicted files / hunks = `B`'s conflict surface.
3. **Merge in descending conflict-surface order** (largest predicted conflict first). Merging the
   most-entangled branch while main is simplest minimizes downstream rebase cascades; each later
   branch rebases onto an already-integrated main via **new branch + cherry-pick** (never
   force-push), at tier ≥ implementer (`references/model-routing.md`).
4. **Escalation trip-wire:** if a predicted conflict touches **any PC-1 sequential-only file**, the
   tasks were **not actually independent** → stop the fleet merge and escalate as **§9 T1** (plan
   ambiguity / hidden dependency). Do not "rebase through" a sequential-only collision.
5. Re-run prediction after each merge (main moved). Cap rebase attempts per branch at **2**
   (`references/state-and-recovery.md#retry-fsm`); a 2nd failure → hidden-dependency §9 T1.

Merge remains strictly serial: predict → order → merge one → §6-VERIFY → predict next.

---

## GW-1: Session-owned merged-branch cleanup backstop

The cleanup backstop must reap **branches**, not just worktrees. Only branches **this session
created** and **confirmed merged** are eligible (A4 carve-out covers session-created branches).

For each session-owned branch after its merge passes §6-VERIFY, and again in the §On Completion
sweep:

```
git fetch --prune origin
git branch --list '<branch>' --merged origin/main   # non-empty output = confirmed merged
git branch -d <branch>                              # local; -d refuses if unmerged
git push origin --delete <branch>                   # remote
```

- Use `-d` (safe delete), not `-D` — it refuses to drop an unmerged branch, which is the guard.
- **Squash/rebase-merge carve-out:** squash- or rebase-merged branches never show in `--merged`
  and `-d` refuses to delete them. ONLY THEN, after confirming the merge via the recorded merge
  SHA or `gh pr view <branch> --json state,mergedAt`, `git branch -D <branch>` is permitted —
  the merge confirmation replaces `-d`'s ancestry guard. Never `-D` without that confirmation.
- **Ignore "not found" / "remote ref does not exist"** errors (already gone is success).
- Never delete a branch this session did not create, and never one not confirmed merged.

---

## GW-5: Robust worktree teardown

Used by §6-VERIFY check 8, the §On Completion sweep, and PC-4 probe cleanup. **Prune first**, then
enumerate, then force-remove only confirmed orphans:

```
git worktree prune                       # 1. drop already-gone administrative entries
git worktree list                        # 2. enumerate what actually remains
git worktree remove --force <path>       # 3. remove each confirmed session-created orphan
```

- Run `prune` **before** `list` so the list reflects reality (avoids removing a path that is already
  gone and erroring).
- **`git worktree remove --force` is A4-EXEMPT**: it is the sanctioned backstop for removing an
  **orphaned agent worktree this session created** (e.g. from a failed/timed-out agent). It deletes
  only that worktree's checkout, not history.
- This is **categorically different from `git reset --hard` (§FORBIDDEN A2)**, which rewrites working
  state/history and is never permitted on main. Never substitute one for the other.
- Ignore "not a working tree" / "not found" errors. Only force-remove paths confirmed to belong to
  this session.

---

## GW-4: Revert context to respawn

When a merged task is reverted (§6-VERIFY failure, §ERROR "broke main"), the successor respawn MUST
carry the failure forward so it does not reproduce it.

1. The **revert agent** records both SHAs: the **reverted merge SHA** and the **revert commit SHA**
   (persist in state, `references/state-and-recovery.md#retry-fsm`).
2. The respawn (new agent — never reuse the failed one) prompt MUST include a
   **`## Prior Attempt — REVERTED`** section containing:
   - the reverted-merge SHA and the revert SHA,
   - **which checks failed** (e.g. §6-VERIFY check 2 TESTS, check 7 SCOPE) with the exact failing
     output,
   - a directive: *do not reproduce this failure; address the named root cause first.*
3. Name the revert in the successor's **`## Depends On`** so it branches from the post-revert main
   HEAD and treats the revert as a predecessor (§7-CHAIN).
4. Attempt/rejection counters continue across the revert per the retry FSM — a revert does not reset
   the ladder.

---

## RR-5: Fleet circuit breaker

After **each fleet wave**, aggregate outcomes before launching the next wave or spawning new agents.

**Trip conditions (either fires → breaker opens):**
- **≥2 tasks in the wave failed with a shared root cause** (same error class, same module, same
  missing interface), or
- **repeated rebase conflicts hit the same file(s)** across branches.

**On trip:**
1. **Freeze new spawns** — no further implementers enter the fleet.
2. **Recompute the dependency graph** for remaining tasks; treat the shared root-cause module / the
   repeatedly-conflicting file as a **hidden shared dependency**.
3. **Serialize the overlap** — tasks touching it run sequentially (and, if a PC-1 file, per PC-1).
4. **Cap each branch at 2 rebase attempts** (`references/state-and-recovery.md#retry-fsm`); a 3rd is
   not attempted.
5. If resolving the overlap needs a **new integration task** (not on the approved list), trigger
   **§9 T6** — surface and request user approval; never add it silently.

The breaker only ever **reduces** concurrency and **adds** serialization — it never raises the cap or
relaxes a guardrail. Once the overlap is serialized and clear, resume the (now smaller) fleet under
the same cap-5 ceiling.
