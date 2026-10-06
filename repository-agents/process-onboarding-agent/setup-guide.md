# AI-DLC Setup Guide

This document explains how to set up the AI-DLC framework in a repository. It works with Claude Code, Cursor, and GitHub Copilot. Everything here is generic — substitute your project's technology stack, domain terms, and rules where indicated.

---

## Framework Root (`{FRAMEWORK_ROOT}`)

Throughout this guide, `{FRAMEWORK_ROOT}` refers to the installation path determined during the **Preliminary Step** of `process-onboarding-agent/onboard.md`. It is set to:

- `{existing-docs-folder}/intent-execution-framework` — if the engineer has an existing process/docs folder, or
- `docs/process/intent-execution-framework` — if no such folder exists yet

All files created by this guide are placed under `{FRAMEWORK_ROOT}/`. Do not create any framework files under `process-onboarding-agent/` — that folder is the bootstrap agent and is deleted after onboarding.

---

## What You Are Building

AI-DLC is a structured operating system for building software with AI assistance. It is not a tool — it is a set of files, conventions, and ceremonies that govern how your team and your AI work together. The output is a repo where:

- Every feature starts with a written intent and testable acceptance criteria
- Every AI interaction is gated by a quality check and logged for audit
- Every failure feeds back into rules that prevent recurrence
- Any engineer (or AI tool) opening the repo knows exactly how to work in it

---

## Before You Begin

### Question 1 — Which AI tool are you using?

This guide uses the term **master rule file** to refer to the file that governs AI behaviour in your repository. Each tool has a different name and location for this file:

| AI Tool | Master rule file | Location |
|---|---|---|
| **Claude Code** | `CLAUDE.md` | Repo root |
| **Cursor** | `.cursor/rules/project-rules.mdc` | `.cursor/rules/` (with `alwaysApply: true`) |
| **GitHub Copilot** | `copilot-instructions.md` | `.github/` folder |

The content of the master rule file is identical across tools. The only differences are the file name, the location, and: for Cursor, the body is wrapped in a `---\nalwaysApply: true\n---` YAML frontmatter block; for GitHub Copilot, internal links to `{FRAMEWORK_ROOT}/` files must use the prefix `../{FRAMEWORK_ROOT}/` since the file lives inside `.github/`.

All subsequent steps in this guide refer to the "master rule file." Substitute the correct name and path for your chosen tool.

> If you want to support more than one AI tool in the same repo, write the master rule file once and create copies for each additional tool. See Step 8.

---

### Question 2 — Fresh or Mature Project?

Before following this guide, the AI must ask the engineer one question:

> **"Is this a fresh project (no significant codebase yet) or a mature project (existing code, conventions, and team practices already in place)?"**

- **Fresh project** → proceed directly to Step 1 below.
- **Mature project** → complete the Mature Project Onboarding phases (Phase M1–M3) first, then continue to Step 1.

---

## Mature Project Onboarding

Onboarding AI-DLC into an existing codebase requires three preparatory phases before any Bolt is written. The goal is to extract what the system already is, encode it as rules, and establish safety controls — so that AI-DLC enhances the project without disrupting what already works.

---

### Phase M1 — Codebase Archaeology Analysis

#### M1.0 — Scope Agreement (Required Before Any Analysis)

Large codebases cannot be analysed in a single context window. Before reading any code, the agent must ask the engineer to define the analysis scope.

Ask the engineer:

> "This codebase may be too large to analyse in one pass without overloading context. To keep each analysis pass focused and accurate, I'd like to work through it in segments.
>
> Please tell me:
> 1. Which modules, services, or folders are the highest priority for AI-DLC onboarding?
> 2. Are there any areas I should skip entirely for now (e.g. legacy code not being actively worked on, third-party code, generated files)?
> 3. Should I analyse one segment at a time and report findings before moving to the next, or would you prefer a summary across all agreed segments?"

Record the engineer's answers. Use them to define **analysis segments** — named, bounded slices of the codebase (e.g. "auth service", "payments module", "shared UI components"). Each segment is analysed independently across M1.1–M1.4, with findings reported to the engineer before moving to the next segment.

**Rules for segmented analysis:**
- Never read beyond the agreed segment boundary in a single pass
- After completing each segment, present a findings summary and ask the engineer to confirm before continuing to the next segment
- If a segment itself is too large for one context window, ask the engineer to break it down further before proceeding
- Keep a running segment log: `Segment | Status | Key findings` — update it after each segment completes

Only proceed to M1.1 once at least one segment is agreed.

#### M1.0-P — Parallel Archaeology (Optional)

After segments are agreed, offer the parallel option:

> "You have [N] segments to analyze. I can work through them one by one in this session, or — if other engineers are available — each person can run the analysis for their segment independently on their own machine and share a structured report back to you. Running segments in parallel has two advantages:
>
> 1. **Speed** — all segments are analyzed simultaneously rather than sequentially.
> 2. **Accuracy** — engineers who work in a specific module every day will surface patterns and risks that a cold analysis might miss. Independent findings that match across sessions have higher confidence than single-session findings.
>
> Would you like to run this archaeology in parallel? If yes, I'll give you a segment assignment brief for each engineer to copy to their own session."

If the engineer says **yes**:

1. Produce a **Segment Assignment Brief** for each segment — a short block the engineer can paste into another AI session to kick off a parallel analysis. Format:

   ```
   Segment Assignment: [Segment name]
   ─────────────────────────────────────────────────
   You are running a codebase archaeology analysis for one segment of a larger project.
   Your scope is limited to: [folders/modules]
   Do not read outside this boundary.

   Run phases M1.1, M1.2, M1.3, and M1.4 from process-onboarding-agent/setup-guide.md for this segment only.
   When complete, produce a Segment Report using the format defined in Phase M1.5 of the setup guide.
   The engineer will copy your Segment Report back to the main session for synthesis.
   ─────────────────────────────────────────────────
   ```

2. Assign each segment to an engineer. If the number of engineers is fewer than the number of segments, assign multiple segments to one engineer — they run them sequentially in their session.

3. Tell the main engineer:

   > "Share each brief with the assigned engineer. When each parallel session is complete, paste its Segment Report back here. I'll run Phase M1.5 to synthesize all reports into the final archaeology output."

4. Pause the main session's M1.1–M1.4 analysis — do not begin reading code in the main session. The main session resumes at Phase M1.5 once all Segment Reports are received.

If the engineer says **no**, proceed directly to M1.1 in the current session.

---

#### M1.1 — Architecture Mapping

- Read the codebase module by module (within the agreed segment)
- Generate business-language descriptions of each module (what it does, not how)
- Produce a capability map: what the system does as a whole
- Identify service boundaries, integration points, and data flows
- Output becomes the foundation for the Bolt Backlog

#### M1.2 — Pattern Extraction

- Read 10–20 representative files across modules
- Extract: naming conventions, error handling style, ORM usage, API response shapes, authentication patterns, test patterns and coverage conventions
- Document all extracted patterns — these go into the master rule file and the `rules/` folder before any Bolt is written

#### M1.3 — Due Diligence Audit

Before classifying work, audit the existing codebase for defects and structural problems that new AI-generated code could inherit. This step must complete before any Bolt is planned.

**What to look for:**

| Category | Examples |
|---|---|
| **Logic defects** | Incorrect business logic, off-by-one errors, wrong conditional branches, silent data loss |
| **Design violations** | Responsibilities mixed across layers, circular dependencies, God classes or functions doing too much |
| **Security gaps** | Unvalidated input, missing auth checks, secrets in code, direct DB calls from the wrong layer |
| **Fragile patterns** | Catch-all error suppression, hardcoded values that should be config, mutable shared state |
| **Test blind spots** | Code paths with no test coverage; tests that assert implementation details rather than behaviour |
| **Consistency breaks** | Naming or structural conventions that differ across modules with no documented reason |

**Output:** A ranked list of findings. For each finding, record:
- Location (file/module/function)
- Category from the table above
- Impact if inherited by new code
- Recommended correction (fix-in-place, quarantine, or encode as a prohibition in the master rule file)

Present the list to the engineer and agree on which items must be corrected before the first Bolt runs and which can be logged as Remediation Bolts.

#### M1.4 — Debt and Gap Mapping

Classify all identified work — including findings from M1.3 — into three Bolt types:

| Bolt Type | Description |
|---|---|
| **Enhancement Bolt** | New capabilities not yet in the system |
| **Remediation Bolt** | Tech debt, coverage gaps, refactoring, and defects found in M1.3 |
| **Migration Bolt** | Architectural changes that enable future work |

**Prioritisation order:** Remediation of blocking defects found in M1.3 first → Enhancement (shows value) → Remediation of non-blocking debt → Migration last.

---

#### M1.5 — Parallel Synthesis (Only runs if M1.0-P parallel mode was chosen)

This phase runs in the **main session** once all parallel engineers have returned their Segment Reports. Do not start synthesis until every assigned segment has a report.

##### Segment Report Format

Each parallel session must produce a report in this exact structure so the main session can parse and synthesize it consistently:

```markdown
# Segment Report: [Segment name]

**Analyst:** [Engineer name]
**AI tool:** [Claude Code / Cursor / GitHub Copilot]
**Date:** YYYY-MM-DD
**Scope:** [folders and modules covered]
**Confidence:** High / Medium / Low
(High = engineer is very familiar with this module;
 Medium = some familiarity;
 Low = cold analysis only, no domain knowledge applied)

---

## M1.1 — Architecture

[Business-language description of each module in scope. What it does, not how.]

**Service boundaries identified:**
- [boundary description]

**Integration points:**
- [integration description]

**Data flows:**
- [flow description]

---

## M1.2 — Patterns Extracted

| Pattern type | Observed convention | Example location |
|---|---|---|
| Naming | [convention] | [file/module] |
| Error handling | [convention] | [file/module] |
| API response shape | [convention] | [file/module] |
| Auth pattern | [convention] | [file/module] |
| Test pattern | [convention] | [file/module] |
| ORM / DB access | [convention] | [file/module] |

---

## M1.3 — Due Diligence Findings

| # | Location | Category | Description | Impact | Recommendation |
|---|---|---|---|---|---|
| 1 | [file/function] | [category] | [description] | [impact] | Fix-in-place / Quarantine / Encode as prohibition |

---

## M1.4 — Debt Classification

| Work item | Bolt type | Priority | Notes |
|---|---|---|---|
| [description] | Enhancement / Remediation / Migration | High / Med / Low | [notes] |

---

## Confidence Notes

[Anything the analyst was uncertain about, code paths not covered, or areas where a second opinion is recommended.]
```

##### Synthesis Protocol

Once all Segment Reports are received:

**Step 1 — Validate completeness.** Check that every agreed segment has a report. If any segment is missing, do not proceed — ask the engineer to follow up with the assigned analyst.

**Step 2 — Merge architecture findings.** Combine the M1.1 sections across all reports into a single capability map. Identify: modules that appear in multiple reports (cross-segment dependencies), integration points that span segment boundaries, and data flows that pass through more than one segment.

**Step 3 — Reconcile patterns.** For each pattern type in M1.2, compare findings across reports:

| Situation | Action |
|---|---|
| Same convention found in 2+ segments | Mark as **Confirmed convention** — high confidence for the master rule file |
| Different conventions found for the same pattern type | Mark as **Inconsistency** — flag to engineer for resolution before writing rules |
| Convention found in only one segment | Mark as **Unconfirmed** — note the segment scope and treat with lower confidence |

**Step 4 — Consolidate due diligence findings.** Merge all M1.3 findings into a single ranked list. Apply a confidence boost to any finding that appears in more than one report independently — these have been confirmed by multiple analysts without coordination and are high-priority. Flag any finding where two reports directly contradict each other (e.g. one reports a pattern as consistent, another reports it as absent) — these require engineer resolution.

**Step 5 — Unify debt classification.** Merge all M1.4 items. De-duplicate by description. Where two analysts classified the same work item differently (e.g. one called it Remediation, another called it Migration), present both classifications to the engineer and ask for a decision.

**Step 6 — Present the synthesis report.** Before proceeding to Phase M2, present:

```
Parallel Archaeology Synthesis
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Segments analyzed:    [N] of [N] assigned
Analysts:             [names]
Combined confidence:  [summary]

Confirmed conventions:     [N]  (found in 2+ segments independently)
Inconsistencies to resolve: [N]  (conflicting findings — need engineer decision)
High-confidence findings:  [N]  (due diligence items confirmed by 2+ analysts)

Items needing engineer resolution before Phase M2:
  [list each inconsistency or conflict with both positions stated]
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

Resolve all conflicts with the engineer before writing any Phase M2 files. Then continue to Phase M2 using the synthesized output as the archaeology result.

---

### Phase M2 — Repository Overlay

Create the governance layer on top of the existing codebase. All five artefacts must exist before the first Bolt runs.

#### M2.1 — Master Rule File

The most important file in the overlay. See **Before You Begin** for the correct file name and location for your AI tool. Must contain:
- Architecture context derived from M1.1
- Coding conventions extracted in M1.2
- Forbidden zones (see M2.2)
- Bolt workflow rules
- Entry points (see M2.3)

Follow the full master rule file authoring instructions in Step 2 of this guide, but populate each section from the archaeology output rather than from scratch.

#### M2.2 — Forbidden Zones

Create `{FRAMEWORK_ROOT}/guidelines/forbidden-zones.md` containing an explicit list of files, modules, or patterns that the AI must **not** modify without senior engineer approval. This protects stable, business-critical code from accidental change.

```markdown
| Zone | Path / Pattern | Reason | Approval required from |
|---|---|---|---|
| <name> | <path or glob> | <why it is protected> | <role> |
```

Reference this file in the master rule file under Code Rules (Section 3) so it is enforced in every session.

#### M2.3 — Entry Point Registry

Create `{FRAMEWORK_ROOT}/guidelines/entry-points.md` containing the approved list of modules and features where AI-DLC Bolts may begin. This controls the expansion boundary. Update the list as the team gains confidence with the process.

```markdown
| Module / Feature | Status | Notes |
|---|---|---|
| <name> | Approved / Pending / Blocked | <any constraints> |
```

#### M2.4 — Coding Conventions File

Create `{FRAMEWORK_ROOT}/rules/code-standards.md` (or populate it if it already exists) entirely from the patterns extracted in M1.2. Every AI session must match the style of the existing codebase — this file is the authoritative source injected into each session.

---

#### M2.5 — Seed Codebase Findings

Copy `_template.md` and `README.md` verbatim from `process-onboarding-agent/ops/inception/codebase-findings/` to `{FRAMEWORK_ROOT}/ops/inception/codebase-findings/`. Then, for each segment analyzed in Phase M1, create one finding file from `_template.md` (e.g. `payments-service.md`) populated from that segment's M1.1–M1.4 output:
- **Summary** — from the segment's architecture mapping (M1.1)
- **Findings** — one dated entry per segment, attributed to `Initial archaeology (M1)` rather than an intent slug, covering the patterns extracted (M1.2), due diligence findings (M1.3), and debt classification (M1.4)
- **Open Questions** — anything M1 flagged with low confidence or could not resolve from code alone

Add one row per segment to the index table in `README.md`. This turns the archaeology output — which would otherwise live only in this onboarding session — into the persistent starting point every future brownfield intent checks before re-analyzing the same code.

---

### Phase M3 — Blast Radius Management

Before any Bolt is executed, three safety controls must be in place. These apply to every Bolt that touches existing code.

#### M3.1 — Test Coverage Gate

Verify that adequate test coverage exists for the target feature or module before a Bolt begins. If coverage is insufficient, a Remediation Bolt to add tests must be planned and executed first. No Bolt modifying existing code starts without this gate passing.

#### M3.2 — Feature Flag Requirement

Every change introduced by AI-DLC into an existing module must be wrapped in a feature flag. This limits production impact if a change behaves unexpectedly and allows rollback without a deployment.

#### M3.3 — Default Acceptance Criterion for Existing-Code Bolts

Every Bolt that affects existing code must carry the following AC by default. It has two forms depending on Bolt classification:

**Standard form** (Enhancement Bolts and all Bolts not classified as Migration or Remediation):
> **"All integration tests for [affected module] pass without modification."**

This AC is non-negotiable and cannot be removed during elaboration.

**Contract-change form** (Migration Bolts and Remediation Bolts that explicitly change contract boundaries — e.g. API shapes, data schemas, inter-module interfaces):
> **"All integration tests for [affected module] pass without modification, except for tests that cover the contract boundaries listed as breaking changes below. Each breaking change must be detailed and approved in the elaboration session before any code is generated."**

When the contract-change form applies, the elaboration session must produce a **Breaking Changes Register** — a table attached to the unit file listing every changed contract boundary, the reason it must change, and the name of the engineer who approved it. No unit using the contract-change form may be executed without a completed and approved Breaking Changes Register.

Add both forms to the master rule file Section 6 (AI-DLC Workflow) and to `{FRAMEWORK_ROOT}/rules/code-standards.md` so the correct form is enforced automatically based on Bolt classification.

---

### Mature Project Onboarding Checklist

- [ ] Architecture map produced (capability map, service boundaries, data flows)
- [ ] Pattern extraction complete (naming, error handling, ORM, API shapes, test conventions)
- [ ] Debt and gap map produced; work classified into Enhancement / Remediation / Migration Bolts
- [ ] Master rule file written from archaeology output (see Before You Begin for file name)
- [ ] `{FRAMEWORK_ROOT}/guidelines/forbidden-zones.md` created and referenced in the master rule file
- [ ] `{FRAMEWORK_ROOT}/guidelines/entry-points.md` created
- [ ] `{FRAMEWORK_ROOT}/rules/code-standards.md` populated from extracted patterns
- [ ] `{FRAMEWORK_ROOT}/ops/inception/codebase-findings/` seeded with one file per M1 segment, indexed in `README.md`
- [ ] Test coverage gate verified for first target module
- [ ] Feature flag approach confirmed with team
- [ ] Default AC for existing-code Bolts added to the master rule file and `code-standards.md`

Once all items above are checked, continue to **Step 1** of this guide to create the full folder structure and remaining artifacts.

---

## Fresh Project — Structured Interview

For fresh projects the agent cannot read a codebase to populate the master rule file. Instead, it must run a structured interview before creating any files. The interview is strictly one question at a time, in this order. The agent must not proceed to the next question until the engineer has answered the current one.

**Do not present all questions at once. Ask them one at a time and wait for the answer.**

---

**Interview sequence:**

1. **Product identity**
   > "In one sentence, what does this product do and who uses it?"

2. **Technology stack — backend**
   > "What language and framework will the backend use? (e.g., Node/Express, Python/FastAPI, .NET/ASP.NET Core)"

3. **Technology stack — frontend**
   > "What language and framework will the frontend use? (e.g., React/TypeScript, Vue, server-rendered only)"

4. **Database and auth**
   > "What database will you use, and how will authentication work? (e.g., PostgreSQL with Supabase Auth, MongoDB with JWT)"

5. **System boundaries**
   > "Are there any hard rules about what must never happen architecturally? (e.g., 'the frontend must never call the database directly', 'the mobile app communicates only through the REST API')"

6. **Domain language**
   > "List the key business terms this system uses — the words that appear in your domain, not generic tech terms. For each one, give a one-sentence definition. (e.g., 'Booking — a confirmed reservation between a guest and a host')"
   >
   > *Keep asking "any more?" until the engineer says done.*

7. **Known constraints and prohibitions**
   > "Are there any technical choices that are already decided and must not be changed by the AI? (e.g., 'we must use REST, not GraphQL', 'no ORM — raw SQL only', 'all prices stored as integers in cents')"

8. **First capability**
   > "What is the first feature or capability you want to build? Give it a name and one sentence describing what it does for the user."

9. **Documentation archive threshold**
   > "AI-DLC generates operational documents over time (retros, improvement files, unit files, bolt files). These can be compacted and archived periodically using the compact-docs skill. How many months old must a document be before it qualifies for archiving? (Common choices: 3, 6, or 12 months. This can be changed later.)"

---

Once all nine questions are answered, the agent has enough to:
- Create the folder structure (Step 1)
- Write the master rule file with Sections 1–5, Section 8, and Process Configuration fully populated
- Write an initial first intent file from the answer to question 8
- Flag Sections 6 and 7 (workflow and review) as pre-populated from the guide defaults

---

## Step 1 — Create the Folder Structure

Create this directory tree at the root of your repository:

```
{FRAMEWORK_ROOT}/
  Instructions2FDE.md        ← main guide for Forward Deployed Engineers (FDEs)
  README.md                  ← how the process achieves quality
  rules/
    prompt-quality-gate.md   ← the 4-component gate run before every code generation
    code-standards.md        ← language/framework conventions and anti-patterns
    security.md              ← never/always security rules
    architecture.md          ← ADRs — decisions and their rationale
    engagement.md            ← engineer engagement monitoring signals and intervention protocol
  skills/
    mob-elab-prompts.md      ← interactive protocol and prompts for elaboration sessions
    review-checklist.md      ← structured lens for reviewing AI output
    compact-docs.md          ← engineer-triggered skill to archive old operational documents
    root-cause-analysis.md   ← skill to analyse incidents and improvements for design, technology, and process gaps
    solution-shaping.md      ← decides generic-vs-specific, simplest-viable approach, and extend-vs-build-vs-buy before design begins
    design-session.md        ← Phase 0 of elaboration — locks API contracts and data model decisions before units are proposed
    bolt-risk-assessment.md  ← blast radius, rollback, and feature flag assessment before a bolt's first unit executes
    progress-digest.md       ← plain-language stakeholder progress summary for a feature intent
    uat.md                   ← acceptance testing protocol; blocks an intent from closing without sign-off
    process-health.md        ← quantitative report on how well the AI-DLC process is functioning
    dependency-audit.md      ← scheduled audit of third-party dependencies by severity
    architecture-review.md   ← monthly read-only whole-codebase health review
    disk-hygiene.md          ← scheduled workstation disk sweep; never touches unmerged work or data
    knowledge-promotion.md   ← classifies retro improvements as generic (promote to base repo) or project-specific
    process-visualization.md ← reconstructs how a bolt actually got delivered as Mermaid diagrams
    new-engineer-induction.md ← walks a new team member through the project's framework
    bug-bolt.md               ← lightweight bolt workflow for fixing a specific, reproducible bug
    hotfix-bolt.md            ← emergency bolt for production incidents
    nfr-bolt.md               ← non-functional quality attribute bolt (performance, security, accessibility)
  guidelines/
    domain-glossary.md       ← canonical business terms used in code and prompts
    edge-cases.md            ← known failure modes to check before generating code
    acceptance-patterns.md   ← rules and anti-patterns for writing ACs
    dev-setup.md             ← environment setup checklist for new engineers
    team-rollout.md          ← scaling to multiple engineers
  prompts/                   ← one file per feature; every AI session logged here
  ops/
    inception/
      intents/               ← one file per feature intent
        _template.md
        README.md
      elaborations/          ← one folder per intent; one file per session
        _template.md
      codebase-findings/     ← one file per module/area; reverse-engineering findings from existing code, accumulated across intents
        _template.md
        README.md
    build/
      backlog.md             ← master status of all units
      id-reservations.md     ← id ledger: unit/bolt/intent/ADR/edge-case ids and migration numbers, each with a "Next free" marker
      units/                 ← one file per atomic unit of work
        _template.md
      bolts/                 ← one file per planned batch of units
        _template.md
    operate/
      retros/                ← one file per completed bolt
        _template.md
      incidents/             ← one file per production issue
        _template.md
      improvements/          ← one file per process change triggered by retro/incident
        _template.md
```

---

## Step 2 — Write the Master Rule File (The Behavioral Contract)

The master rule file is loaded automatically by your AI tool every session. See **Before You Begin** for the correct file name and location for your tool — the content is the same regardless of which tool you use. Every section below is required.

> **Path substitution — critical:** Every template block below contains the placeholder `{FRAMEWORK_ROOT}`. When writing the actual master rule file, replace every occurrence of `{FRAMEWORK_ROOT}` with the real path determined in the Preliminary Step of `process-onboarding-agent/onboard.md`. For example, if `FRAMEWORK_ROOT` = `docs/process/intent-execution-framework`, then `{FRAMEWORK_ROOT}/rules/prompt-quality-gate.md` must be written as `docs/process/intent-execution-framework/rules/prompt-quality-gate.md`. The master rule file must contain real, resolvable paths — never the literal string `{FRAMEWORK_ROOT}`.

### Section 1 — Project Identity

State what the system is and its technology stack. Be specific — the AI needs to know:
- What the product does (one sentence)
- Backend language/framework and key patterns
- Frontend language/framework
- Database and auth approach
- **System boundaries** — what calls what; what must never happen (e.g. "frontend never calls the database directly")

```markdown
## 1. Project Identity

**<ProjectName>** is a <one-sentence description>.
- Backend: <language/framework, key patterns>
- Frontend: <language/framework>
- Database: <database, auth approach>

**System boundaries:**
- <boundary rule 1>
- <boundary rule 2>
```

### Section 2 — Prompt Quality Gate

A two-line routing entry. Do not embed the gate definition — it lives in `rules/prompt-quality-gate.md`.

```markdown
## 2. Prompt Quality Gate

Before every code response: read and enforce `{FRAMEWORK_ROOT}/rules/prompt-quality-gate.md`.
If any of the four components (Context · Constraints · Acceptance Criteria · Output Format) is missing, stop and ask for it. Do not generate code.
```

### Section 3 — Code Rules

Three to five **hard-stop prohibitions** that the AI must have in working memory (no file lookup tolerated) — these are the rules where a single violation causes immediate, serious harm. Then a mandatory-read routing line for everything else.

```markdown
## 3. Code Rules

**Hard stops — memorise, never look up:**
- Never commit secrets, API keys, or connection strings
- Never trust client-supplied IDs without server-side ownership verification
- Never expose internal stack traces to the client
- [1–2 stack-specific absolute prohibitions from the project interview]

**Full rules (read before writing any code):**
- Conventions and patterns: `{FRAMEWORK_ROOT}/rules/code-standards.md`
- Security rules: `{FRAMEWORK_ROOT}/rules/security.md`
- Architecture decisions: `{FRAMEWORK_ROOT}/rules/architecture.md`
```

Keep the hard-stop list to five items maximum. Anything beyond five belongs in `code-standards.md`, not here.

### Section 4 — Domain Language

A mandatory-read instruction plus the two or three terms most likely to cause logic errors if misused. The full glossary lives in `guidelines/domain-glossary.md` — do not reproduce it here.

```markdown
## 4. Domain Language

Read `{FRAMEWORK_ROOT}/guidelines/domain-glossary.md` before every elaboration session and before generating any business logic. Use only the terms defined there — do not substitute synonyms.

**Critical terms (load immediately):**
- **[Term]:** [one-line definition]
- **[Term]:** [one-line definition]
```

Limit inline terms to three. If the project has more critical terms, add them to the glossary file, not to this section.

### Section 5 — Known Edge Cases

A single mandatory-read routing line. No inline table — edge cases are added continuously via the retro loop and must stay in their dedicated file to remain current.

```markdown
## 5. Known Edge Cases

Read `{FRAMEWORK_ROOT}/guidelines/edge-cases.md` before generating code for any unit. Do not skip this step — new edge cases are added after every retro.
```

### Section 6 — AI-DLC Workflow

The turn structure is short enough to hold in working memory — keep it inline. Everything else routes to files.

The delivery and version-control rules at the end of the template are inline too: they govern every commit, push and deploy, so they must be in front of the agent every session rather than behind a routing line. Keep the incident in each one — a rule with no reason is the first one a later session drops. The push and ledger-conflict rules extend the concurrent-sessions rule in `rules/code-standards.md` (Step 3); include them as soon as two sessions ever share the repo.

```markdown
## 6. AI-DLC Workflow

**Session start check:** At the beginning of every session, present the following note to the engineer:
> "At any point during this session, if you have a question about a step, need further clarification, or don't have the exact answer to a question I'm asking — just say so. I'll help you work through it so we don't get blocked."

Then read the `Next dependency audit` date from Section 9 as it stands on the shared remote's main branch, not in the local checkout (fetch, then read the master rule file at `origin/main`). Where sessions run in parallel the checkout is often stale, and another session may already have run the audit or sweep and moved the date. If today is on or after that date, prompt the engineer before any other work:
> "A dependency and security audit is scheduled. Would you like to run it now, or set a new date?"
If the engineer defers, ask for the new date and update Section 9 before continuing.

Then read the `Next disk hygiene sweep` date from Section 9 and apply the same rule. Also run the sweep unscheduled whenever free disk space is observed below the headroom threshold in Section 9.

Then count the improvement files in `{FRAMEWORK_ROOT}/ops/operate/improvements/` whose Status is `Open` (excluding `_template.md`) and report them in three lines, never the full list:
> "**N improvement proposals are Open, the oldest from YYYY-MM-DD.** Three to decide: [expired ones first — past their `Decide by` date — then the oldest]. Would you like to decide any of them now?"

Then grep the backlog for owed hotfix retros (`retro + RCA due YYYY-MM-DD`) dated before today. Say nothing when there are none; otherwise:
> "**A hotfix retro is overdue: [bolt], due YYYY-MM-DD.** Run it now, or say when."

**Elaboration turn structure (strictly one unit per turn):**
1. Propose one unit — name and one-sentence purpose only. Stop.
2. Propose ACs as a numbered list. Stop.
3. Surface edge cases and open questions. Stop.
4. Ask the three observability questions: what confirms this is working in production? What log entry signals failure? What alert threshold makes sense? If the answer represents code behavior, add it as an AC. If not applicable, record "Not applicable" and move on. Stop.
5. Move to next unit. Repeat.
6. After all units agreed, present summary table and ask for sign-off before writing any files.

**Solution shaping:** at the start of every mob elaboration, check the intent for a `## Solution Shape` section. If it is absent and the intent introduces a new capability, a potentially reusable surface, or an expensive-to-reverse decision, ask the engineer once whether to run `{FRAMEWORK_ROOT}/skills/solution-shaping.md` first or proceed straight to design. The engineer decides — run it, skip it, or invoke it directly at any time; never block on it. Skip the prompt entirely for plainly small, feature-specific intents.
**Full elaboration protocol (including design session):** read `{FRAMEWORK_ROOT}/skills/mob-elab-prompts.md` before every elaboration session. The design session runs as Phase 0 of elaboration — it is not invoked separately.
**Codebase findings:** before analyzing existing code to understand a new intent's dependencies on prior implementation, check `{FRAMEWORK_ROOT}/ops/inception/codebase-findings/README.md` for an existing file on that module/area; after any such analysis, record or update the finding there. This is part of the mandatory elaboration protocol above, not a separate skill.
**Bolt risk assessment:** read `{FRAMEWORK_ROOT}/skills/bolt-risk-assessment.md` after elaboration sign-off and before the first unit in a bolt executes. No unit may begin execution without a signed-off risk assessment in the bolt file.
**UAT skill:** read `{FRAMEWORK_ROOT}/skills/uat.md` when all units under an intent are marked Done, or when the engineer invokes it directly. Prompt the engineer to run UAT before setting intent status to Implemented.
**Progress digest skill:** read `{FRAMEWORK_ROOT}/skills/progress-digest.md` when the engineer asks for a stakeholder update, progress summary, or digest for an intent.
**Process health skill:** read `{FRAMEWORK_ROOT}/skills/process-health.md` when the engineer invokes it to audit how well the AI-DLC process is functioning.
**New engineer induction skill:** read `{FRAMEWORK_ROOT}/skills/new-engineer-induction.md` when an engineer says they are new to the project or invokes it directly.
**Knowledge promotion skill:** read `{FRAMEWORK_ROOT}/skills/knowledge-promotion.md` as Step 5 of the Post-Retro Improvement Workflow after all improvements are applied. A retro is not closed until every Applied improvement has a Knowledge Promotion status.
**Process visualization skill:** offer to read `{FRAMEWORK_ROOT}/skills/process-visualization.md` at the start of every retro, before "What Went Well" is discussed. The engineer may accept, skip, or invoke it directly at any time. Never run it without the engineer's go-ahead.
**Dependency audit skill:** read `{FRAMEWORK_ROOT}/skills/dependency-audit.md` when the engineer invokes it, or when the `Next dependency audit` date in Section 9 has been reached. Prompt at session start if the date is due.
**Architecture review skill:** read `{FRAMEWORK_ROOT}/skills/architecture-review.md` when the engineer invokes it, or monthly alongside the dependency audit — offer it whenever the dependency audit is prompted. Read-only: it recommends refactors, it never makes them.
**Disk hygiene skill:** read `{FRAMEWORK_ROOT}/skills/disk-hygiene.md` when the engineer invokes it, when the `Next disk hygiene sweep` date in Section 9 has been reached, or unscheduled whenever free disk space is observed below the headroom threshold in Section 9. Prompt at session start if the date is due.
**Compact-docs skill:** read `{FRAMEWORK_ROOT}/skills/compact-docs.md` when the engineer invokes it.
**Root-cause-analysis skill:** read `{FRAMEWORK_ROOT}/skills/root-cause-analysis.md` when the engineer invokes it, or when an incident is marked Resolved and no RCA has been run on it.
**Bug bolt:** read `{FRAMEWORK_ROOT}/skills/bug-bolt.md` when the engineer says "fix a bug", "there's a bug in X", or "bug: [description]". Do not run a full mob elaboration — follow the bug bolt workflow directly.
**Hotfix bolt:** read `{FRAMEWORK_ROOT}/skills/hotfix-bolt.md` when the engineer says "hotfix", "production issue", "prod is down", or "emergency fix for X". Skip elaboration — begin hotfix intake immediately.
**NFR bolt:** read `{FRAMEWORK_ROOT}/skills/nfr-bolt.md` when the engineer says "improve performance", "harden security", "accessibility improvements", "NFR bolt for X", or "non-functional work on X". Do not create a new intent — follow the NFR bolt workflow.
**Engagement monitoring:** read and apply `{FRAMEWORK_ROOT}/rules/engagement.md` throughout all ceremonies.

**Version control and delivery.** These govern every commit, push and deploy. The hazards are silent — nothing fails, the history is simply wrong — so each is a rule, with its incident kept beside it:
- **The commit's path list is typed, never derived.** Stage explicit paths, never `git add -A`, and name each path you wrote: a list built from `git status`, `git diff --name-only` or a glob is explicit in form and still carries another session's files. Commit with `git commit --only <paths>` (a new file needs `git add -N` first): the index is shared, so a `git diff --cached` review is not atomic with the commit that follows it, and `--only` commits exactly the named paths whatever a peer staged in between. `--only` protects the index, not the file — it commits every hunk in a named file, a peer's unfinished ones included — so review staged changes hunk by hunk for any file you did not edit alone this session; a stat listing does not show a peer's hunk riding in with yours. *(makerclub, 2026-09-23: a peer's `package-lock.json` reached a commit through a list read off `git status`.)*
- **Count the `git status` lines against the files you edited before committing.** Fewer lines than files means a peer's commit absorbed some of yours; record where they landed and do not rewrite the peer's commit. Commit once the scoped tests pass; let the full suite gate the push, not the commit. And because format and lint gates run over the whole repo, run the format gate before every push, so one session's stray unformatted file cannot fail every session's CI.
- **Committing makes you publishable.** Any session's push publishes every commit beneath it. Commit promptly anyway, because uncommitted edits are lost to a peer's rebase or stash; when a commit must not ship yet, make it in a detached worktree. Report a commit with its hash and whether it is pushed — an unpushed commit is visible to every worktree and invisible in chat. When told to pick up another session's work, run `git log --all -- <its files>` first and say which commit you found and whether its author is still running. After reporting a fix other sessions need, `git fetch` again before building it and again before committing: the session you told is often already in that file and lands first.
- **Push from your own worktree, after an ancestor check.** A merge or push run in a tree another session may be working in publishes whatever that tree holds. Work, merge and push in a worktree of your own; before every push run `git fetch && git merge-base --is-ancestor origin/main HEAD`, and stop if it fails; if the push is rejected, rebase and re-gate, never merge. *(makerclub, 2026-09-27: a `merge --ff-only` in the shared checkout refused, the chain carried on, and `git push` published a peer's finished but unpushed commit.)*
- **A ledger conflict is resolved hunk by hunk; "keep both" is mechanical too.** Files several sessions append to (the backlog, the id ledger, the ADR log) conflict on rebase routinely, and your side of the hunk was cut from an older default branch, so it can hold stale copies of a peer's rows. Keep upstream's rows as they are, add only the lines you wrote, and before `rebase --continue` read `git diff origin/main -- <file>`: it must show only your `+` lines, or the resolution is wrong. *(makerclub, 2026-09-28: a "keep both" resolution carried two peer units as "not deployed" while the default branch said deployed.)*
- **Allocate unit, bolt, intent, ADR and edge-case numbers at push time.** Work under placeholders (`U-TBD`, `ADR-TBD`) in filenames, headings and cross-references. Immediately before publishing: fetch, rebase, read the next free numbers at that commit, substitute every placeholder, record the claims in the same commit, push. The push is the only lock a shared repository has, so a lost race becomes a rejected push — re-stamp and retry — instead of a double claim found after other files cite it; reserving at sign-off and re-checking narrows the race but cannot see a rival claim committed and unpushed elsewhere. Never renumber after a successful push. Substitute placeholders by anchored line, never file-wide, then read every changed line of the registry diff and ask of each whether you wrote that placeholder — a substitution count that equals its own pre-count proves the script is consistent, not that it only touched your rows. The ledger (`ops/build/id-reservations.md`) and the stamp are described in `guidelines/dev-setup.md`.
- **A document that a test reads is half a code change, and it lands in the bolt.** Pure documentation (memos, notes) is authored, reviewed and committed. But where a source-of-truth document is parsed by a conformance test, committing the doc change to the trunk ahead of the code that conforms to it leaves the trunk red, which blocks every other session's verify run until the code catches up. Docs-first *ordering* is still right; only the destination changes. Land the doc row and its code in one commit, or keep the doc change on the bolt branch until the conforming unit merges.
- **A CI run that did not START is a red gate — read the default branch's last CI run before every push** (`gh run list -b main -L 1`, or your CI's equivalent). `success`, or a run still in progress, is the only answer that lets the push go ahead unremarked; anything else, read why first. A run that failed in seconds with no steps is the account or the runner, not the code — and it is still a STOP: tell the engineer and record it in the backlog before pushing on the local gate alone. *(makerclub, 2026-09-24: CI refused every run before its first step over billing, and seven pushes by at least three sessions went through it in 45 minutes — a job that never starts has no test output, so it reads to nobody as a failure.)*
- **A script's result is its own exit line, never its wrapper's — and a command that gates another runs in its own tool call.** Read the exit line the script itself prints (e.g. `EXIT <name>: <code>`), not the `[exited with code 0]` a background runner or wrapper prints about it. `;` passes control on regardless, `&&` propagates failure but never judgement, `set -e` in an agent harness's shell may not stop a chain, and a pipe reports its last command's status and hides the exit code of what it pipes (`git rebase … | tail -1` hides a conflict). Read the test summary or the stack listing, then commit or push in a separate call; a multi-step publish script stops on its first failed step. A backgrounded `cmd; echo $?` reports the `echo` — confirm a long-running command from its artifact, a file whose timestamp moved. *(makerclub, 2026-09-27: a runner's "exited with code 0" sat three lines under the script's own "EXIT: 1", twice in one day.)*
- **"Shipped" means reaching a user's device, not merging — so a deploy of the default branch closes the owed-deploy rows of everything it ships, not only its own.** A deploy carries every change pushed since the live tag. At deploy, read `git log --oneline <live tag>..HEAD` against the backlog's rows that owe a deploy, and for each unit it ships either take its observation or mark it _"shipped in `<tag>`, observation owed"_ in the deploy's own commit. **The rows are three: the unit file's Status, its backlog row, and its bolt file's Status line** — the bolt file is the one the next session on that work opens first. *(makerclub, 2026-10-01: a deploy carried a unit whose Definition of Done and backlog row still read "Owed: deploy", and the sweep that followed found 13 more rows already live; on 2026-10-04 a deploy marked the first two rows and left two bolt files saying "not deployed".)*
- **A device observation names the build it was made on, read from that build's own log before it is recorded.** A unit whose behaviour can only be seen on a real device or in a real browser is Done against an observation written into the unit file — date, commit, device, what was done, what was seen — and that observation names the request or job id and the server revision, found in the live log *before* it is written down. A "looks right" with no matching log line on the revision under test is said back to the engineer, not recorded. *(makerclub, 2026-09-28: the engineer's first "it all looked right" came when the log showed no run on the fixed revision at all — it described the old build.)*
```

**Why the Open proposals are read at session start, and only three of them.** Knowledge promotion runs after a retro's improvements are applied, so a proposal still awaiting a decision never reaches it; session start is the one moment every session is known to look. Keep it to a count, an age and three names at any queue length — a long block at every session start is skimmed, and a check that fires on ordinary work trains people to ignore it. The full list belongs in the knowledge-promotion run. Every proposal carries an **Owner** (a person, never "the team") and a **Decide by** date (fourteen days from raising unless it says why not), set by whoever raises it. At that date it is approved, or rejected with the reason recorded; it may be deferred once, with a new date and a reason. Do not back-fill dates onto proposals somebody else raised — that invents a commitment they never made.

### Section 7 — Review Behavior

A single routing line. The checklist lives in its file.

```markdown
## 7. Review Behavior

Before presenting any output, run every item in `{FRAMEWORK_ROOT}/skills/review-checklist.md`. Do not present output that has not passed this checklist.
```

### Section 8 — Reference Map

A table mapping common needs to their files. Engineers and the AI both use this to navigate the framework.

### Section 9 — Process Configuration

A single table of project-level process settings that govern AI-DLC behaviour. Populate from the setup interview answers. Every value here can be changed by the project lead at any time by editing this section.

```markdown
## 9. Process Configuration

| Setting | Value | Notes |
|---|---|---|
| **Archive threshold** | [X] months | Documents older than this qualify for archiving via the compact-docs skill |
| **Last dependency audit** | — | Updated automatically each time the dependency-audit skill runs |
| **Next dependency audit** | YYYY-MM-DD | AI prompts at session start on or after this date; default interval is 30 days |
| **Last disk hygiene sweep** | — | Updated automatically each time the disk-hygiene skill runs |
| **Next disk hygiene sweep** | YYYY-MM-DD | AI prompts at session start on or after this date; default interval is 30 days |
| **Disk headroom threshold** | 25 GiB | Free space below this triggers an unscheduled disk hygiene sweep |
```

The archive threshold is read by the `compact-docs` skill at runtime. If this section is absent, the skill will ask the engineer for the value before proceeding.

The dependency audit dates are read and written by the `dependency-audit` skill. The `Next dependency audit` date is checked at the start of every session — if today is on or after that date, the AI prompts the engineer to run the audit before any other work begins. Set this value during onboarding by asking the engineer:

> "When would you like to schedule the first dependency and security audit? The recommended interval is once a month."

The disk hygiene dates and the headroom threshold are read and written by the `disk-hygiene` skill in the same way. Ask the engineer for the first sweep date (default 30 days out) and keep the 25 GiB threshold unless their builds need more headroom.

---

## Step 3 — Write the Rules Files

### `rules/prompt-quality-gate.md`

Defines the four components in detail with examples of complete and incomplete requests. Include:
- What each component means
- The order to ask for missing components
- An example of an incomplete request and the correct response
- An example of a complete request

### `rules/code-standards.md`

Document your stack's patterns and anti-patterns. Key sections:
- Languages & runtimes
- Naming conventions (per language)
- Framework patterns (backend and frontend)
- Formatting & linting tooling
- Testing requirements and coverage thresholds

**Seed these stack-independent rules on day one.** Each one was learned from a failure in a sibling project and holds on most stacks: a run that looked like a result and was not, with nothing in a later run prompting anyone to look again. Seed the file with the ones your stack can hit, and keep each rule's reason beside it — a rule with no reason is the first one a later session drops as inconvenient.

*Reading a verification:*
- Take a result from the runner's own summary line. No summary line means no run, whatever the exit code said. Read the suite (file) count as well as the test count, because a suite that fails to load reports no failing tests.
- Never read a command's outcome through a pipe. `| tail`, `| grep` and `| head` throw away output before you know which part you need, and the exit status you get is the filter's. Run the command bare, redirect both streams to a file, take `$?` from the command, then filter the file. For a command that changes state (a commit, a deploy, a migration), check the artefact it should have produced, such as the new commit or a timestamp that moved, not what it printed.
- The instrument has its own traps, and each one fakes a result. A wrapper (`nohup`, `&`, a trailing `; echo`) reports its own exit status. A tool run outside its project directory may resolve a different global binary, so call the project's script. `;` carries on after a failed step and `&&` does not. A shell that does not word-split an unquoted variable (zsh) passes a whole list as one argument.
- A scripted edit (sed, perl, a replace script) proves that each substitution matched. Grep for the old text afterwards as well as counting the new. An exit code says nothing about whether a pattern matched, and nested quoting can make a pattern match nothing without any error.
- Run a formatter only on files you created, or files that were already clean at `HEAD`. Run it on someone else's file and the formatting churn buries your change. Before staging, check that `git diff --stat` is about the size of what you wrote.
- A name that only exists as a string is invisible to the type checker: a log event, a test id, an env var key, a route path, a column inside SQL. Open the module that defines it before you write it into code, an AC or an observability check. A green typecheck is not evidence that you looked.
- Never report a fact the code does not have. A response, message or log that says what happened ("created", "expired") gets that fact from the layer that knows it, never by guessing from what the caller sent. If an AC asks for a claim the code cannot know, reword the AC.

*Falsification probes (mutate the code, watch the guard go red, restore):*
- Commit the work before probing it, and restore from an explicit source (`git restore --source=<commit> --staged --worktree <path>`). Restoring uncommitted work from the index deletes that work, and every later probe then measures a tree without it. `git checkout <sha> -- <path>` writes the index as well, so a later plain restore puts back the wrong content. For an untracked file, take a copy with a distinct name before the first mutation. Check each restore by comparing content hashes, never by the restore command's exit code.
- A red counts only when the tests that failed are the ones guarding the AC, so read their names. A mutation that breaks compilation, or turns many unrelated cases red, says nothing about the AC: narrow it and run it again. Pass mutation text through a file or a heredoc, never as a double-quoted shell argument, where a `$1` in SQL expands to nothing.
- Clean up after the mutated run before you run the control, or give each run its own fixtures. Otherwise the control counts the rows the mutation left behind.

*Tests that share infrastructure:*
- Against a shared test database, scope every existence or absence check to the suite's own tenant or fixtures. A table-wide query breaks the first time a neighbouring suite's row exists.
- Let the database compare the timestamps it wrote (`SELECT expires_at <= now()`). Do not compare them against the app's clock: the two clocks differ, and the app's date type may drop precision. Never fix such a flake by loosening `>` to `>=`, because the loosened check cannot see an update that never ran.
- A test that asserts on source text slices between two named anchors, never a fixed-length character window, which a new doc comment can break.
- A shared HTTP or fetch mock answers by endpoint, never with one shape for every request. Otherwise the next call the app makes fails in a way that looks like a product bug.
- A test file never imports another test file. Its top level is its registration, so importing it for a shared value registers its suites again under the importer's name: the count grows and nothing fails. Shared cases live in a fixtures module.
- A test that renders code reading the clock either pins the clock or builds its fixtures from the same pinned "now". A fixture taken from the real clock at load time passes on most days and fails on the ones where it crosses a boundary the code computes (a week, a month, midnight).

*Tests and checks*
- **A verdict over a collection states the count of what it is about — and zero is not a pass.** Any "OK", "none failed" or "GREEN" line after a walk, a comparison or a listing names how many items it looked at; zero is RED, and an absent input is tested for first, because shells and test runners turn absence into emptiness silently (a failed redirect into a loop; `it.each([])` running zero tests and passing). **A floor on a proxy is not a floor** — count the things the verdict is about, not the files opened to find them. *(makerclub, 2026-09-28: an RCA found three checks reading green over nothing, one of them the check that no secret reaches a release.)*
- **A check that compares N values prints their NAMES, never the values.** "1 secret value checked" tells a later reader nothing about which secret was not compared; ending the line with the variable names does, and leaks nothing. *(makerclub, 2026-10-02.)*
- **An exact assertion over a log window first asserts that nothing else can write into it.** `toEqual([...])` over the lines one action logged is a claim about everything alive in the process. List the unsolicited writers (timers, socket events), assert none is alive before the window opens, and make `afterEach` wait until the server has forgotten what the test opened, not merely until the client's promise resolves. Filtering the window to "this request's lines" is the tempting fix, and it weakens the assertion; a window filtered to one id is already safe. *(makerclub, 2026-09-28: a connection left by an earlier test was swept closed inside the window, and an auth assertion went red one run in four, reading as flake.)*
- **A test that accepts several outcomes widens its list only with the code path and event order that produce the new value.** If you cannot name them, the new value is a finding, not a timing — reproduce it before accepting it. *(makerclub, 2026-10-05: a test accepted "device offline" as "it is away" after it had already waited for the device's hello; the value was a reconnect defect, live for a day until CI found it again.)*
- **A comment asserting that a case does not exist is a claim, not documentation.** "No such case exists here", "this cannot happen", "nothing else calls this" are assertions about the codebase, and the next reader treats them as a search already performed and does not repeat it. They also age badly, because the thing they deny is exactly what a later change adds. Require the search to be run, and the surviving claim to say what was searched ("no call site outside `lib/` as of <date>") rather than assert a universal. It is the prose form of *a verification must be able to fail*: an absence claim has no execution at all, so it cannot fail and is never re-run. *(Coconuts, 2026-08-25: a source scanner's header read "does not understand a pattern inside a string literal — no such case exists here" while the file committed alongside it contained exactly that case, which the scanner's own test then flagged.)*
- **An EXEMPTION's precondition is part of the exemption.** A guard with a carve-out ("allowed only when X holds") is a claim about the codebase. If nothing executes X, the carve-out quietly widens to whatever the codebase happens to look like, and the guard stays green. Write the check that enforces the precondition in the same change as the exemption. Prefer a precondition a test can execute over one a reader must believe. When adding to an existing exemption list, ask what the existing entries assume and whether anything still verifies it. *(A guard allowed table truncation only for suites holding a lock, but checked only that the truncating file imported the lock. The complement check — every *writer* of that table holds it — found two unenrolled writers on its first run that a manual sweep had missed.)*
- **A race or timeout test must be checked reachable against the constants that govern it.** Where two timers bound a window — a settle window against a lease ceiling, a retry budget against a request timeout, a debounce against a poll — do the arithmetic before writing the test. If one bound always expires first the scenario cannot occur, the test asserts nothing, and it will usually still pass because the code does the safe thing for an unrelated reason. A permanently-green test that can never observe its condition reads as coverage forever. When a scenario proves unreachable, reach the state another way or delete the test and say why. This matters more with an agent than without one: an acceptance criterion describes a scenario in words, and words do not reveal that one constant is six times the other. *(Coconuts, 2026-08-25: a test asserted that a resource build refuses after a mid-wait revocation; the wait abandons at 2s and the revocation ceiling is 12s, so the build always landed first.)*

*Data, fixtures and adapters*
- **Canonicalizing JSON columns — compare values, not strings** (applies where a column type like Postgres jsonb or MySQL JSON is used): such types do not preserve key order or duplicate keys, so comparing a stored value to a freshly computed one by serialized-string equality silently fails after any round-trip. Compare semantically — key-sorted stable stringify, field-by-field, or the database's own JSON equality. The failure is silent by construction and typically hits dedupe, idempotence checks, and change detection.
- **Provider fixtures mirror captured reality:** when stubbing a third-party API, author the fixture from a captured real response or the provider's current docs — never from the consuming code's own assumptions, or the gate can only ever confirm those assumptions. Encode the provider's quirks explicitly (sort/column order, pagination, envelope shape) and comment the fixture with where its shape came from.
- **Expected values are derived, never assumed — and deriving against runnable code means RUNNING it:** when a test asserts a computed outcome (a threshold effect, a score, a calibrated range, a derived state), derive the expectation from the specification/model and cite the arithmetic in the test or its comment — an intuition-authored expectation tests the author, not the system. When the expectation comes from an executable algorithm (a stub, a scoring function, a parser), a mental model of that algorithm is an assumption with the arithmetic shown: run the real code on the fixture and paste the measured value into the fixture's comment. Companion to the fixture rule: that one governs test inputs, this one outputs. *(Two fixtures were calibrated against a test stub by mental arithmetic: estimates of 0.22–0.31 measured 0.09–0.15, which cost two dead test cycles. A 20-line harness running the stub fixed it first try.)*
- **Async mutations are observed before navigation in E2E tests — and the observation must be FALSIFIABLE:** a page-bound mutation (a framework form/server action, a fetch with no awaited UI effect) is silently aborted when the test navigates away — the write never lands and the failure surfaces downstream of the cause, in a test that reads sequentially correct. After triggering a mutation, assert an observable effect (poll the API, wait for a rendered change) before navigating. That assertion must be one that would fail had the mutation never fired: a state that already holds before the write lands — the input's own just-filled value, or anything a re-render reproduces identically — satisfies the letter of the rule and observes nothing. When the form has no distinct on-success signal, poll server state. To prove the observation gates on the write, disable the submit and watch the test go red *at the observation*.
- **No type assertion closing a shape adapter (statically-typed stacks):** where code reshapes one contract into another — a wire form into a domain model, a row into an entity — a closing cast (`… as TargetType`) removes the only automated check on the reshape, and it is tempting precisely because the reshape does not typecheck. Drive the mapping off a shared field map with a type predicate narrowing each branch, so an unhandled field is dropped rather than written with the wrong shape — a missing optional value in the worst case, never a crash. "The compiler will catch it" is a claim to verify, not a design: a mis-shape written through an indexed or loosely-typed record produces no compile error.
- **A diagnostic that ENUMERATES CAUSES is a contract — adding a cause means extending the list in the same change:** where a behaviour is deliberately silent (a security control that must not reveal why it refused, a rate limit that would otherwise be an enumeration oracle), the failing test's or log's message is the only place a reader learns what to check. Such a list goes stale invisibly: nothing fails when a fourth cause is added and the message still names three. A wrong list of suspects is worse than none, because it is credible. Put the most likely cause first, not the most recently added; give each cause the command that settles it; and whenever a unit adds a refusal, suppression or early return to a silent path, ask "does any diagnostic enumerate the causes of this?" *(A test-helper message written before a rate budget existed pointed readers at two innocent causes twice in two days; the real cause was the spent budget.)*

*Agent shell hygiene*
- **Agent commands carry their own scope — above all, backgrounded ones:** if your agent harness may reset the working directory between tool calls, add a standing rule: a command that edits or verifies tree-scoped state carries its scope IN that command (`cd <tree> && …`, `git -C`, a package manager's `--dir`, or absolute paths), never assumed from an earlier call. Backgrounded or detached commands are the highest-risk case: they inherit an assumed cwd, can start long after the `cd` they relied on, and fail silently into the wrong tree. So every backgrounded command carries its scope as its FIRST token, and the first thing to do after launching one is read the tool's own path line in its output (most test runners and package managers print the directory they ran in). Treat a background launch whose tree you have not read back as unverified. *(Four wrong-tree misses in one session, all backgrounded: one fix was declared ineffective because the test ran in the unfixed tree, and another run left stray edits in a shared tree.)*
- **Message-bearing text never passes through the shell's mouth:** commit messages, PR bodies and any prose handed to a CLI that contain backticks or `$` go through a QUOTED heredoc (`git commit -F - <<'MSG' … MSG`), never `-m "…"`. Inside double quotes the shell executes `` `word` `` and `$(…)` and splices the output into the text, often silently. **Needing interpolation does not license unquoting the heredoc:** keep it quoted and pass dynamic values in from outside as env vars or arguments that the inner interpreter reads (`SHA=$(git rev-parse HEAD) python3 - <<'EOF' … os.environ['SHA'] … EOF`). *(An unquoted heredoc used to splice in one sha executed the prose's backticked words and pushed empty command output into history.)*

*Requests and clients*
- **Authenticate in the earliest request hook, before the body is parsed — and every authenticated route has a test that sends a malformed body WITHOUT credentials and expects 401.** Many frameworks parse and validate the body before their later hooks run, so a credential check placed there answers 400 to an anonymous caller with a bad body and leaks the schema to strangers. It is the test the route's author is least likely to write, because they wrote the fixtures too. *(makerclub, 2026-09-21: an admin route checked its token after validation, and all 71 green tests sent valid bodies.)*
- **A statically served single-page app pins its cache headers with a test: the entry HTML `no-cache`, hashed assets `immutable`.** A static-file server's default lets a returning browser keep an old index whose hashed bundle no longer exists, so each deploy reaches returning users only by luck — and no test is a returning browser unless one is written. *(makerclub, 2026-09-21: the default was `max-age=1h`, found only when a human noticed a missing panel.)*
- **Every text field on a touch client renders inside ONE shared keyboard-safe wrapper, enforced by a source scan, and is walked with the SOFTWARE keyboard up.** One wrapper means every field behaves one way; a screen that cannot use it changes the wrapper, never hand-rolls its own. A simulator types from the host keyboard and never raises the soft one, so a walk there cannot see a control hidden under it. *(makerclub, 2026-09-23: a pairing screen's Connect button sat under iOS's number pad, which has no return key to close it, after a careful simulator walk.)*

*Scripts, start-up and configuration*
- **Configuration files are parsed, never sourced.** `. .env` executes the file: a value with a space breaks it, and a password should never pass through a shell interpreter. Read `KEY=value` lines, split on the first `=`, and pin the space case in a test. *(makerclub, 2026-09-21: a Wi-Fi name with a space broke a provisioning script's first run.)*
- **A loop fed from a file tests the file first — `set -e` does not.** On bash 3.2 (every Mac's `/bin/bash`), `while …; done < "$f"` with `$f` missing prints an error, runs zero times and carries on. Put `[ -f "$f" ] || die …` above every such loop and enforce the text with a scan, floored as below. *(makerclub, 2026-09-24: the check that no secret reaches a release binary printed OK over a missing env file in every git worktree.)*
- **A start-up step that can fail logs WHY, in the failure's own words — never a bare boolean.** *(makerclub, 2026-09-21: a worker failed both warm-up steps and logged `ok=false`, which reads as "slow" while every job after it fails; four lines of logging would have saved a deploy cycle.)*
- **A directory, cache or workspace missing a required part refuses to be used, rather than skipping the part.** Name the parts that must be there, throw when one is absent, and say the environment is wrong. A silent skip turns a missing file into somebody else's bug one deploy later. *(makerclub, 2026-09-21: a skipped template entry surfaced as a compiler error about a component nobody had removed.)*
- **A copy that restores build state preserves modification times, and a test pins it.** Incremental build tools (make, ninja, CMake, bundlers with a file cache) decide what to rebuild by comparing mtimes, so a plain copy of a warm template makes every restored file newer than the outputs that depend on it and turns each incremental build into a cold one — silently, with every build still green. Copy with mtimes preserved (`cp -p`, `rsync -t`, `fs.cp` with `preserveTimestamps`), and assert in a test that a restored file is not newer than its build output. *(makerclub, 2026-09-27: every restore re-ran the build system's configure step, 12 of a 19.6 s compile, on every build for a week; the build tool's own log showed it in minutes.)*
- **A `--dry-run` prints the steps it did not reach, and the first live run is watched at exactly those.** A dry run proves the steps it runs and nothing about the ones it skips — which are the ones that touch the real environment. End it with _"not exercised: <steps>"_, and the unit that ships the script lists those steps as owed for its first live run. *(makerclub, 2026-10-05: a publish script's dry run was green; its first live run failed on the read-back the dry run never reached.)*

**Critically:** add anti-patterns discovered through actual failures — e.g. framework methods that look correct but have unit-testing limitations. These turn retro findings into permanent rules.

**Concurrent sessions — placeholders until push, and serialise shared files.** If your project ever runs two agent sessions against one working tree, add a standing rule: two sessions collide on unit/bolt numbers and on shared cross-cutting files (schema/data-model, the migration journal, a shared domain module), and the interleaved uncommitted state often can't be split cleanly (interactive `git add -p` is unavailable). Mitigate — work under placeholders (`U-TBD`, `B-TBD`, `ADR-TBD`) and allocate the real numbers at PUSH from the id ledger (`ops/build/id-reservations.md`), per the id-ledger rule in `guidelines/dev-setup.md` (Step 5) and the version-control rules in the master rule file Section 6; never claim at elaboration sign-off *(Riley, 2026-08-28: claiming at sign-off failed eight times before it was replaced — the worked example below)*. Serialise edits to shared data-layer files (one stream at a time — migration journals are linear), and commit an edit to a shared ledger file on its own and at once, after reading `git diff -- <file>` for hunks that are not yours: a path-scoped commit (`git commit --only <path>`) protects the INDEX, not the FILE — it commits every hunk in that file, a peer's unfinished ones included. Better still: run concurrent streams on separate git worktrees/branches and merge, so shared-file interleaving can't happen. **And if you do: the ahead-count is not noise.** A staleness check (`git rev-list --left-right --count origin/main...HEAD`) is standard practice in that setup, and its *behind* number answers "is this checkout safe to work in?". Its **ahead** number answers a different question that nothing else in the process asks: **is something finished and unmerged sitting HERE?** A worktree left from a previous session can hold a commit nobody is coming back for. When ahead > 0, look at what the commits are, and either merge them or state in the closing message that they were left and why — not every stranded commit is worth merging, but every one is worth a sentence. *(Coconuts, 2026-08-25: a session read "90 behind, 1 ahead", correctly moved to a fresh worktree, and left the 1 as in-progress scratch. It was a finished two-day-old commit carrying a lint rule; the rule found three live instances of the defect it prevents the moment it finally reached main — every one written during the two days it sat stranded.)*

**Shared-checkout git safety.** If sessions share one working tree, add these to the same rule. Each is cheap, and each was learned by losing or nearly losing someone else's work:
- **`HEAD~1` is not "my commit".** Before any `git reset`, run `git log -1 --format='%h %s'` and confirm the tip carries *your* subject line, then reset by explicit hash. If the tip is not yours, your commit is buried: leave the history alone and say so. The window is not the reset itself; it is the gap between forming the intention and running the command. *(Riley, 2026-09-01: a `reset --soft HEAD~1` meant to withdraw a session's own commit dropped a peer's commit, made minutes earlier. It was recovered from the reflog, which was luck rather than method.)*
- **Never resolve a conflict mechanically across commits you did not write.** `--ours`, `--theirs` and a `rebase --skip` loop decide whose work survives without reading it. A shared registry (backlog, number ledger, edge-case list) is resolved by keeping both sides' rows by hand and reading the result back row by row. When hand resolution is unreasonable, cherry-pick your own commits onto the remote tip in a separate worktree. *(Riley, 2026-09-02: one `--theirs` deleted two sessions' ledger rows, and a `--skip` loop dropped four peer commits the same day. Both were caught afterwards.)*
- **Stage by explicit file path, never by directory**, even when every file you *intend* to touch is under that directory. "Check `git status` first" is not enough: the files a peer is editing are listed there, seen, and swept in anyway. *(Maestro, 2026-09-09: `git add docs/process/…/` swept a peer's two finished files into an unrelated commit, immediately after a `git status` that listed both.)*
- **A question about the repository is answered from the remote, not from this checkout.** "Did we ever do X?" and "is Y still broken?" are questions about what has been pushed, and a shared checkout is routinely many commits behind. Fetch before the first search, and confirm a "we haven't" answer against the remote tip before saying it. *(Maestro, 2026-09-21: a session searched a checkout 34 commits behind, found nothing, and opened a bug bolt to build a fix that had shipped two days earlier. Two checks agreed with each other because both were stale.)*

**Why the number is claimed at push and not before.** A reserved number is invisible in the artifact directory, and the artifact is invisible in the reservation ledger: a number is recorded in the ledger before its artifact exists anywhere, and an artifact can already exist on an unmerged branch, or uncommitted in another worktree, before its reservation is visible on the trunk. Each source is blind to the other, so checking either alone proves nothing, and a number checked in both is still provisional until it lands — it belongs to whoever merges first. A generator that auto-numbers from local state (the highest entry in *your* checkout) hands out a colliding number however carefully the reservation was made. Worked example — four collisions in one night on database migration numbers: one session reserved `0080`, verified it free on the trunk and on every remote branch, and recorded the sha it reconciled against — the full discipline, performed correctly; a second session reserved `0080` against a sha that already contained the first reservation, and merged first; the first renumbered to `0081`, and a third session then took `0081`, because its generator numbered from its own checkout. Three sessions ran the check; all three collided. Hence the rule: placeholders until push, the ledger's "Next free" marker read at the rebased commit, and a lost race rejected by the push itself rather than found after other files cite the number. Prefer artifact kinds that are cheap to renumber (a regenerable file) over ones referenced by prose across the tree. And after any renumber, grep the WHOLE tree for the old id — code and comments included, not only the plan artifacts (e.g. `grep -rn "ADR-0NN\b" --exclude-dir=node_modules .`): a code comment still citing the pre-renumber id points every future reader at the wrong decision.

### `rules/security.md`

Two sections:
- **Never do these** — injection, auth gaps, secrets, data exposure
- **Always do these** — input validation, auth on every endpoint, least privilege

Add a section for your specific platform's security quirks (e.g. Supabase RLS, AWS IAM patterns, OAuth flows).

### `rules/architecture.md`

One ADR per significant decision. Format:

```markdown
### ADR-001 — <Decision title>
**Decision:** <what was decided>
**Why:** <the reasoning>
**Trade-off:** <what you gave up>
```

Add an ADR whenever a new cross-cutting decision is made — especially ones where the AI might make a different choice if not told (e.g. symmetric vs asymmetric JWT signing, SSR vs SPA, monorepo vs polyrepo).

### `rules/engagement.md`

Copy this file verbatim from `process-onboarding-agent/rules/engagement.md` to `{FRAMEWORK_ROOT}/rules/engagement.md`. It defines two monitoring protocols: (1) engineer disengagement signals — when to intervene, what to say, and how to handle continued non-engagement; (2) failed output circuit breaker — if AI output for the same unit is rejected 3 consecutive times with the same underlying failure, execution stops, a diagnostic question is asked, and the outcome is recorded in the unit's Prompt Log. The master rule file Section 6 routes to it with a single mandatory-read line — do not inline its contents.

---

## Step 4 — Write the Skills Files

> **Path substitution — critical:** Any `{FRAMEWORK_ROOT}` placeholder that appears in the content descriptions below (or in "Wire into the master rule file" snippets) must be replaced with the actual resolved path before writing. This applies to generated skills files (`mob-elab-prompts.md`, `review-checklist.md`) and to any routing lines added to the master rule file Section 6. Skills files that are copied verbatim (compact-docs, root-cause-analysis, design-session, uat, etc.) do not contain `{FRAMEWORK_ROOT}` and require no substitution.

### `skills/mob-elab-prompts.md`

The mob elaboration reference. Must include:
- **A census carries the command that produced it.** Where an elaboration counts something the units will be scoped against — call sites, surfaces, tables, endpoints — the plan records the command beside the number, and the bolt risk assessment re-runs that command rather than re-reading the plan. A number in a document header is read as fact by everyone downstream and re-derived by nobody, and sign-off launders it into an agreed constraint that later acceptance criteria depend on. *(Coconuts, 2026-08-25: a plan announced "18 mutation sites" over a subtotal of "10" directly above a table listing 13; it was summarised, signed off, and caught only when the risk assessment re-derived it from the tree — by which point two more sites had moved.)*
- **Solution Shape check (before anything else):** at the very start of every elaboration session, read the intent and check for a `## Solution Shape` section. If it is missing and the intent introduces a new capability, a potentially reusable surface, or an expensive-to-reverse decision, ask the engineer once:

  > "This intent has no recorded solution shape. Run Solution Shaping first (`{FRAMEWORK_ROOT}/skills/solution-shaping.md`) to decide generic-vs-specific, simplest-viable, and extend-vs-build — or proceed straight to design?"

  The engineer decides: run it (then resume elaboration with the recorded shape as binding context), or proceed as-is. Never block. For plainly small, feature-specific intents, skip this prompt and go straight to mode selection.
- **Elaboration Mode Selection:** at the very start of every elaboration session, before Phase 0, ask the engineer which mode they prefer:

  > "Before we begin — which elaboration mode would you like to use?
  >
  > **A — Interactive (turn-by-turn):** I'll propose one unit at a time, confirm the ACs with you, then move to edge cases and observability before proposing the next. Good for working through uncertain scope.
  >
  > **B — Plan-first:** I'll draft a complete elaboration plan — all proposed units with ACs, edge cases, and observability signals — as a single markdown document. You review it, mark changes, and we refine from there. Good when you have a clear picture and want to see everything at once."

  Record the answer. **Mode A** follows the standard turn-by-turn protocol described below. **Mode B** follows the plan-first protocol:
  1. Run Phase 0 (design session) as normal.
  2. Produce the full draft plan as a markdown document written to `{FRAMEWORK_ROOT}/ops/inception/elaborations/YYYY-MM-DD-<unix_timestamp>-[intent-slug]-draft-plan.md`. The document contains: intent recap, all proposed units (each with context, ACs, scope boundaries, edge cases, and observability signals), and a unit summary table.
  3. Ask the engineer to review and respond with any changes, additions, or removals.
  4. Apply changes in a second pass — update the draft document and confirm the revised unit summary table with the engineer.
  5. Proceed to sign-off and the Dependency Map update as normal once the engineer confirms the plan is complete.

  The quality gate, ACs, and sign-off requirements are identical in both modes — Mode B compresses the back-and-forth into a document review cycle, it does not skip any step.

- **Phase 0 — Design Session:** read `{FRAMEWORK_ROOT}/skills/design-session.md` and run it at the opening of every session before proposing any units. The design session scopes the intent's API surface, data model, and architectural patterns, then produces binding constraints that govern every unit and AC in the session. For simple intents with nothing new to design, Phase 0 concludes quickly and flows straight into unit decomposition. If the intent carries a `## Solution Shape` section (recorded by `{FRAMEWORK_ROOT}/skills/solution-shaping.md` before elaboration), Phase 0 treats those decisions as binding context and designs within them.
- **Codebase findings check (brownfield dependency analysis):** whenever Phase 0 or unit decomposition requires understanding existing code — because the intent depends on, integrates with, or is constrained by a prior implementation — first check `{FRAMEWORK_ROOT}/ops/inception/codebase-findings/README.md` for a file already covering that module/area. If one exists, read it as a starting point and verify it still matches the current code before relying on it; findings go stale as code changes. After analyzing any module/area not yet documented there, or finding something that contradicts an existing entry, write or update the corresponding file (append a new dated entry, never overwrite prior ones) before finishing Phase 0, and update the index in `README.md`. This step is mandatory whenever code analysis of an existing module occurs — findings from reading the codebase must never live only in the session's memory.
- **Constraint coverage per SURFACE, before the summary table.** The design's constraint list is read once, for the intent as a whole; the units are then decomposed per surface — per client, per service, per bounded context. That gap is where a signed-off constraint gets implemented in one place and silently skipped in another, and no unit's Definition of Done can see it because the defect lives strictly *between* units. Walk the constraint list once per surface the intent touches and record, for each constraint, which unit carries it there — or "does not bind here", which is a legitimate answer. **Silence is not.** A constraint with no unit on a surface it plainly governs is the finding, and catching it costs one pass over a list that already exists.
  *(Coconuts, 2026-08-25: a constraint governing a shared mixer — "the faders follow the playhead, and the song-wide level stays reachable" — was signed off, implemented on the web client the same day, and never given a mobile AC, because the mobile units were written for a narrower mode and the constraint list was never re-read per surface. Three units two days later put back what one line of a coverage pass would have caught.)*
- **An option put to the engineer that compares to shipped behaviour carries its evidence — `file:line` in the option's own text.** An option is a claim, and the engineer answers the question they were asked: _"capped at 60 characters, like the display name"_ reads as checked and was not (the display name was capped at 30; 60 was a form's `maxlength`). Written with its source — _"like the display name (`DISPLAY_NAME_MAX = 30`, `app.ts:86`)"_ — the mismatch is visible to the writer as they type it and to the engineer as they choose. An option naming existing behaviour without a citation is, by this rule, unverified, and says so.
  *(makerclub, 2026-09-24: two options in one Phase 0 asserted shipped behaviour wrongly — the cap above, and a route change for a phone that only paired. Both were one grep away.)*
- **Pacing:** where an answer is obvious from prior decisions, existing ADRs or the session's own context, present it as settled-unless-objection and move on within the same turn. Reserve full stops for decisions that need the engineer's domain knowledge: scope calls, policies, trade-offs. Never skip a protocol step; compress it.
- The mandatory interactive protocol (turn structure + never-do rules)
- **Draft-plan → AC reconciliation at sign-off.** Before asking for sign-off, read the draft plan against the agreed ACs: every draft-plan item that did not become an AC is read out and recorded in the relevant unit file as "Descoped: [item] — [reason]". Scope that exists only as narrative — neither an AC nor a recorded descope — belongs to nobody and silently evaporates, resurfacing later as a UAT finding.
- **Rule-input provenance check at sign-off.** For any AC that defines a scoring/placement/calibration rule, name each input's evidence path and confirm whether a low-trust (inferred or extracted) source can populate it; if it can, the sign-off records whether that is acceptable for this use. The flaw lives in the spec, so no test of the spec will catch it.
- Facilitation prompts for: proposing units, proposing ACs, edge case check, observability check (success signal / failure signal / alert threshold per unit), generating implementation scaffold, reviewing output
- **Measure an unverified premise first.** Before proposing behaviour, ask whether any unit rests on a fact nobody has measured — how a device, library or third-party API actually behaves. If so, measuring it is unit one: its AC is a number or a comparison, and it runs before the units built on the answer.
- **Sizing — a second caller of a module hard-wired to one endpoint is a generalisation of shipped code, not an addition.** When a unit reuses an existing client or transport for a new endpoint, grep the module for what it names before sizing; if the endpoint is a private constant, the unit changes the shipped caller and is sized, and registered as contract-changing, accordingly.
- **When a unit reverses, narrows or relies on an earlier AC, quote that AC's own text.** A test named after an AC number records what its author believed the AC meant, and its assertion is often wider. The gap cuts both ways: when a test goes red on new work, ask which broke — the AC, or the test's wider restatement of it — before changing the code; if it is the restatement, narrow the test to the AC and assert the extra property where it belongs.
- **Precondition existence check, before the summary table:** every AC's Given either already exists on the current product surface or is delivered by a unit that *executes before* this one. "Delivered somewhere in this plan" is satisfied by a later unit, and leaves the AC untestable when it runs; an AC resting on a capability nobody built is hidden scope or a defect, named and resolved at sign-off, not mid-execution. (This is the reverse of the draft-plan reconciliation above: that checks scope→AC, this checks AC→capability.)
- **Re-planning after sign-off — a withdrawn or renamed mechanism is swept in the same commit.** Grep the plan, every unit file and any document that reasons from it (an endpoint, a table, a cap, a code) for its name, and rescope or mark each hit. A unit that still names a withdrawn mechanism looks correct until the day it is next.
- **At the AC turn — choose the default AC's form after the count/set sweep, not before it.** When a unit adds a member to a set the tests pin (an enum value, a registry entry, a contract method, a menu item), sweep the suites for count and set-membership assertions at the AC turn, and write the default AC's form (M3.3) from what the sweep found: tests that pin the count or the set mean the contract-change form and a Breaking Changes Register. Choosing the standard form first and sweeping later turns one approval into two.
  *(makerclub, 2026-09-28: a unit proposed the standard form, four tests pinned the set's count, and the contract-change form needed a second sign-off after the first.)*
- **At the edge-case turn — an AC triggered by a message names the message's cardinality and cites where it is SENT.** Once per process start, once per connection, or once per event — read from the code that encodes and sends it, not from the code that consumes it. A consumer shows what a field means; only the sender shows how often it arrives. One grep for the message's encoder answers it.
  *(makerclub, 2026-09-28: a device's greeting is sent on every connect while its reset-reason field is per boot, so an AC keyed on that field repeated after every network drop; its fix then assumed a heartbeat's first timing nobody had read in the sender.)*
- **Post sign-off — Dependency Map update:** after the engineer confirms sign-off on the unit summary table and before writing any files, read `{FRAMEWORK_ROOT}/ops/inception/dependency-map.md` and update it: record any prerequisites this intent has on other intents, and any shared interfaces (API contracts, data entities) that cross intent boundaries. Add a row to the Update Log. If a dependency on an incomplete intent is found, flag it to the engineer before proceeding.

### `skills/review-checklist.md`

Structured review sections covering all five AI failure modes:

| Section | Failure mode covered |
|---|---|
| Functional Correctness | Logic errors that look correct |
| Code Quality | Over-engineering; dead code |
| Security | Security vulnerabilities |
| Architecture | Architectural drift |
| Tests | Logic errors; hallucinations |
| AI-Specific Checks | Hallucinated library calls; prompt log; scope creep |
| Observability | Missing production evidence; silent failures |
| Deployment Readiness | Configuration errors; breaking changes |

Key items that must be present:
- Feature verified in a real environment — tests passing alone is not sufficient
- Any AC asserting a page/route is "accessible" is verified by a rendered 200 (following redirects) with the expected content actually visible — text present and legible, images decoded (non-zero natural dimensions) — never a build pass or a 3xx redirect alone
- After any deploy, a post-deploy smoke suite passes against the live deployed URL, asserting key pages render their actual content (text visible, images decoded, primary auth/entry reachable) rather than merely returning a 200 status — a deploy is not done until this passes, and whoever is driving the deploy (the AI assistant included) runs it and reports the result rather than handing a runnable check back to the engineer
- **Every claim that something is covered, enforced or unchanged cites the artifact that proves it — or is marked unverified.** "Asserted in a unit test", "the compiler enforces this", "the guard is untouched" are claims: name the test, paste the output, or run the check; if you cannot, write "unverified" and say what would settle it. A claim marked unverified invites the look that a believed one prevents — a false claim of coverage is often exactly why nobody inspected the seam that later failed
- **LIST every new element the change renders, and for each say whether it is interactive and how a user knows that.** The output is the list, not a tick. Acceptable answers are things a user can SEE: a real control style, or an explicit button or label on a clickable region. "The whole card is clickable" is not one. Also flag the inverse: a non-interactive element styled like a link or button. Behavioural ACs prove a click works; nothing else asks whether anyone will click it, so without this the UAT tester is the first person to look
- **A screen with a text field on a touch client was walked with the SOFTWARE keyboard up.** A simulator types from the host keyboard and never raises the soft one, so a walk there passes with the call-to-action hidden under the keyboard. The shared-wrapper scan catches a missing wrapper; only this walk catches a layout the wrapper does not save
- **Every OK/GREEN line the diff adds over a collection was read for what it prints when the collection is empty.** It names its count, and zero is RED — the weak point of that rule is the moment a new verdict is written
- **A new test is not finished until it has been seen to FAIL against the behaviour it rejects — and the break must remove the RULE, not the feature.** Deleting the feature fails every test in the group at once, so a test that could never have detected anything passes the control by hiding in the crowd. Break the single decision the test names, leave the rest standing, and confirm that test and no other goes red
  - **The red must be the guarded assertion failing, not the code erroring.** A break that makes every query or render throw goes red for a reason unrelated to the guard; read the failure message, not the count
  - **A probe that stays green proves nothing until it is shown to have landed.** Confirm the injected change is in what the guard actually reads — the string in the bytes it scans, the code path the test actually renders. "The guard held" and "the change never reached the guard" produce the same run
  - **A broad red set proves the ACs fail together, not that each is tested.** For every AC claimed covered, at least one probe's predicted red set names that AC alone
  - **Verify the tree after every probe, before the run you quote.** A restore chained onto a command that times out never runs, and the next green is about the unfixed code. Check `git status`/`git diff` first, and restore with `git restore` rather than a temp-file copy
  - **Ask the cheapest way the assertion could pass.** If it is anything other than the guarded mechanism — a race never forced, a fixture refused earlier, an inherited guard from reused code that fires first — the test is decoration until that precondition is established
- **Know whether a suite exercises the working tree or an artifact BUILT from it — and if the latter, rebuild before concluding anything.** A harness that starts a compiled server, serves a bundle or runs a container tests the last build, not your edit, and some reuse an instance started before it. The benign symptom is a red that will not go away; the dangerous one is a false green. A red/green falsifiability exercise needs a build on each side. Record in the project's conventions which suites are source-run and which artifact-run
- **When a change alters an existing RULE, the tests of the old rule are re-proved, not merely re-run.** They keep passing, keep their old names, and can quietly stop guarding anything. If a test's subject has moved, rewrite it to what the rule still decides and re-prove it — never edit its expected values until it is green again
- **A gate's PASS is its exit code, never the absence of matching text in its output.** `<gate> 2>&1 | grep -i error` printing nothing is indistinguishable from the gate never having run — a missing toolchain, a renamed script, a command that failed to start — and the pipe discards the status that would tell you which. Capture the status first (`cmd > log 2>&1; echo "exit=$?"`, or let it break an `&&` chain), then filter the log for detail. "The suite is green" asserted from filtered output is not evidence
- **Name the control that triggers each destructive call** (a delete, an overwrite a person confirms, a send) — its own click or submit, never an event derived from it (a dialog's `close`, a field's `blur`, `beforeunload`, a route change). A derived event can be withheld, or fired by something that is not the decision; the click is the only witness to it. An autosave is not a decision and is exempt
- **Adding an item to a list some test asserts EXHAUSTIVELY is a shape change, and is swept like one** — a registry, a catalog, an enum, an equality assertion over a set of names. Grep the registry's own identifier and the exhaustive-assertion idiom across the suites before pushing. Name this case explicitly rather than leaving it to "function signature": it is missed repeatedly not through carelessness but because adding a registry entry does not feel like changing a shape and breaks no caller, so the generic wording genuinely does not seem to apply. *(Coconuts, 2026-08-25: two occurrences in one bolt, both reddening an exhaustive probe-name assertion at the suite gate.)*
- **A decision with two clauses gets an assertion per clause.** For every "X and not Y", name the test for "not Y" — the negative clause produces nothing visible, so it is the one left untested. A probe that stays green there is a finding, not a failed probe
- **A guard that scans source strips comments first, and carries a self-check.** Otherwise it reddens on the comment explaining the rule, and the tempting fix deletes the explanation. The self-check asserts the pattern is still caught in code and that real declarations survive the stripping, so an over-eager stripper fails rather than passes
- **A guard re-pointed at a moved mechanism is also re-scoped to it.** When a refactor moves a value — a credential, a path, a payload — into a new container (a template, a temp file, a config document), ask what the new container exposes and which assertion was scoped to the old one. A green suite answers only where the property moved
- **Copy assembled from parts is asserted as the full rendered string, anchored at both ends.** Capitalisation, spacing and punctuation at the seam belong to no single part, and an unanchored fragment match passes with the seam broken
- **A loosened assertion says why it was tight.** Single match → any match, equality → contains, an exact count → a range: the strict failure is often the defect reporting itself. One sentence at the call site justifies the looser form, or the strict form stays
- **A test that declares a limit opens an action to close it.** "This cannot exercise X" is paired in the same commit with an owner and a time to cover the gap (a device pass, a UAT step), a DoD line left unticked, or a written acceptance — the declared blind spot is where the next defect lives
- **An intermittent failure naming a different test or file each time is still one class.** Unrelated tests sharing a failure share the harness — timing, ordering, a leaked handle, a contended resource — and the differing file is evidence *for* a shared cause, not against one. Three sightings is a class whether or not they name the same test. Reproduce a suspected race by shrinking its window, not by adding load
- **A sweep names one result its output must contain, and the forms its pattern cannot match.** A broken sweep reports a count that looks exactly like a clean one; if the known positive is missing, the tool is wrong — fix it before reporting. Check the unmatched forms by hand
- **Fix one, grep all — in every lane that fixes a defect.** Grep for the defect's shape, not only the reported site, and record each hit as fixed or safe-and-why. It belongs here as well as in the bug-bolt skill because hotfixes, UAT fixes and in-unit fixes never open that skill, and this checklist is the step every fix passes
- **Name the unit of count before counting.** When a change must reach every place something is used, count the thing the diff edits (write sites, call sites), not the files or modules that contain it; a correct count of the wrong noun misses sites in files already open. Where the count is per call site, the compiler is often a better enumerator than grep
- **A new or tightened constraint is reviewed by its writers.** List every code path that writes the constrained columns and check each has a test at the layer that builds the row; a test against the library below passes while a route drops a field. A UAT step for it asserts a successful write, never that the constraint exists
- Nothing in the diff beyond what the ACs required (no extra abstractions or future-proofing)
- **Every In-scope item in the unit is accounted for, not only every AC.** A Scope line is a deliverable, but the Definition of Done ticks ACs — so a Scope item with no AC behind it has no box whose absence would reveal it. Walk the In-scope list and name the code or test that delivers each item, or state that it was descoped and why. An act with no reachable affordance (a fully tested operation nothing in the UI or API calls) is as unfalsifiable as a behaviour with no assertion
- Diff checked against system boundaries in `architecture.md`
- **A specific contract value a test asserts — a status code, an error message, an enum value, a field name, an order — is read out of the code under test, never recalled from convention or a sibling test.** When such a test fails, open the implementation and decide which side is wrong before touching production code; where the contract encodes a deliberate distinction (two statuses for two refusal reasons), assert the value and its message so the test documents it. The danger is not the red test but "fixing" correct code to match a mis-remembered convention, which quietly erases the distinction
- **Destructive and remote operations.** A destructive operation validates everything it needs BEFORE its first irreversible act, and says what it removed or changed. It defaults to a rehearsal (a dry run, a plan, a diff), so meaning it costs one explicit flag. A call that MUTATES something outside the process checks its exit status before anything reports success: a failed read prints nothing and harms nothing, but a failed mutation that reports success leaves you believing a thing you have not done. Where such a rule is enforced by a scan, the scan names the sites it must reach and floors the count, and that list of sites is itself checked. An offenders-list-is-empty assertion passes trivially when the walk finds nothing; an emptied site list disables the scan silently; and a new site added without a line in the list is exempt by omission
- **A tool's output is evidence only within the semantics its producer states — and a WATCHER's exit code is not the run's verdict.** A streaming/watch command reports on the watch, not the run: a CI watcher can exit 0 for a run whose verify job failed and whose deploy was skipped. Before reporting any CI or deploy outcome, read the verdict from the source of record (the run's recorded conclusion and per-job list) and confirm what is actually running
- Behavioral trade-offs confirmed before accepting output
- **A failing test at a security boundary (auth, session, access control) is never resolved by re-running.** Root-cause it, or open a bug unit, before the gate proceeds. A passing re-run is not evidence of absence for an intermittent failure — the generic flake heuristic must not apply where the failing assertion guards security
- For wrapper/layout components: existing files grepped for patterns the new component will duplicate before generation
- Observability section of the unit file is complete — success signal, failure signal, and alert threshold are recorded; any that represent code behavior are expressed as ACs and implemented in the diff
- **Every "verified" or "Done" claim reflects a check that actually ran this session**, never a templated status. When the behaviour cannot run in the available harness (a native module needing a device build, a flow needing a deploy), the status reads "code + tests green; [device/live] verification pending <gate>" and the DoD box stays unchecked
- **A unit or bolt Status line claiming "committed", "pushed" or "deployed" is reconciled against actual git and pipeline state at close** (SHA present, pushed, CI green for that SHA), never trusted from the document's own earlier text
- **Every finished state has its own content and a positive way out.** Any surface that sets a sent / saved / booked / paid / submitted flag must, in that state, (a) say what happened, (b) offer a positive exit (Done, or a hand-off to the next step), never only Cancel / Back / close, and (c) stop presenting the spent form as live. Moving a form between containers (page → sheet, inline → modal) re-opens this item, because the container decides what its exit means. And a form with more than one submit route passes each field to every route it sits above. *(Maestro, 2026-08-09 and 2026-09-29: a registration QR flow, and later three more surfaces, left the user on a spent form with no positive way out.)*
- **A screen that stacks content above a list has one scroll owner.** Intrinsic-height cards pinned above a flexible scrollable child can grow until the list collapses to zero height and becomes unreachable; a per-card height cap does not prevent it, because it never bounds the sum
- **When a change alters an input's format, a component's test id or props, or a function signature, the tests that drive it are grepped and updated in the same change**, not discovered at the suite gate
- **A test that seeds or asserts second-scale time uses one clock.** Identify the clock domain of the code under test first: database-clocked code (`now()` in SQL) seeds and asserts in SQL, application-clocked code in the application's clock, never mixed
- When the deliverable is a client binary (mobile app, desktop app, installer): its startup path was verified before distribution, by the strongest means the artifact allows. A passing build proves compile/link time only — dynamic-linking and startup failures appear at first launch. Where the artifact can be launched locally, launch it and reach its first screen. Where it cannot (e.g. a store-signed mobile binary, installable only through the store's own distribution channel), substitute **both** a static equivalent — resolving the binary's cross-module/dynamic symbol imports against what its bundled libraries actually export — **and** a staged rollout in which one recipient launches successfully before the rest are notified. This item must never be recorded as satisfied by a launch that cannot physically occur
- **A green suite is not a visual review.** When a unit changes how something looks, report the pass count as evidence of behaviour only and name the visual result as owed to a human
- **Every past-tense claim names the run that produced it, and is written after reading that run's output.** A prediction in the past tense reads as a measurement because of its evidenced neighbours. Write an evidence line in a step after the command, quoting its output; otherwise use the conditional or mark it unconfirmed
- **The gate is run as the project defines it, package-wide and in full, before every push CI will check — never on the changed files only.** Typecheck, lint, format, dead-code, audit, coverage, build, per the project's verify command: run the whole gate, not just the command you changed. A per-file lint or typecheck is a different command with a different answer. If a local dependency the gate needs is unavailable (no container runtime, no local database), the blocked suites may instead be verified by a manually triggered CI run on the branch, confirmed green BEFORE merging to the trunk or cutting a release, with the run's URL recorded in the unit or bolt file; everything that can run locally still runs locally first. Tests may also read non-code artifacts (scripts, manifests, workflows, fixtures) as text, so before pushing a rewrite of one, grep the test directories for its name. A red gate is a finding, never something left for the next session
- **A diff that falsifies a standing claim corrects it in the same commit.** Rule files, ADRs, and comments of the form "X cannot happen yet" state preconditions nothing re-derives. Before Done, ask which claim this diff makes untrue — a new credential class, writer, or trust boundary — and fix it now; prefer a guard that fails loudly to a comment explaining why it cannot fire
- **The backlog row is updated in the same commit as the unit's record.** The row is read without the unit file beside it, and a follow-up commit leaves a window in which the backlog disagrees with the tree

### `skills/compact-docs.md`

The compact-docs skill is engineer-triggered and must never run automatically. It archives operational documents older than the project's configured threshold to keep the active workspace manageable without losing institutional memory.

Copy this file verbatim from `process-onboarding-agent/skills/compact-docs.md` to `{FRAMEWORK_ROOT}/skills/compact-docs.md`. No customisation is needed — the archive threshold is read from the master rule file Process Configuration section at runtime.

### `skills/root-cause-analysis.md`

The root-cause-analysis skill applies structured 5-Whys analysis to resolved incidents and filed improvements to find deeper root causes beyond the immediate fix. It classifies findings into three categories — Solution Design, Technology Selection, and Process — and produces recommendations that map directly to new intent files, ADRs, or improvement files.

Can operate on a single file or across a batch to surface cross-cutting patterns and recurring vulnerabilities.

Copy this file verbatim from `process-onboarding-agent/skills/root-cause-analysis.md` to `{FRAMEWORK_ROOT}/skills/root-cause-analysis.md`. No customisation is needed.

### `skills/solution-shaping.md`

The solution-shaping skill runs before mob elaboration to decide the shape of the solution — generic capability or feature-specific implementation, expected usage and scale, the simplest viable approach, extend-vs-build-vs-buy, and reversibility. The signed-off decision is recorded on the intent as a `## Solution Shape` section (plus a `Shape:` header field) and inherited by the design session as binding context, so Phase 0 designs within an agreed shape rather than an open field.

It runs **at the developer's discretion** — the engineer decides per intent whether to run it or go straight to elaboration, and skipping is a legitimate choice for work that doesn't need it. Invoke it on intents where the shape isn't obvious — new capabilities, candidate platform features, or requests that may be better served by extending an existing module or adopting an existing service. `mob-elab-prompts.md` **auto-prompts** for it at the start of a session when the intent has no recorded shape, but never runs it without the engineer's go-ahead — so the step is offered, not forced.

Copy this file verbatim from `process-onboarding-agent/skills/solution-shaping.md` to `{FRAMEWORK_ROOT}/skills/solution-shaping.md`. No customization is needed.

### `skills/design-session.md`

The design-session skill runs as Phase 0 of mob elaboration to establish an agreed design foundation — API contracts, data model sketch, and architectural pattern decisions — before any units are proposed. Mob elaboration inherits the design as binding constraints, so ACs are written against a concrete interface rather than a vague description.

The skill works through three optional areas (API contract, data model, architectural patterns) based on the intent's scope. It produces a `ops/inception/designs/YYYY-MM-DD-<unix_timestamp>-[slug]-design.md` artifact and links it back to the intent file. Any new architectural patterns agreed during the session are written to `{FRAMEWORK_ROOT}/rules/architecture.md` as ADRs immediately.

Copy this file verbatim from `process-onboarding-agent/skills/design-session.md` to `{FRAMEWORK_ROOT}/skills/design-session.md`. No customization is needed — it is called by mob-elab-prompts.md, not invoked separately.

### `skills/uat.md`

The UAT skill runs when all units under an intent are marked Done. It generates a plain-language demo script from the acceptance criteria of every unit under the intent, guides the engineer through a stakeholder validation session, and records the outcome in the intent file's UAT Sign-off section.

The engineer chooses one of three paths: conduct UAT now (the AI walks through the script step by step and records pass/fail per step), defer UAT to a later date (recorded with a revisit date, intent can still close as Implemented), or mark UAT as not required (a reason must be stated). All three choices are recorded. The intent status cannot move to Implemented without a UAT Sign-off entry.

Any UAT failure automatically creates a draft unit in the backlog for the engineer to confirm.

Copy this file verbatim from `process-onboarding-agent/skills/uat.md` to `{FRAMEWORK_ROOT}/skills/uat.md`. No customization is needed.

**Wire into the master rule file Section 6** by adding one routing line:

```markdown
**UAT skill:** read `{FRAMEWORK_ROOT}/skills/uat.md` when all units under an intent are marked Done, or when the engineer invokes it directly. Prompt the engineer to run UAT before setting intent status to Implemented.
```

### `skills/progress-digest.md`

The progress-digest skill generates a plain-language, one-page progress summary for a feature intent — written for non-technical stakeholders (product owners, clients, leadership) who cannot read engineering artifacts. It translates the intent file, bolt status, and unit statuses into a clear picture of what is being built, what is done, and what comes next.

Engineer-triggered at any point during or after delivery of an intent. Produces a single file at `{FRAMEWORK_ROOT}/ops/inception/intents/[intent-slug]-digest.md`, overwriting any previous digest for that intent. The digest is never archived — it reflects the current state of the intent whenever generated.

Copy this file verbatim from `process-onboarding-agent/skills/progress-digest.md` to `{FRAMEWORK_ROOT}/skills/progress-digest.md`. No customization is needed.

**Wire into the master rule file Section 6** by adding one routing line:

```markdown
**Progress digest skill:** read `{FRAMEWORK_ROOT}/skills/progress-digest.md` when the engineer asks for a stakeholder update, progress summary, or digest for an intent.
```

### `ops/inception/dependency-map.md`

A single file that records which intents depend on which others, and which API contracts, data entities, or shared services cross intent boundaries. Updated automatically by the AI after every elaboration sign-off. Read before every bolt planning step.

Copy this file verbatim from `process-onboarding-agent/ops/inception/dependency-map.md` to `{FRAMEWORK_ROOT}/ops/inception/dependency-map.md`. It contains the empty table structure and update log — the AI populates it as intents are elaborated.

The AI must read this file before planning a bolt and flag: (1) any prerequisite intent not yet Implemented, (2) any units in the planned bolt that touch a shared interface owned by a different intent.

### `ops/inception/codebase-findings/`

One file per module, service, or area of the existing codebase — the accumulated record of what the AI has learned by reading that code. Exists so reverse-engineering done for one intent is never repeated for the next, particularly on brownfield/mature projects where new intents routinely depend on undocumented prior implementation.

Copy `_template.md` and `README.md` verbatim from `process-onboarding-agent/ops/inception/codebase-findings/` to `{FRAMEWORK_ROOT}/ops/inception/codebase-findings/`. The folder starts with only these two files — individual finding files (e.g. `payments-service.md`) are created by the AI the first time it analyzes that module, per the `mob-elab-prompts.md` protocol above. `README.md` contains the index table the AI checks before repeating any code analysis.

### `skills/new-engineer-induction.md`

The new-engineer-induction skill runs when an engineer joins an AI-DLC project for the first time. It reads the project's actual master rule file, domain glossary, quality gate, and backlog — then explains each section in plain language, demonstrates the quality gate with a project-specific example, optionally runs a practice elaboration for two units, and produces a personalized quick-reference card written to `{FRAMEWORK_ROOT}/guidelines/[engineer-name-slug]-quick-ref.md`.

The session takes 30–45 minutes. Engineers who want a faster version say "quick tour" to skip the practice elaboration.

Copy this file verbatim from `process-onboarding-agent/skills/new-engineer-induction.md` to `{FRAMEWORK_ROOT}/skills/new-engineer-induction.md`. No customization is needed — the skill reads the project's own files to personalize the session.

**Wire into the master rule file Section 6** by adding one routing line:

```markdown
**New engineer induction skill:** read `{FRAMEWORK_ROOT}/skills/new-engineer-induction.md` when an engineer says they are new to the project, or invokes it directly.
```

### `skills/knowledge-promotion.md`

The knowledge-promotion skill evaluates each applied improvement to determine whether it is generic (beneficial to all AI-DLC projects) or project-specific. For generic improvements, it drafts the exact change needed in the base repository — file path, current text, and proposed replacement — so the engineer can raise a PR against `ai-dlc-base` without having to reconstruct the context later. The promotion decision and draft are recorded in the improvement file.

Runs automatically as Step 5 of the Post-Retro Improvement Workflow after all improvements are applied. Can also be invoked directly against a specific improvement. A retro is not fully closed until every Applied improvement has a Knowledge Promotion status recorded.

The skill uses the target file as the primary classification signal — files copied verbatim from the base repo (skills, templates, `engagement.md`) are classified generic; files generated per project (code-standards, architecture, domain-glossary, etc.) are classified project-specific. Ambiguous cases are resolved using a content test: would this improvement still apply to a project with a completely different tech stack and domain?

Copy this file verbatim from `process-onboarding-agent/skills/knowledge-promotion.md` to `{FRAMEWORK_ROOT}/skills/knowledge-promotion.md`. No customization is needed.

**Wire into the master rule file Section 6** by adding one routing line:

```markdown
**Knowledge promotion skill:** read `{FRAMEWORK_ROOT}/skills/knowledge-promotion.md` as Step 5 of the Post-Retro Improvement Workflow, after all improvements are applied. A retro is not closed until every Applied improvement has a Knowledge Promotion status.
```

### `skills/process-visualization.md`

The process-visualization skill reconstructs how a bolt (or a whole intent) actually got delivered and renders it as Mermaid diagrams — an actual delivery timeline (`gantt`) and an actual execution path (`flowchart`) — plus a plan-vs-actual deviation table comparing the bolt's recorded `Execution Order` against what really happened. When the project is a git repository, it mines commit history for each unit and bolt file to find the real dates a `Status:` field changed; otherwise it falls back to the dates already recorded in the artifacts and says so plainly.

Offered at the start of every retro, before "What Went Well" is discussed, the same way `solution-shaping.md` is offered before design sessions — the engineer accepts, skips, or runs it later. Output is written into the retro file's "Delivery Flow (What Actually Happened)" section, so the rest of the retro discussion has a factual anchor instead of relying on memory. It complements the retro's Round Trips section: that records what one surface cost; this shows where the whole bolt's time went.

Copy this file verbatim from `process-onboarding-agent/skills/process-visualization.md` to `{FRAMEWORK_ROOT}/skills/process-visualization.md`. No customization is needed.

**Wire into the master rule file Section 6** by adding one routing line:

```markdown
**Process visualization skill:** offer to read `{FRAMEWORK_ROOT}/skills/process-visualization.md` at the start of every retro, before "What Went Well" is discussed. The engineer may accept, skip, or invoke it directly at any time. Never run it without the engineer's go-ahead.
```

### `skills/dependency-audit.md`

The dependency-audit skill audits third-party dependencies for major version drift, end-of-life packages, and known vulnerability patterns. It runs in two phases: AI analysis of manifest files using training knowledge, followed by a tool-assisted scan where the engineer runs their ecosystem's security scanner (npm audit, pip-audit, bundler-audit, etc.) and shares the output. Findings are classified by severity and converted into Remediation Bolts in the backlog.

The skill is scheduled — the next audit date is stored in the master rule file Section 9 (Process Configuration). At the start of every session, the AI checks whether the scheduled date has passed and prompts the engineer if so. The engineer may run the audit immediately or defer it to a new date. The recommended cadence is once a month.

Copy this file verbatim from `process-onboarding-agent/skills/dependency-audit.md` to `{FRAMEWORK_ROOT}/skills/dependency-audit.md`. No customization is needed.

**During onboarding (Step 9 of the Process Configuration section):** ask the engineer when they would like to schedule the first audit and populate the `Next dependency audit` row accordingly.

### `skills/architecture-review.md`

The architecture-review skill reads the whole codebase periodically for anti-patterns, duplication, architectural drift and complexity creep, and produces a ranked, dated report of findings with `file:line` evidence, a severity and a concrete recommendation each. It measures the code against the project's own rules — the anti-pattern catalogue in `code-standards.md`, the ADRs, the hard stops and the glossary — not against generic taste, and it tracks each finding across reviews (fixed / still open / worse / new), which is the part a one-off review cannot give. It is read-only: it proposes the top items as candidate bolts for the engineer to ratify and never refactors itself. Where `review-checklist.md` gates one change, this is the whole-codebase, over-time view.

Recommended cadence: monthly, alongside the dependency audit, and after any bolt that added a large new surface.

Copy this file verbatim from `process-onboarding-agent/skills/architecture-review.md` to `{FRAMEWORK_ROOT}/skills/architecture-review.md`. No customization is needed — it reads the project's rules files at runtime.

**Wire into the master rule file Section 6** by adding one routing line:

```markdown
**Architecture review skill:** read `{FRAMEWORK_ROOT}/skills/architecture-review.md` when the engineer invokes it, or monthly alongside the dependency audit — offer it whenever the dependency audit is prompted. Read-only: it recommends refactors, it never makes them.
```

### `skills/disk-hygiene.md`

The disk-hygiene skill reclaims workstation disk space that AI-assisted development consumes passively — stale worktrees (each with a full dependency install), package-manager caches, container build caches and orphaned volumes, simulator images. Nothing else in the process reclaims it, and a full disk fails builds with misleading errors. The skill works in tiers from safe caches to judgment calls, and its two hard rules are never to delete unmerged or uncommitted work, and never to delete a dev database or untracked local data — including a dev database on an **anonymous** container volume, which it protects by cross-checking every target against every container's mounts. Every destructive step is verified by re-measuring, never by trusting the command's output.

The skill is scheduled like the dependency audit — the next sweep date lives in the master rule file Section 9 — and also runs unscheduled whenever free space falls below the headroom threshold recorded there. Its commands are written for macOS with Docker and the Node, Xcode and Android toolchains; steps for tools a workstation does not have are skipped.

Copy this file verbatim from `process-onboarding-agent/skills/disk-hygiene.md` to `{FRAMEWORK_ROOT}/skills/disk-hygiene.md`. No customization is needed — it reads `guidelines/dev-setup.md` at runtime to learn how the project's dev database is started.

**Wire into the master rule file Section 6** by adding one routing line:

```markdown
**Disk hygiene skill:** read `{FRAMEWORK_ROOT}/skills/disk-hygiene.md` when the engineer invokes it, when the `Next disk hygiene sweep` date in Section 9 has been reached, or unscheduled whenever free disk space is observed below the headroom threshold in Section 9. Prompt at session start if the date is due.
```

### `skills/process-health.md`

The process-health skill analyses the project's operational artifacts and produces a quantitative report on how well the AI-DLC process is functioning. It calculates four metrics — improvement adoption rate, quality gate failure rate, AC revision rate, and bolt velocity trend — and surfaces additional decay signals such as retros with no improvements, stale open improvements, and recurring failure patterns.

The report is presented in-conversation and always saved to `{FRAMEWORK_ROOT}/ops/operate/process-health-YYYY-MM-DD.md` automatically. Reports accumulate over time, making it possible to compare health trends across runs.

Recommended cadence: after every third or fourth bolt, or whenever the team suspects the process has drifted.

Copy this file verbatim from `process-onboarding-agent/skills/process-health.md` to `{FRAMEWORK_ROOT}/skills/process-health.md`. No customization is needed.

**Wire into the master rule file Section 6** by adding one routing line:

```markdown
**Process health skill:** read `{FRAMEWORK_ROOT}/skills/process-health.md` when the engineer invokes it to audit how well the AI-DLC process is functioning.
```

### `skills/bolt-risk-assessment.md`

The bolt-risk-assessment skill runs after elaboration sign-off and before the first unit in a bolt executes. It actively interrogates each unit for blast radius, cross-unit sequencing risks, rollback feasibility, and feature flag requirements — replacing the passive "Risks and Assumptions" table in the bolt file with structured, engineer-signed-off findings.

For mature projects (those with existing code), this assessment is mandatory before any unit executes. For fresh projects with no existing modules affected, it may be brief but must still be completed.

Copy this file verbatim from `process-onboarding-agent/skills/bolt-risk-assessment.md` to `{FRAMEWORK_ROOT}/skills/bolt-risk-assessment.md`. No customization is needed.

**Wire into the master rule file Section 6** by adding one routing line:

```markdown
**Bolt risk assessment:** read `{FRAMEWORK_ROOT}/skills/bolt-risk-assessment.md` after elaboration sign-off and before the first unit in a bolt executes. No unit may begin execution without a signed-off risk assessment in the bolt file.
```

### `skills/bug-bolt.md`

The bug-bolt skill is a lightweight bolt workflow for fixing a specific, reproducible bug. It replaces the full elaboration ceremony with a four-question intake, a recurrence check against retro and incident history, a single focused unit with regression-guard ACs, and a mandatory RCA if the bug is recurring. Skips: design session, dependency map update, mob elaboration. Preserves: pre-generation checks, review checklist, blast radius assessment for shared components, and post-fix retro.

Copy this file verbatim from `process-onboarding-agent/skills/bug-bolt.md` to `{FRAMEWORK_ROOT}/skills/bug-bolt.md`. No customization is needed.

**Wire into the master rule file Section 6** by adding one routing line:

```markdown
**Bug bolt:** read `{FRAMEWORK_ROOT}/skills/bug-bolt.md` when the engineer says "fix a bug", "there's a bug in X", or "bug: [description]". Do not run a full mob elaboration — follow the bug bolt workflow directly.
```

---

### `skills/hotfix-bolt.md`

The hotfix-bolt skill is an emergency bolt for production incidents that cannot wait for a planning cycle. It runs a three-question intake (symptom, severity, rollback availability), creates a minimal unit with a two-AC pair (fix applied / no regressions), confirms the blast radius before any code is generated, and mandates a retro and RCA within 24 hours. Skips: elaboration, design session, dependency map update. Preserves: review checklist, pre-generation checks, incident file creation, retro.

Copy this file verbatim from `process-onboarding-agent/skills/hotfix-bolt.md` to `{FRAMEWORK_ROOT}/skills/hotfix-bolt.md`. No customization is needed.

**Wire into the master rule file Section 6** by adding one routing line:

```markdown
**Hotfix bolt:** read `{FRAMEWORK_ROOT}/skills/hotfix-bolt.md` when the engineer says "hotfix", "production issue", "prod is down", or "emergency fix for X". Skip elaboration — begin hotfix intake immediately.
```

---

### `skills/nfr-bolt.md`

The nfr-bolt skill handles non-functional quality attribute improvements — performance, security hardening, accessibility, reliability, observability, and scalability. It does not create a new intent; it references and improves existing ones. Key controls: measurable threshold ACs are mandatory (vague ACs are rejected), a before/after measurement is required for bolt closure, affected intents are identified upfront and cross-referenced after completion, and bolt-risk-assessment runs before any unit executes because NFR changes often have wider blast radii than they appear.

Copy this file verbatim from `process-onboarding-agent/skills/nfr-bolt.md` to `{FRAMEWORK_ROOT}/skills/nfr-bolt.md`. No customization is needed.

**Wire into the master rule file Section 6** by adding one routing line:

```markdown
**NFR bolt:** read `{FRAMEWORK_ROOT}/skills/nfr-bolt.md` when the engineer says "improve performance", "harden security", "accessibility improvements", "NFR bolt for X", or "non-functional work on X". Do not create a new intent — follow the NFR bolt workflow.
```

---

## Step 5 — Write the Guidelines Files

### `guidelines/domain-glossary.md`

Full definitions for every domain term. Each entry: term, definition, usage notes, and related terms. The glossary in the master rule file is a summary — this file is the authoritative source.

### `guidelines/edge-cases.md`

Full descriptions of each known edge case: the scenario, the required behavior, and which parts of the system must handle it. Start with the ones most relevant to your domain. Add entries from retros and incidents.

Seed it with stack-independent entries as well as domain ones — shapes worth having in the file before a retro finds them the hard way:
- A test whose fixture is derived by the same route as the code under test shares the code's wrong assumption, and is green exactly when the defect ships. State fixtures literally, or derive them by a different route so the two can disagree.
- A property protected by two mechanisms is pinned by an outcome test that stays green while either one survives. Say so on the test, probe with both removed so it is known it can fail, and record it as "pins the property, neither mechanism".
- A configuration fact — a scope, a limit, an endpoint — restated in several documents drifts silently. Keep it in one and link from the rest, above all from anything published.
- Where data arrives at differing trust levels (verified vs extracted or inferred): **an input to a scoring, placement or calibration rule carries the trust of its WEAKEST evidence path, not its best.** When a rule consumes a value an inference or extraction lane can populate, the decision record states whether such values are trusted for that use — or the rule excludes that path. Tests confirm a formula as specified, so a spec that trusts an inferred input passes green while being wrong.
- **In-memory state written after an `await` in a per-event handler is unordered.** Each handler can be correct on its own while the system assumes handlers finish in the order their events arrived. Such state is either **monotonic** (a write that would move it backwards is refused) or **serialised**; a reader that pairs it with a database row reads the in-memory state **first**, before the row; and every new store of this kind names which of the two it is and has a test that delivers two writes out of order. *(makerclub, 2026-09-28: twice in one bolt and once in a peer's handler the same day — a late progress frame overwrote a newer one, and a watcher paired a row read before a state change with a stage read after it.)*

### `guidelines/acceptance-patterns.md`

Rules for writing good Given/When/Then ACs:
- One behavior per criterion (no compound ACs)
- Name the actor in every Given
- Cover at least one unhappy path per unit
- No implementation details in ACs — name the outcome, and a mechanism only when choosing it is the decision being signed (a table, an endpoint, where a token lives). The test: would the engineer care if it were done another way with the same result? If not, it comes out
- An AC that asserts a STRUCTURAL fact (a constraint exists, a column is nullable, a parameter has no safe default, there are N call sites) carries the check that established it in one clause: the command, the file:line or the constraint name. An unchecked structural premise is not a weak AC but a false one, signed off and planned against, then withdrawn mid-execution. The authoring tell is "is enforced", "cannot be null", "there are N" or "always/never has": ask what you looked at, and if the answer is "a similar table" or "it must be", look. The same applies to structural claims in code comments
- Anti-patterns to avoid (vague outcomes, testing implementation not behavior)
- On a web surface, an AC that says a mark is "visible" names every background it sits on and a contrast threshold — 3:1 for a focus ring or other non-text mark (WCAG 1.4.11), 4.5:1 for body text (1.4.3) — and the unit records the measured ratio for each background. A mark that is *there* passes every check that does not compare it with what is behind it
- A degraded path whose trigger is the VIEWER'S ENVIRONMENT — a missing embedded viewer or codec, images or scripts disabled, a blocked font, an offline or aborted request, a reduced-motion or assistive-technology preference, a failed media load — gets an AC that forces that environment and asserts what the user then reads; no amount of data-driven testing reaches it. Beware the element that is present in the DOM but never rendered in a normal run: an existence or visibility assertion on it passes or fails on a property of the test runner, so assert its attribute and text, and note why at the assertion so nobody "fixes" it
- A fixture chosen for the ROLE it plays (the unsupported format, the expired token, the invalid payload) gets one named home, such as a factory or helper documented with why it exists, and every test that needs the role refers to it. Nothing in the tree marks a role, so when the role's meaning changes (the "unsupported" format gains support), every test playing it keeps passing while asserting nothing, and searches by filename or type undercount them. When a role has one candidate left, say so in that home's docstring: losing it means rethinking the path, not repointing it

Questions the AC set must answer before sign-off:
- **Every AC names the control that triggers it.** "None" means the unit is not done. An AC whose Given names the act in the past tense ("given a record one user edited") tests a consequence of the act, and stays true of a product with no way to perform it; pair it with an AC that the act is reachable and works
- **Remembered state has a correction path.** When a unit makes the system remember something from a user action — a learned default, a taught category — the ACs say how a wrong memory is corrected, whether correcting re-teaches, and what happens to items already filed under it. "Remembers" or "never asks again" with no "correct" or "change" is the tell
- **An effect added to a reversible act states what the inverse does.** If retire, remove, disconnect or archive now also changes something, an AC says what restore, re-add, reconnect or unarchive does to that same thing — or "unchanged, because …"
- **"A and B differ" names what makes them differ.** Name the varying input — a clock, a counter, a random source, a signature — and check its precision, since two values made within its resolution are identical; then assert the property (it is recomputed on every read), not the proxy
- **A numeric bound shows its derivation**, or says it is derived in the unit from a named constant and makes that derivation an AC. Either way, a test builds the largest legal input and asserts it fits. A figure with no arithmetic beside it is pasted forward as fact
- **A date-dependent AC names a case either side of a clock change**, and the suite sets the time zone it claims and asserts the setting took — likewise any other ambient process state the claim depends on (locale, clock, working directory). On a runner in another zone, both cases silently run the same one
- **"The user can reach X" is Done only against a recorded observation in a real environment** when the test runner never lays the screen out — it finds an element mid-screen and one past the edge identically. Adding one more control to an existing row is the usual trigger
- **A guard planted for a later unit is declared in that unit.** A test that asserts something is absent so a future unit must turn it red is a good handoff, but it collides with the default "pass without modification" AC. The unit arming it writes the expected red into the target unit's Definition of Done (or the bolt's open items until that unit exists) and names the target in the test's comment; undeclared, a firing tripwire is indistinguishable from a test bent to fit the code

Proving an AC with a test:
- **A green falsification probe can mean the guard is unreachable.** If a defect you proved was installed leaves the suite green, first confirm control flow reaches the changed line (a throw or log in the branch). A guard that cannot fire is a production defect; the fix is structural, not more coverage
- **A timing test owns the quantity it asserts.** Ask what the assertion depends on, and say it at the call site. In order of preference: inject the clock → own the event (fire the interval or release each response by hand — the only option when the value is a count) → test the arithmetic as a pure function → choose a real-time window. An injected seam that reads the ambient source is not owned: would the value change if the machine got busier? Size any real window per test from what it asserts; never assert a transient state with a wait-for-arrival helper, where "not yet" and "already gone" fail identically; and when the harness forces an `await` back into the race, assert the recorded order of events rather than current state
- **An authenticated route's ACs include the anonymous-malformed case:** _Given no credential and a body that would fail validation, when the route is called, then the answer is `401` and says nothing about the body._ A suite whose every request sends a valid body cannot see an auth check that runs after body parsing or validation. *(makerclub, 2026-09-21: a token check placed after body parsing answered anonymous bad bodies with 400, describing the schema to a caller who had not authenticated, while 71 green tests all sent valid bodies.)*
- **A test for a user-facing string pins the AC's words, never the constant the code exports.** Where an AC quotes copy a person reads — an error, a prompt, a button — the test spells out the literal string from the AC, not an import of the constant (or a template built from the same source) the code uses; everywhere else, importing the constant is right. An assertion whose expected value comes from the module under test proves only that the module agrees with itself, and the shape is mechanical enough to sweep for. *(makerclub, 2026-09-21: a falsification probe changed an error message and the suite stayed green, because the test imported the constant the probe had just mutated.)*
- **A test vector for a selection rule puts the winner where the naive rule would not pick it.** "Keep the strongest", "the newest", "the highest score" each have a naive reading — keep the first — that a fixture can pass by accident. List the winner second or later, and say so in a note on the fixture so a reader knows the order is deliberate. *(makerclub, 2026-10-05: a "keep the strongest" vector listed the strongest first, and an implementation that kept the first passed it; reordered, the same mutation went red.)*


### `guidelines/dev-setup.md`

Step-by-step environment setup for a new engineer:
- All prerequisites with version checks
- Configuration files needed and what goes in them (no actual secrets — explain how to get them)
- How to start each service
- Auth verification steps
- Secrets hygiene checklist
- New third-party credential order: mint → validate with a direct provider call (from the terminal; never pasted into chat) → store as a platform secret → wire the env var → verify in-product. A key never proven working turns every downstream failure into a multi-party diagnosis
- How to tell that new code is actually running. A dev server's reload or rebuild call that returns success is not proof. Name a sign that only a real restart produces, such as app state that resets or a build id that changes, and check it before you trust anything you see on the screen
- Where a local run and CI differ: tests skipped locally, dependencies installed locally but not on the runner, leftovers in a shared local database. A local count that matches CI's does not mean the same tests failed. Give a recipe for reproducing CI's install shape, for example a fresh worktree that links only the dependencies CI installs
- Give the gate's build its own output directory, never the one a running dev server serves from. A shared directory corrupts the live server, which then fails every route and looks like a hang rather than a collision
- **A new worktree's bring-up, as a script** — see below
- **How each delivery is taken by the thing that receives it, and how to prove which version it runs.** An over-the-air app update, a CDN-cached web client, a device image: each takes effect on its own schedule, and the record of a delivery says what the receiver has to do (relaunch twice, hard-reload, wait for the reboot) and which log line or screen proves the new version is the one running. A walk that tests the old version reads as a bug in the new one. *(makerclub, 2026-09-26: an OTA app update applies on the second relaunch; the first walk after it ran the old bundle, which behaved correctly for itself and was read as a defect.)*

**The project's gates live in ONE script, and dev-setup names it.** A rule in a document does not run; a script does. Write the gate as `scripts/check` (or the stack's equivalent) on the first day the project has more than one test command, and describe here what it guarantees:

- **It runs every gate and captures each exit code on its own line, before anything can pipe it.** `set -e` that did not stop a shell, a build piped into `tail`, a test run piped into `grep` — each replaces the real status with the status of the last command in the pipe. Report green only from the script's own verdict line, never from a wrapper's "exited 0" printed around it. *(makerclub, 2026-09-21: four swallowed failures across two bolts, one leaving a red test on `main` for eleven minutes — the prose rule was written after the first and the next two happened anyway.)*
- **Each run owns what it uses, and is immune to its own file being edited mid-run.** On a machine where two sessions can gate at once: a database created for this run and dropped on exit, a log directory of this run's own, and the whole script wrapped in one function called on the last line, so the interpreter has parsed all of it before any of it runs. *(makerclub, 2026-09-22: two sessions' gates shared one test database and one log directory; the backend went red on missing tables, and a peer's edit to the script broke the shell mid-read.)*
- **It holds the machine awake for its own run, and says when the machine slept anyway** — a test that "timed out" across a laptop's sleep is indistinguishable from broken code. Print the machine's load too, so a red can be read against it. *(makerclub, 2026-09-24: four 15-second tests reported 22–46 minutes each across a lid close.)*
- **On a developer machine, it runs on the pinned runtime or refuses**, with a distinct exit code, before any gate. A tool version that is merely documented gets missed; a wrong package manager rewrites the lockfile in a commit nobody reads hunk by hunk. CI may run an older runtime on purpose, because CI never writes the lock. *(makerclub, 2026-10-01: a shell on the wrong Node major had its npm rewrite the lock and drop every platform field from it.)*
- **It refuses an installed dependency tree that does not match the lockfile** — the same versions installed as locked, every non-optional entry present, nothing extra — not only a missing one, and it re-checks after every pull. A stale tree fails type-checking in a way that looks like code. *(makerclub, 2026-10-05: two reds on the type checker in one session, once in a fresh worktree and once after a fast-forward.)*
- **Framework hygiene is one of its gates.** Each of these is one grep, and each existed in makerclub until something ran it: a Done unit still holding the unit template's placeholder text; a unit whose bolt file does not exist; **a unit its named bolt does not list, or whose bolt's scope excludes what the unit touches** (a link that resolves is not a bolt that has assessed the unit); a bolt with every unit Done and no retro; an id placeholder surviving in a pushed file. *(makerclub, 2026-09-28: a unit linked an existing bolt whose table did not list it and whose scope said "no firmware", and reached a device under a risk assessment that had never read it.)*
- **The build context is checked, not assumed.** Every file a container image copies is tracked by git *and* survives the ignore file of the build context (`.dockerignore` or equivalent) — tracked is not the same as shipped. Make the check prove it fails on exactly the bad file. *(makerclub: an ignored lock file built only on the machine that had generated it, 2026-09-22; a new script under an excluded directory failed a cloud build four minutes in, 2026-09-27.)*

**A pre-deploy check is a second script, and its verdict is FAIL or not — never a judgement.** It runs the gate on the exact tree it deploys, checks that tree is the published trunk with nothing uncommitted, reads what is live *now* against what the deployer expected to replace, and takes a lock against a concurrent deploy. Rules for its side effects and its reds:
- **A temporary access rule (a firewall opening, a short-lived credential) is named per run — a timestamp and process id — and the script's exit trap removes only its own.** The timestamp lets a stray-rule check tell a peer running now from a leftover. *(makerclub, 2026-09-28: two read-only checks shared one rule name, each one's trap deleted the other's, and the survivor hung waiting for a port.)*
- **Load can make a gate falsely red, never falsely green.** On a machine far above its core count, a red made of timeouts and zero assertion failures measures the machine. The check still runs the gate, and a red under load is reported with the load, as "re-run when quieter", not as a finding. *(makerclub, 2026-09-28: at a load of 174 on 10 cores, 34 timeouts and 0 assertion failures. It first refused to start the gate under load, as a FAIL; on a machine shared across projects that refused deploy after deploy of green trees, and was relaxed to the rule above on 2026-10-04.)*

**The trunk is deployable by anyone only if nothing on it waits for another artefact.** When a deploy ships the trunk, one session's code-complete-but-undeployed commit rides along with every other session's deploy. So a push of code that needs a newer version of a separately deployed artefact (a worker, a schema, a device runtime, a mobile build) either deploys that artefact at the push, or writes in its backlog row which version it waits for — and nobody deploys past such a row. *(makerclub, 2026-09-28: the trunk carried features its live compiler could not build, and the next session's unrelated change had to wait behind them.)*

**A new worktree's bring-up is a script, not a discovery.** A worktree shares the history and none of the git-ignored state: local env files, fetched toolchain components, the installed dependency tree. Each missing piece is otherwise found by a gate going red for a reason that looks like code. The script links (never copies) secrets files to the main checkout's, refuses by name a file the main checkout lacks — linking to nothing recreates a vacuous check — runs the dependency-tree check above, and ends `ready` or `NOT ready`, naming each problem. Re-running it is harmless. *(makerclub, 2026-09-24: three pieces of ignored state stood between a fresh worktree and a runnable gate, and the second failed as if it were code.)*

**Ids and migration numbers are allocated at push, from `ops/build/id-reservations.md`.** Work under placeholders (`U-TBD-1`, `ADR-TBD`, a migration as `NNNN_name`), and in the push cycle — fetch, rebase, read each table's "Next free" marker at that commit, substitute, add the claim rows and advance the markers in the same commit, gate, push. Migration numbers are ids like any other: a plan that hands out numbers hands the same one to two units. **The stamp counts placeholders before and stamped ids after, over the same file list, and requires them equal and above zero** — a count higher than the placeholders you wrote means you stamped a peer's rows, and a count of zero means the substitution read no files, which prints exactly what success prints. *(makerclub: a plan gave migration `0003` to two units, 2026-09-21; a file list passed as one unsplit string made the stamp read nothing and report nothing, 2026-09-28.)*

### `guidelines/team-rollout.md`

Covers: git branching conventions, environment isolation, secrets management, backlog ownership, multi-FDE Bolt rhythm, PR checklist, merge conflict resolution (including AI-assisted resolution), team leader review guidelines, and common anti-patterns.

---

## Step 6 — Write the Ops Templates

Each template file defines the structure for its artifact type. The key templates:

### `ops/inception/intents/_template.md`
Fields: Status, Date, Owner, **AI Risk** (Minimal / Limited / High — see template for definitions), What, Why, Success Looks Like, Assumptions, Open Questions, Out of Scope, Elaboration Sessions, Extracted Units, UAT Sign-off, Implementation Summary.

The **AI Risk** field gates the review level required: Minimal → standard review; Limited → named engineer sign-off on ACs before elaboration and on Implementation Summary before merge; High → senior engineer approval before any elaboration begins, plus all Limited controls.

The **Implementation Summary** section is written by the AI once all units under the intent have been delivered and merged. It must not be filled in earlier. It contains four subsections:

1. **What Was Built** — user-facing description of every feature delivered under the intent; one paragraph per distinct feature
2. **How It Works (Key Design Decisions)** — data model choices, API contracts, edge cases explicitly handled, and non-obvious implementation facts a future engineer must know before modifying the feature
3. **Scope Delivered vs. Original Intent** — any deviations from the original intent (descoped items, changed assumptions, additions); write "Delivered as specified" if none
4. **Known Limitations and Future Considerations** — constraints the current implementation imposes on future changes; write "None identified" if none

When the last unit of an intent is confirmed done, the AI must:
1. Set the intent status to **Implemented**
2. Write the Implementation Summary by reading the elaboration session files, unit files, and bolt retros for this intent
3. Ask the engineer to review the summary before closing the intent

The Implementation Summary is the authoritative reference for future Bolts that modify or extend this feature. Any Bolt touching a feature covered by an intent must read its Implementation Summary before elaboration begins.

### `ops/build/units/_template.md`
Fields: Status, Intent link, Elaboration link, Bolt link, Priority, Context, Acceptance Criteria, Scope (in/out), Dependencies, Pre-generation Checks, Edge Cases to Handle, Definition of Done, Prompt Log link, Notes.

The **Pre-generation Checks** section is critical for wrapper/layout units — list grep patterns to run across existing files before generating to surface duplication.

### `ops/build/bolts/_template.md`
Fields: Status, Goal, Start/Target/Completed dates, Units table, Execution Order diagram, Risks & Assumptions, Definition of Done, Retrospective link.

### `ops/build/id-reservations.md`

The id ledger: one table per id family (units, bolts, intents, ADRs, edge cases, migration numbers), each row a claimed id with the file or artifact that holds it and the commit that claimed it, and a **"Next free"** marker per table. Nothing is numbered from this file at planning time — work is written under placeholders (`U-TBD`, `B-TBD`, `ADR-TBD`, `NNNN_name`) and the ledger is read, advanced and committed in the push cycle described in `guidelines/dev-setup.md` and the master rule file Section 6. Copy the base file verbatim; it starts with every marker at its first value.

### `ops/operate/retros/_template.md`
Sections: Delivery Flow (What Actually Happened — populated by the process-visualization skill when the engineer accepts the offer at retro start), What Went Well, What Didn't Go Well, Round Trips (how many hand verifications and builds one surface took, and what each bought), AI-Specific Observations (prompts that worked / needed revision / quality gate failures / output accepted without enough review), Actions table, Improvements Triggered (**required** — cannot be left blank without a stated reason), New Intents Triggered, Post-Retro Improvement Workflow.

**The Post-Retro Improvement Workflow is mandatory and AI-driven.** Immediately after the retro document is complete, the AI must:
1. Synthesize every finding in "What Went Well" (a discipline the next session would need), "What Didn't Go Well", "Round Trips" and "AI-Specific Observations" into concrete improvement proposals — one per finding — identifying the exact file and text to change
2. Present all proposals to the engineer for approval, rejection, or revision before touching any file
3. **For each approved proposal:** check which open or in-progress units reference the section being changed (Pre-generation Checks, ACs, or referenced rule files) and present the impact list to the engineer before applying. Record affected units in the improvement file.
4. For each approved proposal: create an improvement file, apply the change to the target file, and update mirror files if the master rule file was modified
5. Mark each improvement Applied in the retro and close the retro only when all approved improvements are applied

The intent is that every retro automatically tightens the rules, skills, and guidelines that govern the next bolt. No finding should require the engineer to manually translate it into a file change.

### `ops/operate/improvements/_template.md`
Fields: Triggered by (retro/incident link), Target file, Current text, Proposed replacement, Reason, Validation criteria, Status, Applied date.

---

## Step 7 — Write Instructions2FDE.md

This is the main onboarding document for every engineer. Sections:
1. What AI-DLC is (the loop diagram: Inception → Build → Operate → Improvements)
2. How to invoke each ceremony by talking to the AI (not by following manual steps)
3. What the engineer still owns (review, AC confirmation, running tests, edge case checks)
4. Phase 1 — Inception (how intents and mob elaboration work)
5. Phase 2 — Build (bolts, unit execution order, review before merge)
6. Phase 3 — Operate (retros, incidents, improvements)
7. The Three Non-Negotiables (quality gate, review checklist, prompt log)
8. Using a different AI tool (Cursor → `.cursor/rules/project-rules.mdc`; GitHub Copilot → `.github/copilot-instructions.md`)
9. Common mistakes table
10. Quick reference table (ceremony → what to say to the AI)

---

## Step 8 — Confirm Tool Setup and Add Mirror Files

### Primary tool setup

By this point your master rule file should exist at the correct path for your chosen tool (see **Before You Begin**). Verify:

| Tool | Expected path | Loaded automatically? |
|---|---|---|
| Claude Code | `CLAUDE.md` at repo root | Yes — every session |
| Cursor | `.cursor/rules/project-rules.mdc` (with `alwaysApply: true`) | Yes — every session |
| GitHub Copilot | `.github/copilot-instructions.md` | Yes — every session |

### Supporting multiple tools in the same repo

If your team uses more than one AI tool, create copies of the master rule file for each additional tool. The content is identical — only the file name, location, internal link prefixes, and (for Cursor) a wrapping YAML frontmatter block differ.

**Add Cursor support** (if your primary tool is Claude Code or Copilot):
```bash
mkdir -p .cursor/rules
printf -- '---\nalwaysApply: true\n---\n\n' > .cursor/rules/project-rules.mdc
cat CLAUDE.md >> .cursor/rules/project-rules.mdc
```
Open `.cursor/rules/project-rules.mdc` and, in the copied content below the `---` frontmatter block, update the opening line to reference Cursor.

**Add GitHub Copilot support** (if your primary tool is Claude Code or Cursor):
```bash
mkdir -p .github
cp CLAUDE.md .github/copilot-instructions.md
```
Open `.github/copilot-instructions.md`, update the opening line to reference GitHub Copilot, and change all `{FRAMEWORK_ROOT}/` link prefixes to `../{FRAMEWORK_ROOT}/`.

**Add Claude Code support** (if your primary tool is Cursor or Copilot):
```bash
# From Cursor rule file — strip the YAML frontmatter before copying:
tail -n +5 .cursor/rules/project-rules.mdc > CLAUDE.md
# Or from Copilot:
# cp .github/copilot-instructions.md CLAUDE.md
```
Open `CLAUDE.md`, update the opening line to reference Claude Code, and if copying from Copilot change all `../{FRAMEWORK_ROOT}/` prefixes back to `{FRAMEWORK_ROOT}/`.

### Sync discipline

Whenever the master rule file is updated, all mirror files must be updated in the same PR. Add this to your PR checklist — it is a team discipline, not an automated process.

---

## Step 9 — Initialize the Backlog and First Intent

1. Copy `process-onboarding-agent/ops/build/backlog.md` to `{FRAMEWORK_ROOT}/ops/build/backlog.md` — it already contains the empty status sections and the Reference Link Registry comment at the bottom.
2. Copy `process-onboarding-agent/ops/build/id-reservations.md` to `{FRAMEWORK_ROOT}/ops/build/id-reservations.md` — the id ledger every push cycle reads and advances (see `guidelines/dev-setup.md`, Step 5).
3. Copy `process-onboarding-agent/ops/inception/dependency-map.md` to `{FRAMEWORK_ROOT}/ops/inception/dependency-map.md` — it contains the empty map structure and update log.
4. Identify the first capability to build and write an intent: `ops/inception/intents/YYYY-MM-DD-<unix_timestamp>-<slug>.md`
5. Say to your AI assistant: "Run a mob elaboration for the [intent name] intent"
6. After sign-off, the AI creates unit files, updates the backlog, and updates the dependency map. **When adding any unit or bolt to the backlog, the AI must use reference-style links** — write the display text as `[Unit-name][unit-slug]` in the table and add the path definition to the Reference Link Registry at the bottom of the file. Never use inline URLs in backlog tables.
7. Say: "Plan a bolt from the open units in the backlog" — before creating the bolt file, the AI reads `ops/inception/dependency-map.md` and flags any prerequisite intents that are not yet Implemented, or any units that touch a shared interface owned by a different intent.
8. Say: "Execute unit [name] from bolt [name]"

---

## What Makes This Work

The quality of the framework depends entirely on two things:

**1. The specificity of the master rule file.** Generic rules produce generic output. The more your master rule file encodes your actual domain language, your actual architectural decisions, and your actual learned anti-patterns, the better every AI interaction will be. A master rule file written on day one will be much weaker than one shaped by three bolts of retros.

**2. The discipline of the retro loop.** The retro is the compiler for the process. Every failure that is not encoded into a rule will recur. Every retro that produces no improvement file means the next bolt starts from the same baseline. Run the retro after every bolt, file the improvements, and the framework compounds.

---

## Onboarding Completion — What the Agent Must Produce

When the agent has finished executing this guide, it must output a structured completion report before handing back to the engineer. The report must contain:

### 1. Files Created
A table of every file written during onboarding, grouped by folder.

| File | Status | Notes |
|---|---|---|
| `CLAUDE.md` (or tool equivalent) | Created | Sections 1–9 populated |
| `{FRAMEWORK_ROOT}/rules/...` | Created | … |
| *(etc.)* | | |

### 2. Sections Requiring Engineer Review
List every section or field in the master rule file that the agent could not populate from the codebase and left as a placeholder. The engineer must fill these before the first Bolt runs.

### 3. Open Questions
Any ambiguity the agent encountered that the engineer must resolve — e.g., conflicting patterns found in the codebase, modules where ownership was unclear, or test coverage below the gate threshold.

### 4. First Recommended Action
One sentence: what the engineer should do next before starting the experience agent (e.g., "Review the domain glossary placeholders in Section 4 of the master rule file, then start a new session to begin the first mob elaboration.").

### 5. How to Start the Experience Agent

This is the final and mandatory step. After the report is presented, the agent must say:

---

> **Onboarding is complete. The experience agent is now embedded in your `[master rule file name]`.**
>
> **This session is now finished. Do not continue working in this conversation.** The onboarding session has accumulated context — interview answers, archaeology findings, file creation history — that is no longer needed and will slow down and distort future AI sessions.
>
> **To start the experience agent:**
> 1. Close or end this conversation entirely.
> 2. Open a brand new session inside your project repository using your AI tool ([Claude Code / Cursor / GitHub Copilot]).
> 3. Your AI tool will automatically load `[master rule file path]` at the start of the session.
> 4. Say: **"[first action from item 4 above]"**
>
> From that point forward, every session in this repository is an experience agent session. The onboarding agent is not needed again unless you are onboarding a new project.

---

> The agent must not mark onboarding as complete until all checklist items below are checked and this report — including the handoff instruction — has been presented to the engineer.

---

## Checklist: Ready to Start

**Tool setup**
- [ ] AI tool identified (Claude Code / Cursor / GitHub Copilot)
- [ ] Master rule file created at the correct path for your tool (see Before You Begin)
- [ ] Mirror files created for any additional tools in use (see Step 8)

**Framework files**
- [ ] Folder structure created (`{FRAMEWORK_ROOT}/` tree from Step 1)
- [ ] Master rule file written with all 9 sections (Step 2)
- [ ] `rules/` files written (prompt-quality-gate, code-standards, security, architecture)
- [ ] `skills/` files written (mob-elab-prompts, review-checklist)
- [ ] `guidelines/` files written (domain-glossary, edge-cases, acceptance-patterns, dev-setup)
- [ ] Ops templates written (intent, unit, bolt, retro, incident, improvement, codebase-findings)
- [ ] `Instructions2FDE.md` written
- [ ] `{FRAMEWORK_ROOT}/README.md` written

**First iteration**
- [ ] First intent written and ready for mob elaboration
