# Lifecycle Stages — Agent-Driven SDLC up to the Irreversible Gate

Purpose: extend coordinator coverage across the SDLC — spec elaboration, design discovery,
risk classification, release-readiness, deploy/rollback prep, and multi-phase roadmap chaining —
while **automating only up to the irreversible gate**. Deploy, publish, migrate, and dependency
changes stay human-gated by §9 T4 / §FORBIDDEN. Never propose automating the irreversible action
itself; prepare it, then PAUSE.

Loaded by SKILL.md (Startup, §2-DELEGATE, §5-LIFECYCLE, §9-PAUSE, §ROADMAP, §On Completion) when a
run needs spec/design framing, risk-classified prep tasks, release/deploy readiness, or phase chaining.

**The gate line (memorize):** every stage below produces *reversible artifacts and exact commands*.
The moment an action is irreversible (tag, publish, deploy, run a migration, change a dependency),
STOP and hand the prepared commands to the user. See §9 T4 and §FORBIDDEN A–C.

Cross-links (do not duplicate — defer to the owning file):
- Retry/resume/structured state → `references/state-and-recovery.md#retry-fsm`
- Security/coverage/migration verification gates → `references/verification.md#security-gate`,
  `references/verification.md#migration-verify`
- Fleet safety / file-overlap preflight → `references/parallel-execution.md`

---

## LC-1 — Requirements / Spec Elaboration (ONE upfront round)

Runs **once**, during Startup, BEFORE the task list is presented for approval. This is **not** a
recurring cadence — you ask decision-critical ambiguities in a single batched round, then proceed.

Draft a compact spec (in the running log / DB, not a new repo file unless asked):

| Field | Content |
|-------|---------|
| Problem statement | One paragraph: what outcome the user wants and why |
| Assumptions | Each labeled `ASSUMPTION:` so the user can correct in their approval reply |
| Non-goals | What is explicitly out of scope this run |
| Acceptance criteria | Testable, per-feature; become §6-VERIFY expectations |
| User-visible behavior | CLI/API/UI surfaces touched; error → retry → success flows |
| Constraints | Perf, platform, compat, security, dependencies-frozen |
| Open questions | Only decision-critical ambiguities (see batching rule) |

**Batched-clarification rule (ONE round only):**
1. List every ambiguity where interpretations diverge enough to change the task list or tiering (§9 T1).
2. Ask **all of them together, once.** Do not drip-feed; do not re-ask later phases (that is a cadence — forbidden).
3. For non-critical gaps, record a labeled `ASSUMPTION:` and proceed — do **not** ask.
4. Fold answers + assumptions into acceptance criteria, then present the task list for approval.

If the user already supplied a complete plan/PRD, skip questions entirely and derive the spec from it.

---

## LC-2 — Design / Architecture Discovery Task

When ANY task hits a STANDARD trigger for **structural** reasons — shared interfaces/APIs, schema or data
migrations, auth/security, concurrency/atomicity, or platform-specific behavior — insert a
**first-class design-discovery task** that runs (and merges, if it produces docs) BEFORE the
implementation tasks it informs. Delegate it like any task (it never edits product code beyond docs).

Design-discovery agent output (a doc PR or a structured report):

- **Design constraints** — invariants the implementation must hold (ordering, idempotency, auth boundary).
- **Affected surfaces** — every module/handler/renderer/CLI/API path touched (feeds cross-path tiering).
- **Sequencing** — the order implementation tasks must run; what blocks what.
- **Risks** — failure modes, race windows, compat breaks; map each to a mitigation.
- **Rollback notes** — how each change is undone (informs LC-5 and §6-VERIFY revert).
- **Refined implementation tasks** — concrete, scoped, tiered tasks replacing the placeholder.

Discovery almost always reveals work not in the approved list. **New work → §9 T6**: surface it and
request approval; NEVER add tasks silently. Tier the design task itself TOP (it is structural by
definition). Reviewer ≥ implementer applies to its doc PR too.

---

## LC-6 — Lifecycle Risk Classification

Classify **every** task at planning time. High-risk classes get a dedicated **prep task** (reversible)
plus an explicit approval boundary. Store the class in the DB (extend `todos`; if a custom column/table
is needed use `CREATE TABLE IF NOT EXISTS` and fall back to a `RISK:<class>` tag in the description).

| Class | Examples | Required prep (reversible) | Gate |
|-------|----------|----------------------------|------|
| **code-only** | logic, refactor within scope | standard §6-VERIFY | none beyond verify |
| **docs** | README, changelog, design notes | standard §6-VERIFY | none |
| **dependency** | add/remove/upgrade a package | record exact change + lockfile diff plan | §9 T4 / §FORBIDDEN C2 — STOP |
| **security** | auth, crypto, secrets, input trust | threat notes; run audit/SAST if configured | §9 T4 if secrets/rotation; else verify |
| **config** | app config, feature flags | diff + intended values; note blast radius | verify; T4 if CI/secrets/.github (§FORBIDDEN B2) |
| **infra** | CI, containers, runners, IaC | dry-run plan, rollback path | §9 T4 — STOP before apply |
| **data-migration** | schema change, backfill | up/down dry-run + data-safety plan | §9 T4 — STOP; see `references/verification.md#migration-verify` |
| **public-API** | exported signatures, wire formats | compat/version-impact note, deprecation plan | §9 T4 — STOP before publish |

Prep tasks produce: threat/risk notes, a dry-run plan, a rollback plan, audit/verification commands,
and the explicit approval boundary. They are delegated and verified like normal tasks; they NEVER
perform the irreversible action.

---

## LC-4 — Release-Readiness & Versioning (after all merges)

Runs once the approved list is fully merged and §6-VERIFY-green. All steps are reversible **prep**;
PAUSE at the gate (§9 T4) before any tag/publish/deploy.

Checklist (delegate the doc/version edits; coordinator only reads + assembles):
1. **Detect the release process** — read AGENTS.md, `CONTRIBUTING`, `RELEASING*`, CI release workflows,
   `package.json`/`pyproject.toml`/`Cargo.toml` scripts, tag conventions, changelog format. Skip cleanly
   if the repo has no release process.
2. **Recommend a semver bump** — from the merged change set: breaking → major, additive → minor,
   fixes-only → patch. State the reasoning; do not assume.
3. **Verify changelog / release notes** — spawn an agent to draft/update `CHANGELOG`/release notes from
   merged commits (a normal verified, reversible task). CHANGELOG is sequential-only (§4).
4. **Version bump** — delegate as a normal task IF it is not a dependency change. **A bump that alters a
   dependency manifest's deps still hits §FORBIDDEN C2 → §9 T4 STOP.**
5. **Build artifacts** — run the project build / package step (read-only/dry-run where possible);
   capture output and resulting artifact paths.
6. **Summarize commits** — grouped, human-readable change summary bound to the release SHA.
7. **Prepare exact release commands** — the precise `git tag …`, `npm publish`/`gh release create …`,
   deploy invocations — **as text for the user to run or approve.**
8. **PAUSE (§9 T4)** with artifacts ready: bump recommendation, updated changelog, build output,
   commit summary, and the exact commands. Do not tag, publish, or deploy.

---

## LC-5 — Deploy / Smoke / Observability / Rollback PREP

For deployable projects, after LC-4 prep. Automate **read-only dry-runs only**; require explicit
approval before any real deploy or migration.

Prepare (do not execute the live deploy):
- **Discover deploy docs** — runbooks, `deploy/`, IaC, CI deploy jobs, environment config. Skip cleanly if absent.
- **Smoke-test commands** — the exact post-deploy health/smoke checks to run, with expected output.
- **Monitoring / log checks** — dashboards, log queries, metrics/alerts to watch and their healthy ranges.
- **Rollback command / path** — the precise revert/rollback invocation and where it points (prior version/tag).
- **Abort criteria** — concrete conditions that mean "stop and roll back" (error-rate threshold, failed
  smoke check, migration mismatch).
- **Dry-runs** — run only read-only previews (`--dry-run`, plan/diff outputs, `terraform plan`,
  `helm --dry-run`) and capture results.

**PAUSE (§9 T4 / §FORBIDDEN)** before any real deploy or migration. Hand the user the dry-run output
plus the smoke/monitoring/rollback/abort plan and the exact deploy command. Migrations additionally
require up/down evidence — see `references/verification.md#migration-verify`.

---

## OA-6 — Release-Prep Delegated Up to the Gate

Once the approved list is merged, the coordinator MAY spawn agents for **reversible** release-readiness
work without re-asking for each: docs, changelog, draft release notes, and a version bump as a normal
verified task. Treat release-prep as **in-scope follow-on**, not silent task addition — surface it as a
short "release-prep follow-on" list and respect §9 T6 (it is still new work the user should see).

Boundary: reversible prep proceeds; the irreversible publish/deploy/migration halts at §9 T4 with
artifacts ready (LC-4/LC-5). A version bump that is a **dependency change** still hits §FORBIDDEN C2 →
STOP. Reviewer ≥ implementer and §6-VERIFY apply to every release-prep PR.

---

## OA-3 — §ROADMAP Multi-Phase Chaining Loop

When the plan defines **multiple phases** (a roadmap), execute them as a loop — one approval per phase's
task list, never a blanket approval for the whole roadmap.

Loop:
1. **Plan phase N** — apply LC-1 (spec, once for the run) and LC-2/LC-6 (per phase) to phase N; produce
   its numbered, tiered task list.
2. **Approve phase N's list ONLY** — present and get user approval for *this phase's* tasks. Do not
   present later phases' tasks yet.
3. **Execute** — run the phase via the normal choreography (§2-DELEGATE, §4-PARALLEL, §5-LIFECYCLE).
4. **Verify** — every task in the phase passes all of §6-VERIFY.
5. **On-Completion sweep** — run the §On Completion stale-worktree sweep so no environment survives the phase.
6. **Carry context forward** — write a **phase-to-phase summary** (delivered behavior, new interfaces +
   signatures, schema/config impact, landmines, deferred items) into the DB (§8-PERSIST); seed the next
   phase's task prompts with it (like §7-CHAIN across phases).
7. **Advance or stop** — if the roadmap defines a further phase, return to step 1 for phase N+1. Stop at
   roadmap end, or immediately on any always-stop trigger (§9 T1–T6, §FORBIDDEN, §ERROR).

Rules: NEVER auto-approve a later phase from an earlier phase's approval. NEVER collapse phases to skip a
gate. The per-phase approval is the only added pause — there is **no** periodic cadence between tasks
within a phase (§9 cadence = none).
