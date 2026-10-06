# Id Reservations

> **How this file is used:** ids are never allocated at planning time. Work is written under placeholders (`U-TBD`, `B-TBD`, `I-TBD`, `ADR-TBD`, `EC-TBD`, a migration as `NNNN_name`). In the push cycle — fetch, rebase, read each table's **Next free** marker at that commit, substitute every placeholder, add the claim rows and advance the markers in the same commit, gate, push — the push is the only lock the repository has, so a lost race is a rejected push: re-stamp and retry. Never renumber after a successful push. Resolve a conflict in this file hunk by hunk, keeping upstream's rows as they are and adding only the rows you wrote. See `guidelines/dev-setup.md` and the master rule file Section 6.

## Units

Next free: **U-001**

| Id | Artifact | Claimed in commit | Date |
|---|---|---|---|
| — | — | — | — |

## Bolts

Next free: **B-001**

| Id | Artifact | Claimed in commit | Date |
|---|---|---|---|
| — | — | — | — |

## Intents

Next free: **I-001**

| Id | Artifact | Claimed in commit | Date |
|---|---|---|---|
| — | — | — | — |

## ADRs

Next free: **ADR-001**

| Id | Artifact | Claimed in commit | Date |
|---|---|---|---|
| — | — | — | — |

## Edge cases

Next free: **EC-001**

| Id | Artifact | Claimed in commit | Date |
|---|---|---|---|
| — | — | — | — |

## Migrations

Next free: **0001**

| Number | Migration | Claimed in commit | Date |
|---|---|---|---|
| — | — | — | — |
