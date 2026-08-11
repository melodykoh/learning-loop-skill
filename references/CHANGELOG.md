# learning-loop Version History

> ⚠️ **SUPERSEDED — do not add entries here. The live version history is `README.md` § Version History.**
> This file stopped at v4.3 while releases continued: v4.4, v4.5 and v4.6 each updated README's table and never touched this file. Kept for the pre-v4.4 reasoning trail only. *(Discovered 2026-08-11, when a plan authored against SKILL.md's title line proposed shipping "v4.4" — a number already taken two releases earlier.)*

> Extracted from SKILL.md (2026-07-07 v4.2 restructure). Newest first. Full reasoning trail per change: SESSION_LOG.md.

## v4.3 (2026-07-26 — Step 1b.5 program-concluded exit branch)

- **Behaviour change (one branch).** Step 1b.5 gains a 6th branch: **PROGRAM-CONCLUDED EXIT**, checked *before* the trigger condition. If the latest `phase-1-decision-log.md` entry records the tracked hypothesis/program as closed, retired, or permanently resolved, the trigger is suppressed with a one-line log and the Decision Report is not dispatched.
- **Why:** the trigger is `count ≥ 3` **OR** `≥7 days since ship`, **AND** the latest decision is not <24h old. Once a program concludes, all three conditions are **permanently true and can never again be false.** The 2026-05-20 entry retired Phase 2 ("shadow mode is permanent"), yet 86 logged runs and 89 days later the step still nominally demanded a Decision Report for a decision closed 67 days prior. With no exit condition, the burden of *not* firing fell on each session's private judgment — silently, every wrap-up.
- **How it surfaced:** a wrap-up session evaluated the step, judged the fire pointless, skipped it by judgment, and **announced the skip.** The announcement is the only reason the gap became visible; a silent skip would have kept it invisible indefinitely.
- **Companion watch-list entry `W7.ag`** tracks destination-back-reasoning at mandated steps *outside* Step 3/3a, where the existing scoped STOP does not reach — filed fresh per that graduation's own "if it appears outside Step 3/3a, file fresh" instruction rather than folded.
- **Generic shape, flagged for other skills:** a count/days-since trigger with no already-resolved exit is a skill-authoring gap that can recur anywhere. Worth a cross-skill grep when convenient.

## v4.2 (2026-07-07 — progressive-disclosure restructure)

- **No behavior changes.** Structural: the six sub-agent prompt blocks (SCANNER, CONSOLIDATION, TRIGGER_MOMENT_AUDITOR, WORKFLOW_STEP_ROUTER, PLAN_DRAFTER, PHASE_1_DECISION_REPORT) moved verbatim to `references/prompts/*.md`, read at dispatch time; version history moved to this file. SKILL.md shrank ~2,300 → ~1,250 lines, so every invocation loads the workflow spine + discipline gates without ~1,000 lines of dispatch-time-only content.
- Staleness fix: `/ce:compound` → `/ce-compound` throughout (CE plugin renamed its commands; old form no longer resolves).
- All STOP gates, rationalization counters, zones, and routing tables remain inline in SKILL.md — they are the discipline layer and must load on every invocation.

## v4.1 (2026-06-12 — W7.w Step-3/3a Sub-Agent STOP gate)

- **Step 3 Sub-Agent Gate (W7.w):** discrete STOP at the consolidation entry — closes the gap where end-of-session momentum rationalized consolidating *in the main conversation* and skipping the sub-agent + persona panel. Counters the 4 rationalizations surfaced in RED-GREEN testing (transcript-not-visible → write a richer scan file; small-consolidation; Mod-6-weaponization; late/tired-and-user-waiting) plus the destination-back-reasoning root cause. The pre-existing Sub-Agent Rule (top of skill) only STOPped for Scan mode; this covers the consolidation entry where both W7.w instances happened.
- **Step 3a persona-skip STOP banner:** placed BEFORE the "non-blocking / informational" framing so it's read first — closes the partial-skip loophole ("personas are shadow-mode/non-blocking, so skipping changes nothing / it's just ceremony"). Clarifies that "non-blocking" describes the personas' *output*, never whether you run them; only legitimate skips remain `REVERT` and 0 conclusions; logging a skip does not legitimize it.
- Provenance: watch-list W7.w (2 instances, threshold met 2026-06-10) → graduated G21. Built test-first per `/writing-skills` Iron Law (baseline subagent reproduced the skip; gate added; re-tested to compliance; 3 REFACTOR iterations to close the partial-skip loophole). No new bootstrap/state files added (Skill Version Ship Verification: nothing to bootstrap).

## v4.0 (2026-05-20 hygiene pass)

- Mod 6: Watch-vs-Codify Decision Criterion (codify-now overrides recurrence threshold when mechanism + destination + ≥1 incident all named)
- Mod 7: Granularity Ceiling Rule (≥3 sub-entries sharing mechanism + fix → collapse; tabular incident format; mechanism-first naming)
- Mod 8: Stalled-Deliverable Separation (plan-stale ≠ learning-loop-stale)
- Mod 9: Graduation Ledger Requirement (`graduation-log.md` is mandatory counterpart to `watch-list.md`)
- Mod 10: Path-Drift Detection at Cluster Audit (reference paths must resolve)
- Phase 2 gatekeeper retired — shadow mode is the permanent active state (see `phase-1-decision-log.md` 2026-05-20 entry)

## v3.13 (May 19, 2026)

| Enhancement | Why It Matters |
|-------------|----------------|
| **STOP — Concrete-Anchor Rule for Zone 2 / Zone 3 1-line summaries (Step 4 Verification Detail Floor)** | v3.8 specified "1-line summary + destination" for Zone 2/3 but didn't constrain what the 1-line summary leads with. In the `2026-05-19-session-a` wrap-up, Z2/Z3 rows rendered as cluster-ID-first jargon (e.g., "W1.p new sub-entry `W1.p.aa` — methodology-doc caveat-writing surface") that the user could not verify without re-loading destination context. User pushback: *"There is no description for these Zone 2 and Zone 3 things. I cannot recall or understand what they're about."* The new STOP requires the 1-line to LEAD with a concrete incident, name, verbatim quote, or specific framing from THIS session; cluster IDs and destination paths go at the END as routing metadata. Same "shouldn't need to remember to verify" floor as v3.7, applied at the 1-line layer. Trigger heuristic lets the agent recognize the failing format BEFORE rendering: row contains cluster ID / plan path BUT no specific name / quote / incident phrase from this session → jargon-only → re-render with concrete anchor or auto-expand to full floor. Good/bad examples included so the contrast is unambiguous. |

## v3.12 (May 14, 2026)

| Enhancement | Why It Matters |
|-------------|----------------|
| **STOP — Test-Case Value Check (CONSOLIDATION_PROMPT step 2)** | Tightening complement to the May 7 Resolution-vs-Increment Check. The May 7 rule fires when THIS session's conclusion creates the resolution; the new rule fires when the resolution is ALREADY in place from prior work and the incident doesn't add a new test-case dimension. Discriminator: **"would the fix author learn anything new about how to design the fix from this sub-entry?"** If no → drop. Sub-entries are valuable as test cases (per Mod 3 schema rationale), not as incident counters. Provenance: May 14 wrap-up — proposed W1.r-pre-send increment for Stop-hook re-trip on completion-claim-without-verification. User: *"If the hook fires successfully, why are we incrementing the watchlist? Meaning, what would be the new fix if the watchlist count exceeds the threshold? If there is no new fix, why are we incrementing?"* Stop hook is the structural fix; firing; no new test-case dimension to the in-flight W4 retrofit plan. Same-session result: C1+5 (W1.s) KEPT because new register-translation surface adds a real test-case dimension; C4 (W1.r-pre-send) DROPPED because the Stop hook is firing and the incident is the same surface as prior 4 instances. |

## v3.5 (Apr 28, 2026)

| Enhancement | Why It Matters |
|-------------|----------------|
| **Step 3a — Persona Panel (Phase 1, shadow mode)** | Step 3 (consolidation) reliably extracts the right *facts* from raw signals but produces the *wrong rule architecture* in routing proposals at a high rate. Across 4 mined sessions (Apr 2 – Apr 18 2026), 12 correction rounds were observed; 75% changed trigger-framing and/or destination, not facts. Step 3a runs two adversarial sub-agent personas (Trigger-Moment Auditor + Workflow-Step Router) BEFORE Step 4 surfaces proposals to the user. Phase 1 is shadow mode — personas REPORT but DO NOT BLOCK. User reads both views in Step 4 verification, decides per-row. |
| **Step 1b.5 — Phase 1 Persona Panel Evaluation Check** | Self-evaluating decision gate. Fires automatically inside `/learning-loop wrap-up` after ≥3 prior shadow runs OR ≥7 days post-ship, whichever first. Spawns a Phase 1 Decision Report sub-agent that aggregates match/coverage/noise metrics across shadow runs and recommends GO / HOLD / ITERATE / REVERT. Decision applies to NEXT wrap-up's Step 3a behavior, not the current one (per D2 — eval decisions deserve thought, not in-the-moment pressure). Same enforcement principle as Step 1b: bind eval to a workflow step that fires every wrap-up so it cannot rot. |
| **Step 4c — Capture Phase 1 Eval Data** | After user verification in Step 4, classify each persona challenge against the user's actual decision (full_match / partial_match / mismatch / no_challenge_user_corrected). Write `persona-eval.md` per session + append entry to `persona-eval-runs.txt`. Bootstrap `phase-1-ship-date.txt` on first run. This is the dataset Step 1b.5's evaluation gate operates on. |
| **Three new prompt blocks alongside CONSOLIDATION_PROMPT** | `TRIGGER_MOMENT_AUDITOR_PROMPT` (audits symptom-vs-mechanism rule framing for each conclusion), `WORKFLOW_STEP_ROUTER_PROMPT` (audits decision-changer-vs-recall-fact destination choices, receives Auditor's named triggers as input), `PHASE_1_DECISION_REPORT_PROMPT` (aggregates shadow-run metrics and outputs the GO/HOLD/ITERATE/REVERT recommendation). |
| **Storage and presentation separated (per D3)** | Personas write JSON to `persona-review.json` for clean machine-parsing; Step 4 renders into a markdown verification table for user readability. |

**Provenance for v3.5:** Apr 24 2026 — kids-activities `/learning-loop wrap-up` produced 3 sequential narrowings on C3 (Proactive-Offer Filter → Present Options Before Building → Reason Upstream Before Acting). User raised the structural concern: *"a lot of times learning loop is reasonable at summarizing the actual facts that happened in session, but comes to the wrong conclusion and suggests the routing and documentation."* Apr 24-27 — `ce-session-historian` mined 4 prior wrap-up sessions for failure-pattern dataset (12 correction rounds), confirmed dominant failure mix is trigger-framing (33%) + destination (33%) + their compound (25%). Apr 27 — plan hardened with 6 Resolved Decisions (D1-D6). Apr 28 — Phase 1 ship.

**Phase 2 trigger:** Step 1b.5 fires automatically. If GO, promote Personas 1+2 to gatekeeper mode + ship Persona 3 and/or 4 in shadow per failure-mode distribution at ≥20% incidence (per D4). If HOLD, run 3 more shadow wrap-ups. If ITERATE, refine prompts. If REVERT, rip out Step 3a. *(Superseded 2026-05-20: Phase 2 gatekeeper retired; shadow mode is the permanent active state.)*

**Plan reference:** `~/Documents/claude-projects/claude-skills/plans/2026-04-24-learning-loop-persona-panel.md` — full plan, decision history, evidence base, and Appendix A historian output.

**D5 deviation note:** Plan originally specified v3.4 for Phase 1 ship. Watch-list mods (Mods 1-5) shipped earlier the same day as v3.4, so Phase 1 ships as v3.5. Plan's D5 update logic remains correct (additive minor bump per Phase).

## v3.4 (Apr 28, 2026)

| Enhancement | Why It Matters |
|-------------|----------------|
| **Mod 1 — Root-cause matching, not observation matching (Step 4 watch-list)** | Watch-list sprawled from 1 → 30 entries in 15 days because the prior matching criterion was surface-text-similarity on observations. The user's actual criterion is "is the FIX the same?" — two superficially different observations sharing a cognitive/process origin and remediation path are the SAME watch-list item. New rule forces the sub-agent to articulate root cause + fix BEFORE deciding fold-vs-new, with explicit bias toward folding. |
| **Mod 2 — Watch-list matching invoked inside CONSOLIDATION_PROMPT** | Previously, the sub-agent that did all the rich cognitive work (classify, gates, significance, destination) never read watch-list.md or proposed increments. Match happened post-hoc in main-session against single-line descriptions. New rule: the sub-agent reads watch-list end-to-end, articulates cognitive origin + process origin + proposed fix for each conclusion, then matches against existing entries with explicit increment-vs-new justifications. |
| **Mod 3 — Watch-list schema upgrade (Root cause + Fix + Incidents columns)** | Old schema (`ID \| Observation \| Count \| Threshold \| Escalate to`) made fix-matching a NLP problem on 200-word prose. New schema makes "same fix?" a deterministic check. **Sub-IDs (W_N.a, W_N.b, …) preserve incident-level traceability inside clusters** so the eventual fix author can trace through every test case when crafting the remediation. |
| **Mod 4 — Cluster audit step (Step 4b) at every wrap-up** | Bounded-cost sprawl detector: skip entirely if active entries ≤15 AND no fix-cluster ≥3 (tighter than original 20/5 thresholds). When triggered, surfaces sprawl alert + offers re-consolidation sub-agent. Catches sprawl while it's small instead of waiting for manual user invocation at 30+ entries. |
| **Mod 5 — Threshold-met → auto-draft plan in plan-execution-pipeline schema** | The biggest structural change. Previously, threshold-met meant "route to a destination location" — destination was a place, not a plan. The W4 retrofit plan was hand-drafted weeks after sprawl was visible. Going forward: threshold = automatic plan generation, conforming to `~/Documents/claude-projects/Personal/plan-execution-pipeline/schema/plan-schema.md`. **Every historical incident becomes a Success Criteria checkbox** so the fix author receives the full test-case set, not a vague "fix the recurring drift." This makes the fix tractable and the robustness verifiable. |

**Provenance for v3.4:** Apr 27-28 2026 wrap-up session. The Apr 27 consolidation surfaced that 4 of the 30 active watch-list entries shared the same fix (continuous-rule drift via W4 retrofit) and another 3 shared a different fix (stress-test designs before proposing). User pushback: "There is no point incrementing on very specific downstream scenarios, because the solution is not to fix those symptoms, it's to fix the root cause... If the fix is the same, then they should increment the same watch list item as opposed to sprawling to a bunch of different items." Two parallel sub-agents diagnosed the rule (matching criterion was wrong, CONSOLIDATION_PROMPT didn't invoke matching at all, schema lacked structural anchors for fix-comparison). Mods 1-5 are the structural fix.

**Migration note:** The first wrap-up under v3.4 saw the watch-list re-consolidated from 30 → 5 active clusters + 15 standalone, with new schema applied retroactively. After migration, the sub-agent's increment-or-new decisions are governed by the v3.4 root-cause matching rule.

## v3.3

| Enhancement | Why It Matters |
|-------------|----------------|
| **Significance threshold (Gate 5)** | Gates 1-4 are pass/fail on quality. Gate 5 asks "would a future session go WRONG without this?" — separating interesting observations from consequential learnings. Prevents over-documentation of signals that pass quality gates but aren't worth persisting. |
| **"Noted" routing option** | Explicit acknowledgment for signals that pass quality gates but fall below the persistence threshold. Shown in wrap-up summary but not routed anywhere. Prevents the false binary of "document everything" vs. "lose it." |
| **Behavioral vs. operational split** | Process-level learnings now split into behavioral (changes decisions → CLAUDE.md) vs. operational (changes procedures → project operational docs). Prevents CLAUDE.md from accumulating scheduling heuristics and workflow sequences that belong in playbooks or operational docs. |
| **Repo-adaptive operational routing** | Operational learnings route to playbooks/ if the project has them, otherwise to CLAUDE.md or Memory. The skill is global but adapts to each project's documentation infrastructure instead of assuming playbooks exist. |

## v3.2

| Enhancement | Why It Matters |
|-------------|----------------|
| **Content wedge filter** | Judgment Ledger entries must now pass the content wedge test ("where AI capability meets reality"). Prevents accumulation of operationally useful but non-publishable entries. Borderline cases tagged `⚠️ wedge-check` for user decision. |
| **Content-level quality gate** | Added wedge fit checkbox to content-level quality gates in consolidation prompt. Entries that fail get reclassified as process-level. |

## v3.1

| Enhancement | Why It Matters |
|-------------|----------------|
| **Session-scoped wrap-up** | v3 consolidated ALL accumulated captures regardless of topic. v3.1 defaults to current session only, surfaces other sessions for triage. Prevents cross-polluting unrelated sessions. |
| **Orphan session surfacing** | Capture directories from sessions that closed without wrap-up are shown during triage — user decides to include, skip, or delete. No more silent accumulation. |
| **Sharper content-level routing** | v3 routed learnings *about content work* (editorial rules, scheduling) to Judgment Ledger. v3.1 distinguishes "worldview shifted" (→ Judgment Ledger) from "learned a better way to do content work" (→ Project CLAUDE.md). |

## v3.0

| Enhancement | Why It Matters |
|-------------|----------------|
| **Explicit `/learning-loop` invocation** | v2's description-based matching was non-deterministic — capture phrases matched intermittently, but "wrap up" never triggered reliably. Explicit invocation is deterministic. |
| **Two-mode model (Scan / Wrap-up)** | Scans capture raw signals without judgment; wrap-up resolves hypotheses with hindsight |
| **Smart mode detection** | Minimal friction — context clues route to the right mode, explicit override always available |
| **Memory as routing destination** | Facts (no behavior change) route to MEMORY.md instead of being lost or forced into CLAUDE.md |
| **Auto-memory coexistence** | Complementary design — auto-memory handles quick facts, learning-loop handles structured analysis |
| **User stories documented** | Distinct use cases ("mid-task, save signals" vs "done, consolidate everything") now explicit — prevents future designs from collapsing them |

## Previous Versions

**v2.1:** Real-time micro-logging (Phase 1 scratch files), project-level CLAUDE.md routing
**v2:** Type-specific quality gates, orchestration model, user-initiated triggers
**v1:** Proactive monitoring (failed — Claude can't sense context % in Claude Code)
