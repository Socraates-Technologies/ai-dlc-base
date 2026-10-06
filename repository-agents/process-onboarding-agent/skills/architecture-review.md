# Skill: Architecture & Code-Health Review

**Purpose:** Periodically read the whole codebase for anti-patterns, duplication, architectural drift and complexity creep, and produce **ranked, actionable recommendations** — so the code is refactored deliberately instead of sliding into a mess one reasonable-looking change at a time. The skill **recommends; it does not refactor.** The engineer ratifies each recommendation, and a ratified refactor becomes its own engineering, NFR or Remediation bolt.

**Trigger:** Engineer-initiated ("run an architecture review", "how healthy is the code?"), or monthly alongside the dependency audit — when the AI prompts for the `Next dependency audit` date in the master rule file Section 9, it offers this review in the same breath. Also worth running after any bolt that added a large new surface.

Because the review is read-only analysis, it can run on a cheaper or faster model than code generation.

**This skill does NOT:**

- Change code. Findings and recommendations only. A fix is a separate, engineer-ratified unit or bolt.
- Duplicate `skills/review-checklist.md`, which gates **one change**. This is the **whole-codebase, over-time** view.
- Replace a security review. Where a finding is security-shaped, flag it and defer the depth to the project's security rules and review.
- Open backlog items silently. It **proposes** them; the engineer ratifies.

---

## Step 1 — Anchor in the Project's Own Rules (read before scanning)

Findings are measured against what **this project has agreed**, not against generic style opinion. Read, in this order:

- `rules/code-standards.md` — the **retro-grown anti-pattern catalogue**. Every item there was a real failure, so the review's first job is to check that none has crept back.
- `rules/architecture.md` — the ADRs. Drift from an ADR is a finding, and so is code still honouring a **superseded** ADR.
- `rules/security.md` and the hard stops in the master rule file Section 3.
- `guidelines/domain-glossary.md` — code using a synonym the glossary rejects is a finding.
- `guidelines/edge-cases.md` — code on a path an edge case covers must still implement the agreed handling.
- The **previous** architecture review report, if one exists. **Update the trend; do not start over.** Whether earlier findings were fixed, ignored or got worse is the most useful thing the review produces.

Then fix the scope and write it down: which source directories are in, and what is excluded (dependency folders, build output, generated code, vendored code, any agent worktree directories). The report states the scope.

---

## Step 2 — Scan Across the Dimensions

Work each dimension concretely with search and file reads, and cite `file:line` for every finding. An impression is not a finding.

1. **Load-bearing invariants first — the highest stakes.** List the project's invariants from the hard stops and the ADRs (who may read what, which component may reach which store, where secrets and personal data may and may not appear) and check each one against the code as it is now. Has any route grown an unauthenticated read? Has a client started reaching a data store directly instead of through the API? Has sensitive data reached a log line? A broken invariant is Critical.
2. **Duplication.** Copy-pasted blocks; parallel implementations of one concern that should share a module; repeated query shapes or auth, validation or rate-limit boilerplate; UI components duplicated where a shared one exists. Search for the existing pattern before recommending a new abstraction.
3. **Anti-pattern recurrence.** Walk `rules/code-standards.md` item by item. A recurrence outranks a novel smell of the same size, because it means a rule the team already paid for has stopped holding.
4. **Architectural drift and layering.** Business logic leaking into handlers or UI; god-modules that own several concerns; a low-level module reaching up into route or UI concerns; inconsistent response shapes or error handling across endpoints (in particular, some paths leaking internals that others mask).
5. **Complexity hotspots.** Oversized files and functions, deep nesting, high fan-in or fan-out. Rank by **likelihood of a future bug × change frequency**, not raw size — a large stable file matters less than a medium one that changes every week. Version-control history gives the change frequency.
6. **Dead weight.** Dead or commented-out code, TODOs with no backlog item, unused exports, stale feature flags, leftover probe or debug UI.
7. **Test gaps on critical paths.** Is every load-bearing invariant from dimension 1, and every irreversible operation (deletes, payments, outbound messages), exercised by a test that would fail if the rule were removed? A gap on a load-bearing path is High.

---

## Step 3 — Rank and Recommend

For each finding record:

| Field | Content |
|---|---|
| Severity | Critical / High / Medium / Low |
| Location | `file:line` |
| Defect | One line |
| Why it matters | The failure it invites |
| Recommendation | Concrete, not "consider refactoring" |
| Effort | S / M / L |

Rank most severe first.

**Severity guide:**

- **Critical** — a load-bearing invariant is broken, or one change away from it. Escalate to the engineer immediately; do not wait for the report.
- **High** — real correctness or maintainability risk that will bite soon: spreading duplication, a recurred anti-pattern, an untested critical path.
- **Medium** — drift or complexity worth a planned refactor.
- **Low** — hygiene and consistency; batch opportunistically.

**Be honest about coverage.** State what was read in full and what was sampled ("read all of the service layer; sampled the six largest UI screens"). A partial review that says so is worth more than a "comprehensive" one that was not. Never cap findings silently.

---

## Step 4 — Write the Report and Propose the Work

Write the report to `ops/operate/architecture-review-YYYY-MM-DD.md` under the framework root:

```
Architecture review — YYYY-MM-DD

Scope:    [directories read in full] · [directories sampled, and how] · [exclusions]

Summary:  [2–4 lines: overall health, the single most important thing, the trend since the last review]

Findings: [the ranked table from Step 3]

Trend since [previous report date, or "first review"]:
  Fixed:      [ids]
  Still open: [ids]
  Worse:      [ids]
  New:        [ids]

Recommended next actions:
  1. [finding] → [proposed bolt type] · [High-risk? yes/no, and why]
  2. ...
```

Then **propose** — do not create — the top one to three items as candidate backlog entries for the engineer to ratify. A refactor that touches authentication, authorisation, data integrity, migrations or the deploy path is **high-risk**: say so explicitly, because it needs a signed-off `bolt-risk-assessment` before any unit executes.

---

## Step 5 — Guardrails

- **Read-only.** Never edit product code during a review. Never print a secret value, even one found in the code — report its location only.
- **Recommend, never mutate.** The engineer ratifies every refactor.
- **Measure against the project's rules, not taste.** When the review suggests a *new* rule, route it through the retro and `knowledge-promotion` loop instead of asserting it in the report.
- **Say what was not scanned.**
