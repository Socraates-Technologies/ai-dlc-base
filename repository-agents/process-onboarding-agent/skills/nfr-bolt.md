# Skill: NFR Bolt

**Purpose:** A bolt workflow for non-functional quality attribute improvements — performance, security hardening, accessibility, reliability, observability, or scalability. Operates on existing intents rather than creating a new one. Every AC must carry a measurable threshold; vague targets ("faster", "more secure") are rejected.

**Trigger:** Engineer says "improve performance of X", "harden security for X", "accessibility improvements", "NFR bolt for X", "non-functional work on X", or "reliability improvements to X". Routed from Section 6 of the master rule file. Do not create a new intent file — load this skill and reference the existing affected intents instead.

**Requires:** A fully onboarded AI-DLC project with at least one delivered intent. The master rule file, quality gate (`rules/prompt-quality-gate.md`), review checklist (`skills/review-checklist.md`), dependency map (`ops/inception/dependency-map.md`), unit and bolt templates, and the `ops/` folder structure must all be in place. This skill is a bolt-type variant that operates within the framework — it is not a standalone performance or security checklist. If AI-DLC is not yet installed, start with `process-onboarding-agent/onboard.md` first.

**What this skill does not replace:** intent files for existing features (it references them), mob elaboration (for new features), UAT, or the standard retro. An NFR bolt improves how existing features behave — it does not add new behavior. Any NFR change that requires new behavior visible to users must be captured as a new intent using the standard feature flow.

NFR categories: Performance · Security Hardening · Accessibility · Reliability / Resilience · Observability · Scalability

---

## Step 1 — NFR Intake

Ask the engineer four questions:

> "To scope this NFR bolt accurately:
> 1. **Which quality attribute?** Performance / Security Hardening / Accessibility / Reliability / Observability / Scalability / Other — specify.
> 2. **Which component or feature?** Name the service, module, or user flow being improved.
> 3. **What is the current state?** A measurement or description of today's behavior — e.g. p95 latency is 1.2 s; screen reader navigation is broken on checkout; no retry logic on payment calls.
> 4. **What is the target threshold?** A specific, measurable outcome — e.g. p95 latency < 300 ms; WCAG 2.1 AA compliant; three retries with exponential backoff on payment calls."

Do not proceed without a measurable target threshold. "Better", "faster", and "more secure" are not valid thresholds — ask again until a number or verifiable criterion is provided.

**Scope the threshold to what the change CONTROLS, not to the outcome the engineer wishes for.** A measurable threshold can still be the wrong threshold: if it depends on defects the unit does not touch, then doing the work perfectly cannot satisfy it, and the bolt lands in Step 7's Blocked state for reasons that have nothing to do with the work.

The test at intake: **name the failure modes that could redden this threshold, then strike out every one the change cannot reach.** If any remain, the threshold is measuring someone else's bug. Two consequences worth stating in the bolt file:

- Prefer a threshold on the **specific signal** the change governs over a threshold on a **composite gate result** (e.g. "the whole suite is green"), which aggregates every unrelated defect in the system.
- Where the composite really is the goal, the prerequisite fixes belong in the plan — the same rule this skill already applies to end-to-end thresholds.

Declaring a defect out of scope in the unit's Scope section does **not** protect the threshold from it; scope and threshold must agree, and only the threshold closes the bolt. And if a threshold must be narrowed after measurement, record the original wording, why it failed, and who decided — a threshold quietly rewritten to match its result is not a threshold.

**The struck-out list includes the INSTRUMENT.** After naming the product's failure modes, ask "what would make this threshold fail — or pass — even if the code were perfect?" and answer it about the instrument: the test runner and its timeouts, the machine, **the environment the instrument inherits from whatever runs it**, and **whether the instrument can see itself**. A test run by a script that has loaded real credentials inherits them unless the script clears them first; a process watch that searches for a string its own command line contains will find itself every time. Before believing an instrument's zero, show it a known positive. *(makerclub, 2026-09-23: a pre-deploy script loaded the live estate and then ran the gate, whose test inherited the real secrets and failed on perfect code, 37/41; a `ps` watch reported 266 "leaks" that were two per sample — itself. Both after reading a rule that named the runner and the machine, but not these.)*

**A carried-forward item inherits the PREMISE of the decision that carried it — re-test that premise at intake.** NFR work often arrives as "the thing we noted last time": a memo's carried-forward list, a retro's fast-follow, an ADR's "still outstanding" paragraph. Each item's evidence was gathered in a *state*, and the decision that carried it forward may have changed that state. So for every carried-forward item, name the observation that motivates it and ask **which configuration produced that observation, and does it still hold?** Worked example: a hosting decision carried two items forward. It noted that the decision removed the first item's payoff, then listed the second as "complementary" without applying the same test. The second item's only evidence came from a configuration that same decision had just eliminated. It was elaborated, risk-assessed and signed off before the Step 3 measurement refuted it, because those ceremonies ask how to do the work well and none asks whether the premise is real.

---

## Step 2 — Identify Affected Intents

Read `process-onboarding-agent/ops/inception/dependency-map.md` and the backlog. Identify which existing intents or units touch the component being improved.

State the affected intents to the engineer:

> "This NFR bolt affects the following existing intents: [list]. I will cross-reference this bolt in each intent's Implementation Summary after completion."

If the NFR improvement changes behavior visible to users (e.g. a new error retry message, a changed loading state): flag those intents' ACs for review after the bolt is done.

---

## Step 3 — Establish the Baseline

Before generating any unit, confirm the current measured value:

> "What is the current measured value for [the threshold from Step 1]? This becomes the before-state for verification."

If no measurement exists: the first unit in this bolt must establish the measurement tooling or instrumentation before any improvement work begins. Record the baseline once measured.

**Measure the baseline on the instrument that is slow, and record where the time goes, not only how much.** A stand-in's profile predicts; the production instrument's own record measures. Before the first improvement unit, read that record (a build log, a per-stage timer, a request trace); if none exists, the measurement unit adds one that splits the time by stage. **Set the target after the split shows the bottleneck** — a target derived from one stage's arithmetic assumes that stage is the bottleneck. (makerclub, 2026-09-27: a laptop profile predicted a 10–12 s build on the production builder, which measured 19.6 s; the builder's own log and one dry run on it showed a configuration step re-running on every build, which the laptop never ran. In the same bolt a download target set from the network window's ceiling was missed by a flash-write stage the window did not govern.)

**For a security finding about requests, the instrument includes everything in front of the process.** An in-process test shows what the framework does with a hostile request; in production that request first passes an ingress, proxy, CDN or load balancer, any of which can rewrite it — removing an exposure or adding one. Alongside the in-process test, send the same hostile requests **raw to the live URL** (e.g. `curl --path-as-is`, confirming the raw path in `curl -v`) and record both sets of answers. Where they differ, explain the difference before believing either. (makerclub, 2026-09-30: a path-traversal advisory was assessed in-process as not exposed; the first live probe showed the ingress canonicalising `..`, `//`, `%2E` and `%2F` before forwarding — a layer the assessment could not see. One that decoded `%2F` into a separator would have been just as invisible.)

---

## Step 4 — Create the Units

NFR bolts may contain multiple units if the improvement spans more than one component. **If Step 3 found no existing measurement, the first unit is the measurement tooling and every improvement unit follows it — that ordering is not a preference.** A single unit that both measures and improves yields one number with nothing to compare it to, and its green threshold gets credited to the change rather than tested against a before-state. Apply the standard unit template with these constraints:

**ACs must use measurable thresholds:**
```
Given [the component under load / in its operational context]
When [the NFR scenario — e.g. 100 concurrent users, a screen reader navigating checkout]
Then [the measurable threshold is met — e.g. p95 response time < 300 ms with no errors]
```

Vague ACs such as "the page loads faster" are not accepted.

**Observability is mandatory on every NFR unit:**
- Metric or signal that confirms the threshold is met in production
- Alert threshold that would indicate regression back below the target

**No feature scope creep:** NFR units must not change business logic. If a performance fix requires a data model change that alters behavior, that is a new feature intent — not part of this NFR bolt. Flag it and defer.

---

## Step 5 — Risk Assessment

Run `process-onboarding-agent/skills/bolt-risk-assessment.md` before any unit executes.

NFR bolts often have wider blast radii than they appear: performance changes can alter caching behavior, security hardening can break existing integrations, accessibility changes can shift layout for sighted users. Do not skip this step.

---

## Step 6 — Execute

Execute units in the planned order. Review each unit's output using `process-onboarding-agent/skills/review-checklist.md` before proceeding to the next.

---

## Step 7 — Verify the Threshold

After all units are Done, verify against the baseline established in Step 3:

> "Please measure [the metric from Step 1] and compare it against the baseline: [value from Step 3]."

The bolt is not Done until the threshold is verified. If the threshold is not met: set bolt status to Blocked, state the gap (measured vs. target), and propose the next step. Do not silently mark it Done.

**A measured round changes ONE variable, and its decision rule is written before it runs.** Ask the engineer for the rule — e.g. _better than X: keep; no better: close on X; worse: roll back_ — and put it in the unit file with the round's settings. A round that changes two settings needs a third round to attribute its result. (makerclub, 2026-09-27: a round that raised two network windows together regressed 5.5 s → 17.3 s, and a third round was needed to learn that neither helped; that third round, with its rule agreed first, closed on its result in one message.)

**A result AT the threshold is a repeat, not a pass.** When a measurement lands within the instrument's resolution of its threshold, record it as neither met nor missed: write the repeat's rule first (e.g. _clearly under: met; clearly over: Blocked with the gap stated; at the bound again: back to the engineer_), run it once with **nothing changed**, and record both numbers. Reading it as met credits the change with a number the instrument cannot tell from a miss; reading it as missed sends the bolt after a fix it may not need. (makerclub, 2026-09-28: a jitter threshold of ≤ 1 printed `1.00` to two places — anywhere from 0.995 to 1.005; one unchanged repeat under a rule written first read 0.84.)

---

## Step 8 — Update Affected Intent Records

For each intent identified in Step 2:
- Add a note in its Implementation Summary referencing this NFR bolt and the improvement delivered
- If any of the intent's ACs are now outdated: flag them for revision in the next elaboration session for that intent

---

## Step 9 — Close

1. Mark all units Done. Mark bolt Done. Update the backlog.
2. Record the before/after measurement explicitly in the retro file.
3. Create the retro and run the Post-Retro Improvement Workflow.
