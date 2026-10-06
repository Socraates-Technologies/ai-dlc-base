# Skill: Bolt Risk Assessment

**Purpose:** Actively interrogate a planned bolt for blast radius, cross-unit sequencing risks, rollback feasibility, and feature flag requirements before the first unit executes. Replaces the passive "Risks and Assumptions" table in the bolt file with structured, AI-driven findings that the engineer signs off on.

**Trigger:** Runs after elaboration sign-off and before the first unit in a bolt is executed. Invoked by the mob elaboration protocol automatically after the sign-off summary is confirmed, or directly by the engineer with "run risk assessment for bolt [name]."

**Never skip for mature projects.** For projects with existing code, this assessment is mandatory — no unit executes without a signed-off risk assessment in the bolt file. For fresh projects with no existing modules affected, the assessment may be brief but must still be completed.

**When one question needs the change to exist, split — do not defer the whole assessment.** Some risks are answerable only by a measurement that needs the code (does it fit, does it stay under budget, does it still start). Write the assessment first with that measurement as a named open item — what will be measured, where, and what answer would change the plan — and complete every sweep that does not need the code up front. The engineer can decide on an assessment with one open measurement; one written afterwards only asks them to approve a conclusion.

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

**Test-coupling sweeps.** The default AC predicts that existing tests pass unmodified; check that prediction against the tests themselves before accepting its form:

- **Callers, not only readers.** When a unit may change a function's signature, count its call sites (production and test separately, with the command), not just the consumers of what it returns. A new field that must be resolved from something the function does not already receive is a signature change. Decide required-versus-optional in the assessment: a required parameter makes a forgotten argument a compile error, an optional one a silently wrong answer.
- **Every form of "what this call sends".** A request may be asserted through the client function in one suite and through an injected prop or callback in another, and by a called-with matcher in one place and a deep-equal on recorded calls in another. Search for all names and forms, then read the hits.
- **Suites that render the application root.** An app-level or end-to-end suite names none of the components it exercises, so a symbol search cannot find it. List the root-level suites and ask whether the change is reachable from each.
- **Fixture defaults under a new default.** When a rule changes what renders or is returned by default (a filter, a partition, a narrowed view), read the fixture helpers' default for every field the rule reads. A "neutral" fixture may fall outside the new default and empty every existing case.
- **Fixtures that must reset, not only teardowns that will fail.** When an existing write path starts writing a table it did not write before — new or existing — list every suite that drives that path and check both its teardown (loud: a foreign-key failure names the file) and its per-test reset (silent: rows accumulate across cases until an assertion depends on an absolute count).
- **A Register row names what the assertion checks, not a number it prints.** Open the assertion behind each row. A count over a matcher that enumerates members stays green when a member is added; the row is "the matcher gains the member and the count moves", or the guard goes blind.

**When a change has several equivalent forms, measure them — do not argue them.** Where the plan has a free variable whose options are equally correct for the product (position in an ordered set, which module owns a function, which identifier stem), apply each form to a clean tree, run the affected suites, and record one row per form with its count and command. The engineer decides from the table. State what makes the forms equivalent; a form that is cheaper because it does less is pricing a different change.

**A claim written into a shared document carries the search that established it.** Statements about who consumes an interface, when it became shared, or what is enforced where are reasoned from later by sessions that cannot check them. Record the command beside the claim, as for a measured number; where the search was not run, write the weaker sentence you can support.

**Predict a falsification before running it.** When a finding is mitigated by a new guard, install the defect it guards against and write down how many tests should go red first. Fewer reds than predicted usually means the change dissolved one of the paths it was meant to keep, not that the guard passed. If the guard written for the defect stays green while unrelated tests catch it, its outcome probably depends on which of two concurrent operations settles last: force that ordering in the test (a deferred promise, a sequenced mock) with a comment saying why it is load-bearing.

---

## Step 3 — Cross-Unit Sequencing Risks

Review the bolt's Execution Order and identify risks that arise specifically from the order units run in — not from individual units in isolation.

Look for:

- **State dependency risk:** Unit B assumes a state that Unit A creates. If Unit A fails mid-way, what is the system state?
- **Partial rollout risk:** If the bolt is interrupted after some units are done but before others, is the system in a consistent state?
- **Shared resource contention:** Two units that modify the same file, table, or config. If run in parallel, do they conflict?
- **Contract mismatch risk:** A unit that changes an interface that a later unit in the same bolt consumes. If the producer unit changes shape after the consumer unit is written, do they fall out of sync?
- **Deploy-order risk:** Where client and server ship separately or at different speeds, ask both ways: what does the new client do against the old server, and the old client against the new server? The dangerous answer is not an error but silent wrongness — an old server ignoring an unknown field, or an old client's default branch describing a new enum value with a false sentence. Record what the default actually does, and put the mitigation in the code (refuse the act when the dependency is absent, or hold the value back until the reader ships), because deploy order cannot be enforced by a document.

Produce a sequencing risk table if any risks are found:

```
Sequencing Risks

| Risk | Units involved | Condition | Mitigation |
|---|---|---|---|
| [description] | [unit A → unit B] | [when this occurs] | [how to prevent or recover] |
```

If no sequencing risks exist, state: "No cross-unit sequencing risks identified."

**Wire the entry point early.** Land the control that opens a new surface as soon as that surface can stand up, not as the last unit: everything after it becomes observable in the running product, which catches defects (layout, reachability) that tests cannot see. Check what the new control makes unreachable, and that it does not open a half-built screen.

**Claims about things outside the repo must be probed, not inherited.** For each unit whose sign-off depends on a manual observation, name the artefact it needs (device or simulator build, staging environment, seeded account), confirm it runs the current tree, and say who builds it — found now it is a background task, found at the gate it stalls a finished unit. Infrastructure claims (a variable is set, a resource exists) are re-probed at closeout and amended by striking the old sentence, since no test in the repo can falsify them.

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

HIGH-PRIORITY ITEMS (requiring engineer decision before execution):
[list any High blast radius units, unsafe rollbacks, or open feature flag questions]
[or "None — assessment is clear to proceed"]

Sign off to proceed with unit execution?
```

If the engineer raises concerns or requests changes to the assessment, revise the relevant section and present the summary again. Do not proceed to Step 7 until the engineer explicitly confirms sign-off.

The gate fires twice: no unit executes without a signed assessment, and nothing deploys while one is unsigned. An assessment written after the fact is presented for sign-off before the change ships, while a "no" can still matter; if it deploys unsigned, its open items become backlog rows.

A decision at sign-off that withdraws or replaces a mechanism is a re-plan: search the bolt's units for the withdrawn mechanism and amend each affected Context and AC in the same commit as the assessment. A note beside a stale AC is not an amendment — the AC is what gets built.

Read one bolt ahead. If this bolt's decisions withdraw or rename anything, grep the next bolt's units for it too: a unit that still names the withdrawn mechanism looks fine until that bolt is next, and a grep now costs seconds while the context is still held.

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
