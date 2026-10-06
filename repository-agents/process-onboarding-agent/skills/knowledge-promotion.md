# Skill: Knowledge Promotion

**Purpose:** Evaluate each applied improvement to determine whether it is generic enough to benefit all AI-DLC projects, and if so, draft the exact change needed in the base repository so the engineer can raise a PR. Turns the passive "open a PR if you remember" instruction into a deliberate, structured ceremony that runs after every retro.

**Trigger:** Runs automatically as Step 5 of the Post-Retro Improvement Workflow, immediately after all approved improvements are applied. Can also be invoked directly by the engineer against a specific improvement file: "run knowledge promotion for [improvement]."

**Context:** This skill operates inside a project repository. It cannot modify the base repository directly. Its output is a drafted change — the exact file path and text to add or replace in the base repo — which the engineer carries into a PR against `ai-dlc-base`. The promotion decision and draft are recorded in the improvement file so the context is never lost.

---

## Step 0 — List the Improvements Still Open

An Open proposal decays exactly like a guard deferred to a code comment: a true statement of a hazard, sitting where the next person will not read it. So every run begins here:

1. **List every improvement whose Status is still Open**, oldest first, one line each with the retro or RCA that raised it and its date. Say when a proposal has stayed Open across two or more retros — that is the signal, more than its age.
2. **Put that list where work starts**, such as the backlog, not only in the retro that raised it. A proposal invisible there is one the next bolt rediscovers by shipping the defect.
3. **Treat Open as a decision not yet made.** Each proposal is either approved and applied, or Rejected with the reason recorded — "the cost is not worth it" is a fine reason. Leaving it Open is the only answer that does not close the loop.

This is prose; nothing can fail a build over an undecided proposal. It works only while the list is short and in the way.

---

## Step 1 — Identify the Improvements to Evaluate

If triggered by the Post-Retro Improvement Workflow, evaluate every improvement that was marked Applied in the current retro session, in order.

If invoked directly, ask the engineer:

> "Which improvement file should I evaluate for knowledge promotion?"

Read the improvement file in full before beginning the evaluation.

**Evaluate every generalisation clause, not only every improvement.** The steps below run once per Applied improvement, so a rule is classified, checked and swept only if it has a row of its own. A _"the general shape is…"_ sentence written **inside** another improvement's prose has no row, and so it is never evaluated at all — and any sweep of the artefacts that predate it never reads them against it. Before Step 1.5, read each Applied improvement for a sentence that generalises beyond that improvement, and give each one an entry of its own in this batch, stated from its mechanism rather than from the improvement it sits in. _(makerclub, 2026-09-28: a "a scan over sites floors its count" clause lived as a paragraph inside a dry-run rule; the sweep had one row per improvement, so three sites predating the clause stayed unread — two of them gates, one the check that proves no secret reaches a shipped binary.)_

---

## Step 1.5 — Ask Whether the Improvement Can Be a CHECK Rather Than a Sentence

Before classifying, ask of each improvement: **could this be enforced by something that runs, instead of something someone must remember?**

A retro's natural output is prose — a rule appended to the code standards, a bullet in a checklist. Prose is necessary (it carries the reasoning) and it is **not a mitigation**: it only fires if the next person reads it, recognises their situation in it, and acts. Where the invariant is mechanically decidable, the durable form is a **test, a lint rule, or a gate step**, with the prose kept as the explanation of why the check exists.

For each improvement, record one of:

- **Check landed** — name the test/lint rule/gate step, and confirm it has been seen to FAIL against the defect it prevents (a guard never observed red is not a guard).
- **Check possible, not landed** — say why (usually cost or scope), and raise it as an action so it is a decision rather than an omission.
- **Prose only — not mechanically decidable** — the honest answer for judgement-shaped lessons. Most process improvements land here, and that is fine; the point is to have asked.

A prose-only improvement whose invariant *was* mechanically decidable is the failure mode this step catches.

### A rule is not finished until you can name a compliant instance that achieves nothing

Ask this of the prose itself, including when the answer above was *prose only*: **try to complete the sentence "this rule could be followed completely and still achieve nothing if …".** If it can be completed, the rule is not finished — tighten it until it cannot. Compliance is checkable by reading and effectiveness is not, so a rule whose satisfaction condition is weaker than its purpose is followed by everyone and fails silently, in the direction that looks like success. (For example: "a date-dependent test names one case either side of the clock change" is satisfied in full by a test run in a timezone where both cases have the same offset.)

Ask it here rather than leaving it to the rule's author: the thing the author was holding while living the incident is the thing the sentence omits, and promotion is the first reading by someone who was not there.

**Ask it of the retro's and RCA's own recommendations too.** They are the most exposed, because they are written by whoever has just been the person who noticed. A rule whose trigger is an observation ("when X goes red twice, open a bug") cannot even be seen to be broken, since nothing records an observation not made. Bind such a duty to a moment that already happens — a template field, a session-start check, a Definition of Done line — or record it honestly as advice.

**Where the gate cannot reproduce the defect at all** — the test environment cannot exercise the mechanism (gestures, scrolling, a native or hardware path) — a behavioural test cannot see the class, and the obvious structural test is a pin on the one site that was fixed. It reads as coverage and protects only that site. The check instead:

- **scans every site from source**, parsing where a regex would under-report;
- **guards itself** — an "offenders list is empty" assertion passes trivially when the walk finds nothing, so also assert the sites the scan must reach and a floor on how many;
- **is probed on a site its author did not write**, since generalising beyond the fixed site is the only reason it exists.

---

## Step 2 — Classify: Generic or Project-Specific

For each improvement, determine whether it is **generic** (beneficial to all AI-DLC projects) or **project-specific** (only relevant to this project's stack, domain, or conventions).

**Use the target file as the primary classification signal.** Paths below are where the files live in the project: `{FRAMEWORK_ROOT}` is the framework root set in the Preliminary Step of `repository-agents/process-onboarding-agent/onboard.md` (e.g. `docs/process/intent-execution-framework`). `setup-guide.md` and `onboard.md` have no copy in the project; they are named at their base-repo path.

| Target file type | Classification |
|---|---|
| Any file in `{FRAMEWORK_ROOT}/skills/` | Generic — skills are copied verbatim into every project |
| Any `_template.md` file in `{FRAMEWORK_ROOT}/ops/` | Generic — templates are shared across all projects |
| `{FRAMEWORK_ROOT}/rules/engagement.md` | Generic — copied verbatim into every project |
| `repository-agents/process-onboarding-agent/setup-guide.md` | Generic — the shared framework specification |
| `repository-agents/process-onboarding-agent/onboard.md` | Generic — the shared onboarding protocol |
| `{FRAMEWORK_ROOT}/rules/prompt-quality-gate.md` | Likely generic — evaluate content |
| `{FRAMEWORK_ROOT}/rules/code-standards.md` | Project-specific — encodes the project's tech stack |
| `{FRAMEWORK_ROOT}/rules/security.md` | Project-specific — unless the finding addresses a universal pattern |
| `{FRAMEWORK_ROOT}/rules/architecture.md` | Project-specific — encodes project ADRs |
| `{FRAMEWORK_ROOT}/guidelines/domain-glossary.md` | Project-specific |
| `{FRAMEWORK_ROOT}/guidelines/edge-cases.md` | Project-specific |
| `{FRAMEWORK_ROOT}/guidelines/forbidden-zones.md` | Project-specific |
| `{FRAMEWORK_ROOT}/guidelines/entry-points.md` | Project-specific |
| `{FRAMEWORK_ROOT}/guidelines/acceptance-patterns.md` | Likely generic — evaluate content |
| `CLAUDE.md` / `.cursor/rules/project-rules.mdc` (or any `.mdc` under `.cursor/rules/`) / `.cursorrules` (legacy) / `copilot-instructions.md` | Project-specific |
| New file being created | Evaluate by content |

**For ambiguous cases, apply the content test:**

Ask: *If this improvement were applied to a completely different project — different tech stack, different domain, different team size — would it still be an improvement?*

- Yes, the process or ceremony would be better regardless of context → **Generic**
- No, it relies on knowledge of this project's domain, stack, or specific patterns → **Project-specific**

---

## Step 3 — Handle Project-Specific Improvements

If the improvement is project-specific:

Update the `Knowledge Promotion` field in the improvement file to:

```
**Knowledge promotion:** Project-specific — [one-sentence reason why it does not generalise]
```

State the outcome to the engineer:

> "[Improvement title] is project-specific ([reason]). No base repo change needed."

Run Step 3.5, then proceed to the next improvement.

---

## Step 3.5 — Sweep the Artefacts That Predate the Rule

A new rule shapes the work created after it; nothing re-reads the work created before. For every Applied improvement, project-specific or generic, before recording its status:

1. **Name the set of existing artefacts the rule would have shaped** had it existed earlier — e.g. bolts without backlog rows, units without an evidence section.
2. **Run the sweep**, mechanically where possible, and record the command and its count in the improvement file. Never write "none found" without the command that looked.
3. **Fix what it finds in the same change, or list each miss with an owner.**
4. If a sweep is impractical, **write the rule's effective date into the rule**, so a later reader knows earlier artefacts were never checked.

Generic improvements then continue to Step 4.

---

## Step 4 — Handle Generic Improvements

If the improvement is generic, determine the corresponding file in the base repository. The base repo's framework files live under `repository-agents/process-onboarding-agent/`. Some project files have no file of their own there — they are generated during onboarding from a section of `setup-guide.md` headed with the file's path — so their change goes into that section:

| Project file changed | Base repo file to change |
|---|---|
| `{FRAMEWORK_ROOT}/skills/[skill].md` | `repository-agents/process-onboarding-agent/skills/[skill].md` |
| `{FRAMEWORK_ROOT}/skills/mob-elab-prompts.md` or `review-checklist.md` | `repository-agents/process-onboarding-agent/setup-guide.md` — the section for that file (no standalone base file) |
| `{FRAMEWORK_ROOT}/ops/[path]/_template.md` | `repository-agents/process-onboarding-agent/ops/[path]/_template.md` |
| `{FRAMEWORK_ROOT}/rules/engagement.md` | `repository-agents/process-onboarding-agent/rules/engagement.md` |
| `{FRAMEWORK_ROOT}/rules/prompt-quality-gate.md` | `repository-agents/process-onboarding-agent/setup-guide.md` — the `rules/prompt-quality-gate.md` section (no standalone base file) |
| `{FRAMEWORK_ROOT}/guidelines/acceptance-patterns.md` | `repository-agents/process-onboarding-agent/setup-guide.md` — the `guidelines/acceptance-patterns.md` section |
| — (an improvement to onboarding itself) | `repository-agents/process-onboarding-agent/setup-guide.md` |
| — (an improvement to the onboarding protocol) | `repository-agents/process-onboarding-agent/onboard.md` |
| New skill file | New file at `repository-agents/process-onboarding-agent/skills/[skill].md` |
| New rule or guideline file | `repository-agents/process-onboarding-agent/setup-guide.md` — a new section headed with the file's path, and an entry in Step 1's folder tree |

**Open the base file at that path before drafting.** A draft written against a path that does not exist is a draft nobody can apply _(makerclub, 2026-09-23: two retros drafted against `process-onboarding-agent/…`, which this table used to name, before finding the files under `repository-agents/`)_.

Draft the exact change needed in the base repo. Be precise — draft the exact text to replace and the exact replacement, the same format used in the improvement file itself:

```
Base repo change draft
──────────────────────────────────────────────
File:    [path in base repo, e.g. repository-agents/process-onboarding-agent/skills/…]

Current text (or "New addition"):
[exact text that exists in the base repo file, or "N/A — new addition"]

Proposed replacement:
[exact text to replace it with]

Why this improves all AI-DLC projects:
[one or two sentences — the reason this generalises beyond this project]
──────────────────────────────────────────────
```

Present the draft to the engineer and ask:

> "This improvement appears to apply to all AI-DLC projects. I've drafted the change for the base repository above. Would you like to promote it?
> - **Yes** — I'll record the draft in the improvement file. Raise a PR against `ai-dlc-base` with this content, noting this project and retro as the source.
> - **Revise** — Adjust the draft before recording.
> - **No** — Record as declined with a reason."

Wait for the engineer's response.

---

## Step 5 — Record the Decision

**If promoted:**

Update the improvement file's `Knowledge Promotion` field:

```
**Knowledge promotion:** Promoted — PR to be raised against `ai-dlc-base`

### Base Repo Change Draft

**File:** [relative path in base repo]

**Current text:**
[exact current text or "N/A — new addition"]

**Proposed replacement:**
[exact proposed text]

**Why this generalises:**
[reason]

**Source:** [project name], retro [retro file link]
```

Remind the engineer:

> "The draft is recorded in the improvement file. When you raise the PR against `ai-dlc-base`, copy the draft above into the PR body and link back to this improvement file as the source."

**If declined:**

Update the improvement file's `Knowledge Promotion` field:

```
**Knowledge promotion:** Declined — [reason the engineer gave]
```

---

## Step 6 — Summary

After evaluating all improvements in the batch, present a summary:

```
Knowledge Promotion complete.

Improvements evaluated: [N]
  Generic — promoted:        [N]  [list titles]
  Generic — declined:        [N]  [list titles with reason]
  Project-specific:          [N]  [list titles]

[If any promoted:]
Next step: raise a PR against ai-dlc-base for each promoted improvement.
Include the draft text from the improvement file and link back to this project's retro as the source.
```
