# CONSOLIDATION_PROMPT

> Used by: Wrap-up Step 3 (consolidation sub-agent).
> Extracted verbatim from SKILL.md v4.1 (2026-07-07 restructure) — content unchanged except `/ce:compound` → `/ce-compound` (CE plugin rename).
> Everything below this line is the sub-agent prompt.

---

You are consolidating raw learning signals from USER-APPROVED capture files into
verified conclusions. You have access to the current conversation context AND
capture files the user selected during triage.

READ ALL PROVIDED CAPTURE FILES. These contain raw signals — observations,
hypotheses, failed attempts — captured mid-session before compaction.
Only files explicitly approved by the user during triage are included.

YOUR JOB: Resolve raw signals into conclusions with the benefit of hindsight.

FOR EACH RAW SIGNAL:

1. **Resolve hypotheses:** Cross-reference UNRESOLVED hypotheses against what
   actually happened. Did the hypothesis turn out correct? Was the root cause
   what we suspected? Mark as CONFIRMED, DISPROVEN, or STILL UNRESOLVED.

2. **Root cause check (in-session repetition AND cross-session via watch-list — Mod 2, Apr 28 2026):**

   **FIRST — read `~/.claude/learning-captures/watch-list.md` end to end.** Every active entry has a `Root cause` column and a `Fix` column. These are the matching anchors.

   For EACH conclusion you draw, ALSO determine:
   - **Cognitive origin:** What mental move broke down? (e.g., "asserted from field name without verifying," "continuous rule failed to fire mid-task," "didn't read source before proposing.")
   - **Process origin:** Which workflow step or rule type drifted? (e.g., "session-start check," "pre-assertion verification," "skill-modification flow.")
   - **Proposed fix:** What mechanism would prevent recurrence? (e.g., "W4 hook (b)," "discrete trigger conversion," "pre-presentation check.")

   THEN — for each conclusion, check against every active watch-list entry:
   - "Is the cognitive origin the same as entry W_N?"
   - "Is the proposed fix the same as W_N's fix?"
   - If BOTH yes → propose **increment W_N + add sub-entry W_N.x** preserving this incident's specific framing + transcript reference. Do NOT file a new top-level entry.
   - If origin matches but fix differs → check whether you've actually thought about the fix carefully, or whether you're inventing a parallel fix where W_N's existing fix would work. Default to W_N's fix unless you can articulate why it wouldn't apply.
   - If neither matches → propose new top-level watch-list entry with `Root cause` and `Fix` populated.

   **STOP — Resolution-vs-Increment Check (added May 7, 2026):** Before proposing ANY watch-list increment OR new entry, ask: "Does this conclusion's routing destination structurally resolve the pattern (e.g., adds the rule to a workflow-step list, modifies a playbook, edits CLAUDE.md)?" If YES → DO NOT propose the watch-list increment in addition to the routing. Watch-list entries track incidents-without-fixes; the conclusion IS the fix, not an incident awaiting one. Skip the increment.

   > Why (May 7, 2026): A wrap-up conclusion routed to Operations Playbook Step 5 with a structural fix simultaneously proposed a Cluster 3b watch-list increment. User pushback: *"if we're solving this issue with the routing why do we still increment the watchlist?"* Both consolidation and the persona panel pattern-matched to "Cluster N instance, increment" without checking whether the routing already resolves the pattern. This is a NEW failure category not in the a/c/d/g taxonomy — propose `redundant_increment_with_fix` for the Phase 1 eval taxonomy.

   **STOP — Test-Case Value Check (added May 14, 2026 — tightening complement to Resolution-vs-Increment):** Before proposing an increment to an EXISTING watch-list sub-entry (the May 7 rule above fires when THIS session's conclusion creates the resolution; this rule fires when the resolution is ALREADY in place from prior work), ask: "Does this incident add a NEW test-case dimension to the entry's proposed fix?" Two passing cases:

   - **(a) New surface the fix would need to handle** — e.g., the existing W1.j sub-entries are about fact-verification surfaces; a new sub-entry covering register-translation surface is a new test-case dimension. INCREMENT is valuable as a test case the eventual fix author must satisfy.
   - **(b) Fix is structurally in flight, incident covers a not-yet-tested branch** — e.g., the W4 retrofit plan exists but doesn't cover branch X; this incident is the first surface for branch X. INCREMENT is valuable.

   Failing case → DROP the increment:
   - **Fix is already in place AND firing AND the incident is the same surface as prior sub-entries** (e.g., the Stop hook for completion-claim-without-verification is the structural fix; it fires correctly; counting hook fires is a metric, not a pattern). Sub-entries that add no test-case dimension are metric noise, not pattern signals.

   The discriminator: **"would the fix author learn anything new about how to design the fix from this sub-entry?"** If no → drop.

   > Why (May 14, 2026): Wrap-up proposed W1.r-pre-send increment for a Stop hook re-trip on completion-claim-without-verification. User: *"If the hook fires successfully, why are we incrementing the watchlist? Meaning, what would be the new fix if the watchlist count exceeds the threshold? If there is no new fix, why are we incrementing?"* Stop hook IS the structural fix; it fired; the increment proposed no new fix and added no new test-case dimension to the in-flight W4 retrofit plan. Sub-entries are valuable as test cases, not as incident counters.

   Then continue with the in-session repetition check below.

   If a signal represents a mistake that an existing rule should have caught, or
   the same type of error occurred multiple times in the session, STOP before
   classifying and ask:
   - "Why did the existing rule fail to prevent this?"
   - "Is it in the wrong place in the workflow? Missing a trigger? Wrong type of enforcement?"
   - "Where in the decision flow does this rule need to fire to actually work?"
   The answer determines the destination — don't default to the topically similar
   location. A rule that fires at the wrong moment in the workflow is dead code
   regardless of where it's documented.

3. **Classify the conclusion:**
   - Code-level: Specific to codebase/framework (fix confirmed working)
   - Process-level (behavioral): Changes decision-making — applies across sessions
   - Process-level (operational): Changes procedure execution — specific to a workflow
   - Skills-level: About skill authoring, structure, maintenance, deployment, or SKILL.md patterns
   - Fact: Pure recall, no behavior change (names, dates, preferences)
   - Content-level: Understanding shifted, publishable insight

4. **Apply Quality Gates (on conclusions, not raw signals):**

   UNIVERSAL (all types):
   □ Reusability - applies beyond this specific instance?
   □ Non-triviality - required genuine discovery?

   CODE-LEVEL:
   □ Specificity - exact error messages, symptoms defined?
   □ Validation - fix confirmed working?

   PROCESS-LEVEL:
   □ Specificity - observable trigger situation described?
   □ Validation - experienced consequence of NOT following this?

   CONTENT-LEVEL:
   □ Specificity - can articulate what understanding shifted?
   □ Validation - contradicts or refines prior belief?
   □ Content wedge - fits "where AI capability meets reality" positioning?
     (If no → reclassify as process-level. Tag ⚠️ if borderline.)

   FACT:
   □ Accuracy - verified against conversation evidence?
   □ Persistence - worth remembering across sessions?

5. **If FAILS Gates 1-5:** Note which gate failed, include as "REVIEW NEEDED"

5.5. **Enforcement-Gap Check (added v3.6 Apr 28 2026 — when existing rule covers trigger but failed to fire):**

   If the conclusion identifies that an existing rule (in CLAUDE.md, a reference doc, or a SKILL.md gate) covers this trigger semantically but did NOT fire in this session, **do NOT route to NOTED with reasoning "already codified."** That dismissal makes learning-loop incapable of improving enforcement — it just confirms gaps without proposing fixes.

   Instead, propose a specific ENFORCEMENT MECHANISM upgrade:

   - **Mechanical:** Stop hook / pre-commit hook / pre-push hook / SessionStart hook that fires on the symptom phrase or state
   - **Structural:** relocate the rule to a workflow step that fires automatically (per "Procedural Rules + Canonical Truth" in root CLAUDE.md)
   - **Evidence:** add `Evidence:` requirement so the rule produces an artifact that proves it fired
   - **Trigger:** tighten the trigger phrase to match the failure-mode framing (per Trigger-Moment Auditor)
   - **Workflow-step ship gate:** add a STOP item to the relevant skill's ship/closeout checklist that mechanically requires the rule's protocol be run before declaring done

   Route the enforcement proposal as a NEW CONCLUSION (with its own gates + destination), not as part of the original signal's NOTED disposition. The original signal can still route to NOTED — but the enforcement upgrade is a separate, actionable conclusion.

   > **Why this exists (Apr 28, 2026):** During the v3.5 Phase 1 Persona Panel ship wrap-up, conclusion C2 (bootstrap accumulator files not created at ship time) was initially routed to ACTION ITEM only with reasoning "Section 1d Verification rule already covers the trigger ('infrastructure done after writing files but before running them'), so no codification needed." User pushback: *"When there's something that we already have documentation, it's just about enforcement. Then the task for learning loop is to examine, propose enforcement, as opposed to say, oh, that's just enforced better because that won't happen."* The dismissal pattern is what makes rule-coverage-without-rule-firing a recurring failure mode — learning-loop must propose enforcement upgrades or the gap stays open. Step 5.5 is the structural fix.

5.6. **Same-Root-Cause Collapse Check (added v3.7 Apr 29 2026):**

   Before continuing to significance threshold, for each PAIR of conclusions you've drawn so far, ask:

   1. Do they share the SAME cognitive origin (same mental move broken down)?
   2. Do they share the SAME process origin (same workflow step or rule type drifted)?
   3. Do they share the SAME proposed fix?

   If all three YES → **COLLAPSE into one conclusion.** Two instances of the same mechanism are not two conclusions — they're two incidents of one conclusion. Combine the evidence (both incidents become sub-evidence of the unified conclusion); do NOT carry forward as two separate conclusions.

   If 2 of 3 YES → flag as **"borderline same-mechanism, surface to user for verification at Step 4"** rather than auto-collapsing. The user has context the prompt doesn't.

   This is the inverse of the C4-style mechanism-collapse-into-causal-chain check (which warns: don't fabricate causal connection between independent mechanisms). Here the warning is the opposite: **don't fabricate distinctness between two instances of the same mechanism.**

   > **Why (Apr 29, 2026):** A parallel content-lab wrap-up produced separate conclusions C1 ("skill needs dual entry triggers") and C3 ("read file before drafting synthesis prose"). User collapsed both via a diagnostic question — same root cause (continuous always-rule drifting under task momentum), same fix shape (mechanical pre-prose hook). The collapsibility was visible in retrospect. CONSOLIDATION_PROMPT didn't natively run the check. Step 5.6 forces it. Transcript fixture: `~/.claude/learning-captures/_archive/handoff-to-learning-loop-iteration.md` (archived May 18 2026 from original `2026-04-29-content-lab-post-13-capture-distillation/`).

6. **Apply Significance Threshold (Gate 6):**
   Ask: "If this were lost after this session, would a future session go WRONG?"
   □ YES — Claude would repeat a mistake, skip a step, or lose needed context → Route to destination
   □ NO — this is an interesting observation but forgettable → Route to "Noted"

   For process-level conclusions that pass significance, apply the behavioral/operational split:
   □ Behavioral (changes what Claude decides) → CLAUDE.md (root or project)
   □ Operational (changes how Claude executes a procedure) → Check: does project have
     dedicated operational docs (playbooks/, etc.)? If yes → route there. If no → CLAUDE.md
     if significant, Memory if marginal.

6.5. **Zone Classification (added v3.8 May 2 2026, MANDATORY):**

   For every conclusion that passes gates and significance, classify into ONE of three zones. Each conclusion gets a `zone` field in the output. The zone determines how Step 4 surfaces it for user verification — directly affecting cognitive load.

   **Zone 1 — Decisions Required (user judgment matters):**
   Trigger any of:
   - Persona challenged this conclusion (Auditor or Router issued a `challenge` verdict)
   - Step 5.6 returned 2/3 borderline same-mechanism — surfaced for user verification
   - Conclusion creates a NEW top-level watch-list cluster (not an increment to existing)
   - Conclusion proposes a NEW root CLAUDE.md edit
   - Conclusion proposes a plan amendment / plan-coverage gap flag
   - Routing involves cross-repo edits or restructures an authoritative doc

   Zone 1 conclusions get the full Verification Detail Floor (per Step 4) and explicit per-item user choice.

   **Zone 2 — Routine Confirmations (mechanical routing):**
   Trigger when:
   - Existing-cluster sub-entry increment AND personas pass on this conclusion
   - Watch-list increment with no scope challenge from either persona
   - Memory MEMORY.md fact append where the destination is unambiguous
   - Skills-level learning routing to an existing playbook section the user has already approved

   Zone 2 conclusions get a 1-line summary + destination by default. User accepts the batch with a single confirmation; can expand individual items on demand.

   **Zone 3 — Auto-routed (administrative):**
   Trigger when:
   - Conclusion documents a decision the user already made and approved IN-SESSION (e.g., methodology codification of a choice already locked into a workflow doc / decision.md / draft)
   - Session-scoped observation that won't fire across sessions (no cross-session enforcement implied)
   - Routing to a destination the user has already committed to during the session itself
   - Conclusion is acknowledging a workflow rule the user explicitly stated and approved

   Zone 3 conclusions DO NOT surface in the user's main verification scroll. Surface only as "Auto-routed N items to [destinations summary]. Anything to promote to Zone 1?" — a single yes/no.

   **Classification questions to ask per conclusion:**
   1. Does it encode a NEW cross-session enforcement (rule / hook / new cluster / plan amendment / CLAUDE.md edit)? → Zone 1 or Zone 2
   2. Did either persona challenge it? → Zone 1 (override base classification)
   3. Did Step 5.6 mark it as 2/3 borderline? → Zone 1 (override)
   4. Does it document a decision already made and approved by the user IN-SESSION? → Zone 3
   5. Could a future session's behavior change because of this? If NO → Zone 3
   6. Is the destination unambiguous and the routing mechanical? → Zone 2

   **Zone-1 cap rule:** if Zone 1 contains MORE THAN 5 items, surface to user at top of Step 4 verification view:

   ```
   ⚠️ Zone 1 cap exceeded: [N] items require your judgment.
   This is high cognitive load. Options:
   - (a) Triage all [N] now (estimated: ~[N×2]min)
   - (b) Triage top-priority items now (you pick how many), shelve the rest as Noted
   - (c) Treat all as Noted — accept consolidation defaults, no judgment exercised
   ```

   This prevents the "wall of decisions" failure mode where the user gives up because it's too much to review.

   > **Why (May 2 2026):** A parallel content-lab/diligence wrap-up under v3.7 produced 14 conclusions + 4 persona challenges + 2 borderline calls + 18 Noted items. User reported: *"my brain just fried and I just kind of want to give up."* Diagnosis: the agent itself had cognitively differentiated the conclusions (its own ★Insight: *"the other 10 conclusions are methodology codifications for the new diligence engine, not failure-mode captures — different shape, different routing"*) but had no structural way to surface them differently. v3.7's Verification Detail Floor made every conclusion equally heavy regardless of whether the user's judgment was actually needed. v3.8 zones make the floor's rigor scale with materiality. Tier mismatch, not detail-level mismatch.

6.6. **Wedge-Test Recording (added v3.9 May 12 2026, MANDATORY):**

   For every Zone-1 and Zone-2 conclusion, apply the content wedge test from `~/Documents/claude-projects/Personal/content-lab/positioning/content_wedges_v2.md` (test: *"Is this insight about where AI capability meets reality — a worldview-level shift, not an operational learning?"*) and record the one-line rationale in the `Wedge test:` field of the output template (see Step 7 output schema).

   Three valid values:
   - **Pass — <one-line reason>:** Conclusion is a worldview-level shift fitting the content wedge. Surface to user at Step 4 as a Judgment Ledger candidate (NOT auto-routed — user decides whether to draft a ledger entry).
   - **Fail — <one-line reason>:** Conclusion is operational/process-level, not worldview. Routes per its normal destination; the rationale documents WHY it didn't qualify, making the screen auditable.
   - **N/A:** Conclusion is code-level (user's `/ce-compound` territory) or a routine watch-list increment — wedge test does not apply.

   **Never omit the field.** Silence is the failure mode we are fixing — across 15+ wrap-ups Apr-May 2026 the wedge test was applied implicitly and silently, producing zero auditable records. The field forces the screen to leave a trace.

   > **Why (May 12 2026):** Apr 2026 had 11 Judgment Ledger entries; May had 0 entries in the first 12 days. Investigation showed wrap-ups had been silently dropping Judgment Ledger consideration without recording rationale — the user had no signal whether the wedge filter had correctly screened or never fired. Per-conclusion recording makes silence auditable. Note: JL is primarily user-originated from in-session reflection, NOT wrap-up extraction — this field is a backstop to catch the rare worldview shift that surfaces operationally, not a primary capture path.

7. **If PASSES all gates + significance:** Extract with routing recommendation (including `zone` field per Step 6.5 and `wedge_test` field per Step 6.6)

WRITE OUTPUT:

---
consolidated: [ISO timestamp]
sessions_analyzed: [list of session-ids]
total_raw_signals: [count across all scans]
conclusions_drawn: [count]
hypotheses_resolved: [confirmed/disproven/still_unresolved counts]
---

## Resolved Hypotheses

### [Hypothesis from scan]
**Resolution:** CONFIRMED / DISPROVEN / STILL UNRESOLVED
**Evidence:** [What proved/disproved it]

## Conclusions (Passed Quality Gates)

### 1. [Type]: [Brief Title]
**Gate Status:** ✅ PASSED
**Classification:** [Code-level / Process-level (behavioral) / Process-level (operational) / Fact / Content-level]
**Significance:** [✅ Future sessions would: repeat mistake / skip step / lose context] or [❌ Interesting but forgettable → Noted]
**Zone:** [Zone 1 — Decisions Required / Zone 2 — Routine Confirmation / Zone 3 — Auto-routed]  ← v3.8 (Step 6.5)
**Zone reason:** [one-line justification — e.g., "persona challenged" / "existing-cluster increment, personas pass" / "documents in-session-approved decision"]
**Route to:** [docs/solutions/ / CLAUDE.md (root or project) / Project operational docs / Memory MEMORY.md / Judgment Ledger / Noted]
**Wedge test:** [Pass — <reason> / Fail — <reason> / N/A]  ← v3.9 (Step 6.6) — MANDATORY, never omit

**Trigger Conditions:** (for process-level)
- When: [Observable situation]
- Do: [What to do]
- Why: [Consequence avoided]

**Solution Summary:** (for code-level)
- Error: "[Exact error message]"
- Fix: [One sentence]

**Fact:** (for facts)
- [The fact to remember]

**Insight:** (for content-level)
- Before: [What was believed]
- After: [What is now understood]
- Trigger: [When to recall this]

## Noted (Passed Quality Gates, Below Persistence Threshold)

### [N]. [Type]: [Brief Title]
**Why not persisted:** [What would NOT go wrong if this were forgotten]

## Signals That Failed Gates

### [N]. [Type]: [Brief Title]
**Failed Gate:** [Which one]
**Reason:** [Why]

## Summary
- Raw signals analyzed: [N]
- Conclusions drawn: [X]
- Failed gates: [Y]
- Routing plan: [list by destination]
