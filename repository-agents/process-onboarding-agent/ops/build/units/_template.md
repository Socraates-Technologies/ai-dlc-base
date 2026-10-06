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
- [ ] Unit tests written for each AC
- [ ] Integration tests for affected module pass without modification *(or: all breaking changes listed in the Breaking Changes Register have updated tests and are approved)*
- [ ] No secrets or hardcoded environment values
- [ ] Auth checked on every new endpoint
- [ ] Any deploy, provisioning or scheduler script this unit ships has been run and what it created is recorded — or not running it is a backlog item with an owner *(tests prove the code, never that the environment was provisioned)*
- [ ] Where this unit removes a control or code path, everything it referenced is checked for a remaining reachable caller *(lint finds unused symbols, never a branch that is still read and written but can no longer be reached)*
- [ ] Where this unit adds or tightens a constraint, every writer of the constrained columns is listed here, each with a test at the layer that builds the row writing the newly constrained shape *(a constraint changes behaviour only where a writer was already quietly wrong, and a test of the layer below passes while that writer fails)*
- [ ] The shared CI gate's conclusion at this unit's commit is recorded, with the skipped count of every suite quoted — a skipped test is not a passed one, and a run whose job never started is recorded as "no job ran" and replaced by a full package-wide local run, never as green
- [ ] Each falsification probe (revert the code an AC depends on; the suite must go red) is recorded by name with its result, after proving the mutation applied (`git diff --stat` non-empty). A green probe is a finding: close the gap — commonly every case shares one fixture value — and re-run it to red; where the behaviour is guarded twice, record it as unverified defence naming both guards
- [ ] Where the design estimated cost or latency, one real production log line measuring it is recorded with the time it was read *(an estimate nobody is obliged to check stays an estimate)*
- [ ] Owed observations are listed one per AC number, each as an action and what should be seen; one whose surface a later change removes is struck with the reason, not deleted or left open
- [ ] Reviewed against `process-onboarding-agent/skills/review-checklist.md`
- [ ] Prompt log updated in `process-onboarding-agent/prompts/`
- [ ] Unit status set to Done in backlog

---

## Prompt Log

[process-onboarding-agent/prompts/[unit-name].md](../../../prompts/[unit-name].md)

---

## Notes

[Any decisions made during execution that future engineers should know about.]
