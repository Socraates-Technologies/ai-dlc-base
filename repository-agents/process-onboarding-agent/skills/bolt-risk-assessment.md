# Skill: Bolt Risk Assessment

**Purpose:** Actively interrogate a planned bolt for blast radius, cross-unit sequencing risks, rollback feasibility, and feature flag requirements before the first unit executes. Replaces the passive "Risks and Assumptions" table in the bolt file with structured, AI-driven findings that the engineer signs off on.

**Trigger:** Runs after elaboration sign-off and before the first unit in a bolt is executed. Invoked by the mob elaboration protocol automatically after the sign-off summary is confirmed, or directly by the engineer with "run risk assessment for bolt [name]."

**Never skip for mature projects.** For projects with existing code, this assessment is mandatory — no unit executes without a signed-off risk assessment in the bolt file. For fresh projects with no existing modules affected, the assessment may be brief but must still be completed. This ordering has no momentum exception: work that emerges mid-session — including spillover from a bug or hotfix bolt — gets its bolt file and risk assessment BEFORE its first line of code, not retroactively on pickup.

---

## Step 1 — Read the Bolt and Its Units

Read the bolt file and every unit file listed in the bolt's Units table before saying anything.

Also read:
- `process-onboarding-agent/rules/architecture.md` — to understand existing module boundaries and ADRs
- `process-onboarding-agent/guidelines/forbidden-zones.md` — if it exists, to check whether any unit touches a protected zone
- The design artifact linked from the intent, if one exists

Confirm what you have read:

> "I've read the bolt [name] and its [N] units. I'll now assess the blast radius, sequencing risks, rollback approach, and feature flag requirements before execution begins."

Do not ask any questions yet — proceed directly to Step 2.

---

## Step 2 — Blast Radius Analysis

For each unit in the bolt, work through the following — internally, without asking questions unless something is genuinely ambiguous:

**What to assess per unit:**

| Question | What to look for |
|---|---|
| Which existing files or modules will this unit modify? | Named in the unit's Context or Scope sections |
| What existing behavior depends on those files? | Cross-references in architecture.md; other units in the bolt or backlog that link to the same modules |
| What is the worst-case impact if this unit introduces a defect? | Data loss, broken auth, degraded UI, silent failure, cascading failure in downstream modules |
| Does this unit touch any boundary defined in architecture.md? | API contracts, service boundaries, data ownership rules |
| Does this unit touch a forbidden zone? | Check forbidden-zones.md if it exists |
| Does this unit create a hosted resource of a TYPE this account has never held? | A first use often needs an account-level switch — a cloud provider registration, an API to enable — that fails at the create call and that no fake models, because the fake was written by someone whose account already had it. Record yes / no / unknown; where not "no", the creating code checks and performs the registration itself |
| Does this unit add or depend on a hosted resource provisioned outside the codebase (storage, queue, secret, DB/schema, identity/RBAC, third-party binding, new runtime env var)? | The feature is not Done until the resource is provisioned AND verified in every target environment — CI is green regardless, because tests stub or emulate it. Record what to provision, where it is documented, and the automated deploy signal that fails the deploy if it is absent (e.g. a config-readiness field in the health response asserted by the post-deploy smoke, kept separate from liveness so a misconfig never flaps health). A missing deploy signal is a High-priority sign-off item |
| Does this unit pin an infrastructure ARTIFACT variant — a container image flavor, a base OS, a package distribution? | Verify during this assessment that the variant EXISTS in its registry, as you would a provider API contract — never pin one from habit. If a database image swap crosses a libc boundary (e.g. musl ↔ glibc), name the data directory's collation-compatibility plan (dump/re-init/restore, or reindex); "the volume survives" is a claim, not a plan |

**When the scope is a CLASS of sites, enumerate by the MECHANISM before the blast-radius table, not from the list the finding handed you.** A class scope looks like "every place that reads this column", "every route calling this guard" or "every writer of this state". The finding or intent supplies examples; that list is a **sample**. Treat it as the set and the bolt ships with the class only mostly fixed, which is worse than not starting: moved surfaces now disagree with unmoved ones. Name the mechanism in one line, grep for **it** rather than for the reported files, and record the command and count in the assessment. Do it BEFORE the table, because the table's rows are per unit and a missing site has no row to appear in. Where the bolt will end in a drift guard, consider writing the guard first and reading its red output as the enumeration. (Worked example: a finding named five sites, the assessment found six, and the true count was seven. The seventh surfaced from the same one-line grep after six had already been changed, and the bolt's own drift guard, run against the old tree, listed all seven.)

After assessing all units, produce a blast radius table:

```
Blast Radius — [Bolt name]

| Unit | Modules touched | Existing behavior at risk | Worst-case impact |
|---|---|---|---|
| [unit name] | [files/modules] | [behavior description] | [impact level: High / Med / Low] |
```

Flag any unit rated High immediately before presenting the full table:

> "Unit [name] has a high blast radius — it touches [module] which [existing behavior]. I'll highlight this in the assessment."

**A mechanical sweep encodes the assumptions that were true of the set it was written for — re-verify them when the set grows.** Where a unit will apply the same scripted edit across many files, its safety rests on a pre-check performed on *those* files: that they all share the shape the script assumes. That premise expires silently. A rebase, a merge, or a concurrent session can add one more file that violates it, and re-running the sweep then damages that file while reporting success. So: re-run the sweep's **pre-checks**, not merely the sweep, after any event that grows the target set; prefer a sweep that **fails loudly on an unexpected shape** over one that transforms whatever it is given; and where the invariant must outlive the sweep, land a **drift-guard test** so the gate enforces it rather than the next author remembering — the sweep fixes today, the guard fixes next week. Damage is proportional to distance from a compile error: an unused import costs one gate run, a wrong behavioural assumption costs a debugging session.

**When a unit ADDS A MEMBER to a rendered set, search for an assertion on the SET — not only for a collision with a member.** Blast radius asks what a change *touches*. This covers a change that touches nothing and breaks something anyway: a navigation, a tab row, a menu, a group of controls, a list of statuses. The instinct is to ask *"does my new member's name collide with an existing selector?"* — and the answer is usually no, which is a correct answer to the wrong question. What breaks is a test asserting the set's **membership**: an equality against the full list, a count, a snapshot of a group's children.

The failure mode is **right files, wrong property**, and it is worth naming separately because it reads as thorough. A worked example: an assessment searched the three specs that referenced a tab row, asked whether a new tab's label would collide with their existing selectors, and concluded "collides with nothing". One of those specs asserted the tab list by equality — with its own comment explaining that the equality existed *precisely* so the set could not drift silently. The guard worked; the enumeration did not, and the assessment had been signed on the strength of it.

The search is mechanical: grep the equality and count forms alongside the container's identifier across the test corpus, then read what the matches assert about the container you are adding to. **And when such a guard goes red, UPDATE the set — never loosen it:** an equality converted to a "contains" can no longer catch the next member, which was the whole property it held.

**Counting and enumerating — before any number or "none" enters the assessment:**

**A count is an assumption until it is grepped — never inherit one from the plan, and scope the grep to the INVARIANT, not the mechanism you picture.** A call-site or consumer count written during elaboration came from reading the design, not the tree, and nothing in elaboration tests it. Verify every such number here: before recording it, state which question the grep answers, and paste the command and its hit count beside the number. When a unit imposes a new invariant, write the invariant as a sentence first ("what can make an X exist?") and only then choose the searches. An enumeration built from one verb misses every other verb that reaches the same state: inserts find creators; updates of the discriminating field find things that BECOME the state; re-labelling or classification acts find things that become it by type change; restores find things that return to it. Put the invariant sentence beside the count so a reviewer can check the question, not only the arithmetic. *(An elaboration recorded "~3 call sites" for a symbol that had 22 across 11 files. Separately, an assessment counted 28 write paths with an insert-only grep and was signed off; a sixth path that made the guarded record by UPDATE … SET type surfaced only at execution.)*

**Enumerate by grepping what is USED, not what is DEFINED or remembered.** Search for the symbol at its use sites, or for the discriminating expression itself (the predicate, the flag test, the field access), then decide each hit explicitly — including those needing no change, so the check is visibly complete. A definition count feels authoritative and is the wrong number; correcting a plan's count once does not cover sibling modules of the same shape. Watch for use sites that do not *look* like call sites: markup children, framework fallback slots, dispatch tables, string-keyed registries.

**And enumerate by the BEHAVIOUR, not only the symbol.** A symbol grep finds everything that calls what you are changing. It cannot find a test that encodes the FACT you are reversing, because that test names none of your symbols. Ask in words, "what asserts the thing that is about to stop being true?", and grep for that: the fixture's format, the status code, the sentence, the absent-event assertion. Such a test often keeps passing after the change for a reason it does not assert, so no gate will surface it. *(A bolt gave PDFs a preview. A test whose PDF fixture asserted "no preview event is recorded" named none of the changed symbols, and after the change it still passed only because its fixture bytes were unparseable.)*

**A finding that asserts an ABSENCE needs TWO searches, worded differently, before it is reported.** "This surface has no X", "nothing asserts Y" and "there is no guard here" are the weakest evidence an assessment produces. A single literal grep that returns nothing is one spelling failing to match. Search for the thing's purpose as well as its name, open the file when the claim will drive scope, and state the evidence beside the finding. The consequence is asymmetric: an undercount leaves a gap for a gate to find later, but a false absence creates work that has already been approved. *(An assessment reported a surface had no "open the file" link from one grep for that exact phrase. The surface read "Open full document ↗", and the finding had already become an approved AC.)*

**A stated count needs TWO enumerations, keyed differently.** One grep finds only the spelling you already saw: right scope, incomplete spelling. When an assessment or pre-generation check states a count, record the command AND a second command keyed differently: a synonym or alternative idiom for the same operation, a case-insensitive search for anything a user can name, or the VALUES rather than the identifier. Write down both counts. Where they disagree, the larger is the answer and the gap is the finding. Prose in this family fails repeatedly, so where the invariant admits a structural guard (a test that fails the build on the forbidden pattern), write the guard instead. *(Two consecutive bolts: 18 call sites cited where the mechanism had 42, then 38 files counted by one idiom when two more spelled the identical bug differently. Both times, reading a file caught it, not a better grep.)*

**What the change must prove — reach, observability, and the shipped artifact:**

**A change that makes previously-unreachable code reachable is scoped by what it ENABLES, not by what it edits.** When a fix un-blocks a path that has never executed — a provider call that always failed, a flag that was always off, a quota that always denied — every module downstream of it runs in production for the first time, and its green suite proves only that it survives fixtures. Ask of each unit *"what runs now that never ran before?"*, name that downstream path in the blast-radius table, and require a test pass over it (a real end-to-end call where possible) in the same bolt. A dormant path has no production baseline, so "no change in behaviour" is unverifiable for it by definition.

**Ask of every AC whether its THEN is OBSERVABLE, not only whether its GIVEN exists.** Blast radius enumerates what a change *touches*; nothing in it checks what a change must *prove*. Ask: **is there a surface on which this AC's outcome can be asserted, and can that assertion FAIL?** Three failure shapes recur. First, **no falsifiable form exists**: an absence assertion about something the change deletes passes for ever, so the AC needs re-expressing, often with a new test hook, which is scope. Second, **the surface has no existing coverage**: "this area is well covered" is a claim about the specs that exist, and the answer is sometimes zero. Third, **the behaviour is new**, so no assertion can exist yet. For each AC record one of: the existing test that will assert it (named and grepped for, not assumed); the new test to be written; or "no assertable surface; here is what must be built to make it assertable", which changes the unit's scope. An AC whose THEN nobody can observe is unfalsifiable, and it will be ticked on the strength of the diff.

**Two further questions belong to the same pass.**
- **Is the AC's GIVEN CONSTRUCTIBLE on the surface you just named?** Any claim about what the gate environment can do (scrollbar style, timezone, locale, colour scheme, viewport, network shape, filesystem case-sensitivity, CPU count) is a measurement. Take it with one probe; do not recall it. Where the answer is no, record the AC's disposition here as "closed by inspection or UAT, not by the gate". *(An assessment asserted the gate browser had space-occupying scrollbars. Its gutter measured 0, and the assertion passed with the guarded code deleted.)*
- **Is the AC's WHEN PERFORMABLE?** The observability question asks whether the outcome can be seen; it never asks whether the trigger can be reached. An AC whose trigger no user and no test can reach (e.g. "navigate away while the modal is open" when the modal intercepts every pointer event) will be quietly dropped or, worse, faked.

**Ask which of your verification routes actually exercise the SHIPPED ARTIFACT.** Routes that share a module resolver, a runtime or a build step are **one route wearing three hats**. Listing them as three manufactures confidence the assessment has not earned. The sharpest case is a build tool rewriting code between the source you test and the artifact you ship. Bundlers, transpilers, minifiers and native compilers claim certain constructs as their own (module resolution, `import.meta.url` / `__dirname`, a computed dynamic `import()`, inlined environment variables) and emit a *different but plausible* value. The source stays correct, so reviewing it can never find the defect. Required:
1. Where a build step stands between source and artifact, at least one verification runs the **artifact**, and the assessment names which one.
2. Where a build-owned construct is load-bearing, **read the emitted output** rather than inferring it from the source.
3. Make the runtime value **self-verifying**: check that a resolved path holds a file only the right target holds, rather than trusting that a value came back.

Worked example: a font path resolved via `require.resolve` passed type-checking, a large unit suite and an in-image probe. The bundler replaced the call with a numeric module id, and every rendered page shipped blank with the fix visibly in place.

---

## Step 3 — Cross-Unit Sequencing Risks

Review the bolt's Execution Order and identify risks that arise specifically from the order units run in — not from individual units in isolation.

Look for:

- **State dependency risk:** Unit B assumes a state that Unit A creates. If Unit A fails mid-way, what is the system state?
- **Partial rollout risk:** If the bolt is interrupted after some units are done but before others, is the system in a consistent state?
- **Shared resource contention:** Two units that modify the same file, table, or config. If run in parallel, do they conflict?
- **Contract mismatch risk:** A unit that changes an interface that a later unit in the same bolt consumes. If the producer unit changes shape after the consumer unit is written, do they fall out of sync?

Produce a sequencing risk table if any risks are found:

```
Sequencing Risks

| Risk | Units involved | Condition | Mitigation |
|---|---|---|---|
| [description] | [unit A → unit B] | [when this occurs] | [how to prevent or recover] |
```

If no sequencing risks exist, state: "No cross-unit sequencing risks identified."

---

## Step 4 — Rollback Assessment

For the bolt as a whole, determine the rollback approach. Ask the engineer only if the answer cannot be determined from the unit files and architecture:

Assess each of the following:

**Code rollback:** Can reverting the commits for this bolt's units restore the previous behavior without side effects? Look for: database migrations in any unit (once applied, may not be reversible), external API calls that mutate state (webhooks sent, emails triggered, records created in third-party systems), or file system changes outside the repo.

**Data rollback:** Does any unit write to a database schema or seed data in a way that cannot be undone by reverting the code? If yes, a data rollback script or migration must be part of the unit's Definition of Done.

**Partial rollback:** If only some units in the bolt are merged when a problem is found, can those units be reverted independently, or do they form an atomic group that must be reverted together?

**Image rollback across a migration:** If any unit adds a migration, can the PREVIOUS deployed build start against the migrated schema? Answer it by reading the migration runner, not from "migrations are additive": a runner that refuses a recorded migration it has no file for makes every rollback past a migration an outage. And a rollback that has never been run is a hypothesis — say when it was last rehearsed against the real environment, or that it never has been.

Produce a rollback summary:

```
Rollback Assessment

Code rollback:   [Safe — revert commits restores prior state]
                 [Unsafe — [reason]: engineer must [action] before reverting]

Data rollback:   [Not required — no schema or data changes]
                 [Required — migration needed: [description]]

Partial rollback:[Each unit is independently revertible]
                 [Units [X] and [Y] must be reverted together — [reason]]
```

**Who else deploys:** If more than one session or person can deploy to the same environment, the deploy re-reads what is live IMMEDIATELY before it acts — an unreadable reading is a stop, not "no change" — and takes a lock the others can see. The rollback target named above is read again at deploy time; the one read during this assessment may already be out of date.

---

## Step 5 — Feature Flag Requirement

Determine whether any unit in the bolt changes behavior that an existing user would observe in production — even if the change is intentional. The question is not whether the change is correct, but whether it should be introduced progressively.

Assess each unit against these criteria:

| Criterion | Feature flag required? |
|---|---|
| Changes the behavior of an existing feature visible to end users | Yes |
| Adds a new capability that replaces or competes with an existing one | Yes |
| Changes an API response shape consumed by existing clients | Yes |
| Modifies internal logic with no user-visible effect | No |
| Adds a new endpoint or screen that does not affect existing flows | No — unless explicitly required by project policy |

If a feature flag is required for any unit, state which unit(s) and what behavior the flag gates. If the project has a feature flag convention (in architecture.md or code-standards.md), reference it. If no convention is established, flag this as an open question for the engineer:

> "Unit [name] changes [behavior] which existing users will observe. A feature flag is required. Does this project have an established feature flag pattern, or should I log establishing one as an open question?"

---

## Step 6 — Risk Assessment Sign-off

Present the full assessment before writing anything to the bolt file:

```
Risk Assessment — [Bolt name]

BLAST RADIUS
[paste blast radius table from Step 2]

SEQUENCING RISKS
[paste sequencing risk table from Step 3, or "None identified"]

ROLLBACK
[paste rollback summary from Step 4]

FEATURE FLAGS
[list units requiring flags, or "None required"]

INFRA DEPENDENCIES
[resource · where documented · deploy signal — or "None"]

HIGH-PRIORITY ITEMS (requiring engineer decision before execution):
[list any High blast radius units, unsafe rollbacks, or open feature flag questions]
[or "None — assessment is clear to proceed"]

Sign off to proceed with unit execution?
```

If the engineer raises concerns or requests changes to the assessment, revise the relevant section and present the summary again. Do not proceed to Step 7 until the engineer explicitly confirms sign-off.

---

## Step 7 — Write to the Bolt File

Replace the bolt file's "Risks and Assumptions" section with the signed-off assessment. Do not keep the original passive table — it is superseded by this output.

Write the following structure into the bolt file:

```markdown
## Risk Assessment

**Assessed:** YYYY-MM-DD  
**Status:** Signed off by engineer before execution

### Blast Radius

| Unit | Modules touched | Existing behavior at risk | Impact |
|---|---|---|---|
| [unit] | [modules] | [behavior] | High / Med / Low |

### Sequencing Risks

| Risk | Units involved | Condition | Mitigation |
|---|---|---|---|
| [description] | [units] | [condition] | [mitigation] |

_(or: No sequencing risks identified.)_

### Rollback

- **Code rollback:** [safe / unsafe — reason and required action]
- **Data rollback:** [not required / required — description]
- **Partial rollback:** [independently revertible / must revert together — units and reason]

### Feature Flags

- [Unit name]: gates [behavior description]
_(or: No feature flags required.)_

### Open Items

- [Any items the engineer must resolve before or during execution, or "None"]
```

After writing, confirm:

> "Risk assessment written to the bolt file. All [N] open items (if any) must be resolved before the affected units execute. Ready to begin unit execution — say 'execute unit [first unit name]' to start."
