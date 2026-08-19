---
name: learning-loop
description: Two-mode learning system — raw signal scanning before compaction, quality-gated consolidation at session end. Invoke with /learning-loop.
version: 4.7.0
allowed-tools:
  - Agent  # sub-agent dispatch (formerly Task, renamed in harness v2.1.63)
  - Read
  - Write
  - Edit
  - Bash
  - Grep
  - Glob
  - Skill
---

# learning-loop Skill v4.7

> **Version history:** the live table is `README.md` § Version History. (v4.7 = failure-class gate before prose + a fourth state for concurrent ownership; v4.6 = zone criteria re-keyed onto WHO CAN ANSWER; v4.5 = Step 6a cleanup guard hardened; v4.4 = sub-agent briefs stop claiming inherited context; v4.3 = Step 1b.5 program-concluded exit branch; v4.2 = progressive-disclosure restructure, no behavior change.) **`references/CHANGELOG.md` is superseded — it stopped at v4.3 while releases continued in README.**
>
> **Sub-agent prompt library:** the six dispatch prompts live in `references/prompts/` and are read AT DISPATCH TIME — see "Sub-Agent Prompt Library" section below. **STOP if a prompt file is missing: never improvise a replacement prompt from memory** — surface the missing file to the user instead.

**Purpose:** Two-mode learning capture — raw signal scanning mid-session, quality-gated consolidation at session end. Handles process-level and content-level capture; code-level capture is the user's responsibility via direct `/ce-compound` invocation mid-session (peak-fresh context).

> **Fan-out is this skill's documented behavior.** Invoking it constitutes the user's request for that dispatch — do not stop to ask permission per session, and do not silently downgrade to doing it inline (ratified by this skill's owner, 2026-07-26). *(Attribution left un-named: this repo is PUBLIC and names no individual anywhere else in it; the private copies of this convention carry the named ratification.)* **Cap: none documented — state your agent count and why before dispatching.**

---

> **Model choice per dispatch:** choose each sub-agent's model from what *that* sub-agent
> is being asked to do, and **print each choice + a one-line why in the visible reply**.
> Criterion and rationale: root `CLAUDE.md` → **Model-tiering**. Never pin a model in an
> agent file — it is decided at dispatch, where the task is visible.

## Mode Detection

When `/learning-loop` is invoked, determine which mode to run:

| User says | Mode | Why |
|---|---|---|
| `/learning-loop` + "before I clear" / "context long" / "mid-session" | **Scan** | Context clues indicate mid-session |
| `/learning-loop` + "wrap up" / "done" / "end session" / "consolidate" | **Wrap-up** | Context clues indicate session end |
| `/learning-loop scan` | **Scan** | Explicit override |
| `/learning-loop wrap up` | **Wrap-up** | Explicit override |
| `/learning-loop` (no context clues) | **Ask** | "Mid-session scan or session-end wrap-up?" |

**Wrap-up scope:** Wrap-up always consolidates the **current session's captures**. If other session directories exist, they are surfaced for triage (see Wrap-up Step 2) — the user decides whether to include, skip, or delete them.

**Detection logic:**
1. Check if the user passed an explicit argument (`scan` or `wrap up`)
2. If no argument, scan the user's recent messages for context clues:
   - Scan clues: "before I clear", "context getting long", "going to compact", "mid-session", "save progress"
   - Wrap-up clues: "wrap up", "done for now", "end session", "ending", "consolidate", "finally done"
3. If ambiguous or no clues, ask: "Mid-session scan (preserves raw signals) or session-end wrap-up (consolidates everything)?"

---

## ⚠️ MANDATORY PROCEDURES

### Sub-Agent Rule
**When running Scan mode:**
1. **DO NOT** do capture work in main conversation — this wastes context
2. **SPAWN** an Agent sub-agent (formerly the Task tool) using the SCANNER_PROMPT
3. Sub-agent writes to `~/.claude/learning-captures/[session-id]/scan-NNN.md`
4. Main conversation waits for completion, then confirms capture is done

**STOP:** If you're about to scan for signals yourself in the main conversation, you're doing it wrong. Spawn a sub-agent.

### User Verification (Critical)
**Before routing ANY learning to a destination:**
1. **Present a summary** of captured/consolidated signals to the user
2. **Explicitly ask for verification** — "Does this accurately reflect what happened?"
3. **Wait for user confirmation** before routing to CLAUDE.md, Judgment Ledger, or Memory

> **Why this exists (Jan 29, 2026):** AI-generated captures can contain hallucinations — wrong names, fabricated premises, misremembered details. A capture once got the user's husband's name wrong and claimed constraints that didn't exist.

**DO NOT** update any destination based on captures without user sign-off.

### Skill Version Ship Verification (added v3.6 Apr 28 2026)

**When shipping a new version of this skill (or any skill that adds bootstrap state files, accumulator logs, or one-time initialization):**

1. **Enumerate every bootstrap step the new version added** — accumulator file creation, sentinel file writes, decision-log initialization, schema migrations, etc.
2. **Run them, OR confirm they will be triggered by a downstream workflow step that has actually fired**.
3. **Verify each artifact exists on disk** before declaring the version "shipped."

**STOP and correct if you're:**
- About to declare a new skill version "shipped" / "live" / "deployed" without checking that the version's added bootstrap files exist
- Assuming "Step N will create the file when it first runs" without confirming Step N has run at least once after the ship

> **Why (Apr 28, 2026):** v3.5 Phase 1 Persona Panel shipped with three new files supposed to be bootstrapped by Step 4c (`persona-eval-runs.txt`, `phase-1-ship-date.txt`, eventually `phase-1-decision-log.md`). At ship time none existed, because Step 4c only runs DURING a wrap-up — and no wrap-up had run yet under v3.5. Step 1b.5 read these files at the next wrap-up start, found them missing, and silently took the "skip Phase 1 evaluation entirely" branch. The Phase 1 self-evaluation gate would have stayed dormant indefinitely. Caught only by explicit consolidation analysis. Section 1d Verification rule in root CLAUDE.md covered this trigger semantically ("infrastructure done after writing files but before running them") but didn't fire on this ship — this STOP is the enforcement upgrade.

---

## Core Insight

> **"The human shouldn't need to remember."**

Context compaction and `/clear` destroy details. Files persist. This skill ensures:
1. **Learnings are captured** before compaction erases them
2. **The right tool is named for the right capture type** — process/content capture happens in wrap-up; code-level capture is user-invoked `/ce-compound` mid-session (this skill does NOT orchestrate `/ce-compound`)
3. **Nothing falls through the cracks** — even when you forget to document

---

## Why Explicit Invocation

v2 relied on a `triggers` YAML field, but `triggers` is not a supported field in Claude Code's SKILL.md spec. The only auto-invocation mechanism is **description-based matching** — Claude's LLM matches user requests to the skill's `description` field. This is non-deterministic: distinctive phrases like "run a capture" matched often enough to produce capture files, while common phrases like "wrap up" were too generic and consistently failed to invoke the skill — handled conversationally or intercepted by auto-memory instead.

**Result:** Scan-like invocations worked intermittently, but wrap-up never triggered reliably. Capture files accumulated across sessions with no consolidation phase to process them.

**v3 fix:** Explicit `/learning-loop` invocation is deterministic. Both modes fire exactly when intended — no reliance on LLM matching heuristics.

### Auto-Memory Coexistence

Claude Code also has a built-in auto-memory feature that intercepts natural-language phrases like "capture" and "remember this." Explicit invocation avoids this secondary concern too:

| Feature | Auto-Memory | Learning-Loop |
|---------|-------------|---------------|
| **Invocation** | Natural language ("remember this", "capture") | Explicit `/learning-loop` command |
| **Scope** | Quick facts, preferences | Multi-signal session analysis with quality gates |
| **Output** | `MEMORY.md` entries | Routed to 6 destinations based on type (or Noted/dropped) |
| **Quality gates** | None (direct write) | Type-specific gates + user verification |

**Complementary, not competing:**
- Auto-memory handles quick "remember X" requests — let it
- Learning-loop handles structured session analysis — explicit invocation ensures it runs when intended
- Memory (MEMORY.md) is one of learning-loop's routing destinations, but learning-loop applies quality gates first

---

## Operation 1: Scan Mode

**User story:** "I'm mid-task, context is getting long, but I'm not done yet."

### What Scan Does
- Spawn sub-agent → scan conversation for **raw signals** (unresolved observations, not conclusions)
- Write to `~/.claude/learning-captures/[session-id]/scan-NNN.md`
- **No routing, no conclusions, no quality gates on the signals themselves.** Back to work.

### Scan Process

1. Determine session-id (date + brief context, e.g., `2026-02-24-learning-loop-v3`)

2. Create session directory:
   ```bash
   mkdir -p ~/.claude/learning-captures/[session-id]
   ```

3. Determine scan number:
   ```bash
   ls ~/.claude/learning-captures/[session-id]/scan-*.md 2>/dev/null | wc -l
   ```
   Next scan = count + 1, zero-padded (scan-001.md, scan-002.md, etc.)

4. Spawn scanning sub-agent with SCANNER_PROMPT (Read `references/prompts/scanner-prompt.md` at dispatch)

5. After capture completes, confirm:
   ```
   "Scan complete — captured [N] raw signals to scan-[NNN].md.
   These are unresolved observations, not conclusions.
   Safe to continue or compact. I'll consolidate everything at wrap-up."
   ```

### SCANNER_PROMPT (Raw Signal Mode)

**Read `references/prompts/scanner-prompt.md` at dispatch time** and use its content (below the header) as the sub-agent prompt, substituting `[session-id]` and `[NNN]`. Verbatim v4.1 content — includes the mandatory Recurrence Test (v3.9), the seven signal types, scratch-file cross-referencing, and the scan output schema. **STOP if the file is missing — surface to user; do not improvise a scanner prompt.**

---

## Operation 2: Wrap-up Mode

**User story:** "I'm finally done — maybe sessions later. Make sense of everything."

### What Wrap-up Does
1. **Scan current session** — this context hasn't been captured yet
2. **Triage captures** — show this session's captures + surface any orphaned sessions for user decision
3. **Resolve with hindsight** — which hypotheses were right? What actually worked?
4. **Apply quality gates** on conclusions (not raw signals)
5. **User verification** — present summary, wait for confirmation
6. **Route to destinations** based on type
7. **Clean up capture files**

### Wrap-up Process

#### Step 1: Scan Current Session

Run a scan of the current session first (same as Scan mode), since this context hasn't been captured yet. This ensures the final session's signals are included.

#### Step 1b: Deferred Methodology Check (MANDATORY)

Before consolidating, scan for failures in **this session** that match the `revisit_triggers` of any deferred methodology memory. This binds deferred-methodology review to the session-end event so deferred investigations accumulate real production evidence instead of rotting.

**STOP before proceeding to Step 2 unless all of the following have been produced:**

1. **Enumerate deferred methodologies.** Evidence: paste the output of

   ```bash
   for f in ~/.claude/projects/*/memory/*.md; do
     [ "$(basename "$f")" = "MEMORY.md" ] && continue
     if grep -q "^status:[[:space:]]*deferred[[:space:]]*$" "$f" 2>/dev/null; then
       echo "$f"
     fi
   done
   ```

   If no deferred methodologies exist, state "No deferred methodologies found" and skip to Step 2.

2. **Scan the session for failure events.** Review the session transcript for:
   - User pushback calling out premature completion ("said done but wasn't", "skipped a step", etc.)
   - Stop hook catches (`{"ok": false, "reason": "..."}` messages)
   - Your own acknowledged mistakes that fit a deferred methodology's failure mode

3. **Cross-reference.** For each deferred methodology, check its `revisit_triggers` (regex, case-insensitive) against the session's failure events. Evidence: paste each memory's triggers list and the matched session evidence.

4. **If any match, surface to the user in this shape:**

   ```
   ⚠️ Deferred methodology resurfaced: [name from memory]
   - Matched trigger: [regex pattern]
   - Session failure evidence: [specific quote/line from transcript]
   - This methodology was deferred pending production data; the session provided fresh evidence.
   - Decision: (a) implement now this session, (b) next session, (c) keep deferred + append incident
   ```

5. **If user chooses "keep deferred + append incident" OR "next session",** append a dated entry to the memory file's `## Incidents` section:

   ```markdown
   ### [YYYY-MM-DD] [session-id-short] — [skill involved, if identifiable]
   - **Failure:** [quote or paraphrase]
   - **Gate form at failure:** [compact-imperative / descriptive-label / none / N/A]
   - **Would the deferred methodology have prevented it?** [yes / no / unclear — with reasoning]
   ```

6. **If user chooses "implement now",** scope it as part of this wrap-up OR spin out an immediate task.

7. **If no matches,** say so in one line and proceed to Step 2. No noise when nothing matches.

**Complement:** the `~/.claude/hooks/deferred-methodology-detector.py` UserPromptSubmit hook fires the same check *in-session* when failure phrases are typed in real time. This step is the session-end retrospective backstop for cases the in-session hook missed or where the user wants a consolidated review.

> **Why this step exists (Apr 22, 2026):** Memory entries that say "revisit later" rot without an active trigger. The gate-template production test memory was the motivating case — deferred pending production data, but prior to this wiring had no mechanism to resurface automatically. Generalized to any `status: deferred` memory so future deferrals inherit the behavior.

#### Step 1b.5: Phase 1 Persona Panel Evaluation Check (added v3.5 Apr 28 2026)

Active when Phase 1 shadow mode is in flight. Skip entirely if `~/.claude/learning-captures/persona-eval-runs.txt` does not exist (Phase 1 not yet shipped or already past Phase 1).

If the file exists:

1. **Read accumulators:**
   - Count entries (excluding the header line) in `persona-eval-runs.txt`
   - Read first-entry timestamp from same file
   - Read `~/.claude/learning-captures/phase-1-ship-date.txt` for Phase 1 ship date (set in Step 4c bootstrap)
   - Read `~/.claude/learning-captures/phase-1-decision-log.md` if it exists — most recent decision marks where Phase 1 stands today

2. **Trigger condition (per D2):** evaluation fires if BOTH:
   - `count ≥ 3` (≥3 prior shadow runs accumulated, not counting this current wrap-up) OR `(today - phase_1_ship_date) ≥ 7 days`, whichever first
   - **AND** the latest entry in `phase-1-decision-log.md` is NOT a recent decision (< 24 hours old) — prevents re-firing within the same 24-hour window after a decision was just made

3. **If trigger condition met:**
   a. Spawn Phase 1 Decision Report sub-agent with `PHASE_1_DECISION_REPORT_PROMPT` (Read `references/prompts/phase-1-decision-report-prompt.md` at dispatch). Pass it: all `~/.claude/learning-captures/*/persona-eval.md` files, plus the `persona-eval-runs.txt` log.
   b. Surface the report to the user — show match rate, coverage rate, noise rate, failure-mode distribution, and the GO/HOLD/ITERATE/REVERT recommendation.
   c. Capture user's decision in `~/.claude/learning-captures/phase-1-decision-log.md`. Append in this format:

      ```markdown
      ## <YYYY-MM-DD HH:MM ET> — Decision: <GO|HOLD|ITERATE|REVERT>
      **Eval window:** <first_run_date> to <latest_run_date> (<N> runs)
      **Metrics:** match=<X%>, coverage=<X%>, noise=<X%>
      **Sub-agent recommendation:** <GO|HOLD|ITERATE|REVERT>
      **User decision:** <accepted|overrode-to-X>
      **Reasoning:** <user's note or "accepted recommendation">
      ```

   d. **Per D2: the decision applies to the NEXT wrap-up's Step 3a behavior, not this current wrap-up.** This wrap-up proceeds as scheduled (consolidation + Step 3a personas in shadow mode + Step 4).

4. **If trigger condition not met but Phase 1 is active:** append a one-line note to the user — `Phase 1 still accumulating data: <N> of 3 runs, <X> days since ship.` Proceed to Step 2.

5. **If `persona-eval-runs.txt` does not exist:** Phase 1 not yet shipped (or already past Phase 1 — gatekeeper or revert). Skip this sub-step entirely.

6. **🛑 PROGRAM-CONCLUDED EXIT — check this BEFORE evaluating the trigger condition in step 2 (added 2026-07-26).** Read the latest entry in `phase-1-decision-log.md`. **If it records the tracked hypothesis/program as closed, retired, or permanently resolved** — e.g. *"Phase 2 gatekeeper retired (shadow mode is permanent)"* — then **suppress the trigger, emit one line, and proceed to Step 2:**

   ```
   Phase 1 eval: SUPPRESSED — program concluded <YYYY-MM-DD> ("<decision text>"). <N> runs logged; no open decision for a report to inform.
   ```

   Do **not** dispatch the Decision Report. Do **not** fall through to steps 2–4.

   **Why this branch exists (2026-07-26):** the trigger is `count ≥ 3` **OR** `≥7 days since ship` **AND** the latest decision is not <24h old. Once a program concludes, **all three conditions are permanently true and can never again be false** — the 2026-05-20 entry retired Phase 2 as "shadow mode is permanent," yet 86 runs and 89 days later the step still nominally demands a Decision Report for a decision closed 67 days prior. There was no exit condition, so the burden of not firing fell on **each session's private judgment, silently, every wrap-up.** That is the failure this branch removes: a documented step whose only mitigation was being routed around. *(Caught 2026-07-26 when the dispatching session skipped it by judgment and announced the skip — the announcement is what made the gap visible. Companion watch-list entry tracks destination-back-reasoning at mandated steps outside Step 3/3a, where G21's scoped STOP does not reach.)*

> **Why this step exists (Apr 28, 2026):** The persona panel needs a self-evaluating decision gate at 4 wrap-ups or 7 days post-ship. Without binding the eval trigger to a workflow step that fires every wrap-up, the eval would rot the same way deferred methodology memories do. Step 1b.5 binds the eval to `/learning-loop wrap-up` itself — the same enforcement principle that worked for Step 1b.

#### Step 2: Triage Captures

```bash
ls -la ~/.claude/learning-captures/*/scan-*.md ~/.claude/learning-captures/*/scratch.md 2>/dev/null
```

List all capture files with timestamps, then present a **triage view**.

**Archive-dir convention (added May 18, 2026):** Directories under `~/.claude/learning-captures/` whose names start with `_` (e.g., `_archive/`) are **archive locations for preserved reference fixtures**, NOT active sessions. They MUST be excluded from the "Other Sessions Found" listing — they will not contain scan-*.md or scratch.md files, but a careless `ls -la` of the parent dir will surface them as "unknown session dirs" and trigger recurring "what is this?" confusion at every wrap-up. When constructing the triage view, filter out any directory whose basename starts with `_`. The glob above naturally skips them (no matching scan/scratch files inside) — preserve this property; do not switch to a broader directory listing without the `_*` prefix filter. **Why this exists:** cited fixtures (e.g., the v3.7/v3.8 + Step 5.6 design provenance handoff doc at `_archive/handoff-to-learning-loop-iteration.md`) need to persist so SKILL.md citations remain checkable, but they also need to stay outside the active triage scan so they don't generate recurring noise. The `_archive/` dir + `_*` prefix-skip convention is the structural fix.

**⚠️ Concurrent-ownership check — the FOURTH state (added v4.7, 2026-08-19, MANDATORY, run BEFORE building the table).**

**Purpose before procedure:** *consolidated findings + zero inbound references* is produced identically by a run that was **abandoned** and by a run **currently in flight in a parallel session**. Listing the second as an include / hold / delete decision puts another session's live evidence on the table — and the "safe" branch is not safe either: a `RETAINED:` record is a write onto a **shared** surface asserting a hold the owning session never agreed to. One session unilaterally annotating another's work.

```bash
THIS_SID="[this-session-id]"
IN_FLIGHT_MINUTES=120

# Run over every OTHER session dir before any of them becomes a row in the table below.
# Stat the WHOLE directory, not the triage glob: that glob lists only scan-*.md and
# scratch.md, and in the founding incident the freshest file was the persona review.
for d in ~/.claude/learning-captures/*/; do
  b=$(basename "$d")
  case "$b" in _*) continue ;; esac      # archive / deferred parents, per the convention above
  [ "$b" = "$THIS_SID" ] && continue     # this session's own dir is never "elsewhere"
  # NEVER let an error read as "no recent files". An unreadable directory is
  # UNCLASSIFIABLE, and unclassifiable must fail toward do-not-touch — the same
  # "the one error you must never hide is the one that makes absence look proven"
  # rule Step 6a enforces. `find` exits non-zero and prints nothing to stdout on a
  # permission error, so a swallowed stderr turns it into a confident "triageable".
  recent=$(find "$d" -type f -mmin -"$IN_FLIGHT_MINUTES" 2>&1); rc=$?
  if [ "$rc" -ne 0 ]; then
    echo "🔵 UNCLASSIFIABLE — TREAT AS IN FLIGHT — $b: $recent"
  elif [ -n "$recent" ]; then
    echo "🔵 IN FLIGHT ELSEWHERE — $b (written within ${IN_FLIGHT_MINUTES}m)"
  else
    echo "·  triageable — $b"
  fi
done
```

**An in-flight directory is a STATE TO REPORT, not a decision to surface.** It never becomes a row in the table below. Render it separately, and offer nothing:

```
## In flight elsewhere — no action available, nothing to decide
- [other-session-id] — last write [N] min ago; another session is working in it.
  Not included, not held, not deleted, not read. Returns to normal triage once it goes quiet.
```

**Do NOT, for an in-flight directory:** offer include / hold / delete · write a `RETAINED:` record for it · copy or archive its files · read its `consolidation.md` and fold its conclusions into this wrap-up. Each is a write or a race against a session that never agreed to it, and the last one routes the same conclusions into the same destination files twice.

**Why the threshold is 120 minutes.** It is a heuristic, so it is tuned by which error costs more. A false *in-flight* costs one wrap-up of delay and **self-clears** — an abandoned directory's mtime keeps aging, so it becomes triageable on its own. A false *not-in-flight* opens the delete and annotate branches on live evidence, and that loss is total. Asymmetric miss costs, so the window is generous rather than tight: a live run can sit well over an hour at Step 4, which blocks on the user.

**Why not a lock file — considered and rejected, 2026-08-19.** A marker written at Step 1 carries exactly the information the directory's own existence already carries — *a run started here and has not finished* — so it cannot separate live from abandoned. Only **recency of activity** can. A heartbeated lock could, but that is real engineering for a once-observed failure, and this skill has a documented history of new-version bootstrap steps silently not firing (see **Skill Version Ship Verification**). A marker that never gets written fails in the **dangerous** direction — read as "nobody owns this." The mtime check needs no bootstrap and fails safe.

> **Why this exists (2026-08-19).** A wrap-up's Step 2 read a sibling capture directory holding a finished consolidation and a completed persona review — Steps 1 through 3a done — with zero inbound references, and surfaced a three-way fold-in / hold / delete decision. **It was not abandoned. A concurrent session was mid-flight in it, and finished cleanly about an hour later.** The disambiguator was already on screen: the directory's newest file was six minutes old, printed by the very `ls -la` that produced the listing. The data was read for *contents* and not for *recency*. **Confirmed by test before this branch was written:** three independent baseline runs against that exact listing all saw the six-minute timestamp, all three still classified it "abandoned run," and all three proposed at least one write against the live directory — two of them a `RETAINED:` record, one of them folding its conclusions into the reading session's own wrap-up. Seeing the disambiguator is not the fix. Having a state to put it in is.

Triage view shape:

```
## This Session: [session-id]
- scan-001.md (captured [time] ago, [N] signals)
- scratch.md ([N] micro-signals)

## Other Sessions Found
| Session ID | Last Capture | Age | Signals |
|-----------|-------------|-----|---------|
| [other-session-id] | [date] | [N days ago] | [count] |

→ These may be from sessions that closed without wrap-up.
   For each: include in this wrap-up, skip for now, or delete?
```

**Default:** Only this session's captures proceed to consolidation. Other sessions require explicit user opt-in.

**If no other sessions exist**, skip the triage table and proceed directly.

#### Step 3: Consolidate with CONSOLIDATION_PROMPT

**🛑 STOP — Sub-Agent Gate at the consolidation entry (W7.w, MANDATORY).** Before anything else at this step: are you about to consolidate the signals *yourself, in this main conversation,* instead of spawning the Step 3 consolidation sub-agent and then running the Step 3a persona panel? **That is the W7.w failure mode — do not.** It is still wrong even if:

- **"Sub-agents can't see the transcript, so consolidating in-context is more accurate / a blind sub-agent reasoning over my paraphrase is lossier."** The hand-off file IS the mechanism. Write this session's signals into `scan-NNN.md` (Step 1) richly enough to stand on their own, then pass that file. A thin hand-off is a reason to write a *better scan file* — never a reason to skip the sub-agent. "I have the live context so I'm more accurate" is exactly the self-trust the persona panel exists to check.
- **"It's only 2 signals / the consolidation is tiny / personas would produce 0 challenges / personas are non-blocking so skipping changes nothing."** Run them anyway. Size is not a skip condition, and **"non-blocking" means the personas don't *block* progression to Step 4 — it never makes running them optional.** The *only* legitimate skips are Phase decision = `REVERT` or 0 conclusions (see Step 3a). This is the confirmed recurring rationalization `step3a_skipped_rationalization`.
- **"Mod 6 says don't run heavy machinery when mechanism + destination + incident are already in hand."** Mod 6 governs **watch-vs-codify routing of a conclusion** (Step 4b), not whether to spawn the consolidation/persona sub-agents. No skill rule licenses skipping Step 3/3a. Citing one to justify in-context consolidation is a misread — run the protocol, don't reason around it.
- **"It's late / the user's waiting / they asked for a quick wrap-up."** End-of-session momentum IS the W7.w trigger. "Quick" means fewer signals, not a skipped protocol; the full pass costs ~2 minutes and protects the highest-correction-rate failure modes at exactly the tired moment they slip.

**Reasoning from a predicted outcome ("the sub-agent/personas would add nothing" / "I'd be more accurate doing it myself") back to skipping the step is the W7.w root cause itself.** You cannot know the output is worthless until you produce it.

**Required next action — no exceptions:** confirm/write the `scan-NNN.md` hand-off → spawn the Step 3 consolidation sub-agent against it → run the Step 3a persona panel. Only then proceed.

> **Why this gate exists (W7.w, threshold met 2026-06-10):** Two instances of skipping Step 3/3a under end-of-session momentum. Instance 1 (`session-a`, 2026-05-19): skipped personas, rationalized "personas would produce 0 challenges." Instance 2 (`session-b`, 2026-06-10): consolidated in-context, rationalized "sub-agents can't see the transcript / it's small" — the proper re-run then caught **two real consolidation errors** the in-context pass missed (a missed cluster match + a duplicate git rule already living in `reason-upstream.md`), direct evidence the skipped step had value. The Sub-Agent Rule at the top of this skill only STOPs for Scan mode; this gate covers the consolidation entry, where both skips actually happened.

Spawn a sub-agent with CONSOLIDATION_PROMPT (Read `references/prompts/consolidation-prompt.md` at dispatch). Pass only the **approved captures** (this session + any user-selected others). This is where raw signals become conclusions.

#### Step 3a: Persona Panel (Shadow Mode — Phase 1, added v3.5 Apr 28 2026)

**🛑 STOP — running the persona panel is MANDATORY in shadow mode (W7.w). Read this BEFORE the "non-blocking" framing below.** You ALWAYS run both personas. The *only* legitimate skips are Phase decision = `REVERT` or 0 conclusions (see Skip-this-step conditions). Personas being **"non-blocking / informational / not a gate"** describes what happens to their *output* — it does not *halt* progression to Step 4 — and it **never** means running them is optional. Skipping because *"it's 2 routine Zone 2 items / they'd produce 0 challenges / the user's waiting / it's just ceremony / it's shadow-mode by design"* is the `step3a_skipped_rationalization` failure mode that W7.w exists to fix — NOT a design-sanctioned shortcut. (Instance 1, `session-a` 2026-05-19: skipped personas on exactly this reasoning; user caught it with one question.)

After consolidation produces its draft, run a two-persona adversarial review BEFORE Step 4 surfaces the proposal to the user. The personas target the dominant failure modes observed across 4 prior wrap-up sessions (12 correction rounds, ~75% concentrated in trigger-framing + destination-routing).

**Phase 1 mode:** personas REPORT but do NOT block — i.e., their *output* doesn't halt progression to Step 4. **Per the STOP above, this is not permission to skip them: you still always run them.** Their output appears as additional columns in Step 4's verification view. User reads both views, decides per-row.

**Phase mode resolution** — read `~/.claude/learning-captures/phase-1-decision-log.md` if it exists:
- File missing OR latest decision = `HOLD` OR latest decision = `GO` OR latest decision absent → **shadow mode** (run personas + report, do not block — this is the permanent active mode)
- Latest decision = `REVERT` → **skip Step 3a entirely** (proceed to Step 4 unchanged)
- Latest decision = `ITERATE` → shadow mode, but flag in output that prompts may be in revision

**Note on retired Phase 2 gatekeeper mode (2026-05-20):** A planned Phase 2 gatekeeper variant (challenges block Step 4 until resolved) was retired after analysis showed the binding constraint on learning-loop effectiveness is downstream throughput (watchlist → codified rule graduation), not upstream catch-rate. Shadow mode catches 70%+ of user corrections at zero workflow friction; gatekeeper mode would amplify catches without addressing the graduation gap. See `~/.claude/learning-captures/phase-1-decision-log.md` 2026-05-20 entry for full rationale. **Treat any `GO` entry in the decision log as historical context — shadow mode is the permanent active mode.**

**Sequence (sequential, not parallel — Router depends on Auditor's output):**

1. **Trigger-Moment Auditor** runs first. Spawn as sub-agent with `TRIGGER_MOMENT_AUDITOR_PROMPT` (Read `references/prompts/trigger-moment-auditor-prompt.md` at dispatch). Pass: full consolidation output (`consolidation.md`) + current root CLAUDE.md contents. Returns per-conclusion JSON: `{id, named_trigger_moment, framing_assessment, broader_trigger_class_if_applicable, verdict, challenge_reasoning, counter_proposal}`.

2. **Workflow-Step Router** runs second. Spawn as sub-agent with `WORKFLOW_STEP_ROUTER_PROMPT` (Read `references/prompts/workflow-step-router-prompt.md` at dispatch). Pass: same inputs as Auditor PLUS Auditor's JSON output. Returns per-conclusion JSON: `{id, classification, existing_workflow_step_list_at_trigger, recommended_destination_ranked, consolidation_destination_assessment, verdict, challenge_reasoning, re_route_proposal}`.

3. Both outputs written to `~/.claude/learning-captures/[session-id]/persona-review.json` as a single combined file:

   ```json
   {
     "session_id": "...",
     "phase": "1-shadow",
     "trigger_moment_auditor": [...],
     "workflow_step_router": [...]
   }
   ```

**Skip-this-step conditions:**
- Phase decision = `REVERT` (per phase mode resolution above)
- Consolidation produced 0 conclusions (no input to review)

**NOT a legitimate skip condition** (added May 19, 2026; reframed May 20, 2026 after Phase 2 retirement):
- Personas-would-produce-zero-challenges rationalization — **always run personas in shadow mode regardless of expected output.** The rationalization pattern ("personas would produce 0 challenges anyway given Zone-2-only consolidation") is a confirmed recurring failure mode (`step3a_skipped_rationalization` flagged across 6+ sessions since 2026-05-12). Reasoning from probable outcome back to skip the protocol is exactly the failure personas exist to catch — trusting consolidation's framing/destination is the failure mode personas were built for. This rule is independent of the retired Phase 2 gatekeeper decision; it remains valid for shadow-mode operation.
- **"Personas are shadow-mode / non-blocking / informational, so skipping them changes nothing / it's just ceremony" rationalization** (added v4.1, 2026-06-12, REFACTOR pass). "Non-blocking" describes only that personas do **not block** progression to Step 4 — it does **NOT** make running them optional. Shadow mode is the *permanent active mode* precisely so the panel runs every wrap-up; skipping because it's non-blocking is `step3a_skipped_rationalization` in a new costume. It also **biases the shadow-eval dataset** — the runs you'd skip (routine, low-stakes) are exactly the observations the eval needs, so "these are routine" is a reason to run, not skip. **Logging the skip does not make it legitimate.** The only legitimate skips remain `REVERT` and 0 conclusions.

**STOP and surface if:**
- Either persona sub-agent returns malformed JSON (capture verbatim, surface to user, fall through to Step 4 without persona columns)
- Either persona sub-agent times out or errors (same handling — Phase 1 personas are non-blocking by design)

#### Step 4: Present for User Verification

**v3.8 zoned presentation (May 2 2026; criteria re-keyed v4.6 2026-08-11):** The verification view is structured by **zone**, NOT by content-type-flat-sections. This scales the user's cognitive load with materiality — Zone 1 items get full attention; Zone 2 batch-confirm; Zone 3 auto-routed.

> ⚠️ **The zone CRITERIA are not in this file.** They live at step 6.5 of `references/prompts/consolidation-prompt.md`, which the consolidation sub-agent reads at dispatch. This file governs how a zoned item is *rendered*; that file governs how it is *sorted*. **An edit here cannot change any zone assignment** — the classifier never reads SKILL.md.

**Zones sort on WHO CAN ANSWER, not on how risky the edit is.** Zone 1 = only the user can answer (publishability under their name · scope crossing onto a shared or external surface (the rule's REACH widens onto other people or other projects — not merely that a shared file gets edited) · irreversible-or-intolerable-if-wrong · a genuine preference with no defensible default). Zone 2 = they should know, nothing to decide. Zone 3 = mechanism calls you made, logged in the routing record. Full criteria and provenance at consolidation-prompt step 6.5.

**Persona challenges are adjudicated, not forwarded.** A `challenge` verdict asks a mechanism question — *is this framing right? is this the right destination?* — which is yours to answer. Resolve it before rendering, then zone the RESULT. **State every resolution**: an overruled challenge renders as a Zone 2 line naming the challenge and your reason, so ruling on your own work is never silent. It reaches Zone 1 only if what survives adjudication is itself something only the user can answer. *(Replaces the v3.8 auto-promote, which sent every challenge to Zone 1 regardless — that rule is what left the panel's output with no adjudicator.)*

**Zone-1 cap check:** before rendering, count Zone 1 items. If COUNT > 5, surface the cap warning at top of the view (per consolidation-prompt step 6.5). **Treat a large Zone 1 as a sorting smell first** — under WHO-CAN-ANSWER criteria most sessions produce very few. Re-run the sorting question on each before showing a wall of decisions.

Present the consolidated summary in zone order:

```
## Session Learning Signals — v3.8 Zoned Verification

From [N] scans, consolidation produced [X] conclusions ([Z1] Zone 1, [Z2] Zone 2, [Z3] Zone 3), [H] resolved hypotheses, [N] Noted observations. Cluster audit (Step 4b): [N] active clusters, [sprawl alert? yes/no].

[IF Zone-1 cap exceeded:]
⚠️ Zone 1 cap exceeded: [N] items require your judgment. This is high cognitive load. Options:
- (a) Triage all [N] now (estimated: ~[N×2]min)
- (b) Triage top-priority items now (you pick how many), shelve the rest as Noted
- (c) Treat all as Noted — accept consolidation defaults, no judgment exercised
[/IF]

---

### Zone 1 — Decisions Required ([Z1] items)

[For each Zone 1 conclusion, render with FULL Verification Detail Floor:]

**[C-id] [Type]: [Brief Title]**

- **What happened in this session:** [1-3 sentences with specific incident or pattern. Quote the user or quote yourself if a direct exchange triggered the signal. Concrete event, not abstracted rule.]
- **What's wrong / what's missing:** [explicit gap or failure mode]
- **What the fix does:** [concrete before/after. If destination is a watch-list cluster or sub-entry, NAME what's already in that cluster and how this addition interacts.]
- **Why this destination:** [why this cluster/file/section vs alternatives. Don't reason from secondary constraints (e.g., "root CLAUDE.md is at line budget") when the rule's logic dictates a destination.]
- **Persona challenges (if any):**
  - **[Trigger-Moment Auditor]** ⚠️ challenge: [one-sentence reasoning]
    - Original framing: "[from consolidation]"
    - Counter-proposal: "[from persona]"
    - Broader trigger class (if applicable): "[from persona]"
  - **[Workflow-Step Router]** ⚠️ challenge: [one-sentence reasoning]
    - Original destination: "[from consolidation]"
    - Re-route to: [destination + section]
- **Why only you can answer this:** [the ONE thing you hold that I don't — publishability under your name / scope crossing onto a shared or external surface (the rule's REACH widens onto other people or other projects — not merely that a shared file gets edited) / irreversible-or-intolerable-if-wrong / a preference with no defensible default. **Name it in those terms.** If the honest answer is "it edits an authoritative doc" or "a persona challenged it," this is not a Zone 1 item — adjudicate it and re-zone.]
- **Options, tradeoff, consequence:** [what you are choosing between; what each costs relative to the other; what happens if you do nothing]

**Your choice for [C-id]:** (a) accept consolidation, (b) accept persona counter-proposal, (c) write your own

[Repeat for each Zone 1 conclusion]

---

### Zone 2 — Notify and Confirm ([Z2] items, accept-all default)

Two kinds share this tier: things that **change what you should expect** from the model, the harness, or the workflow (no action required — you are being told, not asked), and routine factual writes you were present for and can spot-check.

**(v3.13) The 1-line Conclusion column MUST lead with a concrete incident / name / verbatim quote / specific framing from THIS session.** Destination column carries the cluster ID / plan path. Do NOT put cluster IDs or shorthand jargon in the 1-line — that defeats verification. See Step 4 Verification Detail Floor "Trigger heuristic" + good/bad examples.

| # | Conclusion (1-line, concrete-anchor-first) | Destination | Personas |
|---|---------------------------------------------|-------------|----------|
| C5 | "[specific incident/name/quote from this session]" | [cluster ID + file] | ✅✅ |
| C6 | "[specific incident/name/quote from this session]" | [cluster ID + file] | ✅✅ |
| ... | ... | ... | ... |

**Default action:** accept the batch.
- Reply **"y"** to confirm all Zone 2 items.
- OR list specific items to expand into full Verification Detail Floor (e.g., "expand C5, C7").

---

### Zone 3 — Decided ([Z3] items, mechanism calls logged not surfaced)

[Z3] items auto-routed to: [destinations summary, e.g., "workflow doc default rules (×6), MEMORY.md (×2), Noted (×4)"].

**(v3.13) When listing individual Z3 items (e.g., in response to "expand Z3"), apply the same concrete-anchor rule as Z2** — the 1-line leads with the session-specific incident/name/quote, not the destination ID.

**Anything to promote to Zone 1?** Reply **"y"** to accept the auto-routing OR list specific item IDs to promote (e.g., "promote C9, C12 to Zone 1").

---

### Cluster + Watch-List State

[Brief summary of cluster audit result + any new entries proposed. Single paragraph or compact table.]

[IF Mod 5 auto-drafted any plans this wrap-up (clusters that met BOTH gates: ≥5 sub-IDs AND no active plan), surface as Zone-3-style single-line notification — NEVER as Zone 1 decision:]

```
✓ Auto-drafted plans (matured clusters, v3.11 Mod 5):
  - <Cluster ID> → <plan path> (N Open Qs)
  - <Cluster ID> → <plan path> (N Open Qs)
Review when ready; promote to `ready-for-autonomous` after answering any Open Questions.
```

[/IF]

---

### Resolved Hypotheses ([H] total)

| # | Hypothesis | Resolution |
|---|------------|------------|
| 1 | "[from scan]" | CONFIRMED / DISPROVEN / STILL UNRESOLVED |

---

### Phase 1 Eval Status

[1-line: "Run #[N], [days] post-ship — Phase 1 Decision Report [trigger met / not met]." If trigger met, surface Decision Report inline below.]

---

### Noted ([N] items collapsed — reply "expand noted" to see)

---

⚠️ **VERIFICATION REQUIRED:**
- Zone 1: explicit per-item choice for each (above)
- Zone 2: "y" to accept batch OR list items to expand
- Zone 3: "y" to accept auto-routing OR list items to promote
- Names, facts, premises in Zone 1 — anything wrong?

Once confirmed, I proceed to Step 4b (cluster audit if not done) → Step 4c (eval data capture) → Step 5 (route).

---

[LEGACY FORMAT REFERENCE — only used if zone classification unavailable, e.g., persona-review.json missing AND consolidation predates v3.8:]

### Ready for Documentation (Passed All Gates) — flat fallback

| # | Type | Classification | Summary |
|---|------|----------------|---------|
| 1 | Discovery | Code-level | "P2024: Timed out fetching connection" — connection pooling fix |
| 2 | User pushback | Process-level (behavioral) | Hypothesis testing before fixes |

### Resolved Hypotheses

| # | Original Hypothesis | Resolution | Evidence |
|---|---------------------|------------|----------|
| 1 | "Root cause might be connection pooling" | CONFIRMED — pool exhaustion under load | Fixed with pool size increase |

### Watch List (Recurring Candidates)

Before finalizing the Noted bucket: check `~/.claude/learning-captures/watch-list.md`.

⚠️ **Root-cause matching, not observation matching (Mod 1, Apr 28 2026).** The match criterion is **"is the fix/remediation the same?"**, not "does the observation text look similar?" Two superficially different observations with the same underlying cognitive or process origin and the same remediation path are **the same watch-list item — increment its incident list, do not create a new entry.**

For each candidate from this session:
1. State the candidate's **root cause** in one sentence (cognitive origin: what mental move broke down? + process origin: which workflow step / rule type drifted?)
2. State the candidate's **proposed fix** (what mechanism would prevent recurrence?)
3. Read every active watch-list entry's `Root cause` and `Fix` columns. For each, ask: "Would the same fix close both?" If yes → **increment the cluster, append this incident as a sub-entry (W_N.x) preserving the specific framing/transcript ref**. If no → new entry with count=1.
4. If the fix is "the W4 retrofit plan" or another already-known plan, the new instance is an instance of that cluster. Add as sub-entry, do not file standalone — even if the surface framing is novel.
5. **When in doubt between fold and new: fold.** Sprawl is the bigger cost. Sub-IDs (W_N.a, W_N.b, …) preserve incident-level traceability inside the cluster so the eventual fix author can trace through every test case.

| # | Root cause (cognitive + process origin) | Fix | Incident summary | Aggregated count | Threshold | Action |
|---|----------------------------------------|-----|------------------|------------------|-----------|--------|
| [N] | [One-sentence origin] | [Remediation mechanism] | [W_N.x: brief framing + date + transcript ref] | [sum] | [2 or 3] | [Escalate → plan generation per Mod 5 / Still watching] |

- After user verification, update `watch-list.md` (increment + sub-entry, or new entry). Move escalated entries to the Archived section.
- **Threshold escalation now triggers Mod 5 (auto-draft plan)** — see Step 4b cluster audit + Step 5 routing for the plan-generation flow.

### Noted (Passed Quality Gates, Below Persistence Threshold)

| # | Type | Summary | Why Not Persisted |
|---|------|---------|-------------------|
| [N] | [Type] | [Brief description] | [What would NOT go wrong if forgotten] |

### Needs Review (Failed Gate)

| # | Type | Failed Gate | Reason |
|---|------|-------------|--------|
| 3 | Failed attempt | Verification | Never confirmed fix works |

### Persona Panel Review (Phase 1 shadow mode — informational, not a gate)

If `~/.claude/learning-captures/[session-id]/persona-review.json` exists, render its contents into a per-conclusion table. The personas reviewed each routed conclusion. Their output is informational — user reads both views and decides per-row.

| # | Conclusion (consolidation summary) | Trigger-Moment Auditor | Workflow-Step Router |
|---|-----------------------------------|------------------------|---------------------|
| 1 | "..." | ✅ pass | ✅ pass |
| 2 | "..." | ⚠️ challenge: scope-too-narrow → broader trigger "<phrase>" | ✅ pass |
| 3 | "..." | ⚠️ challenge: symptom-anchored → mechanism framing "<phrase>" | ⚠️ challenge: re-route from <X> to <Y> |

For each challenged conclusion, expand the counter-proposal underneath:

```
**C2 — Trigger-Moment Auditor counter-proposal:**
- Original framing: "<from consolidation>"
- Proposed mechanism framing: "<from persona>"
- Broader trigger class: "<from persona, if applicable>"

**C3 — Workflow-Step Router counter-proposal:**
- Original destination: "<from consolidation>"
- Re-route to: <destination + section>
- Reasoning: <one sentence>
```

Per-conclusion choices for the user: (a) accept consolidation, (b) accept persona counter-proposal, (c) write your own. The user's choice for each conclusion is captured in Step 4c for eval purposes.

If `persona-review.json` does not exist (Phase 1 not yet shipped, REVERT in effect, or persona run failed and surfaced as malformed), this section is omitted entirely.

---

⚠️ **VERIFICATION REQUIRED:** Does this summary accurately reflect what happened?
- Are names, facts, and premises correct?
- Did I miss anything important?
- Did I capture something that didn't actually happen?
- For each persona challenge: do you accept the counter-proposal, accept the consolidation, or write your own?

Please confirm accuracy before I proceed to documentation.
```

**WAIT FOR USER CONFIRMATION before proceeding to Step 4c.**

#### Step 4 — Verification Detail Floor (originally added v3.7, scoped v3.8 May 2 2026)

The Verification Detail Floor scales rigor with zone (classification criteria: `references/prompts/consolidation-prompt.md` step 6.5):

| Zone | Floor requirement |
|---|---|
| **Zone 1** | **MANDATORY full Verification Detail Floor** — per-conclusion narrative block with all 7 fields (what happened, what's wrong, what fix does, why destination, persona challenges, why only you can answer, options/tradeoff/consequence) |
| **Zone 2** | 1-line summary + destination by default — **the summary MUST lead with a concrete incident/name/quote/specific-framing from THIS session** (v3.13). Full floor available on user request ("expand C5, C7"). |
| **Zone 3** | Destination + 1-line summary only — **same concrete-anchor rule as Zone 2** (v3.13). No floor. User can promote to Zone 1 to see full floor. |

**Required fields for Zone 1 conclusions:**

```
**[C-id] [Title]**
- **What happened in this session:** [1-3 sentences with specific incident or
  pattern. Quote the user or quote yourself if a direct exchange triggered the
  signal. Not the abstracted rule — the concrete event.]
- **What's wrong / what's missing:** [explicit gap or failure mode]
- **What the fix does:** [concrete before/after. If destination is a watch-list
  cluster or sub-entry, NAME what's already in that cluster and how this
  addition interacts (sub-entry W_N.x increment vs new cluster).]
- **Why this destination:** [why this cluster/file/section vs alternatives.
  Don't reason from secondary constraints (e.g., "root CLAUDE.md is at line
  budget") when the rule's logic dictates a destination.]
- **Persona challenges (if any):** [what the persona objected to, your
  concrete adjudication, and your reason — challenges are resolved before
  rendering, never forwarded as the reason this is Zone 1]
- **Why only you can answer this:** [the ONE thing the user holds that you
  don't — publishability under their name / scope crossing onto a shared or
  external surface (the rule's REACH widens onto other people or other
  projects — NOT merely that a shared or PR-gated file gets edited) /
  irreversible-or-intolerable-if-wrong / a preference with no defensible
  default. If the honest answer is "it edits an authoritative doc" or "a
  persona challenged it," it is not Zone 1 — re-zone it.]
- **Options, tradeoff, consequence:** [what they are choosing between; what
  each costs relative to the other; what happens if they do nothing]
```

**STOP and correct if you're:**
- Presenting a Zone 1 conclusion without full Verification Detail Floor (Zone 1 = mandatory floor)
- Mixing Zone 1 and Zone 2 in the same section of the verification view
- Surfacing Zone 3 items in the user's main verification scroll (Zone 3 = single-line summary + "anything to promote?" prompt only)
- Failing to apply zone classification (`references/prompts/consolidation-prompt.md` step 6.5) before rendering Step 4
- **Writing a Zone-1 justification that describes the EDIT rather than what the user holds** — "widens an authoritative doc," "proposes a root CLAUDE.md edit," "a persona challenged it," "Step 5.6 returned 2 of 3." Each is answerable by you. Adjudicate it and re-zone; a Zone 1 slot filled this way costs the user attention that the genuinely-theirs items then compete with.
- Skipping the Zone-1 cap check when Zone 1 has >5 items
- Defaulting to "compressed format because the table is cleaner" for Zone 1 items — Zone 1 always gets the floor
- **(v3.13) Writing a Zone 2 or Zone 3 1-line summary that leads with a destination cluster ID (`W_N.x`, plan path, reference doc path) or shorthand jargon (e.g., "methodology-doc caveat-writing surface") WITHOUT a concrete anchor from THIS session.** The 1-line must lead with a specific name, quote, incident, or framing the user can recognize from this session's exchange. Cluster ID + destination go at the END as routing metadata, not the START as the summary.

**(v3.13) Trigger heuristic — recognize the failing format BEFORE rendering:**

For each Zone 2 / Zone 3 row, check the 1-line summary text:

| Contains | Missing | Verdict |
|---|---|---|
| Cluster ID (`W_N.x`), plan path (`P_N`), or reference-doc path | A specific name / verbatim quote / specific incident phrase from THIS session | **JARGON-ONLY — re-render with concrete anchor, OR auto-expand to full floor** |
| Specific name / verbatim quote / specific incident phrase from THIS session | (anything else) | OK to render compressed |
| Neither cluster ID nor specific anchor | — | Under-specified; re-render with concrete anchor |

If the row is jargon-only, the user cannot verify without re-loading the destination cluster's context into their head — which defeats the "shouldn't need to remember anything to verify" floor that motivated v3.7. Either re-render the 1-line with a concrete session anchor leading, or auto-promote that row to full floor for this wrap-up.

**(v3.13) Good vs. bad examples:**

```
❌ FAILING (cluster-ID-first, jargon-only):
| C1 | W1.p new sub-entry `W1.p.aa` — methodology-doc caveat-writing surface;
       directionally-opposed pair compressed into single causal chain
     | `watch-list.md` W1.p cluster + W4 plan P4 author note | ✅✅ |

User reaction: "There is no description for these Zone 2 and Zone 3 things.
I cannot recall or understand what they're about."

✅ WORKING (concrete-incident-first):
| C1 | Caveat #3 conflated correctly-excluded execs (Trent Charlton, Oleksandr
       Fedorov) with Harmonic-miscategorized real founder (Kerry Lu) — adds
       methodology-doc surface to W1.p
     | `watch-list.md` W1.p cluster + W4 plan P4 author note | ✅✅ |
```

The working version leads with the specific incident (the named people + the specific framing that came up in this session); the destination metadata (`W1.p`, `P4`) comes at the end as routing context, not as the summary itself. The user can recognize "Kerry Lu" / "Trent Charlton" because they came up in this session's exchange; they cannot recognize `W1.p.aa` without re-loading the watch-list.

> **Why v3.7 (Apr 29 2026):** Compressed format passed 4/4 consolidation errors through Step 4 unchallenged in a parallel content-lab wrap-up. User: *"It needs to actually tell me what the thing is that we're trying to analyze... I shouldn't have to remember anything to verify."* Verification under those conditions is performative. The Detail Floor was the v3.7 fix.

> **Why v3.8 amendment (May 2 2026):** v3.7 made the floor MANDATORY for ALL conclusions. Result: a single wrap-up with 14 conclusions (3 Zone-1-shape + 10 Zone-3-shape methodology codifications + many Noted) became 14× the cognitive load. User: *"my brain just fried and I just kind of want to give up."* The floor is right; the scope was wrong. v3.8 makes the floor's rigor scale with zone — so the user's attention scales with where their judgment actually matters. Transcript fixture: `~/.claude/learning-captures/_archive/handoff-to-learning-loop-iteration.md` (v3.7 evidence; archived May 18 2026 from original `2026-04-29-content-lab-post-13-capture-distillation/`) + the May 2 wrap-up output that motivated v3.8 zones.

> **Why v3.13 amendment (May 19, 2026):** v3.8 specified "1-line summary + destination" for Zone 2/3 but didn't constrain WHAT the 1-line summary must contain. In the `2026-05-19-session-a` wrap-up, consolidation produced 4 conclusions (0 Z1, 3 Z2, 1 Z3) and Step 4 rendered the Z2/Z3 rows with cluster-ID-first phrasing like "W1.p new sub-entry `W1.p.aa` — methodology-doc caveat-writing surface." User pushback: *"There is no description for these Zone 2 and Zone 3 things. I cannot recall or understand what they're about."* Same root cause as v3.7 (verification requires remembering), but at a finer grain — the summary itself was jargon-only. After re-expanding with the full floor (which forced concrete incidents back to the surface), verification worked. The fix is to require the concrete-anchor lead at the 1-line layer too, not only the full-floor layer. Trigger heuristic + good/bad examples make the failure recognizable BEFORE rendering, not only after user pushback.

#### Step 4b: Watch-List Cluster Audit + Threshold-Met Plan Generation (MANDATORY — Mod 4 + Mod 5, Apr 28 2026)

Before routing, run a cluster check on the active watch-list:

1. Read `~/.claude/learning-captures/watch-list.md`.
2. Count active entries. **Sprawl alert thresholds (tighter, Apr 28 2026):**
   - **>15 active top-level entries** → sprawl alert
   - **>3 entries sharing the same `Fix` field value** → cluster collision alert

   If either threshold is hit, surface to user:
   ```
   ⚠️ Watch-list cluster check:
   - [N] active entries (threshold: 15)
   - [Cluster name] has [M] entries sharing fix "[fix value]" (threshold: 3)

   Recommend re-consolidation pass before more entries accumulate.
   Consolidate now (spawn sub-agent), defer to next wrap-up, or skip?
   ```
3. If user opts in, spawn re-consolidation sub-agent with the same root-cause-matching prompt as Mod 2 — applied to existing entries, not new candidates. Output: proposed merges with justification.
4. User approves merges; update watch-list.md (preserve sub-IDs for incident traceability).

Skip this step entirely if active entry count ≤15 AND no fix-cluster ≥3.

**Mod 5 — Threshold-met → child sub-agent auto-drafts plan (v3.4 Apr 28, refined v3.11 May 12 2026):**

When a watch-list cluster meets BOTH gates, spawn a child sub-agent to draft a plan in PEP schema:

1. **Maturation gate (v3.11):** ≥5 sub-IDs in the cluster. Tightened from the original v3.4 thresholds (2 or 3) per the May 12 2026 user directive — recurrence evidence is the threshold, not first-pattern speculation.
2. **No-active-plan gate (v3.11):** no plan currently exists for this cluster. Verify both:
   - Grep cluster ID (e.g., `W7.c`) across known plan directories: `~/Documents/claude-projects/claude-skills/plans/`, `~/.claude/plans/`, `~/Documents/claude-projects/Personal/*/plans/`, `~/Documents/claude-projects/Personal/plan-execution-pipeline/plans/`
   - Check the cluster's Fix field for an explicit plan-path reference (e.g., "see plan at /path/to/X.md")

When BOTH gates pass, spawn a **child sub-agent** with `PLAN_DRAFTER_PROMPT` (Read `references/prompts/plan-drafter-prompt.md` at dispatch). The main wrap-up sub-agent does NOT do the drafting work — child sub-agent extraction keeps the main wrap-up context clean (matches the v3.5 persona-panel architecture).

The child sub-agent receives: cluster ID + cluster header + Fix field + all sub-entries + PEP plan schema. It parses Fix for plan location, drafts the plan with each historical incident as a Success Criteria checkbox, handles ambiguous Fix-field areas via `## Open Questions (blocking)` section, writes the file, and reports back.

Plan content schema:

| Plan Field | Source from watch-list cluster |
|---|---|
| `## Objective` | Cluster's root cause + fix in one paragraph |
| `## Success Criteria` | **Every historical incident reframed as a test case checkbox**: "Would this fix have prevented incident W_N.x ([date], [transcript ref])?" Each W_N sub-entry becomes a criterion. The fix must trace through all test cases and demonstrate robustness against all of them. |
| `## Context` | Aggregated incident notes + transcript references + dates + N-failure count + cognitive/process origins |
| `## Open Questions (blocking)` (v3.11) | Every plan section that the drafter couldn't fill with high confidence from the Fix field. Blocks transition to `ready-for-autonomous` until user fills them. The drafter must never fabricate file paths or technical specifics not present in the Fix field. |
| YAML `status` | `draft` |
| YAML `plan_kind` | `executable` (queues into autonomous pipeline) or `pr-spec` if scoped narrowly |
| YAML `priority` | Based on cluster size: ≥10 sub-IDs → high; 5-9 → medium |
| YAML `priority_rationale` | "Watch-list cluster `<ID>` matured to N sub-IDs; fix unimplemented as of `<date>`" |
| YAML `created` | Today's date |
| YAML `project_bucket` | Inferred from chosen plan directory |

Plan file location (Fix-field auto-routing, v3.11):
- Fix field mentions `~/.claude/` paths → `~/.claude/plans/`
- Fix field mentions `~/Documents/claude-projects/claude-skills/` → `~/Documents/claude-projects/claude-skills/plans/`
- Fix field mentions `~/Documents/claude-projects/Personal/<X>/` → `~/Documents/claude-projects/Personal/<X>/plans/`
- No file path mentions in Fix → fallback to `~/Documents/claude-projects/claude-skills/plans/` (most common case)

Filename: `YYYY-MM-DD-<cluster-id>-<short-slug>.md`.

The main wrap-up sub-agent surfaces drafted plans in Step 4's **Cluster + Watch-List State** section as Zone-3-style single-line notifications — NEVER as Zone 1 decisions (would defeat the cognitive-load-reduction goal). Render shape:

```
✓ Auto-drafted plans (matured clusters, v3.11):
  - W7.c → ~/.claude/plans/2026-05-15-w7c-auto-mode-classifier.md (3 Open Qs)
  - W37 → ~/Documents/claude-projects/claude-skills/plans/2026-05-15-w37-hook-false-positive.md (0 Open Qs)
Review when ready; promote to `ready-for-autonomous` after answering any Open Questions.
```

**Why v3.4 → v3.11 refinement (May 12 2026):** Original v3.4 thresholds (count ≥2 or 3) were tuned aggressively to catch sprawl early. In practice, drafting plans on thin evidence created speculative debt. User directive (May 12): *"we want recurrence evidence before adding to the watch-list."* Tightening to ≥5 sub-IDs + no-active-plan aligns Mod 5 with the Workstream A recurrence-test philosophy. Fix-field auto-routing + Open Questions handling were also missing — plans landed in inconsistent locations and over-fabricated specifics from vague Fix fields. Child sub-agent extraction matches the v3.5 persona-panel architecture and keeps wrap-up context lean.

**STOP and surface if:**
- A cluster hits BOTH gates but the PLAN_DRAFTER_PROMPT child sub-agent is not spawned (default action is to draft, not defer)
- The child sub-agent reports back without writing a file (means the drafter failed or refused to run)
- The auto-drafted plan is missing test cases for incidents present in the watch-list cluster (every sub-ID must become a Success Criteria checkbox)

**Mod 6 — Watch-vs-Codify Decision Criterion (added v4.0, 2026-05-20 hygiene pass):**

Before adding any signal to the watch-list, apply this decision criterion:

| Condition | Action |
|---|---|
| **All three present:** mechanism named (the WHY of the failure) + canonical destination identifiable (specific file + section) + ≥1 prior incident | **CODIFY DIRECTLY.** Write the rule into the destination. Log to `graduation-log.md`. Skip the watch-list entirely. |
| **Any one missing:** mechanism unclear (multiple competing hypotheses) OR destination unknown OR pattern not yet trusted (need ≥N instances to distinguish from noise) | **WATCH.** Add to watch-list with threshold-based escalation (Mod 5). |
| **Recurrence of already-codified rule:** matching graduation-log.md entry exists | **INCREMENT `incidents_since_codification`** on the matching graduation entry. If 2+ → re-open trigger (enforcement gap — usually Cluster 1 register-translation pattern). |

**Why (2026-05-20):** Cluster 2 sat at 8 incidents waiting for "more evidence" when the fix was knowable at incident 1 (mechanism = pre-elicit constraints; destination = `pm-partnership.md`). Threshold-based escalation makes sense when the pattern is unknown; it becomes waste when the pattern is already named. The codify-now criterion overrides Mod 5's recurrence threshold (≥5 sub-IDs) when all three conditions are met.

**Mod 6 × failure class (added v4.7, 2026-08-19).** Mod 6 answers **WHEN** — codify now, or watch for more evidence. The Step 5.0 failure class answers **WHAT FORM** a codification may take. They compose; they do not compete. Run Mod 6 first. If it returns **CODIFY DIRECTLY** *and* the conclusion's class is `no-moment`, "codify" must **not** mean writing a paragraph into the destination — it means a mechanism, a verification step whose negative result changes the outcome, a test, or an explicit `HUMAN-CAUGHT:` record. Mod 6's *"canonical destination identifiable"* condition is satisfied by a mechanism's install point just as well as by a file + section, so a `no-moment` finding is never forced onto the watch-list for want of a prose home.

**Mod 7 — Granularity Ceiling Rule (added v4.0, 2026-05-20 hygiene pass):**

Sub-entries within a cluster must not exceed these granularity ceilings:

- **Sub-entry sprawl threshold:** When ≥3 sub-entries share the same root cause + fix shape (only the surface differs), **collapse to 1 entry + "surfaces observed" list/table.** Do NOT create per-surface sub-IDs for the same mechanism.
- **Mechanism-first naming:** Entry header must name the mechanism (WHY) before the surface (WHAT). Surface variants become facets inside the entry, not separate sub-entries.
  - ❌ Wrong: `W1.u.i — Visual surface: cover-image misread` (surface-first, encourages per-surface proliferation)
  - ✅ Right: `W1.u — Surface-signal-as-confident-claim` + table of 8 surfaces (mechanism-first, surfaces are facets)
- **Tabular incident format:** Sub-entries with ≥3 incidents use a compact table (Sub-ID | First-seen | Surface | Count | Routing | Status), not prose paragraphs. Full narrative reserved for entries with unique detail (consolidated chronologies, graduation traces, mechanism explanations).

**Why (2026-05-20):** W1.u accumulated 8 sub-entries (i-viii), W1.k accumulated 4 (parent + .iv with 2 instances), Cluster 1 prose ballooned to ~200 lines. All of W1.u shared the same mechanism + same fix. The granularity decision should happen at *capture* time (when writing the entry), not at *consolidation* time (when sprawl is already visible).

**Mod 8 — Stalled-Deliverable Separation (added v4.0, 2026-05-20 hygiene pass):**

When a watch-list cluster has an associated plan and the plan has been in active-remediation status for >4 weeks without ship:

1. Move the plan's status into the watch-list entry's Status field (e.g., "Plan executed 2026-05-19; registration blocked on /update-config from main session").
2. Watch-list keeps only the **incidents-since-fix counter** and the link to the plan file.
3. Granular sub-IDs preserved but compressed to tabular form (Mod 7).
4. **Trigger attention on the *plan* (deployment target date?), not on the *watch-list* (which is correctly tracking).**

**Why (2026-05-20):** Cluster 1 had a "5+ weeks active remediation" status, but the actual state was "implementation 80% done, blocked on one main-session command." The watch-list framing made it look like learning-loop hadn't acted; the truth was the plan was executing. The watch-list should reveal that distinction so user action goes to the right surface.

**Mod 9 — Graduation Ledger Requirement (added v4.0, 2026-05-20 hygiene pass):**

**Companion file:** `~/.claude/learning-captures/graduation-log.md` (created 2026-05-20).

Every codification action MUST atomically:

1. Update the canonical destination file with the new rule
2. Write a graduation-log.md entry with: mechanism, destination, incident count at codification, re-open trigger
3. Update watch-list entry status to **GRADUATED** (with link to graduation-log.md entry)

A codification that doesn't write to graduation-log.md is **incomplete**. The codification action is the atomic unit; partial codifications are the failure mode this rule prevents.

**Per-wrap-up monitoring (also fires at Step 4c):** After eval data capture, scan that session's incidents against graduation-log.md. If any incident matches a graduated rule's surface, increment `incidents_since_codification` on the matching graduation entry. ≥2 post-fix incidents = re-open trigger (usually Cluster 1 enforcement-gap pattern — rule exists, doesn't fire under task momentum, needs discrete hook).

**Why (2026-05-20):** Zero rules graduated from watch-list to graduation-log in 5 weeks despite multiple Cluster 1 + Cluster 2 fixes shipping at the source (CLAUDE.md, reference docs). The fix shipped; the watch-list never updated. Without a ledger, "did the fix work?" is unanswerable; entries accumulate as a one-way archive.

**Mod 10 — Path-Drift Detection at Cluster Audit (added v4.0, 2026-05-20 hygiene pass):**

During Step 4b's cluster audit, verify referenced paths exist:

- For each cluster with a Fix-field path or plan-path reference, run a one-shot existence check (`test -f` or equivalent).
- If the path doesn't resolve, surface as a STOP item: the reference is broken and the actual file location should be hunted with `find` before the audit proceeds.

**Why (2026-05-20):** Watch-list Cluster 1 referenced the W4 plan at `~/Documents/claude-projects/claude-skills/plans/2026-04-18-w4-discrete-trigger-retrofit.md`, but the actual file lived at `~/.claude/plans/2026-04-18-w4-discrete-trigger-retrofit-plan.md`. Path drift had been invisible for 5+ weeks. Anyone following the reference would fail.

---

#### Step 4c: Capture Phase 1 Eval Data (added v3.5 Apr 28 2026)

Skip this step entirely if `~/.claude/learning-captures/[session-id]/persona-review.json` does not exist (Phase 1 not active, REVERT in effect, or persona run failed).

Otherwise, after user verification completes:

1. **For each conclusion the personas reviewed**, classify the user's actual choice in Step 4:
   - `accepted_consolidation` — user kept consolidation's original framing/destination
   - `accepted_persona_counter` — user accepted the persona's challenge counter-proposal
   - `wrote_own` — user wrote a third version (counts as a correction the persona did not match)
   - `no_correction_needed` — consolidation was right, no challenge issued, no correction made

2. **Compute match classification per persona challenge:**
   - `full_match` — persona's counter-proposal matches user's actual final decision
   - `partial_match` — persona challenged the right conclusion but with different specifics from user's correction
   - `mismatch` — persona challenged but user's correction went a different direction
   - `no_challenge_user_corrected` — persona issued no challenge but user corrected anyway (counts toward coverage miss)

3. **Write `~/.claude/learning-captures/[session-id]/persona-eval.md`** in this exact YAML schema:

   ```yaml
   ---
   session_id: <session-id>
   eval_date: <YYYY-MM-DD>
   phase: 1-shadow
   consolidation_proposals: <int>
   persona_challenges_issued: <int>  # sum across both personas, deduped per conclusion
   user_corrections_made_in_step4: <int>
   ---

   ## Per-Challenge Records

   - challenge_id: 1
     persona: trigger-moment-auditor
     conclusion_id: C1
     consolidation_proposed: "..."
     persona_counter_proposed: "..."
     user_final_decision: "..."
     match_class: full_match | partial_match | mismatch | no_challenge_user_corrected
     failure_category_observed: a | c | d | g | other  # which class the user's correction targeted

   ## Aggregate Metrics

   match_rate: <float, 0.0-1.0>
   coverage_rate: <float, 0.0-1.0>
   noise_rate: <float, 0.0-1.0>

   match_rate_calc: "<numerator>/<denominator>"
   coverage_rate_calc: "<numerator>/<denominator>"
   noise_rate_calc: "<numerator>/<denominator>"
   ```

4. **Append a one-line entry to `~/.claude/learning-captures/persona-eval-runs.txt`** with the format:

   ```
   <YYYY-MM-DD> <session-id> challenges=<N> matches=<N> noise=<N>
   ```

   If the file does not exist, create it with a one-line header: `# Phase 1 persona-panel shadow run log (created <YYYY-MM-DD>)` followed by the entry.

5. **Bootstrap-on-first-run note:** if this is the first Phase 1 wrap-up (file did not exist before this step), also write a sentinel file at `~/.claude/learning-captures/phase-1-ship-date.txt` containing today's date in `YYYY-MM-DD` format. Step 1b.5 reads this to compute the 7-day-since-ship trigger.

**STOP and surface if:**
- Step 3a wrote `persona-review.json` but Step 4 did not present persona columns to the user (means render step is broken — eval would be invalid)
- Per-challenge records reference conclusion IDs that don't exist in the consolidation output (means JSON is malformed — surface verbatim)

#### Step 5: Route to Destinations

**⚠️ Step 5.0 — FAILURE-CLASS GATE (added v4.7, 2026-08-19, MANDATORY — runs BEFORE any routing below).**

**Purpose before procedure:** this loop answers every finding with prose, because prose is what the loop can write. A finding whose trigger is *"any sentence, anywhere"* cannot be fixed by any sentence anywhere. This gate is what stops the loop producing more of what demonstrably does not fire.

**First, the presence check. Run it — do not eyeball it:**

```bash
SID="[consolidated-session-id]"
CONS=~/.claude/learning-captures/"$SID"/consolidation.md
[ -f "$CONS" ] || CONS=~/.claude/learning-captures/"$SID"/consolidation-final.md

# Count conclusions by an EXISTING mandatory per-conclusion field, never by heading shape.
n_concl=$(grep -cE '^\*\*Zone:\*\*' "$CONS")
n_class=$(grep -cE '^\*\*Failure class:\*\*' "$CONS")

if [ "$n_concl" -eq 0 ]; then
  echo "❌ HALT — no '**Zone:**' fields in $CONS. Wrong file, or malformed consolidation output."
  exit 1
fi
if [ "$n_class" -ne "$n_concl" ]; then
  echo "❌ HALT — $n_concl conclusions (by '**Zone:**' count) but $n_class 'Failure class:' fields."
  echo "   Do not route. Return to the consolidation output and classify the missing ones."
  exit 1
fi
echo "✅ failure-class field present on all $n_concl conclusions"
```

⚠️ **Count on `**Zone:**`, not on the heading — this was tested, not assumed.** Across five real consolidation files the conclusion headings took at least three different shapes (`### 1.`, `### C1.`, `#### C1.`) and hypothesis blocks reused the same shapes: a heading regex read **25** conclusions in a file that had **14**, so it would have HALTed a correct wrap-up. `**Zone:**`, `**Route to:**` and `**Wedge test:**` each appear exactly once per conclusion and agreed with one another in **all five** files. Do not "simplify" this back to counting headings.

> **Consolidation files written before v4.7 carry no `Failure class:` field and will HALT here — that is correct, not a bug.** They are historical artifacts. A session opted in at triage is re-consolidated by a fresh Step 3 sub-agent, which produces a new `consolidation.md` that does carry the field; point this check at that file, never at the archived one.

**Then, per conclusion, CONFIRM OR OVERRIDE the class with the destination file actually open.** The consolidation sub-agent read blind and may have marked a class `(provisional)`; you have the destination it could not see. **State which you did** — *"confirmed `shape`"* or *"overrode `placement` → `no-moment`, because …"*. An unstated confirmation is the same as not having done it.

| Class | The remedy may be | The remedy may NOT be |
|---|---|---|
| `placement` | Re-home the rule to a surface loaded at the failing moment · widen the scope key · fix the ritual that enumerated the wrong inputs. **Name where it goes wrong.** | Making the existing text **louder** — bolding it, adding MANDATORY, appending another example — without changing *where it lives* or *what it matches*. Editing the trigger phrase so it reaches this case IS a placement fix; emphasis is not. |
| `shape` | Re-shape the trigger so it can match its own case. | Adding a second rule beside the one that cannot match. |
| `no-moment` | Exactly one of the four below. | **Any new or clarified paragraph at the destination.** |
| `n/a` | Normal routing per the tables below. | — |

**A `no-moment` finding routes to one of exactly four things, and nothing else:**

1. **A mechanism** — a hook, script, pre-gate or installer: something that runs *without being consulted*.
2. **A separate verification step WHOSE NEGATIVE RESULT CHANGES THE OUTCOME.** A step that can only ever confirm is not one of these.
3. **A test** that fails when the case recurs.
4. **An explicit human-caught record** — the honest option when no mechanism is cheap, and a *routing*, not a failure to route. One line on `watch-list.md`:
   `HUMAN-CAUGHT: <mechanism> — the question that catches it: "<the literal question>" — no mechanism proposed because <reason>.`
   Recording the **question** is the payload. Recording that the surface is unmechanized is what makes it countable instead of invisible.

**Never another paragraph for a `no-moment` finding.** That is the rule accretion that root `CLAUDE.md` § 7 names as a failure signal, arriving through the routing step. If you catch yourself writing *"the rule should say more clearly that…"*, re-read the class.

**Composition with Mod 6 (Step 4b):** they are different axes and they run in a fixed order. **Mod 6 decides TIMING** — codify now, or watch for more evidence. **The failure class decides FORM** — what a codification is allowed to *be*. Run Mod 6 first; if it returns CODIFY DIRECTLY, the class then constrains what "codify" means. A `no-moment` conclusion that Mod 6 sends to CODIFY DIRECTLY does **not** get a paragraph written into the destination — it gets one of the four above. Mod 6's *"destination identifiable"* condition is satisfied by a mechanism's install point exactly as well as by a file + section.

**Honest limit of this gate.** The presence check is mechanical; the *correctness* of a class, and the choice among the four, are not. This gate makes omission impossible to do silently — it does not make misclassification impossible. **And on current evidence it is not correcting a demonstrated tendency to answer `no-moment` findings with prose**: pre-change testing across two fixtures and six independent runs never once produced that. Its measured value is narrower and real — the class becomes *recorded*, so it can be counted, and silent divergence between runs becomes visible. Full test result and its limits: consolidation-prompt step 5.5b.


After user confirms, route each learning to its proper destination:

| Type | Destination | Handler |
|------|-------------|---------|
| **Code-level** (confirmed fixes) | `docs/solutions/` via user-invoked `/ce-compound` | Wrap-up surfaces nudge only; user invokes `/ce-compound` mid-session |
| **Process-level (behavioral)** | CLAUDE.md (root or project) | Learning-loop direct (with consolidation discipline) |
| **Process-level (operational)** | Project operational docs* | Learning-loop direct |
| **Skills-level** (skill building/authoring/maintenance) | claude-skills repo (CLAUDE.md or playbook) | Learning-loop direct |
| **Facts** (pure recall, no behavior change) | Memory MEMORY.md | Learning-loop direct |
| **Content-level** (understanding shifted) | Judgment Ledger | Learning-loop direct |
| **Watch List** (noted, but may recur) | `~/.claude/learning-captures/watch-list.md` | Learning-loop direct (increment or add entry) |
| **Noted** (below persistence threshold) | Not persisted | Acknowledged in wrap-up summary |

*Operational docs routing (adapts to repo infrastructure):
1. Does the project have dedicated operational docs (playbooks/, OPERATIONS_PLAYBOOK, etc.)? → Route there
2. No dedicated docs? Is it significant enough for CLAUDE.md? → Add to project CLAUDE.md under operational section
3. Not significant enough for CLAUDE.md? → Memory as operational note, or Noted/drop

**Memory Routing Decision Test:**

> "Does this change how Claude should behave?"

| Answer | Destination | Example |
|--------|-------------|---------|
| **Yes — changes decisions globally** | Root CLAUDE.md | "Always read the full file before editing" |
| **Yes — changes decisions in this project** | Project CLAUDE.md | "Develop when you have lived material" |
| **Yes — changes procedure execution** | Project operational docs* | "Reactive before evergreen scheduling" |
| **Yes — changes how skills are built/maintained** | claude-skills CLAUDE.md or playbook | "Always include Gotchas section" |
| **No — it's a fact** | Memory MEMORY.md | "User's spouse's name", "Dave is L1-L2" |
| **No — worldview/judgment shifted** | Judgment Ledger | "The Leash Length Problem" insight |
| **No — career/life-level context changed** | PERSONAL_CONTEXT.md | "Content is career infrastructure, not a side project" |
| **Interesting but forgettable** | Noted (not persisted) | "Research sprints need different cognitive mode" |
| **Interesting but only 1 occurrence** | Watch List (`watch-list.md`) | "Editorial review missed structural-fit check" |
| **Code fix** | `docs/solutions/` | Connection pooling timeout fix |

**⚠️ Content-Level vs. Process-Level Distinction (Critical):**

A learning *about content work* is NOT automatically "content-level." Apply this test:

> "Could this become a published insight — did my worldview or judgment framework shift?"
> vs.
> "Did I learn a better way to do content work — an operational/editorial rule?"

| Test Result | Destination | Example |
|-------------|-------------|---------|
| **Worldview shifted** — publishable insight | Judgment Ledger | "Consensus vs. non-consensus is the real AI division of labor" |
| **Operational improvement** — how to do content work better | Project CLAUDE.md (Content Lab) | "Diversify publishers", "Same-day launch for timely content", "Cover images must match framing" |

The Judgment Ledger is for **judgment shifts that could become content**. Editorial rules, scheduling heuristics, and production guidelines are process-level learnings — they go to CLAUDE.md even if the project is Content Lab.

**⚠️ Content STRATEGY insights are process-level, not content-level.** Insights about what type of content generates what type of engagement ("infrastructure-gap posts attract builders," "dense content has longer tail," "body-of-work coherence drives reader synthesis") are operationally valuable but they're about HOW CONTENT WORKS, not about how AI capability meets reality. They route to performance_log patterns or editorial strategy, not the Judgment Ledger. The test: "Would this insight be interesting to someone who doesn't publish content?" If no, it's content strategy, not a worldview shift.

> Why this exists (Post #11, Apr 8 2026): "Infrastructure-gap posts function as builder bat signals" and "body-of-work coherence confirmed by cross-post synthesis" both passed the "could this be published?" test (a meta-post about content strategy COULD be published) but failed the content wedge test. The first test is too permissive — always apply the wedge filter before routing to Judgment Ledger.

**⚠️ Content Wedge Filter (Judgment Ledger entries only):**

Before routing to the Judgment Ledger, apply the content wedge test from `positioning/content_wedges_v2.md`:

> "Does this insight fit the positioning: *where AI capability meets reality — from an investor who builds*?"

| Wedge Fit | Action |
|-----------|--------|
| **Yes** — insight about AI capability vs. reality gap | → Judgment Ledger |
| **No** — operational process, editorial craft, or unrelated domain | → Route to appropriate CLAUDE.md instead |
| **Borderline** — tag with `⚠️ wedge-check` for user decision during wrap-up verification |

This prevents the Judgment Ledger from accumulating entries that are genuinely useful learnings but would never become published content under the Ground Truth positioning.

**Process-Level Routing (Behavioral — Global vs. Project):**

For behavioral learnings, apply a second test: *"Would this apply if I was working in a completely different project?"*

| YES → root CLAUDE.md (with size gate below) | NO → project CLAUDE.md |
|---|---|
| "When building APIs, ask about naming convention first" | "This repo's API uses camelCase for all endpoints" |
| "Before modifying functions, read entire implementation" | "Supabase RPCs in this project use `verb_noun` naming" |

**⚠️ CLAUDE.md Size Gate (MANDATORY — applies to ALL CLAUDE.md files):**

Before writing ANY behavioral learning to a CLAUDE.md file (root or project), apply the **trigger-vs-protocol split**:

> "Is this a trigger/routing keyword (2-5 lines) or a detailed protocol (>5 lines, STOP lists, checklists)?"

| Answer | Action |
|--------|--------|
| **Trigger** (2-5 lines, tells Claude *when* to act) | Write to the target CLAUDE.md |
| **Protocol** (>5 lines, tells Claude *how* to act in detail) | Write to a reference/companion file, add 2-3 line trigger + pointer in CLAUDE.md |
| **Extends existing rule** (new STOP item or provenance for an existing section) | Add to the existing rule's reference/companion file, NOT to CLAUDE.md |

**Reference file locations by target:**
- Root CLAUDE.md → `~/.claude/reference/<topic>.md`
- Project CLAUDE.md → project's own docs folder, or `_meta/<topic>.md`, or inline in SESSION_LOG.md if it's a one-off

**Line count check before writing:** Read the target CLAUDE.md and count lines. If at or near its budget, force extraction regardless of learning size.
- Root CLAUDE.md target: <250 lines (force extraction at ≥230)
- Project CLAUDE.md: no fixed number, but apply judgment — if the file requires scrolling to scan, it's too long. Extract.

> Why this exists (Apr 3, 2026): Root CLAUDE.md grew from 173 to 351 lines over 6 weeks. Every learning routed without a size gate. The same mechanism applies to project CLAUDE.md files — any destination that receives learnings without a size check will bloat.

#### Step 5b: Reverse-Check Consumers (Added Apr 17, 2026)

If Step 5 routed any learning that **restructures an authoritative doc** (creates a new source-of-truth section, moves a principle between files, consolidates duplicates, cuts a section), run the reverse-check before cleanup:

1. Enumerate downstream consumers — grep for references to the restructured section/file across skills, agents, playbooks, reference files, hooks
2. Verify each reference still resolves correctly after the restructure
3. Update routing logic in every affected consumer

See `~/.claude/reference/procedural-rule-routing.md` "Reverse-Check: Consumers After Doc Restructure" for the full protocol. Global principle lives in root CLAUDE.md ("Procedural Rules Belong at the Workflow Step").

**STOP and correct if:**
- A learning restructured a rule's home but the skills/agents that invoke that rule still route to the old home
- The distillation output updates the authoritative doc but doesn't mention which consumers need follow-up updates

#### Step 6: Clean Up (MANDATORY — verify cleanup fired before declaring wrap-up complete)

After documentation is confirmed, clean up only the **sessions that were consolidated** (this session + any user-approved others).

#### ⚠️ Step 6a — INBOUND-REFERENCE CHECK, before any `rm` (added 2026-07-28, MANDATORY)

**This step deletes things other files point at.** Before removing anything, grep the durable surfaces for references into the directory you are about to destroy:

```bash
SID="[consolidated-session-id]"

# Assert the named surfaces EXIST before trusting a clean result. A renamed or moved
# watch-list used to be swallowed by `2>/dev/null` and read as "no references" — the one
# error you must never hide is the one that makes absence look proven.
for f in ~/.claude/learning-captures/watch-list.md \
         ~/.claude/learning-captures/graduation-log.md; do
  [ -f "$f" ] || { echo "❌ HALT: expected surface missing: $f — cannot conclude 'no references'"; exit 1; }
done

# Search everywhere referrers ACTUALLY live, not a hand-written three-item list.
grep -rn "$SID" \
  ~/Documents/claude-projects \
  ~/.claude/plans \
  ~/.claude/learning-captures \
  ~/.claude/projects/*/memory \
  --include="*.md" --include="*.json"
```

⚠️ **Why the path set is this wide (verified 2026-07-30).** The previous version grepped
exactly three surfaces — `watch-list.md`, `graduation-log.md`, `~/.claude/plans/*.md` — and
nothing in any workspace repo. Simulated against a real retained session:
**seven files of preserved evidence for it lived in a separate project repo, which the old
guard never looked at.** Its only
inbound match was a plan file — and plans get marked `complete` and archived, at which point
the match vanishes and the directory becomes deletable while seven copies still point at it.
**The protection was incidental, not structural.** Same shape as the named
`ALREADY-TRACKED-CORPUS` limit: a scan that iterates your own working list can validate bad
content in known files but can never find a bad file.

**If anything matches, the referenced CONTENT must be lifted out and written into the referring surface before the delete — not re-pointed, not re-derived.** A pointer whose target this step deletes is a guaranteed dangling reference on a fixed timer.

#### ⚠️ Step 6a-ii — ROUTING-COMPLETION CHECK: an abandoned run is not a completed one (added 2026-07-30)

**This is a SEPARATE check from the inbound-reference grep above, and it catches the case that
one structurally cannot.** The reference check only finds a referrer if some earlier session
*wrote* one. An ABANDONED run writes none — so the grep comes back cleanest exactly when
nothing was done with the findings. Its protection is strongest when a human already noticed
and absent when nobody did.

**⚠️ Ownership precondition — run this FIRST (added v4.7, 2026-08-19, MANDATORY).** Everything in Step 6 — this check and every `rm` below — applies **only to directories THIS wrap-up consolidated**: this session, plus any the user explicitly opted in at Step 2. For any other directory, *"abandoned"* is not an available verdict and neither is deletion.

```bash
THIS_SID="[this-session-id]"

# The abandoned signature and the in-flight signature are IDENTICAL — consolidated findings,
# no inbound references. Only recency separates them, and only for a dir that isn't yours.
if [ "$SID" != "$THIS_SID" ]; then
  # Same error-safety as Step 2: unreadable must never read as "nobody owns this".
  recent=$(find ~/.claude/learning-captures/"$SID" -type f -mmin -120 2>&1); rc=$?
  if [ "$rc" -ne 0 ] || [ -n "$recent" ]; then
    echo "🔵 IN FLIGHT ELSEWHERE (or unreadable) — $SID is not this wrap-up's."
    echo "   Not abandoned. Not deletable. Not annotatable. Report it and move on."
    exit 0
  fi
fi
```

Run this before the HALT below, or the HALT fires on a live session's evidence and hands the user a decision whose safe-looking branch — writing a `RETAINED:` record — is a write onto a shared surface about work that is not theirs to hold.

```bash
# A directory holding consolidated findings whose session id appears on NO durable surface is
# an ABANDONED run, not a completed one. Do not delete it. Surface it.
if [ -f ~/.claude/learning-captures/"$SID"/consolidation.md ] || \
   [ -f ~/.claude/learning-captures/"$SID"/consolidation-final.md ]; then
  if ! grep -rq "$SID" ~/Documents/claude-projects ~/.claude/plans \
        ~/.claude/learning-captures/watch-list.md \
        ~/.claude/learning-captures/graduation-log.md --include="*.md" 2>/dev/null; then
    echo "❌ HALT — ABANDONED RUN: $SID holds consolidated findings that are referenced NOWHERE."
    echo "   This is not a finished session awaiting cleanup. Nothing was routed."
    echo "   Surface the findings to the user and get a routing decision. Do NOT delete."
    exit 1
  fi
fi
```

> **Why (verified 2026-07-30, and it corrects an earlier read of this step).** Step 6a's
> founding incident WAS a deliberate deferral whose pointer sat in the watch-list header — so
> the reference grep *would* have fired for it. **Step 6a does close its founding incident.
> What it never closed is abandonment.** One real wrap-up ran its first few steps, wrote
> seven artifacts, and stopped: no routing, no deferral note, no reference anywhere. A guard
> that asks "does anything point at this?" returns clean, and the evidence is deleted for the
> precise reason that nobody had acted on it yet.

**The general rule (write it into the deferral, not just here): a deferred decision must carry its PAYLOAD on a durable surface, never a path into a volatile store.** The moment you write "revisit this at a future wrap-up, see `<capture-dir>/...`", you have created a note that outlives its own referent.

> **Why (2026-07-28):** the watch-list header carried a note deferring four persona-reviewed source-fidelity guardrails, with the instruction "route or explicitly discard at a future wrap-up; do NOT re-derive." The wrap-up that finally read the note found the capture directory **already deleted by this step**, and the guardrails' content unrecoverable — verified against a full recursive listing including `_archive/`, a case-insensitive grep, and a home-wide find. The deferral surface worked exactly as designed: the note was where the future decision would read it. What failed is that the note held a *pointer*, and this step deletes pointees. Disposition had to be recorded as CLOSED-UNRECOVERABLE — scope unknown.
>
> **Scope note:** root CLAUDE.md's `Trace Before Removing` covers this class conceptually, but its trigger is proposal-shaped ("before the words *we should remove X*") — a scripted `rm` inside a mandated skill step never consults it. This is that rule applied at an automated-removal moment; see `~/.claude/reference/trace-before-removing.md`.

```bash
# Delete consolidated session files individually (safer than rm -r; transparent;
# preserves explicit per-file accountability)
rm ~/.claude/learning-captures/[consolidated-session-id]/scan-*.md 2>/dev/null
rm ~/.claude/learning-captures/[consolidated-session-id]/consolidation.md 2>/dev/null
rm ~/.claude/learning-captures/[consolidated-session-id]/persona-review.json 2>/dev/null
rm ~/.claude/learning-captures/[consolidated-session-id]/persona-eval.md 2>/dev/null
rm ~/.claude/learning-captures/[consolidated-session-id]/scratch.md 2>/dev/null
rmdir ~/.claude/learning-captures/[consolidated-session-id]/

# Verification (MANDATORY — confirms cleanup actually fired)
ls ~/.claude/learning-captures/ | grep "[consolidated-session-id]" && echo "❌ CLEANUP FAILED — session dir still exists; re-run Step 6" || echo "✅ Session dir cleaned"
```

**Do NOT delete sessions the user chose to "skip for now"** — they remain for future wrap-ups.

**STOP and correct if you're:**
- About to declare wrap-up complete (Step 11 Report) without running the cleanup verification line above
- Reading sub-agent's "done" report (Step 3 / Step 3a return) and drafting the user-facing summary WITHOUT executing Step 6 cleanup in the main session
- Treating "eval entry appended to runs log" (Step 4c) as proof that cleanup happened — Step 4c writes the log, Step 6 removes the dir; both are required and they are NOT the same command

**End-of-wrap-up cleanup gate — FOUR states (third added 2026-07-30; fourth added 2026-08-19):** Before writing the Step 11 Report message to the user, run a final `ls ~/.claude/learning-captures/ | grep [consolidated-session-id]`. If the session dir is still listed, it is exactly one of:

1. **Cleaned** — not listed. Proceed.
2. **Silently skipped** — listed, and NO retention record exists (see below). This is the failure this gate was built for: execute the per-file rm + rmdir block above before proceeding to Step 11.
3. **DELIBERATELY RETAINED** — listed, AND a retention record exists on a durable surface naming the reason and the unblock condition. **This is a valid terminal state. Do NOT delete, and do NOT treat it as a skip.** Note it in the Step 11 report and proceed.
4. **IN FLIGHT ELSEWHERE / NOT YOURS** — listed, and it is **not a directory this wrap-up consolidated.** This is not a state of *your* cleanup at all: nothing to clean, nothing to record, nothing to decide. **Do NOT delete it, do NOT write a `RETAINED:` record for it, and do NOT read it as state 2.** Note it in the Step 11 report as one line — "[N] other session dir(s) present, untouched" — and proceed.

```bash
# Distinguish the states — OWNERSHIP first, then a RECORD (not just the directory's presence).
THIS_SID="[this-session-id]"   # the session this wrap-up is FOR; $SID is the dir being judged
if [ "$SID" != "$THIS_SID" ]; then
  echo "🔵 state 4 — not this wrap-up's directory; leave it entirely alone"
elif grep -rq "RETAINED: $SID" ~/.claude/learning-captures/watch-list.md \
        ~/.claude/plans --include="*.md" 2>/dev/null; then
  echo "✅ state 3 — deliberately retained; leave it alone"
else
  echo "⚠️ state 2 — no retention record found; cleanup was skipped"
fi
```

A retention record is one line on a durable surface:

`RETAINED: <session-id> — <reason> — unblocks when <condition> — recorded by <session-id of the wrap-up that consolidated it>`

Writing it is what converts "I decided to keep this" from a private judgment into a checkable fact.

**Only a session that CONSOLIDATED the directory may write its retention record (owner field added v4.7, 2026-08-19).** In the founding incident of state 4 the dangerous branch was not deletion — it was the option that looked safe: writing a `RETAINED:` line onto a shared surface for a directory another session owned, asserting a hold that session never agreed to. The rule makes writing it wrong; the owner field makes a non-owner's record visible as one. Records written before v4.7 carry no owner field and remain valid — the state-2/3 grep matches on the `RETAINED: <session-id>` prefix, which is unchanged.

> **Why the third state (2026-07-30).** With only two states, a reasoned retention — evidence deliberately kept because a routing decision is still open — is **indistinguishable from the failure the gate exists to catch**, so the gate can only be satisfied by deleting the thing you meant to keep, or by routing around the gate. That is the same "the burden of not firing falls on each session's private judgment" defect that the Step 1b.5 program-concluded exit branch was added to remove. A gate with no branch for the legitimate case teaches sessions to ignore it. If cleanup verification still fails (e.g., permission error), surface to user verbatim: "Cleanup of [session-id] failed: [error]. Please run manually." Do NOT proceed to summary report without resolving cleanup state.

> **Why this rule exists (May 19, 2026):** Audit of `~/.claude/learning-captures/` surfaced 4 wrap-up-completed orphan session dirs (2026-05-14-learning-loop-mechanism-refinement / 2026-05-18-country-of-geniuses-linkedin-and-cover / 2026-05-19-session-a / 2026-05-19-pep-executor-and-interaction-rework) — all had eval entries in `persona-eval-runs.txt` (Step 4c writes work) but their session dirs were never deleted (Step 6 silently skipped). Common pattern in eval log: `deviation_noted=phase2_gatekeeper_implementation_gap`. Failure mode: combined-subagent dispatch returns "done" → main session reads report → main session drafts user-facing summary → Step 6 `rm -r` command never fires. Old `rm -r` form gave no verification feedback (silent success or silent skip indistinguishable). Per-file rm + explicit verification line forces the main session to confirm cleanup actually happened before declaring wrap-up complete. User directive: *"the right move is to use rm to delete individual files."* Same family as W1.r-pre-send (completion-token rendering before per-output-type verification) — workflow rule drifts under end-of-task momentum; verification gate before declaring done is the structural fix.

#### Step 6b: Cross-Reference _ideas/

After routing learnings, check if any frustration or bottleneck signal from this session matches a parked idea in `_ideas/`. Use the synthesis summary in `_ideas/CLAUDE.md` for matching — not filenames.

```
1. Read _ideas/CLAUDE.md "Current Synthesis Summary" table
2. For each routed learning (especially process-level frustrations):
   ask: "Does this pain connect to a parked idea?"
3. If match found, note:
   "This pain connects to `_ideas/[slug].md` — that idea may be gaining energy."
```

This is how Type 1 (Infrastructure) and Type 3 (Study/Learn) ideas gain priority — through accumulated pain signals, not scheduled reviews.

#### Step 7: Mention Flagged Insights

If any content-level insights were classified as Pattern or Principle:

```
"Note: You have [N] Pattern/Principle insight(s) flagged for content development
in the Judgment Ledger. These are ready for a dedicated Content Development Session
when you have time — they won't be processed in this session."
```

**Do NOT offer to process them now.** Content development requires dedicated thinking time.

#### Step 8: Git Commit & Push Check

After all learnings are routed and capture files cleaned up, check if the current repo is a git repository. If not a git repo, skip this step entirely.

```
0. Run `git rev-parse --is-inside-work-tree` — is this a git repo?
   ├── NO → Skip Step 8 entirely
   └── YES → Continue

1. CHECK .gitignore exists:
   ├── .gitignore EXISTS → Continue to step 2
   └── .gitignore MISSING → Prompt:
       "This repo has no .gitignore. Without one, system files (.DS_Store,
        editor temp files, log files) will clutter commits.

        Create a standard .gitignore? (Y/skip)"

       If Y → Create .gitignore with:
         .DS_Store
         **/.DS_Store
         .playwright-mcp/
         *.swp
         *.swo
         *~

       Then run `git rm --cached .DS_Store 2>/dev/null` to untrack if
       already committed. Stage the new .gitignore.

2. Run `git status` — are there uncommitted changes?
   ├── NO changes → Skip (nothing to commit)
   └── YES changes → Continue

3. Show the user a summary:
   "SESSION WORK TO COMMIT:
    - [N] files modified, [N] files added, [N] files deleted
    - Key changes: [1-2 sentence summary of what changed this session]

    Commit and push to [remote URL]? (Y/skip)"

4. If user approves:
   └── Stage all relevant files (.gitignore handles exclusions)
   └── Commit with descriptive message summarizing session work
   └── Push to remote if one exists (`git remote -v` to check)
   └── **VERIFY the push landed — don't assume the global auto-push hook fired** (it has silently no-opped ≥3× across distinct causes, incl. a plain main-checkout commit 2026-06-18). Run `git rev-list --left-right --count origin/main...main` → expect `0 0`; if `main` is ahead, `git push origin main`. This is a 1-line self-check, NOT a permission ask. See memory `feedback_git_auto_push_hook`.
   └── Confirm: "Committed and pushed: [short hash] [message]" — only after the `0 0` verify

5. If user skips:
   └── "Skipped. You have uncommitted changes — run `git status` to review later."

6. CHECK cross-repo modifications:
   For each repo that received a routed learning (other than current repo):
   ├── cd to that repo
   ├── git status → show changes
   ├── Prompt: "Also modified [repo-name] with skills-level learning. Commit and push? (Y/skip)"
   └── If Y → stage, commit, push, return to original repo

   Build the list of repos to check in TWO passes:

   Pass A — Hardcoded known repos (always check):
   - ~/Documents/claude-projects/claude-skills/ (if skills-level learnings were routed)
   - ~/.claude/ (if reference docs or settings changed → claude-config auto-push handles this)
   - ~/.claude/skills/learning-loop/ (if routing rules were updated → needs push to claude-learning-loop)

   Pass B — Dynamic discovery (catches nested repos not in the hardcoded list):
   For each file written or modified during Step 5 routing:
   ├── cd to the file's containing directory
   ├── Run: git rev-parse --show-toplevel 2>/dev/null
   │   ├── Returns a path → that path is the repo this file belongs to. Add to the check list.
   │   ├── Returns nothing → file is outside any repo (local-only, e.g. watch-list.md). Skip — but log as "intentionally local-only" in the wrap-up summary.
   │   └── Returns a path DIFFERENT from the parent repo's toplevel → this is a nested repo. It would be invisible to `git status` run from the parent. Flag clearly.
   └── Union Pass B results with Pass A. Deduplicate by toplevel path.

   Why Pass B exists: Pass A is a fixed list and goes stale when new nested repos are added. Pass B treats the routed files themselves as the source of truth — whichever repo git actually assigns them to is the repo that needs a commit. Without Pass B, a newly-introduced nested repo would silently miss commits until someone notices and adds it to Pass A.

   Expected output of the combined pass:
   "Routed files belong to N repo(s): [repo1, repo2, ...].
    Local-only files (skipped): [paths].
    Proceeding to per-repo commit prompts."

   Pass C — Session-wide sub-repo scan (added May 18, 2026):

   Pass A is anchored on "known repos." Pass B is anchored on "files routed by
   this skill during Step 5." Both miss a real case: edits made anywhere in
   the session BEFORE the wrap-up phase that didn't pass through Step 5
   routing. Pass C is anchored on "git state at session end" — three different
   triggers, three different blind spots; all needed.

   For the canonical map of sub-repos and their tracking status, see
   ~/Documents/claude-projects/_meta/REPO_STRUCTURE.md "Per-Sub-Repo Git
   Tracking Status" section. Pass C uses dynamic discovery to stay robust
   against drift in that table — but the table is the human-readable
   architecture context (which repos are PUBLIC, which have remotes, etc.).

   Procedure:
   1. Enumerate every git repo in the ecosystem:
      find ~/Documents/claude-projects -maxdepth 5 -name '.git' -type d 2>/dev/null
      (also include ~/.claude and any other known repo roots — symlinked CLAUDE.md
      and REPO_STRUCTURE.md targets land in ~/.claude/workspace/)
   2. For each discovered sub-repo, cd in and run: git status --short
   3. For each repo with uncommitted changes (modified / deleted / untracked):
      ├── Show user a per-repo summary (path + file count + 1-line of what
      │   changed at file level)
      ├── Check repo visibility BEFORE prompting commit:
      │   gh repo view <remote> --json visibility -q .visibility
      │   If PUBLIC → flag explicitly and run PII-policy diff scan per
      │   ~/.claude/reference/public-repo-pii.md before committing
      ├── Prompt: "Commit and push [repo-name]? (Y/skip)"
      └── If Y → standard commit flow (HEREDOC commit message,
          Co-Authored-By, auto-push fires globally via core.hooksPath)
   4. Local-only repos (no remote per REPO_STRUCTURE.md) — commit locally with
      no push expectation; auto-push hook won't fire there.

   Why Pass C exists: substantive edits made during the session but outside
   Step 5 routing fall through Pass B's filter. Without Pass C, the user has
   to manually ask "did we commit in those repos?" to surface uncommitted
   work. Pass C closes that loop without depending on user prompting.

   Reconciliation with Pass A/B: union all three pass results, deduplicate by
   toplevel path. If Pass C surfaces a repo not in Pass A AND not in Pass B,
   that's a genuine session-wide-only edit — process per step 6's flow.

7. RECONCILE Pass A drift (runs AFTER all commits in step 6 succeed):
   Compare Pass B's discovered set against Pass A's hardcoded list.
   For each repo Pass B found that is NOT in Pass A, check the
   Pass-A-worthy criteria BEFORE prompting the user.

   Pass A exists to solve DISCOVERY problems that Pass B alone might miss
   or make implicit. It is NOT a registry of every project repo the user
   touches. A standard top-level project repo where files resolve cleanly
   via `git rev-parse --show-toplevel` is already handled by Pass B —
   adding it to Pass A duplicates coverage and bloats the list toward
   "every repo ever" until the list is meaningless.

   Pass-A-worthy criteria (at least ONE must be true to prompt for addition):

   a. **Nested repo** — the discovered repo is inside another git repo
      (e.g., `~/.claude/skills/learning-loop/` inside `~/.claude/`). Running
      `git status` in the parent does NOT surface the child. Pass B's
      `git rev-parse` catches this ONLY if a file under the child was
      touched this session; if routing doesn't happen to touch the child
      next session, the check silently skips. Pass A makes the nested
      dependency explicit.

   b. **Non-obvious or symlinked path** — the repo lives somewhere
      unexpected (e.g., symlinked in from outside the usual project tree,
      or at a path that looks like a file rather than a repo). Pass B
      finds it when files are written, but a human reading the SKILL.md
      wouldn't predict it's a destination.

   c. **Routing-named destination** — the repo is NAMED explicitly in the
      skill's routing rules (e.g., `claude-skills` appears in the routing
      table for skills-level learnings). Pass A documents the rule→repo
      mapping for readability, not for discovery.

   If NONE of a/b/c apply:
   └── DO NOT prompt. Log as "Pass B will continue to handle [repo]
       naturally — no Pass A entry needed (standard top-level project repo)."
       Move on.

   If AT LEAST ONE of a/b/c applies:
   └── Prompt the user:

   "Pass B discovered 1 repo not in Pass A's hardcoded list:
      - [toplevel path]
        Files touched: [file1, file2, ...]
        Routing context: [which destination added these files]
        Pass-A-worthy reason: [a / b / c with specifics]

    Add to Pass A? (Y/skip/one-off)

      Y       → edit SKILL.md to add the repo to Pass A with a dated note
                  (e.g. '- [path] (added [date]: [reason: nested/symlink/routing-named]')
      skip    → leave as-is; Pass B keeps handling discovery, but the
                  discovery risk flagged above remains
      one-off → don't prompt for this repo again this session (use for
                  temporary worktrees, migration scratch repos, or other
                  genuinely one-time destinations)"

   If Y:
   ├── Edit this SKILL.md file to add the repo under Pass A with a dated note
   ├── The SKILL.md edit is itself a file touched this session — it will be
   │   committed via the normal Step 6 flow on the NEXT wrap-up, since
   │   Pass A already covers ~/.claude/skills/learning-loop/
   └── Confirm: "Added [repo] to Pass A. Will be committed to learning-loop-skill
       next wrap-up."

   Why this criteria gate: without it, "add to Pass A?" becomes a reflexive
   "Y" for every new repo, and the list grows into a useless enumeration
   of every project destination. The value of Pass A is catching what Pass B
   misses or makes implicit — NOT documenting what Pass B already handles.
   Source: Apr 18 2026 session — I offered to add `openclaw-ops` to Pass A
   after it was Pass-B-discovered; user correctly pushed back ("why don't
   we add every single repo in there?"). Standard project repos resolve
   cleanly via git rev-parse — Pass B is enough. Pass A should be reserved.
```

**Why this exists:** Most local repos were set up for git-based safety (backup + rollback) but may lack .gitignore. Without a session-end prompt, operational doc changes accumulate uncommitted across sessions, losing the backup and history benefits. The .gitignore check ensures each repo only needs one-time setup — after that, commits are clean automatically.

### Sub-Agent Prompt Library (extracted v4.2 — read at dispatch time)

Each prompt below is verbatim v4.1 content, moved to `references/prompts/` so it loads only when its step dispatches. **At each dispatch step: Read the file, use everything below its header line as the sub-agent prompt, substitute bracketed placeholders.**

| Prompt | File | Dispatched by |
|---|---|---|
| SCANNER_PROMPT | `references/prompts/scanner-prompt.md` | Scan mode step 4; Wrap-up Step 1 |
| CONSOLIDATION_PROMPT | `references/prompts/consolidation-prompt.md` | Wrap-up Step 3 |
| TRIGGER_MOMENT_AUDITOR_PROMPT | `references/prompts/trigger-moment-auditor-prompt.md` | Wrap-up Step 3a (persona 1, runs first) |
| WORKFLOW_STEP_ROUTER_PROMPT | `references/prompts/workflow-step-router-prompt.md` | Wrap-up Step 3a (persona 2, receives Auditor output) |
| PLAN_DRAFTER_PROMPT | `references/prompts/plan-drafter-prompt.md` | Wrap-up Step 4b Mod 5 (child sub-agent, both gates met) |
| PHASE_1_DECISION_REPORT_PROMPT | `references/prompts/phase-1-decision-report-prompt.md` | Wrap-up Step 1b.5 (eval trigger met) |

**STOP — never improvise a replacement if a prompt file is missing.** A missing file means the skill install is broken; surface it to the user. The prompts carry accumulated STOP gates and rationalization counters (e.g., CONSOLIDATION_PROMPT's Resolution-vs-Increment and Test-Case Value checks) — an improvised prompt silently loses them, which is exactly the failure class this skill exists to prevent.

---

## Phase 1: Continuous Micro-Logging (Safety Net)

This phase runs continuously during the session as a background safety net, independent of explicit `/learning-loop` invocations.

### When to Log (Not Every Prompt)

Log ONLY after:
- Non-obvious debugging (more than 2-3 attempts)
- Error resolution where root cause wasn't immediately apparent
- Trial-and-error discovery
- User correction or pushback that changed approach

### The Evaluation

```
Internal check:
1. Did this require significant investigation? (>15 min equivalent)
2. Was the solution non-obvious?
3. Could future sessions benefit?

If all yes → Append one line to scratch file (silent)
If any no → Continue without logging
```

### Scratch File Mechanism

**DO NOT interrupt the user's flow.** Write silently via Bash echo append.

**First write in a session** (creates directory + header):
```bash
mkdir -p ~/.claude/learning-captures/[session-id] && echo "## Scratch Log (unverified micro-signals — input for scan/wrap-up, not a source of truth)\n---" > ~/.claude/learning-captures/[session-id]/scratch.md
```

**Subsequent writes** (append one-liner):
```bash
echo "[ISO-timestamp] [source] Signal summary | Key detail or corrective action" >> ~/.claude/learning-captures/[session-id]/scratch.md
```

### Source Tags

| Tag | Meaning | Example |
|-----|---------|---------|
| `[self]` | Claude's own mistake or discovery | `[self] Tried WebFetch on X URL, blocked \| Always use Playwright MCP for social media` |
| `[user]` | User correction or pushback | `[user] Corrected: got spouse's name wrong \| Verify personal names before persisting` |
| `[env]` | Tooling or environment surprise | `[env] Playwright MCP doesn't support PDF export \| Use sips or wkhtmltopdf instead` |

### Scratch File Rules

1. **One line per signal** — no multi-line entries. Format: `[timestamp] [source] summary | detail`
2. **Silent execution** — no announcement to user, no confirmation message
3. **Not a source of truth** — scratch lines are *input* for Scan/Wrap-up, not verified learnings
4. **Survives compaction** — the entire point; when context is compacted, scratch.md preserves the specifics
5. **Use Bash echo** as primary mechanism. Fall back to Write tool only if echo fails.
6. **Session-id** = use a short identifier for the current session (date + brief context)

---

## Code-Level Capture: User-Invoked, Not Orchestrated

Code-level learnings (codebase-specific bugs, fix-and-confirm moments) belong in `/ce-compound`, which produces schema-validated `docs/solutions/` entries via a 7-agent flow. **Learning-loop does NOT orchestrate `/ce-compound`** — code-level capture wants peak-fresh context, which means invoking `/ce-compound` mid-session right after the fix is confirmed, not at wrap-up.

Wrap-up's role for code-level: if a confirmed code-level fix appears in scan signals but `/ce-compound` was not invoked during the session, surface it in the Zone-1 user-attention block with the prompt: *"Worth a delayed `/ce-compound` while context is still warm?"* — but treat it as a one-line nudge, not orchestration.

---

## Post-Clear Recovery

### Hook Configuration (in ~/.claude/settings.json)

```json
{
  "hooks": {
    "SessionStart": [
      {
        "matcher": "clear",
        "hooks": [
          {
            "type": "command",
            "command": "if [ -d ~/.claude/learning-captures ] && [ \"$(ls -A ~/.claude/learning-captures 2>/dev/null)\" ]; then echo 'LEARNING_CAPTURES_EXIST: Found learning captures from before clear. Ask user if they want to review/consolidate before continuing.'; fi"
          }
        ]
      }
    ]
  }
}
```

### When Claude Sees "LEARNING_CAPTURES_EXIST"

```
"I see there are learning captures from before the clear. These contain
raw signals that were preserved before context was lost.

Want me to:
1. Review and consolidate them now (runs wrap-up mode)
2. Leave them for later
3. Delete them (if no longer relevant)"
```

---

## Quality Gates (Type-Specific)

Quality gates apply during **Wrap-up consolidation**, not during Scan mode. Scans capture raw signals without filtering.

### Universal Gates (Apply to All Types)

**Gate 1: Reusability**
> "Would this help in a future session with a similar *situation*?"

- ✅ Pattern applies beyond this specific instance
- ❌ Solution only works for this exact file/config/moment

**Gate 2: Non-Triviality**
> "Did this require genuine discovery (not just obvious or already documented)?"

- ✅ Required investigation, trial-and-error, or hypothesis testing
- ❌ Answer was in first Google result or official docs

### Type-Specific Gates

| Type | Gate 3: Specificity | Gate 4: Validation |
|------|---------------------|-------------------|
| **Code-level** | Can I define exact trigger conditions (error messages, symptoms)? | Did I confirm the fix works? |
| **Process-level** | Can I describe the observable situation that triggers this rule? | Did we experience the consequence of *not* following this? |
| **Content-level** | Can I articulate what shifted in my worldview or judgment framework? | Could this become published content — does it contradict or refine a prior belief about the world (not just about how to work)? |
| **Fact** | Can I verify this against conversation evidence? | Is this worth persisting across sessions? |

**⚠️ Common misclassification:** A learning about content *operations* (editorial rules, scheduling, platform strategy) is **process-level**, not content-level. It routes to Content Lab CLAUDE.md or its playbooks, not the Judgment Ledger. Content-level means your understanding of the *world* shifted — not your understanding of *how to do content work*. **This includes content STRATEGY insights** — "what types of posts generate what types of engagement" is operationally valuable but is about content mechanics, not AI capability meeting reality. Route to performance_log patterns or editorial strategy.

**If a signal fails any gate → Don't extract. Note why in consolidation output for review.**

### Gate 5: Significance Threshold (Applied After Gates 1-4 Pass)

Gates 1-4 are pass/fail on quality. Gate 5 determines WHETHER to persist — not just WHERE.

> Ask: "If this observation were lost after this session, what's the consequence?"

| Consequence | Action |
|------------|--------|
| Claude would make the **same mistake** again | Persist → route to destination |
| A workflow step would **execute in the wrong order** | Persist → route to destination |
| A fact would need to be **re-discovered** | Persist → Memory |
| **Nothing** — it's an interesting observation | → Noted (don't persist) |

**The bar:** "Would a future session go WRONG without this?" not "Is this interesting?"

### Behavioral vs. Operational Split (Process-Level Only)

When a process-level conclusion passes all 5 gates, apply one more split:

> "Does this change how Claude makes DECISIONS, or how Claude executes a PROCEDURE?"

| Answer | Sub-type | Example |
|--------|----------|---------|
| Changes decisions | Process-level (behavioral) | "Develop when you have lived material" |
| Changes procedure execution | Process-level (operational) | "Reactive posts before evergreen in scheduling" |

**Operational routing adapts to repo infrastructure:**
1. Does the project have dedicated operational docs (playbooks/, OPERATIONS_PLAYBOOK, etc.)? → Route there
2. No dedicated docs? Significant enough for CLAUDE.md? → Add to project CLAUDE.md under operational section
3. Not significant enough for CLAUDE.md? → Memory as operational note, or Noted/drop

---

## Process-Level Consolidation Discipline

⚠️ **CONSOLIDATION REQUIRED:** Before adding to any CLAUDE.md:

1. **Read the entire document first** — understand what's already there
2. **Search for similar rules** — does this concept already exist in some form?
3. **If similar exists → Merge/strengthen** the existing section instead of adding a duplicate
4. **If truly new → Add in the appropriate section** with proper context
5. **Consider pruning** — if the document is getting long, can older rules be consolidated?

### Scenario QA for New Rules (MANDATORY after writing process-level rules)

> **Why this exists (Content Lab, Feb 6 2026):** Rules that exist but aren't surfaced at the point of execution are dead code paths in your process.

After writing any new rule to CLAUDE.md:
1. Generate 3-5 scenarios where this rule should trigger
2. For each scenario, trace the workflow — would Claude encounter the rule before taking the wrong action?
3. Identify gaps and present fixes to the user

### Template for Process-Level Entries

```markdown
### [Section Name]

[Rule in imperative form]

> **Why this exists ([Project], [Date]):** [Brief context]
> **Trigger:** [When this rule applies — specific observable situation]

**STOP and correct if you're:**
- [Warning sign 1]
- [Warning sign 2]
```

### Content-Level → Judgment Ledger Template

```markdown
## [Date] - [Project/Context]: [Title]

**Core mistake:**
[What assumption or approach was wrong]

**Core realization:**
[What you now understand differently]

**Reusable principle:**
[Decision rules that apply beyond this project]

**What must stay human:**
[Judgment that couldn't be automated]

**What AI helped with:**
[Where AI added value]

**Trigger for future reference:**
[Observable situation that should remind you of this insight]

**Classification:** [Local / Pattern / Principle]

**Content potential:** [Low / Medium / High] — [Rationale]

**Content Development Status:** [Pending / Not applicable]
```

⚠️ For **Pattern/Principle** insights: Write to Judgment Ledger with "Content Development Status: Pending". **DO NOT** prompt to run content development inline. Mention at wrap-up that flagged insights exist. Content development happens in a dedicated session.

---

## Learning Classification & Routing

| Type | Definition | Handler | Destination |
|------|------------|---------|-------------|
| **Code-level** | Specific to codebase/framework | User-invoked `/ce-compound` mid-session (not learning-loop) | `docs/solutions/` with schema-validated YAML |
| **Process-level (behavioral)** | Changes decision-making across sessions | Learning-loop direct | CLAUDE.md (root or project) with trigger + warning signs |
| **Process-level (operational)** | Changes procedure execution in a workflow | Learning-loop direct | Project operational docs* or CLAUDE.md |
| **Fact** | Pure recall, no behavior change | Learning-loop direct | Memory MEMORY.md |
| **Content-level** | Publishable insight, understanding shifted | Learning-loop direct | Judgment Ledger with future trigger |
| **Noted** | Interesting but below persistence threshold | Acknowledged in summary | Not persisted |

*Operational docs = playbooks/ if project has them, otherwise project CLAUDE.md or Memory.

---

## Multi-Session Flow

```
Session 1: Working on feature...
  [context getting long]
  User: /learning-loop           → detects "before I clear" → Scan mode
  → Sub-agent captures raw signals to scan-001.md
  [compaction happens, work continues...]

Session 2: Still working...
  User: /learning-loop scan      → explicit scan mode
  → Sub-agent captures raw signals to scan-001.md (new session dir)
  [compaction happens]

Session 3: Finally done!
  User: /learning-loop wrap up   → explicit wrap-up mode
    1. Scan THIS session (captures final signals)
    2. Triage: show this session's captures + surface sessions 1-2 as "other sessions found"
       → User opts in to include session 2 (same project), skips session 1 (different topic)
    3. Consolidate approved captures (session 2 + 3)
    4. Resolve hypotheses → draw conclusions → apply quality gates
    5. Present summary → user verifies
    6. Route to destinations → clean up sessions 2 + 3 (session 1 preserved)
```

---

## Version History

Moved to `references/CHANGELOG.md` (v4.2 restructure). Newest-first: v4.2 structural extraction, v4.1 W7.w STOP gates, v4.0 Mods 6-10, v3.x lineage back to v1. Per-change reasoning: SESSION_LOG.md.

---

## Design Principles

| Principle | How It's Implemented |
|-----------|---------------------|
| **Explicit invocation, deterministic behavior** | Description-based matching was non-deterministic (asymmetric failure); `/learning-loop` is deterministic and can't be intercepted |
| **Scans are raw, wrap-up draws conclusions** | Mid-session hypotheses resolved at session end with hindsight |
| **Smart default, explicit override** | Context clue detection with fallback to asking |
| **Memory is a routing destination** | Facts route to MEMORY.md; behavioral changes route to CLAUDE.md |
| **User-invoked, not orchestrated** | `/ce-compound` is invoked directly by the user mid-session; learning-loop wrap-up surfaces a one-line nudge if a code-level fix was missed, but does not auto-invoke |
| **Consolidation over accumulation** | CLAUDE.md edits require reading and merging, not just appending |
| **Persistence over memory** | Scratch lines and scan files survive compaction; mental notes don't |
| **Resilience over rigidity** | Complements auto-memory, adapts when system behaviors change |
| **Every gate enumerates the states it can observe** | A gate with no branch for a legitimate state teaches sessions to route around it — and a routed-around gate is indistinguishable from a working one. See *Gate-state enumeration* below. |


### Gate-state enumeration — a standing check when adding or editing ANY gate in this skill

**Three gates in this skill have now been found missing a branch for a legitimate case:**

| Gate | The legitimate state it had no branch for | Fixed in |
|---|---|---|
| Step 1b.5 Phase-1 evaluation trigger | its program had concluded, so all its trigger conditions were permanently true and could never again be false | v4.3 |
| End-of-wrap-up cleanup gate | a directory **deliberately retained** because a routing decision was still open | v4.5 |
| Step 2 triage + Step 6a-ii + the cleanup gate | a directory a **concurrent session still owned and was writing to** | v4.7 |

In every instance the gate fired **correctly by its own logic**, and the burden of not acting on it fell to **each session's private judgment, silently.** A gate with no branch for the legitimate case teaches sessions to route around it — and **a routed-around gate is indistinguishable from a working one**, so the defect is invisible from the outside and the gate keeps reporting success.

When adding or editing any gate here:

1. **Enumerate every state the gate can OBSERVE** — not the states you designed it for.
2. For each, ask: **is this state legitimate?** Would a careful session want the gate not to fire here?
3. **Every legitimate state gets its own branch and its own terminal disposition.** "Use judgment" is not a branch; it is the absence of one.
4. Where a legitimate state has no cheap branch, **say so inside the gate**, so routing around it is a documented act rather than a private one.

> **The failure is never the state you designed for.**


---

## The Meta-Principle

> **"The human shouldn't need to remember."**

This is the foundation. Everything else supports it:

- **Explicit invocation** → You don't need to remember trigger phrases; `/learning-loop` is unambiguous
- **Scan mode** → You don't need to remember to document before compaction; one command captures everything
- **Wrap-up mode** → You don't need to remember what happened across sessions; captures persist
- **Smart routing** → You don't need to remember where each learning type goes
- **Quality gates on conclusions** → You don't need to worry about capturing too much mid-session

v2 added: **"...and the system should be able to find what it learned."**
v2.1 added: **"...and nothing is lost between the moment of learning and the moment of capture."**
v3 added: **"...and the system adapts to its environment rather than fighting it."**

The orchestration exists so you can focus on the work. Learning-loop handles the meta-work of ensuring nothing valuable is lost.
