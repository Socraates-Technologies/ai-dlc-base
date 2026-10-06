# Skill: Bug Bolt

**Purpose:** A lightweight bolt workflow for fixing a specific, reproducible bug. Replaces the full mob elaboration ceremony with a focused intake, recurrence check, and a single unit — while preserving all quality gates (prompt gate, pre-generation checks, review checklist) and the post-fix retro.

**Trigger:** Engineer says "fix a bug in X", "there's a bug in X", or "bug: [description]". Routed from Section 6 of the master rule file. Do not run a full mob elaboration when this trigger is detected — load this skill and follow its steps instead.

**Requires:** A fully onboarded AI-DLC project. The master rule file, quality gate (`rules/prompt-quality-gate.md`), review checklist (`skills/review-checklist.md`), unit and bolt templates, and the `ops/` folder structure must all be in place. This skill is a bolt-type variant that operates within the framework — it is not a standalone bug-tracking tool. If AI-DLC is not yet installed, start with `process-onboarding-agent/onboard.md` first.

**What this skill does not replace:** intent files, mob elaboration (for new features), UAT, the retro, or the Post-Retro Improvement Workflow. A bug bolt is strictly scoped to fixing a known defect — use the standard feature flow for any work that adds or changes behavior.

---

## Step 1 — Bug Intake

Ask the engineer four questions in one message:

> "To scope this fix accurately, please tell me:
> 1. **What is the bug?** Describe the symptom — what the user or system experiences.
> 2. **Where does it occur?** Name the component, file, function, or user flow.
> 3. **How is it reproduced?** List the steps or conditions that trigger it.
> 4. **What is the expected behavior?** What should happen instead?
> 5. **How does it vary?** Slower or faster, first time or every time, one direction or both, more data or less."

Question 5 matters most for reports of feel (lag, jank, a pause, a jump), which always happen and are diagnosed by how they vary: a cost gets worse when the action is slower, a discontinuity gets better. Ask it before writing any diagnosis.

Record all five answers before proceeding.

**Take a baseline of the whole gate before the fix.** Run the full test suite once on the unmodified main branch and record the totals, not only the named failures. A reporter describes what they noticed; the tree may hold other reds, and the closing claim should be a delta against this baseline rather than "the named tests now pass".

**A red the baseline finds that is not this bug gets a backlog row now.** Name it and file it (or name the existing row) before proceeding. "Pre-existing" says whose it is not, never whose it is — recorded only in the unit, it belongs to nobody.

---

## Step 2 — Recurrence Check

First grep the artefact's own names — the failing file, the test title, the function under suspicion — across the backlog, `ops/operate/rca/`, retros and incidents. A row is written in its raiser's vocabulary and a description search uses yours; the artefact's name is the one string both used. If an RCA names the artefact, read its Recommendations before writing the fix — it may already say this is one instance of a class and what fixes the class.

Then search the project's retro files and incident files for similar descriptions.

- If a similar pattern appears in more than one retro or incident: note it as **recurring**. Root Cause Analysis is mandatory after the fix.
- If no similar pattern is found: proceed without the RCA flag.

State the recurrence finding to the engineer before continuing.

---

## Step 3 — Scope the Fix

Identify the minimum set of changes needed to fix the bug without side effects. Ask if unsure:

> "Is this fix isolated to [component], or are there other places where the same logic runs?"

If multiple components are affected, each becomes a separate unit.

**Ask it again as a control when the diagnosis is a shape.** Before writing the fix, name a sibling site with the same shape and state both branches first: the sibling fails too, so the shape is confirmed; the sibling works, so something protects it you have not found, and the fix waits. Stated after the answer, the question only confirms.

**Taking the cheap fix over the structural one.** When you knowingly choose a local fix over a structural one, write its expiry condition into the unit — and phrase it against the cause left in place, not the symptom just fixed. "If the flash returns" never fires, because the fix makes that one symptom unlikely; "if anything else traceable to [the mechanism] appears" does.

**At the third unit, ask whether this is still a bug bolt.** Each added unit can be individually justified while the total becomes a redesign. Write one line in the bolt file: is this still a defect being fixed, or a change being designed? "Still a bug bolt" is a legitimate answer. If the answer is no, the next unit goes in a new feature or NFR bolt — recording the answer and carrying on is not an option.

---

## Step 4 — Create the Unit

Create a single unit file using the standard unit template with these constraints:

**Context:** What is broken and why — a precise technical statement, not a vague description.

**Acceptance Criteria:**
```
Given [the conditions that trigger the bug]
When [the action that exposes it]
Then [the system behaves correctly — the expected behavior from Step 1]

Given [the same triggering conditions]
When [the fix is NOT present — regression guard]
Then [the broken behavior is detectable — confirm a test would catch a regression]
```

At least one unhappy-path AC must be included.

**Scope:**
- In scope: the specific function, query, component, or flow causing the bug
- Out of scope: refactoring unrelated code, adding features, "while we're here" changes

**Pre-generation checks:** Grep for the affected function or component before generating. Confirm no duplicate fix already exists.

**Fix one, grep all.** Grep the codebase for the defect's *shape*, not only the reported site, and record every hit in the unit with its disposition — fixed, or safe and why. The list, not "I checked". Each "safe" argues the risk: state the condition under which that site would fail and why it cannot arise. "Same shape as the one I fixed" is not a disposition, because sites that look alike can differ in exactly the operator that matters.

**Observability:** Define a log entry or metric that confirms the bug no longer occurs in production.

---

## Step 5 — Risk Assessment

Before execution, ask one question:

> "Does this fix touch any shared component, database schema, or API contract used by other intents or services?"

If yes: run `process-onboarding-agent/skills/bolt-risk-assessment.md` and record the blast radius in the bolt file.
If no: record "Blast radius: isolated to [component]. No shared interfaces affected."

---

## Step 6 — Execute and Verify

Execute the unit. Review output using `process-onboarding-agent/skills/review-checklist.md`.

After output is accepted, confirm the fix:

> "Can you verify the bug is resolved against the reproduction steps from Step 1?"

Do not close the unit until the engineer confirms the fix is verified.

**And when an attempt is refuted, strike it everywhere it was stated as the cause.** A fix that fails verification is recorded as refuted in the bolt file — correctly. But the same hypothesis had also been written, as fact, into the unit's notes, into a code comment above the function it blamed, and into an edge case's prose. Each outlived its refutation, and the next attempt read three confident statements of a cause the bolt file had already withdrawn. The bolt entry that records a refutation lists every other place the hypothesis is stated and corrects them **in the same commit** — a code comment becomes *"a real latent defect, not this one"*, a notes paragraph is struck or prefixed with the date it was refuted, an edge case is re-pointed at what it actually contains. A refuted cause left standing anywhere is a pointer to the wrong place, and the reader who follows it will be the one under the most time pressure.

**A user-reported symptom is fixed only when it is verified against that exact symptom on a real device or environment, and the observation is recorded in the unit.** A green local gate plus a plausible mechanism is "shipped", not "fixed". A unit test can prove a prop is set or a handler fires; it cannot prove the behaviour on real hardware (that a native player releases, that an embedded third-party page initialises, that a native gesture or a real scroll event lands) — the gate asserts around these, not through them, and a suite green for the parts it can reach says nothing about the part it cannot. Write the observation into the unit — date, build or commit, what was done, what was seen — not into a chat message. A drag-to-reorder once reached the trunk behind sixteen green tests having never once worked on a phone; the observation that would have caught it took two minutes.

**Confirm where an error's text/code originates before attributing a cause.** When the symptom has specific wording or an error code, first establish whether it comes from app code, a library, or an embedded third-party page — do not anchor on the first plausible mechanism. *(Maestro, 2026-07-16: a fix targeted the app's native video player when the "Error 153" text on screen was the embedded YouTube player's own overlay. The fix shipped, and the error stayed.)*

Evidence rules for the verification:

- **Falsify the instrument before trusting it.** Run the reproduction with the fix reverted: if it is green, it cannot validate the fix. If it reddens but names different cases on each run, it reproduces without attributing and cannot validate a fix either.
- **A harness you built is a model.** When the defect will not reproduce on demand and you construct one (a stall, a frozen clock, a stubbed network), write one line naming what it does not model, and run at least one condition it cannot produce before writing "fixed".
- **Read the suite total before and after the fix, not the target's colour.** Green on the target and red elsewhere is a fix plus a new defect, and the new one is yours.
- **A run with no summary line ran nothing.** A runner or script that reports a result must check the test summary exists first; an empty run is invalid, re-run, and recorded — never counted as green.
- **A refuted repair is reverted, not softened.** If the fix fails the same way it did before, the diagnosis was wrong: revert the change, record the attempt in the bolt file, and do not keep it with a weaker comment because it is defensible on its own terms.
- **A determinism claim needs a second varied condition.** Before writing "fails only under X", vary one more thing (another host, ordering, worker count) — or write what was observed: "reproduced 2/2 in a full run, 0/3 in isolation, not seen elsewhere".

---

## Step 7 — Close

1. Mark unit Done. Mark bolt Done. Update the backlog.
2. Create a retro file and run the Post-Retro Improvement Workflow. Every action the retro hands on gets a backlog row in the same commit; a retro's actions table is not reopened by anything, the backlog is.
3. If the bug was flagged as **recurring** in Step 2: read `process-onboarding-agent/skills/root-cause-analysis.md` and run it now. Do not skip.
   Record that the RCA is owed and why, never what it will find. A conclusion written into the bolt, backlog or commit message before the RCA runs takes away its independence.
4. If the bug caused a production impact: create or update an incident file at `ops/operate/incidents/`.
