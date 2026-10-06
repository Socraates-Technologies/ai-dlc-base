# Skill: Bolt Risk Assessment

**Purpose:** Actively interrogate a planned bolt for blast radius, cross-unit sequencing risks, rollback feasibility, and feature flag requirements before the first unit executes. Replaces the passive "Risks and Assumptions" table in the bolt file with structured, AI-driven findings that the engineer signs off on.

**Trigger:** Runs after elaboration sign-off and before the first unit in a bolt is executed. Invoked by the mob elaboration protocol automatically after the sign-off summary is confirmed, or directly by the engineer with "run risk assessment for bolt [name]."

**Never skip for mature projects.** For projects with existing code, this assessment is mandatory — no unit executes without a signed-off risk assessment in the bolt file. For fresh projects with no existing modules affected, the assessment may be brief but must still be completed.

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
| Does an AC name a key, a signature, a proof or a hash? | **Which side holds which value?** Write down, per side, what it actually stores; an AC whose checking side cannot hold the value it names is re-derived here, before sign-off, not at build time *(makerclub, 2026-10-05: an AC keyed an HMAC by a device secret the server only ever stores the hash of)* |
| Does the unit state a fact that comes from build or framework configuration? | Read it from the **generated, resolved** configuration the build actually uses, never from the overrides or defaults file a person edits — the framework's own defaults never appear there *(makerclub, 2026-10-05: an assessment read a console port from the defaults file and missed a framework default that also routed the logs to the cable)* |
| Does this unit create or resize a resource that bills while idle? | Write its **idle cost** as a number — what it costs with nobody using it — and **who owns its schedule** (scale-to-zero, hours, shutdown). A bolt that creates infrastructure without both hands the bill to whoever notices first *(makerclub, 2026-09-21: a 2-vCPU worker ran always-on from the moment it was created, against an ADR that said scale to zero out of hours; nobody had decided that)* |
| Does this unit change code that a still-owed proof of a SAFETY mechanism covers? | Re-read the owed proof's plan against this change: list the unit in the proof's Owed line, and make the plan name every output or path added since it was written. A proof run against the plan as first written proves the mechanism someone remembers, not the one that ships *(makerclub, 2026-10-02: an emergency stop was proved on a light; three later units added motors and servos to the cut path before its full bench ran, and the bench plan still named only the light)* |

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

**When a unit adds a NAME to a set, search every test suite for the new name itself — and read the whole output.** A new command, block, key, enum member or event name may already appear in the tests as their example of something *unknown* or *invalid*, and those tests go red the moment the name becomes real. Search the tree for the literal name before the change, not only for the set's other members, and do not truncate the output (`head`, a pager, a result limit): the row the truncation hides is the one the next assessment misses *(makerclub, 2026-09-28: two tests used a not-yet-real name as their unknown example, and a truncated search hid one of them from a later bolt)*.

**A Breaking Changes Register is measured on the WHOLE change, renames included.** A rename that any unit in the bolt requires is a contract change for every caller of the old name, tests included. Measure with every rename the bolt's ACs require already applied — not only the frames, fixtures and shapes the unit set out to change — or the Register misses the lines a rename breaks and they surface at a later unit's execution *(makerclub, 2026-10-02: a Register measured on the wire frames missed two test lines a required method rename broke; the next unit found them and had to keep them green with a deprecated alias until the engineer approved the rows)*.

---

## Step 3 — Cross-Unit Sequencing Risks

Review the bolt's Execution Order and identify risks that arise specifically from the order units run in — not from individual units in isolation.

Look for:

- **State dependency risk:** Unit B assumes a state that Unit A creates. If Unit A fails mid-way, what is the system state?
- **Partial rollout risk:** If the bolt is interrupted after some units are done but before others, is the system in a consistent state?
- **Shared resource contention:** Two units that modify the same file, table, or config. If run in parallel, do they conflict?
- **Contract mismatch risk:** A unit that changes an interface that a later unit in the same bolt consumes. If the producer unit changes shape after the consumer unit is written, do they fall out of sync?
- **Unobserved-step risk:** A unit whose step the gate cannot reach — a hardware reset, a flash or firmware write, a radio, a third-party callback, a real device — is walked on the real thing before the unit that builds on it starts, or the dependency is named here as a risk with its fallback. Withholding Done until an observation is recorded protects the unit; this protects the units after it — nothing builds on an unobserved step *(makerclub, 2026-10-05: a chip read ended with a reset that left the device in its bootloader; the write was built on the same call the same day, and the first real device found it)*.

Produce a sequencing risk table if any risks are found:

```
Sequencing Risks

| Risk | Units involved | Condition | Mitigation |
|---|---|---|---|
| [description] | [unit A → unit B] | [when this occurs] | [how to prevent or recover] |
```

If no sequencing risks exist, state: "No cross-unit sequencing risks identified."

**Each sequencing row is re-read at delivery, and ticked or its skip written with a reason.** A row such as "observe the old path before the new one is first used" buys attribution: if the first real use fails, it says which of several new pieces is at fault. Skipped in silence, the attribution is spent for nothing even when everything works — and a skip that went unnoticed once is skipped again *(makerclub, 2026-10-02: the same "observe the old path first" row was skipped silently in two consecutive bolts)*.

---

## Step 4 — Rollback Assessment

For the bolt as a whole, determine the rollback approach. Ask the engineer only if the answer cannot be determined from the unit files and architecture:

Assess each of the following:

**Code rollback:** Can reverting the commits for this bolt's units restore the previous behavior without side effects? Look for: database migrations in any unit (once applied, may not be reversible), external API calls that mutate state (webhooks sent, emails triggered, records created in third-party systems), or file system changes outside the repo.

**Data rollback:** Does any unit write to a database schema or seed data in a way that cannot be undone by reverting the code? If yes, a data rollback script or migration must be part of the unit's Definition of Done.

**Partial rollback:** If only some units in the bolt are merged when a problem is found, can those units be reverted independently, or do they form an atomic group that must be reverted together?

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

**An AC that names an instrument the assessment knows is unavailable is amended here, not overridden at delivery.** If a finding records that a gate, environment, device or service the ACs rely on will not be available (CI refused, a staging environment down, hardware not yet delivered), amend each AC that names it in the same sign-off: say what stands in for the instrument and let the engineer sign the substitute with the assessment. An override at the deploy is honest, but it was a decision the assessment already had the facts to put in front of the engineer *(makerclub, 2026-10-02: a risk row recorded that CI was refused while two ACs still required a CI-backed pre-deploy check to be green; the deploy went ahead on an override)*.

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
