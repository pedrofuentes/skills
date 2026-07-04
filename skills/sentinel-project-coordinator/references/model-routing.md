# Model Routing — refinements ON TOP of the STANDARD-default policy

Cost/latency refinements that **never weaken** the §2-DELEGATE tier policy. The default tier
stays **STANDARD**; the **ALL-FIVE LIGHT gate** and **reviewer tier ≥ implementer tier** are
inviolable. Nothing here permits LIGHT-by-default or a FAST-tier model for implementation or
review.

> Loaded by SKILL.md §2-DELEGATE (live routing + rejection severity + reviewer diversity),
> §5-LIFECYCLE (FAST tier for operational agents), and §8-PERSIST (where routing state lives).

**Tier names and the current model mapping are defined ONLY in SKILL.md §2-DELEGATE** (tier
order FAST < LIGHT < STANDARD < TOP; TOP is availability-resolved once per run at Startup).
This file never names models. **FAST** is allowed **only** for the operational ops
whitelisted in [§ FAST tier](#mr-4-fast-tier-for-operational-agents-only).

---

## MR-2: Intra-session live routing (a learning loop)

When a task **escalates after a lower-tier failure** (retry ladder bump, or the Sentinel-rejection
override), the lower tier has been empirically disproven for that area. Record it and route future
sibling tasks straight to **TOP** — don't re-pay for the same failure.

**On every escalation, record a live-routing entry in state** (the `coordinator_routing` table —
see `references/state-and-recovery.md` Storage model for the schema; do not duplicate it here).
Capture at minimum:

| Field | Value |
|-------|-------|
| `module` | path/glob the failing task touched (e.g. `src/auth/**`) |
| `pattern` | concern that broke (e.g. `TOCTOU`, `cross-path-propagation`, `migration`) |
| `failed_tier` | the tier that failed |
| `routed_tier` | `TOP` |
| `reason` | one line + originating task id |

**Pre-spawn live-routing check** (run inside the §2-DELEGATE pre-spawn complexity check, before
applying the ALL-FIVE LIGHT gate):

1. Query the live-routing table.
2. If the new task's files match a recorded `module` **OR** its work matches a recorded
   `pattern` → **force TOP**, regardless of what the ALL-FIVE gate would have said.
3. Note the override in the spawn log (`live-routing: <module/pattern>`).

This is a **session-scoped** loop that complements — never replaces — the end-of-run
tier-accuracy retro. Live routing reacts within the run; the retro adjusts assignment heuristics
for next time. Entries do not persist past the session.

---

## MR-3: Severity-calibrated Sentinel-rejection escalation

The §2-DELEGATE override jumps **every** REJECTED PR to TOP. Refine: a trivial rejection
doesn't need a model change — only substantive defects skip the ladder. The coordinator judges
severity from the **🔴 findings in the rejection report**.

| Rejection severity (from 🔴 findings) | Examples | Routing |
|---------------------------------------|----------|---------|
| **Trivial** — mechanical, no design judgment | lint/format failure, syntax error, missing import, unused var, typo, formatting | **Retry at the SAME tier** with the fix delta + report id |
| **Substantive** — correctness/design risk | logic error, wrong/edge-case behavior, architecture/contract violation, race/TOCTOU/concurrency, security/auth/input-trust, data-safety/migration | **Skip the ladder → TOP** (the existing override) |

Rubric:

- **ALL** 🔴 findings are trivial → trivial path (same tier).
- **ANY** 🔴 finding is substantive → substantive path (TOP). Mixed = substantive.
- Cannot classify a finding confidently → treat as **substantive** (fail safe upward).
- This refines *which tier* the respawn uses; it changes nothing else. Still fix **🔴 only**
  (never 🟡/🟢 in-PR), still carry the prior **Report ID + `git diff <prev-SHA>..HEAD`**, still
  **max 5 rejection cycles → escalate to user**, and a substantive rejection still feeds MR-2
  live routing.

---

## MR-4: Fast tier for operational agents ONLY

A FAST-tier model is permitted **only** for agents that neither implement nor review — purely
mechanical, no design judgment, no code authored against acceptance criteria. The FAST-tier
ban for **implementation** and the **reviewer ≥ implementer** rule are unchanged.

**ALLOWED on FAST tier** (operational, deterministic):

- Conflict-**free** merge of an already-reviewed SHA (no edits, no resolution).
- Worktree / branch teardown and the §On-Completion sweep.
- The isolation smoke probe (PC-4, `references/parallel-execution.md`).
- Read-only mechanical reporting (e.g. `git worktree list`, `git log` collection).

**FORBIDDEN on FAST tier** (must be tier ≥ implementer; never FAST):

- Any **implementer** (TDD / code authoring) — FAST tier is permanently banned here.
- **Sentinel / any reviewer** — governed by reviewer ≥ implementer + MR-5.
- A **conflict-resolving rebase** (new branch + cherry-pick) — it edits code and **re-invokes
  Sentinel**, so it is NOT an operational op. It must run at **tier ≥ the implementer**; see
  `references/parallel-execution.md` for the rebaser tiering rule. ⚠ This is the one boundary
  agents most often get wrong: "merge agent" ≠ "FAST tier" the moment a conflict appears.
- Anything that re-invokes Sentinel or produces a diff that will be reviewed.

Decision rule: *does this agent edit code, or cause a (re-)review?* If **yes** → not FAST tier.
If **no** (pure plumbing) → FAST tier allowed.

---

## MR-5: Reviewer model diversity

To reduce shared blind spots between author and reviewer, the **Sentinel reviewer** (or any
fallback reviewer) **SHOULD** use a **different model line** from the implementer — at
**equal or higher tier** (never lower; reviewer ≥ implementer always wins). E.g., a TOP-tier
model reviewing STANDARD-tier work satisfies this.

- **Gated on availability.** If no equal/higher-tier model from a different line is available,
  use the same model at tier ≥ implementer. Diversity is a preference, never a reason to drop
  below the implementer's tier.
- Applies to Sentinel and to any fallback reviewer the coordinator spawns.
- Record the reviewer model alongside the Sentinel Report ID in state (§8-PERSIST) so the
  tier-accuracy retro can see whether cross-family review caught more.
