# Unit: [name]

**Status:** Open | Planned | In Progress | Done | Blocked | Deferred
**Intent:** [ops/inception/intents/YYYY-MM-DD-<unix_timestamp>-[slug].md](../../inception/intents/YYYY-MM-DD-<unix_timestamp>-[slug].md)
**Elaboration:** [ops/inception/elaborations/YYYY-MM-DD-<unix_timestamp>-[slug]-session-N.md](../../inception/elaborations/YYYY-MM-DD-<unix_timestamp>-[slug]-session-N.md)
**Bolt:** [ops/build/bolts/YYYY-MM-DD-<unix_timestamp>-[bolt-slug].md](../bolts/YYYY-MM-DD-<unix_timestamp>-[bolt-slug].md)
**Priority:** High | Medium | Low

---

## Context

[One paragraph. Who needs this, what it does, and why it is a discrete unit. Written for an engineer who has not read the intent.]

---

## Acceptance Criteria

1. Given [precondition], when [action], then [outcome].
2. Given [precondition], when [action], then [outcome].
3. Given [precondition — unhappy path], when [action], then [outcome].

---

## Scope

**In scope:**
- [What this unit covers]

**Out of scope:**
- [What is explicitly excluded — handled by another unit or not planned]

**Declined findings:** a problem found during this unit and not fixed here gets a backlog row in the same commit as the unit — a note in this file is not a record, because nobody rereads a closed unit.

---

## Dependencies

| Dependency | Type | Status |
|---|---|---|
| [Unit or external system] | Unit / API / Config | Done / Pending |

---

## Pre-generation Checks

Before generating code for this unit, the agent must run these checks:

- [ ] Grep for existing implementations of [pattern] across the codebase to avoid duplication
- [ ] Confirm the module entry point is listed in `process-onboarding-agent/guidelines/entry-points.md`
- [ ] Confirm no files in scope appear in `process-onboarding-agent/guidelines/forbidden-zones.md`
- [ ] Verify test coverage for affected module meets the gate threshold
- [ ] Confirm everything an AC relies on exists where the AC's code runs — each client function it calls, each control it taps, each named constant or limit it quotes. A server capability is not a client control, and a constant on one side is not available on the other: if absent, the unit is bigger than written, or the AC records whether the value is imported, duplicated with its source named, or asked of the server
- [ ] Where an AC adds a second instance of a pattern already on the screen ("as X already does"), write the new labels into the AC and confirm they differ from the existing ones *(identical labels break existing text queries and invite a mis-tap)*
- [ ] List the design premises and binding-constraint titles that touch this unit's files, each marked **kept** or **departed**; put any departure to the engineer before code *(a premise no AC restates is otherwise checked by nothing)*
- [ ] If this unit changes, moves or deletes anything an existing test could pin, count the pins by sweeping every test file at execution start — ids and selectors, mocked-call arguments, and the property's own name and the helper that reads it — and size the Breaking Changes Register from that count, with its command *(a count inherited from the design is a draft, and a move that leaves behaviour identical is the change most often mis-scoped as a refactor)*
- [ ] Before a new test queries a control, selector or route, check the plan's later units for its removal; if one removes it, anchor on what survives or write the coupling into that unit's Register in the same commit
- [ ] If this unit moves a component to a different host or container, list what the old host supplied implicitly (layout insets, context, error boundaries, wrappers) and confirm the new one supplies it *(the component's own code and tests do not change, so nothing else looks)*
- [ ] If an existing action starts writing to a table it did not write before, grep the suites that perform that action and tear down its parent rows, and confirm each cleanup covers the new table
- [ ] If meeting an AC literally would do something the intent, an ADR or another AC forbids, put both readings to the engineer before code and record the answer here *(an interpretation written only into the evidence reaches the engineer after it ships)*
- [ ] Grep the backlog and earlier units' notes for findings **declined to this unit** — a finding another unit declined and assigned here is a handover, not a note, and is this unit's scope. List each one here, or write "None". _(makerclub, 2026-09-21: one unit declined a defect to a named later unit, which then executed without doing it, because nothing obliged it to look.)_
- [ ] If this unit resolves a contradiction with, or overrules, a **signed design**, it amends the design document itself **in this unit's own commit** and says so here. Recording the change only in the unit says *that* it changed; the design is where the next reader finds out *what it now is*. _(makerclub, 2026-09-21: two signed design rows were overruled inside unit files, correctly, and the design still said the old thing four units later.)_

---

## Edge Cases to Handle

| ID | Scenario | Required behavior |
|---|---|---|
| EC-001 | [scenario] | [what the code must do] |

---

## Observability

> What evidence must exist in production to know this unit is working correctly? Any entry that represents code behavior the AI must implement should be expressed as an AC in the Acceptance Criteria section above.

**Success signal** — what metric or log entry confirms this unit is operating normally?
- [ ] [e.g. "A counter increments on every successful call to /api/bookings"]
- [ ] [or: "Not applicable — this unit has no production traffic or async process to monitor"]

**Failure signal** — what log entry confirms something has gone wrong?
- [ ] [e.g. "A WARNING-level log entry is written when payment processing fails, including the order ID and error code"]

**Alert threshold** — at what point should an on-call engineer be notified?
- [ ] [e.g. "Alert fires if the error rate for /api/bookings exceeds 5% over a 5-minute window"]
- [ ] [or: "Not applicable"]

---

## Breaking Changes Register

> **Complete this section only for Migration or Remediation Bolts that change contract boundaries. Leave blank (write "N/A — not a contract-change Bolt") for all other Bolts. This section must be completed and approved before any code is generated.**

| # | Contract boundary changed | Reason it must change | Approved by | Date |
|---|---|---|---|---|
| 1 | [API endpoint / data schema / inter-module interface] | [why the change is necessary] | [engineer name] | YYYY-MM-DD |

---

## Definition of Done

- [ ] All ACs implemented and traceable to code
- [ ] Any recorded **deviation from an AC** names the case that distinguishes it from the AC as written, and either shows that case impossible or accepts it. A written deviation reads as a decision somebody made, which is what makes a wrong one expensive. _(makerclub, 2026-09-21: "three resets in 60 s" was implemented as "three crashes, none reaching 5 s of life" and called stricter; a program crashing at eight seconds cleared its own count every boot and looped for ever.)_
- [ ] Unit tests written for each AC
- [ ] Any procedure this unit writes that DELETES or REPLACES data (a restore, a wipe, a migration rollback, a cleanup job) has been rehearsed in this unit, and the rehearsal is recorded with its date and its result — including a first attempt that failed. A procedure is a hypothesis until it has been run
- [ ] If this is the last unit of the last bolt of an intent: the intent's own file is closed in the same commit — its status set, what is Owed listed, and what would reopen it stated
- [ ] If the tests **fake an external tool** (a compiler, a CLI, a third-party service), the real tool has been exercised once **through the same code path that ships** — not run directly beside it. Name the command here, or record its absence as an owed action with an owner. _(makerclub, 2026-09-21: every test replaced the compiler with a script and the unit's verification ran the compiler directly; an environment variable set only in the shipped path made every real build fail, found at the first deploy.)_
- [ ] Integration tests for affected module pass without modification *(or: all breaking changes listed in the Breaking Changes Register have updated tests and are approved)*
- [ ] Any failure in the full run that is not this unit's and does not reproduce is recorded as one line — date, suite and test, the error's first line, elapsed time — on the backlog's open row for intermittent failures (or a new row if none fits) *(a single sighting is a data point, and an intermittent class is made only of single sightings; left in this unit, nobody who later diagnoses it will see it)*
- [ ] No secrets or hardcoded environment values
- [ ] Any resource this unit creates that **bills while idle** (an always-on worker, a provisioned database tier, a reserved instance) has its idle cost recorded here and **names who owns its schedule** — so the bill is somebody's decision rather than a surprise. Write "No idle-billing resources" otherwise. _(makerclub, 2026-09-21: a 2 vCPU worker ran around the clock against an architecture decision that said scale to zero out of hours, and nobody had decided that.)_
- [ ] Auth checked on every new endpoint
- [ ] Any AC whose behaviour the test gate cannot reach (a device, a real network, a person's screen) is Done only against an observation recorded in this unit — date, build or commit, what was done, what was seen. **Where the behaviour is a delivery** (something reaching a person: a message, a notification, a screen state, a file on a device) **the observation is taken at the receiving end**, or from the server's record of the delivered item — never from the sender's own log, which records the attempt whatever happened to it
- [ ] If this unit ran a deploy or published an OTA: every OTHER unit whose owed deploy/OTA it carried is recorded as delivered — test each owing unit's landing commit with `git merge-base --is-ancestor <landing> <the commit you shipped>`, and write the revision or update id and the date into that unit and its backlog row *(a ship from a shared branch carries every commit beneath it; without this, other units go on saying "deploy owed" for weeks after it happened)*
- [ ] Any deploy, provisioning or scheduler script this unit ships has been run and what it created is recorded — or not running it is a backlog item with an owner *(tests prove the code, never that the environment was provisioned)*
- [ ] Where this unit removes a control or code path, everything it referenced is checked for a remaining reachable caller *(lint finds unused symbols, never a branch that is still read and written but can no longer be reached)*
- [ ] Where this unit adds or tightens a constraint, every writer of the constrained columns is listed here, each with a test at the layer that builds the row writing the newly constrained shape *(a constraint changes behaviour only where a writer was already quietly wrong, and a test of the layer below passes while that writer fails)*
- [ ] The shared CI gate's conclusion at this unit's commit is recorded, with the skipped count of every suite quoted — a skipped test is not a passed one, and a run whose job never started is recorded as "no job ran" and replaced by a full package-wide local run, never as green
- [ ] Each falsification probe (revert the code an AC depends on; the suite must go red) is recorded by name with its result, after proving the mutation applied (`git diff --stat` non-empty). A green probe is a finding: close the gap — commonly every case shares one fixture value — and re-run it to red; where the behaviour is guarded twice, record it as unverified defence naming both guards
- [ ] Where the design estimated cost or latency, one real production log line measuring it is recorded with the time it was read *(an estimate nobody is obliged to check stays an estimate)*
- [ ] Owed observations are listed one per AC number, each as an action and what should be seen; one whose surface a later change removes is struck with the reason, not deleted or left open
- [ ] Reviewed against `process-onboarding-agent/skills/review-checklist.md`
- [ ] Prompt log updated in `process-onboarding-agent/prompts/`
- [ ] Anything this unit leaves **owed** — a device observation, a step only a named person can take, a question awaiting a ruling — has its own row in the backlog, naming who owes it and what would close it *(a mention inside this file or a status cell is not a row: after the unit closes, nobody reopens this file, and the backlog is what gets read)*
- [ ] Unit status set to Done in backlog

---

## Prompt Log

[process-onboarding-agent/prompts/[unit-name].md](../../../prompts/[unit-name].md)

---

## Notes

[Any decisions made during execution that future engineers should know about.]
