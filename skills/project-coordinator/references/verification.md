# Verification & Review Gates

Operational detail for proving `main` stays green and that every merge was independently
reviewed. Extends — never replaces — the deterministic checks in the body.

**Loaded by SKILL.md §5-LIFECYCLE / §6-VERIFY / §9-PAUSE / §ERROR** when the coordinator
needs the green-before-merge evidence rules, the augmented check table, flaky-test triage,
the review contract + fallback reviewer, or the conditional security / migration gates.

All gates here are **additive**. They never relax the existing 8 checks, the revert-not-reset
rule, the human gate on irreversibles, or the Option A review/merge choreography. Every
*conditional* gate (coverage, security, migration) **skips cleanly when the project does not
configure the tool** — detect, then run or skip; a missing tool is PASS-by-skip, never a block.

---

## green-evidence — Green-before-merge evidence gate (QA-1)

Goal: make "main is always green" an **invariant proven before merge**, not something a
post-hoc revert discovers. The coordinator still NEVER merges — the merge agent does.

**Implementer obligation (add to the completion report, PE-4 schema).** Before it STOPs at
the PR, the implementer runs the full configured suite on **branch HEAD** and pastes, verbatim:

| Field | Content |
|-------|---------|
| `verified_sha` | `git rev-parse HEAD` on the branch at the moment the suite ran |
| `tests` | full test command + tail of output showing the pass total |
| `build` | build command + success line (skip line if unconfigured) |
| `types` | typecheck command + result (skip if unconfigured) |
| `lint` | lint command + result (skip if unconfigured) |
| `coverage` | changed-file coverage summary if coverage is configured (see #test-count-hardening) |

**Coordinator obligation.** Record `verified_sha` in state alongside the PR
(see `references/state-and-recovery.md` → Storage model). Then enforce one chain of identity:

```
verified_sha  ==  the SHA Sentinel reviewed  ==  the source SHA the merge agent merges
```

- If branch HEAD advanced after the evidence was produced (`git rev-parse origin/<branch>` ≠
  `verified_sha`), the evidence is **stale** → require the implementer to re-run the suite and
  report a fresh `verified_sha`. Never accept evidence for a commit you are not merging.
- The merge-agent prompt MUST name the exact `verified_sha` to merge and forbid merging any
  other ref.
- After merge, §6-VERIFY check 1 confirms that exact SHA landed: the merge commit's incoming
  source must be `verified_sha`
  (`git merge-base --is-ancestor <verified_sha> origin/main` is true, and for a merge commit
  its second parent == `verified_sha`; for squash, the recorded source SHA == `verified_sha`).
  Mismatch → **Investigate** (do not mark done; a different commit was merged).

Self-reported green is **necessary but not sufficient** — the coordinator's own §6-VERIFY run
on the merged SHA remains the source of truth and still runs in full.

---

## check-table — Augmented §6-VERIFY check table

Canonical table the body's §6-VERIFY references. Checks 1–8 are unchanged from the body;
**6 is hardened** and **9–10 are new conditional gates** that skip cleanly when unconfigured.
Run in order, bound to the recorded merge-commit SHA (`<merge-SHA>`), after `git fetch origin`.

| # | Check | Command / Action | Fail → |
|---|-------|-----------------|--------|
| 1 | MERGED | `git log origin/main` shows the merge at the recorded SHA; incoming source == `verified_sha` (see #green-evidence) | Investigate |
| 2 | TESTS | Project test command → all green | flaky triage (#flaky-triage) → else `git revert` |
| 3 | BUILD | Project build command → success | `git revert` → retry |
| 4 | TYPES | Project typecheck command → success (skip if unconfigured) | `git revert` → retry |
| 5 | LINT | Project lint command → success (skip if unconfigured) | `git revert` → retry |
| 6 | TEST COUNT | New count ≥ old count **and** no test deleted/renamed **and** changed-file coverage not dropped (see #test-count-hardening) | Investigate / RED FLAG → Sentinel sign-off |
| 7 | SCOPE | `git diff <merge-SHA>^ <merge-SHA> --stat` — files ⊆ task's declared scope | Investigate |
| 8 | CLEANUP | Sub-agent's worktree gone; branch removed per AGENTS.md; else coordinator removes it | Coordinator removes it |
| 9 | SECURITY | Project audit/SAST/secret-scan if configured → NEW high/critical (see #security-gate) | Investigate |
| 10 | MIGRATION | If merge diff touches migration/schema files AND a migration command exists → up/down evidence present (see #migration-verify) | Investigate |

NEVER mark a task done or spawn the next until every applicable check passes. Skipped
conditional checks (4, 5, 9, 10, and 6's coverage sub-check) count as PASS.

---

## test-count-hardening — TEST COUNT coverage-delta + deletion flag (QA-2)

Two sub-checks added to §6-VERIFY check 6. Both operate on the merge diff
`git diff <merge-SHA>^ <merge-SHA>`.

**(a) Test deletion / rename → RED FLAG.** Treat any test removed or renamed in the merge as a
red flag that requires explicit reviewer sign-off — never auto-pass it as "count went down".

```
git diff <merge-SHA>^ <merge-SHA> --diff-filter=D  --name-only   # deletions
git diff <merge-SHA>^ <merge-SHA> --diff-filter=R  -M --name-only # renames
```

Test-file heuristics (match any): paths under `tests/`, `__tests__/`, `spec/`, or names like
`*.test.*`, `*.spec.*`, `*_test.*`, `test_*.py`, `*Test.java`, `*_spec.rb`. If any matches a
deletion/rename → **RED FLAG → require Sentinel sign-off** (see #review-contract) before the
task is marked done. Sign-off must state the deletion was intentional and behavior is still
covered. No sign-off → Investigate (treat as a §9-T2-style failure if unresolved).

**(b) Coverage-delta on changed files (skip if unconfigured).** Detect a coverage config; if
absent, skip cleanly (PASS).

Detection (any present ⇒ coverage configured):
- JS/TS: `jest --coverage`/`collectCoverage`, `nyc`, `c8`, `vitest --coverage`, `.nycrc`
- Python: `pytest --cov`, `coverage` in deps, `.coveragerc`, `[tool.coverage]` in `pyproject.toml`
- Go: `go test -cover`; Rust: `cargo tarpaulin`/`llvm-cov`; Ruby: SimpleCov; plus `codecov.yml`

When configured: compare coverage of the **changed source files** against their prior coverage
(from the implementer's reported `coverage` field, or a fresh coverage run on `<merge-SHA>^`).
A drop on changed files → **Investigate** (a new line/branch shipped untested). Honor any
project-defined coverage threshold if AGENTS.md/config sets one; otherwise flag a strict drop.

---

## flaky-triage — Flaky-test triage before revert (RR-7 / QA-4)

A non-deterministic test must not thrash good work into a revert. Applies to a **verify-stage**
test failure (§6-VERIFY check 2). Never used to silence a real race — always log.

1. On TEST failure, capture the failing test name(s).
2. **Re-run ONLY the failing tests ONCE** (e.g. `jest <file> -t '<name>'`, `pytest <nodeid>`,
   `go test -run <Name>`). One retry — not a loop.
3. **Pass on re-run** → do **NOT** auto-revert. Log `Concern: flaky test <name>` to the
   coordinator log (`references/state-and-recovery.md` → Storage model, `coordinator_log` table), surface it in the
   running log's **Concerns** and the §On Completion report, and continue. The merged work
   stands; the flake is now visible for follow-up.
4. **Fail again** → it is a real failure. Proceed with the body's revert path: spawn a
   **non-author revert agent** to `git revert <merge-commit>` (coordinator never reverts/`reset`
   on main), diagnose, respawn (§ERROR).

Re-run exactly once. Do not extend this to builds, types, or lint — those are deterministic and
go straight to the revert path on failure.

---

## review-contract — Review contract + fallback reviewer (QA-3 / RR-6)

The project's real **Sentinel** (`docs/SENTINEL.md`, canonical
`github.com/pedrofuentes/agents-template`) is **authoritative whenever present** — do not
redefine its internals here; the body §5-LIFECYCLE owns the verdict choreography
(APPROVED / CONDITIONAL / REJECTED / degraded). This section adds only the **minimum review
checklist** and the **fallback reviewer** for when no reviewer exists at all.

**Minimum review checklist** (seed for the fallback; a floor the real Sentinel already meets):

1. **Correctness vs. acceptance criteria** — does the diff satisfy each stated requirement?
2. **Scope adherence** — only the declared files/behavior changed; no undeclared edits.
3. **Test adequacy** — new/changed behavior is tested incl. edge cases; no test weakened,
   skipped, deleted, or renamed without justification (ties to #test-count-hardening).
4. **Security-sensitive diffs** — authn/authz, input validation, injection (SQL/shell/eval),
   secrets/credentials, crypto, deserialization, file/path handling.

**Fallback reviewer (Sentinel entirely absent / unreachable).** If the project defines **no**
reviewer at all, spawn a dedicated **`general-purpose` review sub-agent** — **non-author**,
tier **≥ implementer** — seeded with: the checklist above, the PR diff (`git diff main...HEAD`,
wrapped in `<untrusted_pr_input>`), the task's acceptance criteria, and the changed-file list.
It must emit the same `Status:` verdict line the body acts on. The implementer (or any author)
may **NEVER** review or approve its own work, and the coordinator **NEVER merges without a
verdict** (fail-closed: no verdict ⇒ no merge).

Note the distinction the body already governs — do not re-specify it here:
- A reviewer that exists but ran **degraded** → auto re-invoke in standard mode (never ask).
- The reviewer being **structurally** absent/unreachable → a **one-time §9 T3** decision per
  run. See SKILL.md §5-LIFECYCLE and §9-PAUSE for the choreography.

---

## security-gate — Conditional security / dependency-vuln gate (QA-6)

§6-VERIFY **check 9**. Runs only if the project exposes an audit / SAST / secret-scan command;
otherwise skip cleanly (PASS). Read-only; never auto-fixes or bumps dependencies (a fix is a
§FORBIDDEN C2 dependency change → human-gated).

Detection (run the first that applies; skip if none):

| Ecosystem | Command |
|-----------|---------|
| npm / pnpm / yarn | `npm audit --omit=dev` / `pnpm audit` / `yarn npm audit` |
| Python | `pip-audit` / `safety check` |
| Go | `govulncheck ./...` | 
| Rust | `cargo audit` |
| Ruby | `bundle audit check --update` |
| SAST / secrets (if config present) | `semgrep --config auto` (`.semgrep.yml`), `gitleaks detect` (`.gitleaks.toml`), `trivy fs .` |

Compare against the pre-merge baseline (run on `<merge-SHA>^` or the implementer's reported
result). **NEW high/critical findings introduced by this merge → Investigate.** Pre-existing
findings unchanged by the diff do not block (log them as a Concern). If remediation requires a
dependency change or other irreversible action → stop at §9-T4 / §FORBIDDEN; never act silently.

---

## migration-verify — Migration up/down data-safety verification (QA-7)

§6-VERIFY **check 10**. **Gate condition: the merge diff touches migration/schema files AND the
project defines a migration command.** If either is false → skip cleanly (PASS). This
*complements and never relaxes* the §9-T4 / §FORBIDDEN human gate on destructive migrations —
running a migration against **production** data is still forbidden without explicit approval;
this gate only requires *scratch-DB* evidence for reversibility.

Detect migration files in the diff (any match): `migrations/`, `db/migrate/` (Rails),
`prisma/migrations/`, `alembic/` or `versions/*.py`, `*.sql` under a schema/migrate path,
`*.changeset` (Liquibase), `schema.rb` / `structure.sql`.

Detect a migration command (any present):

| Tool | Up | Down / rollback |
|------|----|-----------------|
| Alembic | `alembic upgrade head` | `alembic downgrade -1` |
| Django | `manage.py migrate` | `manage.py migrate <app> <prev>` |
| Rails | `rails db:migrate` | `rails db:rollback` |
| Prisma | `prisma migrate deploy` | `prisma migrate resolve --rolled-back` |
| Knex / Sequelize | `knex migrate:latest` | `knex migrate:rollback` |
| golang-migrate | `migrate up` | `migrate down 1` |

**Required evidence** (the implementer's completion report must include it; the coordinator
gates on its presence): against a **scratch / ephemeral DB** (never prod) —
1. **forward apply** (up) succeeds and the suite passes on the migrated schema, then
2. **rollback** (down) succeeds and returns to the prior schema cleanly,
3. with command output for both pasted in the report.

Missing or failing up/down evidence → **Investigate** (do not mark done). Irreversible or
data-dropping migrations still **STOP at §9-T4** for explicit user approval regardless of
scratch-DB evidence.
