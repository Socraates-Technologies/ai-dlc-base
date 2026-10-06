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

**And when the answer names a view controller, window, root or container, name WHICH INSTANCE.**
*"iOS asks the view controller's `supportedInterfaceOrientations`"* is a correct answer that hides the
whole defect: the controller actually asked was the modal host's own controller, whose phone default
is portrait-only, and the feature's app-level setting could not reach it. A mechanism named in the
abstract is not yet a mechanism located. Write down **which instance is topmost at the moment the
feature runs**, and read that instance's default out of the library's source rather than assuming it
inherits — a modal, an embedded web view, a gesture root and a navigation container each carry their
own copy of whatever the app configured globally. *(Maestro, 2026-09-23: landscape video viewers
shipped inert. It was the fourth defect in that repo with this shape.)*

**And for every new or moved presented surface (a sheet, modal or dialog), name WHERE IT MOUNTS in
its presenter's component tree, and which ancestors wrap it.** The instance question above is the
native half: what the modal does *not* inherit. This is the component-tree half: what still reaches
in, because in many UI frameworks a modal is separate from the native view tree and nested in the
component tree at the same time. An ancestor scroller can take its touches, an ancestor that switches
layout at a breakpoint can remount it, and an ancestor gesture handler can claim its gestures. The
default answer is **beside** every scroller, at the presenter's root. Where the stack allows it, a
test that fails on a presented surface mounted inside a scroller makes the answer permanent.
*(Maestro, 2026-09-30: onboarding sheets on the home screen sat inside a scroller for seven weeks,
with every first tap swallowed, because their placement read as layout.)*

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
- **Check every data field it shows exists.** For each concrete field the design displays ("date · duration · uploader"), confirm it is in the relevant API response or derivable on the client. If not, decide now: omit it, derive it, or open a follow-on task. It must not surface for the first time mid-implementation.
- **Check every asset it specifies is achievable.** Separate what existing code or configuration can produce (a background colour) from what needs a **generated asset** (a composed splash graphic, a new icon set), and name each of the latter as a follow-on task. Record unmet items in the design summary's "Provisional" line; never resolve them silently during the build.

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

**A picker or dialog the PLATFORM owns is designed from what the platform will LIST, and from the filter its API offers.** A browser's device or port chooser, an OS file dialog, a native share sheet or contact picker, a permission prompt: the product cannot style it or reorder it, and a handoff tends to draw it with the one right entry in it. The real list is whatever the user's machine holds. So for every such picker the feature consumes, treat it as an external interface and answer two questions in this step, recorded in the design artifact beside the call that opens it:

> "On a typical user's machine, what will this picker list?"
> "Which filter or option does its API offer to narrow that list to what the thing can actually be — and are we passing it?"

*(makerclub, 2026-10-05: the handoff drew a serial-port picker holding the dev kit alone; the browser listed seven ports, six of them Bluetooth speakers, and the engineer's first walk asked for the filter the API had offered all along.)*
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
**Vendor claims:** [each thing a vendor is relied on to do, with the date it was checked — or "none"]
```

**An ADR that names what a VENDOR will do carries the date that was checked, exactly like a version pin.**
A scaling behaviour, a retention window, a quota, a protocol version, a tool's compatibility: each is a claim
about somebody else's system, and it can be wrong on the day it is written or become wrong later. An undated
mechanism is a belief, and beliefs are what the next unit builds on. If the claim was not checked, the ADR
says so, and checking it is the first thing the unit that depends on it does.
Write a trade-off that leaves a case deliberately unserved as the exact user action that meets it, so UAT can turn it into a step.

Write confirmed ADRs to `process-onboarding-agent/rules/architecture.md` before moving to the next pattern.

After each pattern, ask:

> "Is there another pattern to document, or is this step complete?"

---

## Step 5.5 — Where new code will LIVE, and what its host already decides

Before sign-off, any part of the design that says **where** new code goes gets two checks. The first is ordinary: a location claim is a claim about the tree, so verify it — who already imports the module you named, and does the codebase have a convention for this kind of thing? A location claim reads as incidental detail, so it **inherits the credibility of the verified sections beside it** and nobody checks it. Where a location cannot be verified in the session, mark it provisional and say the executing unit decides it — an unmarked wrong location is followed.

The second is the one designs miss. **When the location is INSIDE an existing component, read that component's own render conditions — its early returns, its guards, the props it keys them on.** The first check asks whether the module can legally host the code. This asks a different question, one altitude up: *for which states does the host render at all?* A component is not a neutral container — it already decided who it appears for, and a new control inherits that decision silently.

Worked example (Ascent, 2026-08-12), a near-miss caught only because implementation looked. A design placed a "Remove" control inside an existing per-row controls component, correctly naming the file, its props and its client/server status. That component opens `if (!canResendInvite && !canRevokeAccess) return null` — and **both are false for a person with no platform access**, which is precisely the most removable kind of record (a test entry, a duplicate from an import). Following the design as written would have shipped a control that appeared for every row EXCEPT the ones it existed for, and a test written against the obvious fixture would have passed.

Note why review does not catch this: the design is *right about everything it says*. Nothing in "add a Remove control to `<Component>`, which takes `{id, status}` and is already permission-gated" is false. The omission is a fact about the host that the design never had a slot for. So the check is mechanical — **quote the host's guard conditions into the design, and state which of them the new affordance needs changed** — and where the answer is "none", say so, because "the early return is unchanged" is a claim worth having on the record.

Generalises past components to any host with an admission rule: a route with a redirect gate, a menu that renders per status, a card that hides when empty.

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

**A number written into the design artifact is computed by a command, and the command is kept beside it.** A count, a distance, a ratio — anything the artifact states as measured — reads as measured whether or not it was, and the next reader reasons from it. Keep the one-liner that produced it next to it, so it is re-run rather than trusted.
*(makerclub, 2026-09-24: a design's values review said "17 have a token" and "within 6/255 of every tint" beside a scripted extraction; the numbers themselves were made in the head, and were 18 and 8.)*

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

**A constraint about USER-FACING COPY is written against the copy that already exists on the adjacent surfaces — never against the principle alone.** A copy rule derived from a doctrine reads as rigorous and can forbid the product's own best sentence, because the doctrine is about meaning and the constraint gets written about words.

Worked example (Ascent, 2026-08-12). An ADR recorded that withdrawing a document destroys nothing, and that *"the surface copy must say so plainly, or a user may believe they destroyed a file they did not."* The constraint written from it became **"the word *delete* appears nowhere"**, which travelled into the design, the plan and an AC and was reviewed three times. A sibling dialog two tabs away says **"nothing is deleted"** — the exact denial the ADR asks for, and a phrase the constraint forbade. It took *writing the copy* to notice, and the AC had to be amended mid-execution.

So: before writing a copy constraint, **grep the adjacent surfaces for the phrase you are about to rule on** and quote what you find into the constraint. Then state the rule as a claim about MEANING with the mechanism named — "must not assert deletion, and must state the denial explicitly, as `<sibling surface>` already does" — rather than as a banned token. A constraint phrased as a word ban is testable and wrong; one phrased as a claim needs a slightly cleverer guard and is right.

The general form: **a rule about one fact, written twice from different starting points, will eventually contradict itself.** The surfaces are the source of truth for how a product says a thing; the ADR is the source of truth for what must be true.

---

## Step 7 — Handoff to Unit Decomposition

Once the design artifact is written (or confirmed as not needed), state clearly:

> "Design foundation is set. I'll now propose units — one at a time — starting with the first logical slice of [intent name]."

Continue directly into the mob elaboration turn structure from `mob-elab-prompts.md`. Do not pause for engineer acknowledgment before proposing the first unit.

---

## Step 8 — After sign-off: a design a unit overrules is CORRECTED, and a review that asks for decisions is CLOSED

The design artifact authorises every unit built against it, so it must keep describing the software that is actually running. Two failures of the same kind — a document that authorised work and then went stale under it — are guarded against here.

**A unit that overrules a signed design amends the design artifact in its own commit.** A unit may find that a design row contradicts an ADR, or that a value in it would be wrong in production, and decide against the design. That call may well be right. But the unit file records only *that* it changed something; the design is the only place that records *what it now is*. So the same commit that ships the overruling code edits the affected row of the design artifact — the new value, one line on why, and a link to the unit — and the unit's review checks that it did. *(makerclub, 2026-09-21: two signed design rows — an error contract, and a rate limit that would have locked a whole classroom out on one child's typos — were overruled correctly inside units' pre-generation checks, and four units later the design still said the old thing, so every later reader inherited a document describing software nobody was running.)*

**A review that asks the engineer for decisions carries a decision line per item, and is closed only when the last one lands.** A design review, a handoff review or a contradiction list raised against signed decisions gets a per-item state table — item, recommendation, decision (accepted / rejected / open), who decided, date — and its header says _Closed_ only when no item is open. A recommendation that is followed without being decided leaves code that is right by luck: correct today, with nothing to say why, and nothing to stop the next reader taking the other branch. **An undecided item older than the units built from it is a blocker, not a footnote** — raise it with the engineer before the next unit in that area starts. *(makerclub, 2026-09-21: a review raised four contradictions; three were decided the same day, and the fourth was recommended on, obeyed and never accepted — its review file still read "not yet accepted, no code has been written from it" long after a unit had built the editor from it.)*
