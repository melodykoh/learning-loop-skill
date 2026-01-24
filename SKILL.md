---
name: learning-loop
description: Orchestration layer that monitors sessions for learning signals, prompts for /workflows:compound when appropriate, and ensures no valuable learning is lost to compaction
version: 2.0.0
allowed-tools:
  - Task # Spawn sub-agent for scanning
  - Read # Access capture files, transcripts
  - Write # Create capture files, judgment entries
  - Edit # Update CLAUDE.md files
  - Bash # File operations, directory creation
  - Grep # Search existing docs
  - Glob # Find existing solution files
  - Skill # Invoke /workflows:compound for code-level learnings
triggers:
  # All triggers are USER-INITIATED (Claude does not receive system signals for context limits)
  - "let's wrap up" / "wrap up the session"
  - "run a capture" / "capture learnings"
  - "before I clear" / "going to clear"
  - After /clear (via SessionStart hook reminder)
  - After significant debugging/investigation (code-level check — user can request capture anytime)
---

# learning-loop Skill v2

**Purpose:** Orchestrate learning capture across a session — ensuring `/workflows:compound` runs when it should, and nothing valuable is lost to compaction or `/clear`.

---

## ⚠️ MANDATORY PROCEDURE (Read Before Anything Else)

**When user says "capture" / "capture first" / "run a capture":**

1. **DO NOT** do capture work in main conversation — this wastes context
2. **SPAWN** a Task agent using the SCANNER_PROMPT from Phase 3
3. Sub-agent writes to `~/.claude/learning-captures/[session-id]/capture-NNN.md`
4. Main conversation waits for completion, then confirms capture is done

**STOP:** If you're about to scan for signals yourself in the main conversation, you're doing it wrong. Spawn a sub-agent.

**When user says "wrap up" / "consolidate":**
- THEN you can read capture files and route to destinations (Judgment Ledger, CLAUDE.md)
- This is consolidation, not capture — different phase

---

## Core Insight

> **"The human shouldn't need to remember."**

Context compaction and `/clear` destroy details. Files persist. This skill ensures:
1. **Learnings are captured** before compaction erases them
2. **The right tools get invoked** (`/workflows:compound` for code-level, direct edits for process/content)
3. **Nothing falls through the cracks** — even when you forget to document

## The Problem This Solves

> "Sometimes I remember to run `/workflows:compound`, and sometimes I forget before context diminishes or I just clear without thinking."

Learning-loop sits at a **higher level** and decides:
- What has happened this session?
- Is it worth capturing?
- Should `/workflows:compound` run now (while context is fresh)?
- What needs to be preserved before compaction?

## The Orchestration Model

```
┌─────────────────────────────────────────────────────────────┐
│                    LEARNING-LOOP (Orchestrator)             │
│                                                             │
│  Monitors session → Detects signals → Routes appropriately  │
└─────────────────────────────────────────────────────────────┘
                              │
            ┌─────────────────┼─────────────────┐
            ▼                 ▼                 ▼
     ┌──────────┐      ┌──────────┐      ┌──────────┐
     │Code-level│      │Process-  │      │Content-  │
     │          │      │level     │      │level     │
     └────┬─────┘      └────┬─────┘      └────┬─────┘
          │                 │                 │
          ▼                 ▼                 ▼
   /workflows:compound   CLAUDE.md      Judgment Ledger
   (deep documentation)  (direct edit)  (direct write)
```

**Learning-loop doesn't replace `/workflows:compound` — it ensures it gets run.**

---

## Code-Level Orchestration: When to Prompt for /workflows:compound

For **code-level learnings**, learning-loop's job is to prompt you to run `/workflows:compound` while context is still fresh.

### The Decision Flow

```
Code-level fix just confirmed working
         │
         ▼
    Has this content been compacted yet?
         │
    ┌────┴────┐
    │         │
    NO        YES
    │         │
    ▼         ▼
PROMPT IMMEDIATELY:              TOO LATE for full treatment:
"Want to run /workflows:compound Save detailed signal to capture file
while details are fresh?"        for manual documentation later
    │
    ├── User: "Yes" → Invoke /workflows:compound (7 agents, schema-validated)
    │
    └── User: "No/Later" → Capture signal, remind at wrap-up
```

### Timing Matters

| Trigger | Action |
|---------|--------|
| **Code-level fix just confirmed** | Prompt for `/workflows:compound` immediately (context freshest) |
| **User requests capture** ("run a capture", "before I clear") | "Any undocumented code-level fixes? Last chance for `/workflows:compound`" |
| **After compaction** | Too late for multi-agent treatment; capture what you can for manual documentation |

### Why Prompt Immediately?

`/workflows:compound` launches **7 parallel subagents** that inherit the current conversation context:
- Context Analyzer
- Solution Extractor
- Related Docs Finder
- Prevention Strategist
- Category Classifier
- Documentation Writer
- Specialized Agents (based on problem type)

If context has already been compacted, these subagents inherit a **summary**, not the details. They won't have what they need to do deep documentation.

**The best time to run `/workflows:compound` is right after a fix is confirmed working.**

### The Prompt

After detecting a code-level fix that passes quality gates:

```
"That fix involved some non-trivial investigation. Want me to run
/workflows:compound now to document it properly?

This is best done while details are fresh — the multi-agent
documentation will capture root cause, prevention strategies,
and cross-references."
```

### If User Declines

Capture enough signal in the learning-loop file that:
1. You can manually document later
2. Or run `/workflows:compound` in a future session with the capture file as context

---

## What's New in v2

| Enhancement | Why It Matters |
|-------------|----------------|
| **Explicit orchestration role** | Clear that learning-loop prompts for `/workflows:compound`, doesn't duplicate it |
| **Type-specific quality gates** | Different validation criteria for code/process/content learnings |
| **User-initiated triggers** | Reliable capture model based on user phrases ("run a capture", "wrap up") rather than unreliable context detection |
| **Trigger-condition descriptions** | Process/content learnings formatted for future retrieval |
| **Safety net capture** | When `/workflows:compound` can't run, preserve enough signal for later |

---

## Architecture Overview

```
Session Start
     │
     ├──► Check for existing captures (post-clear safety net)
     │
     ▼
[Work happens...]
     │
     ├──► LIGHTWEIGHT EVALUATION ◄─── After significant debugging
     │    "Did this investigation produce non-obvious knowledge?"
     │    If yes → Queue for capture (don't interrupt flow)
     │
     ▼
User sees context warning OR wants to preserve learnings
     │
     ▼
User requests capture ◄──── PRIMARY TRIGGER (user-initiated)
  ("run a capture", "capture learnings", "wrap up", "before I clear")
     │
     ▼
Sub-agent spawns (sees full context)
Scans for learning signals
Applies QUALITY GATES
Writes to: ~/.claude/learning-captures/[session-id]/capture-001.md
     │
     ▼
Compaction happens (if user proceeds)
Details lost from context
BUT preserved in capture file
     │
     ▼
[User can request additional captures as needed]
     │
     ▼
Session wrap-up ("let's wrap up")
     │
     ▼
Consolidate all capture files
WEB RESEARCH for code-level learnings
Route to proper destinations with SEMANTIC DESCRIPTIONS
Clean up capture directory
```

### Why Triggers Are User-Initiated

**Key insight from investigation:**
- Claude does NOT receive a system signal when context is low
- PreCompact hooks cannot spawn sub-agents (ruled out in V1 design)
- The only reliable triggers are user-initiated phrases

**What this means in practice:**
- When user sees "Context is getting long" warning, they can say "run a capture"
- Before user runs `/clear` or `/compact`, they can request capture
- At session end, "wrap up" triggers the consolidation process

---

## Quality Gates (Type-Specific)

Before extracting any learning, verify it passes the appropriate gates for its type.

### Universal Gates (Apply to All Types)

**Gate 1: Reusability**
> "Would this help in a future session with a similar *situation*?"

- ✅ Pattern applies beyond this specific instance
- ❌ Solution only works for this exact file/config/moment

**Gate 2: Non-Triviality**
> "Did this require genuine discovery (not just obvious or already documented)?"

- ✅ Required investigation, trial-and-error, or hypothesis testing
- ❌ Answer was in first Google result or official docs

---

### Type-Specific Gates

| Type | Gate 3: Specificity | Gate 4: Validation |
|------|---------------------|-------------------|
| **Code-level** | Can I define exact trigger conditions (error messages, symptoms)? | Did I confirm the fix works? |
| **Process-level** | Can I describe the observable situation that triggers this rule? | Did we experience the consequence of *not* following this? |
| **Content-level** | Can I articulate what changed in my understanding? | Does this contradict or refine something I previously believed? |

---

### Gate Details by Type

**Code-Level Gates:**
- **Specificity:** Exact error messages, stack traces, observable symptoms
- **Validation:** Fix was tested and confirmed working

**Process-Level Gates:**
- **Specificity:** Observable situation that triggers the rule (e.g., "when modifying existing functions", "when user pushes back")
- **Validation:** We experienced the failure that makes this rule necessary — this isn't theoretical

**Content-Level Gates:**
- **Specificity:** Can articulate the before/after of understanding — what shifted?
- **Validation:** This contradicts or significantly refines a prior belief — not just confirming what we already thought

---

**If a signal fails any gate → Don't extract. Note why in capture file for review.**

---

## Learning Signals to Detect

| Signal Type | Patterns | Significance | Gate Risk |
|-------------|----------|--------------|-----------|
| **Direction Changes** | "actually let's try...", pivots | High | Check reusability |
| **Failed Attempts** | errors, "that didn't work" | Medium | Ensure verified fix exists |
| **Discoveries** | "oh!", "turns out", root cause | High | Usually passes |
| **User Pushback** | corrections, disagreements | High | Usually passes |
| **Hypothesis Validation** | confirmed/disproven | High | Usually passes |
| **Decisions w/ Tradeoffs** | "went with X because" | Medium | Check non-triviality |
| **Process Observations** | "we should always" | High | Usually passes |

---

## Phase 1: Lightweight Continuous Evaluation (NEW)

### When to Evaluate (Not Every Prompt)

Evaluate ONLY after:
- Non-obvious debugging (more than 2-3 attempts)
- Error resolution where root cause wasn't immediately apparent
- Trial-and-error discovery
- User correction or pushback that changed approach

### The Evaluation (Mental, Not Interrupting)

```
Internal check:
1. Did this require significant investigation? (>15 min equivalent)
2. Was the solution non-obvious?
3. Could future sessions benefit?

If all yes → Flag for capture at next opportunity
If any no → Continue without flagging
```

**DO NOT interrupt the user's flow.** Just mentally queue for capture.

---

## Phase 2: Responding to User Capture Requests

### Trigger Phrases

User initiates capture with phrases like:
- **"run a capture"** / **"capture learnings"** — explicit capture request
- **"before I clear"** / **"going to clear"** — pre-clear preservation
- **"wrap up"** / **"let's wrap up"** — session-end consolidation

### When User Requests a Capture

When user says any trigger phrase, Claude responds:

```
"I'll scan the session for learning signals. Based on what I can see,
there are [N] potentially valuable signals, including [brief summary
of highest-value one].

Running capture now — this preserves details that would otherwise be
lost to compaction, formatted for easy retrieval in future sessions."
```

### Capture Process

1. Create session directory:
   ```bash
   mkdir -p ~/.claude/learning-captures/[session-id]
   ```

2. Spawn scanning sub-agent (see Phase 3)

3. After capture completes:
   ```
   "Captured [N] signals to capture file.
   - [X] passed quality gates
   - [Y] flagged for review (didn't meet all gates)

   Safe to continue or compact. I'll consolidate at session wrap-up."
   ```

### Reminder: User Drives the Timing

Claude does NOT proactively detect context limits. If user sees a context warning and wants to preserve learnings, they should say "run a capture" before continuing or compacting.

---

## Phase 3: Sub-Agent Scanning (Enhanced)

### SCANNER_PROMPT (Updated)

```
You have access to the full conversation context. Your job is to identify
learning signals, apply quality gates, and write them to a capture file.

SCAN FOR THESE SIGNALS (in order of importance):
1. User pushback - corrections, disagreements
2. Direction changes - pivots, abandoned approaches
3. Discoveries - realizations, root causes found
4. Failed attempts - what was tried and why it failed
5. Decisions with tradeoffs - choices made and reasoning
6. Process observations - meta-learnings

FOR EACH SIGNAL:

1. Classify the signal type: Code-level / Process-level / Content-level

2. Apply Quality Gates (type-specific):

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

3. If PASSES all gates, extract with TRIGGER-CONDITION DESCRIPTION:
   - What error message or symptom triggers this?
   - What context/environment is this relevant for?
   - What's the solution in one sentence?

3. If FAILS any gate:
   - Note which gate failed and why
   - Include anyway but mark as "REVIEW NEEDED"

WRITE OUTPUT using this format:

---
captured: [ISO timestamp]
session_id: [from path]
context_state: [Full / Partial]
signals_found: [total]
signals_passed_gates: [count]
signals_need_review: [count]
---

## Signals Captured

### 1. [Signal Type]: [Brief Title]
**Gate Status:** ✅ PASSED / ⚠️ REVIEW NEEDED ([which gate failed])

**Trigger Conditions:**
- Error: "[Exact error message if applicable]"
- Symptom: [Observable behavior]
- Context: [Framework, environment, situation]

**Quote:** "[Relevant quote from conversation]"

**Solution Summary:** [One sentence]

**Classification:** [Code-level / Process-level / Content-level]

**Notes:** [Additional context that might be lost]

---

### 2. [Signal Type]: [Brief Title]
...

---

## Summary
- Total signals: [N]
- Passed quality gates: [X]
- Need review: [Y]
- Recommended for documentation: [list with classifications]
```

---

## Phase 4: Session Wrap-Up Consolidation (Enhanced)

### Process

1. **Check for capture files:**
   ```bash
   ls ~/.claude/learning-captures/*/capture-*.md 2>/dev/null
   ```

2. **If captures exist, consolidate and present:**

```
## Session Learning Signals (Consolidated)

Across [N] captures, I found these signals:

### Ready for Documentation (Passed All Gates)

| # | Type | Classification | Trigger Condition Preview |
|---|------|----------------|--------------------------|
| 1 | Discovery | Code-level | "P2024: Timed out fetching connection" |
| 2 | User pushback | Process-level | Hypothesis testing before fixes |

### Needs Review (Failed Gate)

| # | Type | Failed Gate | Reason |
|---|------|-------------|--------|
| 3 | Failed attempt | Verification | Never confirmed fix works |

What should we do?
1. Document all that passed gates
2. Review flagged signals first
3. Select specific ones
4. Skip documentation this session
```

3. **For Code-Level Learnings: Invoke /workflows:compound**

Code-level learnings get the full `/workflows:compound` treatment:

```
"I found [N] code-level signals that passed quality gates.
Want me to run /workflows:compound for each?

This will invoke the multi-agent documentation system for:
- Root cause analysis
- Prevention strategies
- Cross-references
- Schema-validated output to docs/solutions/
```

If context is too low for `/workflows:compound`, fall back to manual capture with enough detail for later documentation.

4. **For Process-Level and Content-Level: Direct Documentation**

These types don't use `/workflows:compound` — learning-loop handles them directly with trigger-condition-rich descriptions:

**Process-level → Root CLAUDE.md**

⚠️ **CONSOLIDATION REQUIRED:** Before adding to root CLAUDE.md:

1. **Read the entire document first** — understand what's already there
2. **Search for similar rules** — does this concept already exist in some form?
3. **If similar exists → Merge/strengthen** the existing section instead of adding a duplicate
4. **If truly new → Add in the appropriate section** with proper context
5. **Consider pruning** — if the document is getting long, can older rules be consolidated?

The goal is a **streamlined, scannable document** — not a changelog of every learning.

**Template for new process-level entries:**

```markdown
### [Section Name]

[Rule in imperative form]

> **Why this exists ([Project], [Date]):** [Brief context]
> **Trigger:** [When this rule applies — specific observable situation]

**STOP and correct if you're:**
- [Warning sign 1]
- [Warning sign 2]
```

**Content-level → Judgment Ledger**

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
← Add "Pending" for Pattern/Principle insights; "Not applicable" for Local
```

⚠️ **Pattern/Principle Insights: Capture and Defer**

For insights classified as **Pattern** or **Principle**:
- Write to Judgment Ledger with "Content Development Status: Pending"
- **DO NOT prompt to run content development inline** — this interrupts session flow
- Mention at wrap-up that flagged insights exist
- Content development happens in a **dedicated Content Development Session** (per content-lab methodology)

Learning-loop's job is capture and flag, not content creation.

5. **Clean up capture files** after documentation confirmed

6. **Mention flagged Pattern/Principle insights**

If any content-level insights were classified as Pattern or Principle:

```
"Note: You have [N] Pattern/Principle insight(s) flagged for content development
in the Judgment Ledger. These are ready for a dedicated Content Development Session
when you have time — they won't be processed in this session."
```

**Do NOT offer to process them now.** Content development requires dedicated thinking time.

---

## Phase 5: Post-Clear Recovery

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
            "command": "if [ -d ~/.claude/learning-captures ] && [ \"$(ls -A ~/.claude/learning-captures 2>/dev/null)\" ]; then echo 'LEARNING_CAPTURES_EXIST: Found learning captures. Ask user if they want to review/consolidate.'; fi"
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
signals that were preserved before context was lost.

Want me to:
1. Review and consolidate them now
2. Leave them for later
3. Delete them (if no longer relevant)
```

---

## Learning Classification & Routing

| Type | Definition | Handler | Destination |
|------|------------|---------|-------------|
| **Code-level** | Specific to codebase/framework | `/workflows:compound` (multi-agent) | `docs/solutions/` with schema-validated YAML |
| **Process-level** | How to work better | Learning-loop direct | Root CLAUDE.md with trigger + warning signs |
| **Content-level** | Publishable insight | Learning-loop direct | Judgment Ledger with future trigger |

---

## Semantic Retrieval Optimization (Process & Content Only)

> **Note:** This section applies to **process-level and content-level** learnings only. Code-level learnings use `/workflows:compound`, which already produces schema-validated, semantically-rich documentation.

### Why This Matters

The goal isn't just to *store* learnings — it's for future sessions to *find* them.

For process and content learnings that learning-loop writes directly, use trigger-condition-rich descriptions:

**Bad:**
> "Remember to test before committing"

**Good (observable trigger):**
> "When modifying existing functions: Read the entire implementation first and check git history before changing"

### Description Formula for Process-Level

```
[When/Observable situation] + [What to do] + [Why/Consequence avoided]
```

### Description Formula for Content-Level

```
[What shifted in understanding] + [Observable trigger for future reference]
```

---

## The Feedback Loop (Updated)

```
Session in progress → Learning signal detected →
     │
     ▼
Apply type-specific quality gates →
     │
     ▼
Code-level fix?
     │
     ├─ YES + Context fresh → Prompt for /workflows:compound NOW
     │                        (7 agents, schema-validated docs/solutions/)
     │
     └─ YES + Context low → Capture signal for later
     │
     ▼
Process/Content-level? → Capture signal for wrap-up
     │
     ▼
Session wrap-up →
     │
     ├─ Code-level → Run /workflows:compound (if not already done)
     ├─ Process-level → Edit CLAUDE.md directly (with consolidation)
     └─ Content-level → Write Judgment Ledger entry
          │
          └─ If Pattern/Principle → Flag as "Content Development: Pending"
                                    Mention at wrap-up, DO NOT process inline
     │
     ▼
Pattern/Principle insights wait for dedicated Content Development Session
     │
     ▼
Next session starts →
     │
     ├─ Check docs/solutions/ for known errors (schema-validated, searchable)
     ├─ CLAUDE.md rules have observable triggers
     └─ Judgment Ledger has flagged insights ready for content development
     │
     ▼
Mistake avoided because learning was FINDABLE
Content developed when there's dedicated time (not mid-session)
```

---

## Design Principles

| Principle | How It's Implemented |
|-----------|---------------------|
| **Orchestration over duplication** | Prompts for `/workflows:compound` instead of reimplementing code-level documentation |
| **Type-specific quality** | Different validation criteria for code/process/content learnings |
| **Trigger-rich descriptions** | Process and content learnings formatted for future retrieval |
| **Immediate prompting** | Code-level fixes prompt for documentation while context is fresh |
| **Consolidation over accumulation** | CLAUDE.md edits require reading and merging, not just appending |
| **Safety net capture** | Pre-compaction capture ensures nothing is lost even if documentation is deferred |

---

## The Meta-Principle

> **"The human shouldn't need to remember."**

This is the foundation. Everything else supports it:

- **Pre-compaction capture** → You don't need to remember to document before context is lost
- **Prompting for `/workflows:compound`** → You don't need to remember to run it
- **Type-specific routing** → You don't need to remember where each learning type goes
- **Trigger-condition descriptions** → Future sessions can find learnings without you remembering they exist

v2 adds: **"...and the system should be able to find what it learned."**

The orchestration exists so you can focus on the work. Learning-loop handles the meta-work of ensuring nothing valuable is lost.
