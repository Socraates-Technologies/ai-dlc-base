# Bolt: [name]

**Status:** Planned | In Progress | Done | Blocked
**Bolt Type:** Feature | Bug | Hotfix | NFR
**Goal:** [One sentence — what this bolt delivers when complete]
**Start date:** YYYY-MM-DD
**Target date:** YYYY-MM-DD
**Completed date:** —
**Retrospective:** [ops/operate/retros/YYYY-MM-DD-<unix_timestamp>-[bolt-slug].md](../../operate/retros/YYYY-MM-DD-<unix_timestamp>-[bolt-slug].md)

---

## Units

| # | Unit | Status | Priority |
|---|---|---|---|
| 1 | [units/[unit-file].md](units/[unit-file].md) | Open | High |

---

## Execution Order

```
[Unit A] → [Unit B] → [Unit C]
                    ↘ [Unit D]
```

[Describe any parallelism or hard sequencing constraints between units.]

---

## Risk Assessment

> Populated by `process-onboarding-agent/skills/bolt-risk-assessment.md` before the first unit executes. For Bug and Hotfix bolts, record blast radius inline without running the full skill unless shared interfaces are affected.

| Risk | Blast radius | Rollback option | Mitigation |
|---|---|---|---|
| [description] | Isolated / Component / Service / Cross-service | [yes — how / no] | [what to do if it materialises] |

**Feature flag required:** Yes / No — [reason if yes]

---

## Definition of Done

- [ ] All units in this bolt have status Done
- [ ] Integration tests for all affected modules pass
- [ ] No regressions in modules outside this bolt's scope
- [ ] Retrospective file created and filed
- [ ] Backlog updated
- [ ] Dependency map current — ticked with its evidence: the commit that changed the dependency map, or "no new shared interface — <why>". A box ticked from memory ("updated at sign-off") is the failure this line exists to stop.
- [ ] The merge/closeout commit (or the same push) updates every status line the merge falsifies — each unit file's `Status:`, the bolt's unit-table rows, and this header. A Done bolt containing a Planned unit is the tell: the next session reads the docs as ground truth and gets "is this built?" wrong. *(The unit template's no-blank-DoD-box rule catches unevidenced work; this catches statuses that lag evidenced work.)*
