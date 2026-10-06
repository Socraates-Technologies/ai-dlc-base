# Skill: Architecture Review

**Purpose:** Periodically read the whole codebase for anti-pattern recurrence, duplication, architectural drift, and complexity creep, and produce ranked, actionable recommendations — so the code is refactored deliberately rather than sliding into a mess one change at a time. The skill **recommends; it never refactors.** The engineer ratifies findings, and a fix graduates into its own unit, NFR bolt, or Remediation Bolt.

**Trigger:** Engineer-initiated ("run an architecture review"), or monthly alongside the dependency audit. Also worth running after any bolt that added a large surface. Because it is read-only analysis, it can run on a cheaper or faster model.

**This skill does not:**
- Change code. Findings and recommendations only.
- Replace the per-change `review-checklist.md` (which gates one diff) or a dedicated security review. This is the whole-codebase, over-time view. Where a finding is security-shaped, flag it and defer the depth to the security review.
- Create backlog items silently. It proposes them; the engineer ratifies.

---

## Step 1 — Anchor in the Project's Own Rules

Read these before scanning, so findings are measured against what this project has agreed rather than against generic style preference:

- `process-onboarding-agent/rules/code-standards.md` — every anti-pattern listed there was a real failure; checking none has crept back is the review's first job.
- `process-onboarding-agent/rules/architecture.md` — the ADRs. Drift from an ADR is a finding, including code still honouring a **superseded** ADR.
- `process-onboarding-agent/rules/security.md` and the master rule file's hard stops.
- `process-onboarding-agent/guidelines/domain-glossary.md` — code using a term the glossary rejects is a finding.
- `process-onboarding-agent/guidelines/edge-cases.md` — code on a path an edge case covers must still implement the agreed handling.
- The **previous** architecture review in `process-onboarding-agent/ops/operate/improvements/`, if any — update it rather than duplicate it; the trend is the point.

Agree the scope with the engineer (which source directories, which languages). Exclude dependency directories, build output, generated files, and agent worktrees. State the scope in the report.

---

## Step 2 — Scan Across the Dimensions

Work each dimension concretely — search the code and cite `file:line` — not impressionistically:

1. **Load-bearing invariants first.** The project's hard stops and security rules: authentication on every endpoint that returns protected data, tenancy derived from the credential, clients reaching data only through the API, no secrets in client-shipped code or logs. A breach here is Critical.
2. **Duplication.** Copy-pasted blocks and parallel implementations of one concern that should share a module. Look for repeated query shapes, repeated auth or validation boilerplate, and duplicated UI components a shared one already covers.
3. **Anti-pattern recurrence.** Walk `code-standards.md` item by item and check each is still honoured. A recurrence outranks a novel smell of the same size, because the team already paid for that lesson once.
4. **Architectural drift and layering.** Business logic in request handlers or UI instead of the domain layer; god-modules that own many concerns; a lower layer reaching up into a higher one; inconsistent response shapes or error handling across endpoints.
5. **Complexity hotspots.** Oversized files and functions, deep nesting, high fan-in or fan-out. Rank by likelihood of a future bug × change frequency (`git log` churn), not raw size — a large stable file matters less than a medium file changed every week.
6. **Dead weight.** Commented-out code, TODOs without a backlog item, unused exports, stale flags, and throwaway debug or probe code that should not ship.
7. **Test gaps on critical paths.** Authentication, tenancy boundaries, data-integrity paths, and irreversible operations (deletes, payments, sends). A gap on a load-bearing path is High.

---

## Step 3 — Rank and Recommend

For each finding record: **severity**, **file:line**, a one-line statement of the defect, **why it matters** (the failure it invites), a concrete **recommendation**, and a rough **effort** (S / M / L). Sort most severe first.

| Severity | Criteria |
|---|---|
| **Critical** | A load-bearing invariant is broken or nearly so — an unauthenticated data path, a secret exposure, a client bypassing the API. Escalate immediately |
| **High** | Real correctness or maintainability risk that will bite soon — spreading duplication, a recurred anti-pattern, an untested critical path |
| **Medium** | Drift or complexity worth a planned refactor |
| **Low** | Hygiene and consistency; batch opportunistically |

State plainly what was **not** covered — "scanned all of `lib/`; sampled the six largest screens" — rather than implying a complete review. A partial review that says so is worth more than a "comprehensive" one that was not.

---

## Step 4 — Write the Report and Propose the Work

Write the report to `process-onboarding-agent/ops/operate/improvements/YYYY-MM-DD-architecture-review.md`:

```
# Architecture Review — YYYY-MM-DD

Scope: [directories and languages scanned; what was sampled; what was excluded]

## Summary
[2–4 lines: overall health, the single most important finding, the trend since the last review]

## Findings
| # | Severity | Location | Defect | Why it matters | Recommendation | Effort |
|---|---|---|---|---|---|---|

## Trend
[Each finding from the previous review: fixed / still open / worse — and which findings are new]

## Recommended next actions
[The top 1–3 to graduate into engineering work, each marked high-risk or safe cleanup]
```

Then present the top findings and ask:

> "Which of these should become backlog items? You can say 'all Critical and High', list numbers, or 'none' to keep the report only."

A refactor that touches authentication, tenancy, data integrity, or the deploy path is **high-risk** — say so explicitly, and route it through a bolt with a signed-off risk assessment.

---

## Guardrails

- **Read-only.** Never edit product code during a review, and never print a secret value found while scanning.
- **Measure against the project's rules, not taste.** When a review suggests a *new* rule, route it through the retro and improvement loop rather than asserting it in the report.
- **Be honest about coverage.** Name what was scanned, sampled, and skipped.
