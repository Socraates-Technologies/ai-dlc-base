# Skill: Design Session (Phase 0 of Mob Elaboration)

**Purpose:** Establish an agreed design foundation — API contracts, data model, and architectural pattern decisions — at the opening of a mob elaboration session before any units are proposed. The design output becomes binding constraints that govern every unit and AC produced in the session.

**Trigger:** Runs automatically at the start of every mob elaboration session, before the first unit is proposed. It is not a separate ceremony — it is the opening phase of elaboration. The `mob-elab-prompts.md` skill calls this phase explicitly.

The phase may be brief or extensive depending on the intent's complexity. For a simple, well-understood intent with no new interfaces or data entities, it may conclude in two or three questions. For an intent introducing new API surfaces or data structures, it will take longer. The length is determined by the scope questions in Step 2 — not by a fixed number of turns.

---

## Step 1 — Read and Anchor the Intent

Read the intent file fully before saying anything. Then open the session by reflecting the intent back to the engineer in plain language:

> "I've read the intent for [intent name]. As I understand it, we're building [one-sentence summary of goal]. Before we break this into units, I want to establish the design foundation — the interfaces, data shapes, and patterns we'll be building against. This becomes the baseline for every AC we write.
>
> If the intent is more complex than it appears, or if you want to adjust the goal before we design, now is the time."

Wait for the engineer to confirm or correct the understanding. Do not proceed until the intent goal is agreed.

**Read the code at the remote tip, and cite the commit.** Fetch, and read the files the design will touch from a fresh checkout of the default branch (e.g. a detached worktree at `origin/main`), not from a working copy that may be behind; write the short commit hash beside every file and line the session quotes. A shared or long-lived checkout can be many commits stale, and `git fetch` moves the ref, not the files you are reading — an AC written against a screen that has since changed is signed off and wrong.

---

## Step 2 — Scope the Design

Ask the engineer four questions to determine which design areas are relevant. Ask all four together — this is the one exception to the one-question-per-turn rule, because the answers are interdependent:

> "Before we design, I need to scope the work:
> 1. Does this feature expose or consume API endpoints or external interfaces?
> 2. Does it introduce new data entities or change the shape of existing ones?
> 3. Does it require an architectural pattern not already established in this codebase?
> 4. Does it drive the platform imperatively — scroll, focus, layout, navigation — while a user interaction or another operation is already in flight?"

Question 4 is about the feature's **own** events, not the user's: code that scrolls or refocuses mid-gesture raises events the platform routes through the same handling as the user's input, and can end the interaction it serves. If the answer is yes or unsure, name the platform mechanism that receives those events in **Elaboration Constraints**, with a pointer into the framework source.

Record the answers. Use this to decide which steps to run:

| Answer | Steps to include |
|---|---|
| API: yes or unsure | Step 3 |
| Data model: yes or unsure | Step 4 |
| Architectural pattern: yes or unsure | Step 5 |
| Platform events: yes or unsure | An Elaboration Constraints entry naming the mechanism that receives the feature's own events |

Skip any step answered definitively "no". For any "unsure", include the step and mark the relevant section as provisional in the design artifact.

If all four are "no", confirm with the engineer:

> "This intent doesn't appear to introduce new interfaces, data entities, or architectural patterns — the design foundation is inherited from existing conventions. Shall I move straight to unit decomposition?"

If confirmed, skip to Step 6 (no design artifact is needed).

**When a design is supplied** — a mockup, prototype, artboard or handoff bundle — two more checks, whichever steps run:

- **Read its source, not its picture.** Extract colours, sizes, radii and spacing from the markup or design file and map each to the codebase's tokens. A rendered frame and its prose agree closely enough to feel like confirmation, so reading the picture yields confident, wrong numbers rather than visible gaps. Where the design deviates from a repo rule, surface it as a decision for the engineer.
- **Mark each structural premise decided or inherited.** Name the shapes the plan is about to implement — the page has two tabs, the actions are a row, the list is a sheet — and say whether each is on the record (intent, ADR, engineer sign-off) or merely present in the artifact. Inherited premises go to the engineer now, while overturning one costs a sentence rather than a bolt.

---

## Step 3 — API Contract

*Run only if Step 2 scoped in API endpoints or external interfaces.*

Work through each interface one at a time. **Never work on more than one endpoint per turn.**

For each endpoint, ask the following in sequence — one question per turn:

1. > "Describe this endpoint in one sentence — what triggers it and what does the caller get back?"

2. > "What is the method and path? (or the equivalent if this is not a REST interface)"

3. > "What does the request contain? List the field names and types. Mark any that are optional — and for each optional field, say what the server does when it is absent and what it does when it is empty."

   These are two different facts. When the plan splits the seam into a server unit and a client unit, each author otherwise decides one of them alone.

4. > "What does a successful response look like? List the fields and types."

5. > "What error states must the caller handle?"

6. > "Who can call this? What authentication or authorization does it require?"

After each endpoint, ask:

> "Is there another endpoint to design, or is the contract complete?"

**Domain term check:** Read `process-onboarding-agent/guidelines/domain-glossary.md` if it exists. If any field name is a synonym or informal variant of a glossary term, flag it before moving on:

> "The field [name] looks like a synonym for [glossary term]. Should I use [glossary term] to stay consistent with the domain language?"

**Named-act check:** A contract line describing something a person chose, selected, approved or confirmed must name the surface where that happens, or say it is out of scope and which unit owns it. A noun with no referent passes "every AC is testable", because each reader supplies the missing mechanism from their own head.

---

## Step 4 — Data Model

*Run only if Step 2 scoped in new or changed data entities.*

Work through each entity one at a time. **Never work on more than one entity per turn.**

For each entity, ask the following in sequence — one question per turn:

1. > "Name this entity and describe what it represents in one sentence."

2. > "What are its fields? List each field's name, type, and whether it is required or optional."

3. > "How does it relate to existing entities?"

4. > "Are there any uniqueness constraints, indexes, or a soft-delete pattern?"

5. > "Is any field derived or computed rather than stored? If so, which ones and how?"

After each entity, ask:

> "Is there another entity to design, or is the data model complete?"

**Precedent check:** Before closing the data model, for every field whose nullability, default or "means none" representation you are about to decide, search the existing schema and migrations for a field that already answers the same question. Follow it, or state why this case differs — a convention settled in a migration comment is not an ADR, so the conflict check below will not see it.

**A claim about what an existing module contains is a precedent claim too.** Any design sentence saying what another module includes, returns, walks or exposes cites the file and line it was read from. Where it was not read, write the weaker sentence you can support, or make reading it the unit's first check — a sentence from memory becomes an AC that cannot be built.

**Shared-interface check:** Read `process-onboarding-agent/ops/inception/dependency-map.md` § Shared Interfaces and, for every entity this design creates, name each row that reaches it — a row reading "every intent that adds X" binds this design as soon as it adds an X. Write those rows into Elaboration Constraints and give the unit that adds the entity the AC the row requires. The map is otherwise read only at sign-off, after every AC is fixed.

**Conflict check:** Before closing the data model, read `process-onboarding-agent/rules/architecture.md`. If any proposed entity name, field name, or relationship contradicts an existing ADR, surface the conflict immediately:

> "The proposed [entity or field] conflicts with ADR-[N] which states [decision]. How should we resolve this before we continue?"

Do not proceed to Step 5 until all conflicts are resolved.

---

## Step 5 — Architectural Patterns

*Run only if Step 2 scoped in new architectural patterns.*

For each new pattern, ask the following in sequence — one question per turn:

1. > "Describe the pattern in one sentence — what problem does it solve and where does it apply?"

2. > "Is there an existing pattern in this codebase that is similar? If yes, why is it insufficient for this case?"

3. > "What are the trade-offs of this approach versus the most obvious alternative?"

4. > "Should this become an ADR?"

If the engineer confirms an ADR, draft it immediately and present it for confirmation before writing:

```markdown
### ADR-[next number] — [Decision title]
**Decision:** [what was decided]
**Why:** [the reasoning]
**Trade-off:** [what you gave up]
```

Write a trade-off that leaves a case deliberately unserved as the exact user action that meets it, so UAT can turn it into a step.

Write confirmed ADRs to `process-onboarding-agent/rules/architecture.md` before moving to the next pattern.

After each pattern, ask:

> "Is there another pattern to document, or is this step complete?"

---

## Step 6 — Design Sign-off and Artifact

Before presenting the summary, check the premises the design rests on:

- **Environment-gated behaviour states when the variable is read.** For any behaviour "enabled when `X` is set", say whether `X` is read at build time or at request time, and prove it by running the built artifact with `X` unset. The answer decides where the variable is set, the order of deploy steps, and which test is true.
- **A measurement of an external system uses an instrument that can falsify it, and a sample drawn from the users' own sources.** If the premise is "the provider treats a server differently from a browser", measure in a browser — a tool the provider refuses cannot tell a refusal from an answer. A probe that lands on a 404 measures routing and is discarded, not counted. Put the sites, services and apps the users actually use into the sample first; a category sample answers "does this work in general?", and a user's own source that fails is a design decision for the engineer, not a footnote.

Present a design summary before writing any files:

```
Design Foundation — [Intent name]

API Contract:    [N endpoint(s): list METHOD /path]
Data Model:      [N entity/entities: list names]
Patterns:        [N pattern(s): list names and ADR numbers]
Provisional:     [anything marked unsure, or "none"]

Shall I record this as the design artifact and move to unit decomposition?
```

If the engineer confirms, create the design artifact at `process-onboarding-agent/ops/inception/designs/YYYY-MM-DD-<unix_timestamp>-[intent-slug]-design.md` and link it from the intent file under a `Design:` field in the intent header.

The artifact uses this structure:

```markdown
# Design: [Intent name]

**Status:** Agreed
**Date:** YYYY-MM-DD
**Intent:** [relative link to intent file]
**Elaboration:** _(to be linked after elaboration runs)_

---

## API Contract

### [Endpoint name] — [METHOD /path]

**Auth:** [requirement]

**Request:**
| Field | Type | Required | Notes |
|---|---|---|---|

**Response (success):**
| Field | Type | Notes |
|---|---|---|

**Error states:**
| Status | Condition |
|---|---|

---

## Data Model

### [Entity name]

[one-sentence description]

**Fields:**
| Field | Type | Required | Notes |
|---|---|---|---|

**Relationships:** [description]
**Constraints:** [or "none"]

---

## Architectural Patterns

### [Pattern name]

[description and trade-off]
**ADR:** [ADR-N, or "no ADR created"]

---

## Elaboration Constraints

The following decisions are binding during unit decomposition.
ACs must not contradict them. Surface any conflict before writing a unit — do not work around a constraint silently.

- [one binding decision per bullet]
```

---

## Step 7 — Handoff to Unit Decomposition

Once the design artifact is written (or confirmed as not needed), state clearly:

> "Design foundation is set. I'll now propose units — one at a time — starting with the first logical slice of [intent name]."

Continue directly into the mob elaboration turn structure from `mob-elab-prompts.md`. Do not pause for engineer acknowledgment before proposing the first unit.
