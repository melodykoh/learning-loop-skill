# PHASE_1_DECISION_REPORT_PROMPT (added v3.5 Apr 28 2026)

> Used by: Wrap-up Step 1b.5 — Phase 1 persona-panel evaluation gate, when the trigger condition is met.
> Extracted verbatim from SKILL.md v4.1 (2026-07-07 restructure) — content unchanged.
> Everything below this line is the sub-agent prompt.

---

You are evaluating Phase 1 shadow-mode performance of the persona panel and producing a GO/HOLD/ITERATE/REVERT recommendation.

INPUTS (will be passed to you):
- All persona-eval.md files at ~/.claude/learning-captures/*/persona-eval.md (the per-wrap-up eval data)
- The persona-eval-runs.txt log at ~/.claude/learning-captures/persona-eval-runs.txt (one-line entries per shadow run)

YOUR JOB:

1. AGGREGATE METRICS across all shadow runs in the log:
   - Match rate = (challenges that match user's actual Step 4 corrections) / (challenges issued)
     - "Match" includes full_match (persona named exact issue) AND partial_match (persona challenged the right conclusion with different specifics).
   - Coverage rate = (user corrections caught by some persona) / (total user corrections in Step 4)
     - Counts no_challenge_user_corrected toward the denominator (user corrected but personas missed).
   - Noise rate = (challenges user did NOT act on, classified as mismatch where user kept consolidation's framing) / (challenges issued)

2. AGGREGATE FAILURE-MODE DISTRIBUTION from the failure_category_observed values:
   - Category (a) Wrong destination
   - Category (c) Missed dedup/umbrella
   - Category (d) Scope too narrow
   - Category (g) Performative / agreed-without-thinking
   - Other (premature conclusion, etc.)

3. APPLY DECISION LOGIC:

| Outcome | Action |
|---|---|
| Match ≥60% AND Coverage ≥60% AND Noise ≤30% | GO to Phase 2 |
| 1 of 3 metrics borderline (within 10% of threshold) | HOLD — run 3 more shadow wrap-ups |
| 2 of 3 below threshold but failure mode consistent | ITERATE prompts |
| Match rate <40% across the eval window | REVERT — rip out Step 3a |

4. PHASE 2 PERSONA SELECTION (only if GO, per D4):
   - If category (c) ≥20% incidence → recommend ship Persona 3 (Dedup/Umbrella Checker) in shadow at Phase 2 promotion time
   - If category (g) ≥20% incidence → recommend ship Persona 4 (Evidence Sufficiency Challenger) in shadow at Phase 2 promotion time
   - Both above → ship both
   - Neither above → Phase 2 = promotion-only (Personas 1+2 to gatekeeper, no new personas)

OUTPUT FORMAT (markdown report):

```markdown
# Phase 1 Persona Panel Decision Report

**Eval window:** <first_run_date> to <latest_run_date> (<N> shadow runs)
**Generated:** <YYYY-MM-DD HH:MM ET>

## Metrics

| Metric | Value | Target | Status |
|---|---|---|---|
| Match rate | X% | ≥60% | ✅ / ⚠️ / ❌ |
| Coverage rate | X% | ≥60% | ✅ / ⚠️ / ❌ |
| Noise rate | X% | ≤30% | ✅ / ⚠️ / ❌ |

Detailed calc: match=<num>/<denom>, coverage=<num>/<denom>, noise=<num>/<denom>

## Failure-Mode Distribution

| Category | Count | Share |
|---|---|---|
| (a) Wrong destination | N | X% |
| (c) Missed dedup/umbrella | N | X% |
| (d) Scope too narrow | N | X% |
| (g) Performative agreement | N | X% |
| Other | N | X% |

## Decision: <GO | HOLD | ITERATE | REVERT>

**Reasoning:** <2-3 sentences explaining which thresholds were met/missed and what pattern emerged>

## Phase 2 Persona Selection (if GO)

- Persona 3 (Dedup/Umbrella): SHIP in shadow / HOLD — based on category (c) at X% (threshold ≥20%)
- Persona 4 (Evidence Sufficiency): SHIP in shadow / HOLD — based on category (g) at X% (threshold ≥20%)

## Recommended Next Action

<Specific next step the user should take. For HOLD: "run 3 more shadow wrap-ups, re-evaluate." For ITERATE: "review TRIGGER_MOMENT_AUDITOR_PROMPT or WORKFLOW_STEP_ROUTER_PROMPT against the X% mismatched cases." For REVERT: "remove Step 3a from SKILL.md, rerun historian on a richer dataset before redesigning." For GO: "promote Personas 1+2 to gatekeeper mode, ship Phase 2 part B per persona selection above.">
```

**Note on the retired Phase 2 gatekeeper (2026-05-20):** shadow mode is the permanent active state; treat any `GO` recommendation as historical decision-logic context (see SKILL.md Step 3a phase-mode resolution).
