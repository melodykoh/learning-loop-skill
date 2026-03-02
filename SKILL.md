---
name: learning-loop
description: Two-mode learning system — raw signal scanning before compaction, quality-gated consolidation at session end. Invoke with /learning-loop.
version: 3.1.0
allowed-tools:
  - Task
  - Read
  - Write
  - Edit
  - Bash
  - Grep
  - Glob
  - Skill
---

# learning-loop Skill v3.1

**Purpose:** Two-mode learning capture — raw signal scanning mid-session, quality-gated consolidation at session end. Ensures `/workflows:compound` runs when it should, and nothing valuable is lost to compaction or `/clear`.

---

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
2. **SPAWN** a Task agent using the SCANNER_PROMPT
3. Sub-agent writes to `~/.claude/learning-captures/[session-id]/scan-NNN.md`
4. Main conversation waits for completion, then confirms capture is done

**STOP:** If you're about to scan for signals yourself in the main conversation, you're doing it wrong. Spawn a sub-agent.

### User Verification (Critical)
**Before routing ANY learning to a destination:**
1. **Present a summary** of captured/consolidated signals to the user
2. **Explicitly ask for verification** — "Does this accurately reflect what happened?"
3. **Wait for user confirmation** before routing to CLAUDE.md, Judgment Ledger, Memory, or running `/workflows:compound`

> **Why this exists (Jan 29, 2026):** AI-generated captures can contain hallucinations — wrong names, fabricated premises, misremembered details. A capture once referenced "Henry" when the user's husband is "Ted" and claimed constraints that didn't exist.

**DO NOT** update any destination based on captures without user sign-off.

---

## Core Insight

> **"The human shouldn't need to remember."**

Context compaction and `/clear` destroy details. Files persist. This skill ensures:
1. **Learnings are captured** before compaction erases them
2. **The right tools get invoked** (`/workflows:compound` for code-level, direct edits for process/content)
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
| **Output** | `MEMORY.md` entries | Routed to 5 destinations based on type |
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

4. Spawn scanning sub-agent with SCANNER_PROMPT (see below)

5. After capture completes, confirm:
   ```
   "Scan complete — captured [N] raw signals to scan-[NNN].md.
   These are unresolved observations, not conclusions.
   Safe to continue or compact. I'll consolidate everything at wrap-up."
   ```

### SCANNER_PROMPT (Raw Signal Mode)

```
You have access to the full conversation context. Your job is to identify
RAW LEARNING SIGNALS — unresolved observations, not conclusions.

FIRST: Check for scratch file at ~/.claude/learning-captures/[session-id]/scratch.md
If it exists, read it. Each line is an unverified micro-signal logged in real-time.
Cross-reference against conversation context:
- If confirmed → incorporate into your signal list
- If contradicted or unverifiable → discard

SCAN FOR THESE RAW SIGNALS:
1. "Tried X, didn't work because Y" — failed attempts with reasons
2. "User corrected assumption about Z" — pushback or corrections
3. "Hypothesis: root cause might be..." — unresolved hypotheses (mark as UNRESOLVED)
4. "Direction change: pivoted from A to B" — approach changes with reasons
5. "Discovery: turns out X works because Y" — findings (may or may not be confirmed)
6. "Process observation: we should always..." — meta-observations about workflow

FOR EACH SIGNAL:
- Capture the raw observation, not a conclusion
- Mark hypotheses as UNRESOLVED (wrap-up will resolve them with hindsight)
- Include enough context to be useful even after compaction
- Quote relevant conversation excerpts where possible

DO NOT:
- Draw conclusions about what "should" happen
- Apply quality gates (that's wrap-up's job)
- Route to destinations (that's wrap-up's job)
- Filter out signals that seem minor (wrap-up will triage)

WRITE OUTPUT to ~/.claude/learning-captures/[session-id]/scan-NNN.md:

---
captured: [ISO timestamp]
session_id: [from path]
mode: scan
context_state: [Full / Partial — has compaction happened?]
signals_found: [total]
---

## Raw Signals

### 1. [Signal Type]: [Brief Title]
**Status:** [Observed / UNRESOLVED hypothesis / User-corrected]
**Quote:** "[Relevant quote from conversation]"
**Detail:** [What happened, what was tried, what was observed]

### 2. [Signal Type]: [Brief Title]
...

## Scratch Lines Incorporated
[List which scratch lines were confirmed and included]

## Scratch Lines Discarded
[List which were contradicted or unverifiable, with reasons]
```

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

#### Step 2: Triage Captures

```bash
ls -la ~/.claude/learning-captures/*/scan-*.md ~/.claude/learning-captures/*/scratch.md 2>/dev/null
```

List all capture files with timestamps, then present a **triage view**:

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

Spawn a sub-agent with the consolidation prompt (see below). Pass only the **approved captures** (this session + any user-selected others). This is where raw signals become conclusions.

#### Step 4: Present for User Verification

Present the consolidated summary:

```
## Session Learning Signals (Consolidated)

From [N] scans ([this session only / this session + M others]), I found these conclusions:

### Ready for Documentation (Passed All Gates)

| # | Type | Classification | Summary |
|---|------|----------------|---------|
| 1 | Discovery | Code-level | "P2024: Timed out fetching connection" — connection pooling fix |
| 2 | User pushback | Process-level (global) | Hypothesis testing before fixes |

### Resolved Hypotheses

| # | Original Hypothesis | Resolution | Evidence |
|---|---------------------|------------|----------|
| 1 | "Root cause might be connection pooling" | CONFIRMED — pool exhaustion under load | Fixed with pool size increase |

### Needs Review (Failed Gate)

| # | Type | Failed Gate | Reason |
|---|------|-------------|--------|
| 3 | Failed attempt | Verification | Never confirmed fix works |

---

⚠️ **VERIFICATION REQUIRED:** Does this summary accurately reflect what happened?
- Are names, facts, and premises correct?
- Did I miss anything important?
- Did I capture something that didn't actually happen?

Please confirm accuracy before I proceed to documentation.
```

**WAIT FOR USER CONFIRMATION before proceeding to Step 5.**

#### Step 5: Route to Destinations

After user confirms, route each learning to its proper destination:

| Type | Destination | Handler |
|------|-------------|---------|
| **Code-level** (confirmed fixes) | `docs/solutions/` | `/workflows:compound` (7 agents, schema-validated) |
| **Process-level (global)** | Root CLAUDE.md | Learning-loop direct (with consolidation discipline) |
| **Process-level (project)** | Project CLAUDE.md | Learning-loop direct |
| **Facts** (pure recall, no behavior change) | Memory MEMORY.md | Learning-loop direct |
| **Content-level** (understanding shifted) | Judgment Ledger | Learning-loop direct |

**Memory Routing Decision Test:**

> "Does this change how Claude should behave?"

| Answer | Destination | Example |
|--------|-------------|---------|
| **Yes — globally** | Root CLAUDE.md | "Always read the full file before editing" |
| **Yes — in this project** | Project CLAUDE.md | "This repo uses camelCase for API endpoints" |
| **No — it's a fact** | Memory MEMORY.md | "User's husband is Ted", "Playwright MCP tools are globally allowed" |
| **No — worldview/judgment shifted** | Judgment Ledger | "The Leash Length Problem" insight |
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

**Process-Level Routing (Global vs. Project):**

Decision boundary test: *"Would this apply if I was working in a completely different project?"*

| YES → root CLAUDE.md | NO → project CLAUDE.md |
|---|---|
| "When building APIs, ask about naming convention first" | "This repo's API uses camelCase for all endpoints" |
| "Before modifying functions, read entire implementation" | "Supabase RPCs in this project use `verb_noun` naming" |

#### Step 6: Clean Up

After documentation is confirmed, clean up only the **sessions that were consolidated** (this session + any user-approved others):

```bash
# Delete consolidated session directories only — NOT sessions the user skipped
rm -rf ~/.claude/learning-captures/[consolidated-session-id]/
```

**Do NOT delete sessions the user chose to "skip for now"** — they remain for future wrap-ups.

#### Step 7: Mention Flagged Insights

If any content-level insights were classified as Pattern or Principle:

```
"Note: You have [N] Pattern/Principle insight(s) flagged for content development
in the Judgment Ledger. These are ready for a dedicated Content Development Session
when you have time — they won't be processed in this session."
```

**Do NOT offer to process them now.** Content development requires dedicated thinking time.

### CONSOLIDATION_PROMPT

```
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

2. **Classify the conclusion:**
   - Code-level: Specific to codebase/framework (fix confirmed working)
   - Process-level (global): How to work better — applies across projects
   - Process-level (project): Repo-specific conventions, preferences
   - Fact: Pure recall, no behavior change (names, dates, preferences)
   - Content-level: Understanding shifted, publishable insight

3. **Apply Quality Gates (on conclusions, not raw signals):**

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

   FACT:
   □ Accuracy - verified against conversation evidence?
   □ Persistence - worth remembering across sessions?

4. **If PASSES all gates:** Extract with routing recommendation
5. **If FAILS any gate:** Note which gate failed, include as "REVIEW NEEDED"

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
**Classification:** [Code-level / Process-level (global) / Process-level (project) / Fact / Content-level]
**Route to:** [docs/solutions/ / Root CLAUDE.md / Project CLAUDE.md / Memory MEMORY.md / Judgment Ledger]

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

## Signals That Failed Gates

### [N]. [Type]: [Brief Title]
**Failed Gate:** [Which one]
**Reason:** [Why]

## Summary
- Raw signals analyzed: [N]
- Conclusions drawn: [X]
- Failed gates: [Y]
- Routing plan: [list by destination]
```

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
| `[user]` | User correction or pushback | `[user] Corrected: husband's name is Ted not Henry \| Verify names before persisting` |
| `[env]` | Tooling or environment surprise | `[env] Playwright MCP doesn't support PDF export \| Use sips or wkhtmltopdf instead` |

### Scratch File Rules

1. **One line per signal** — no multi-line entries. Format: `[timestamp] [source] summary | detail`
2. **Silent execution** — no announcement to user, no confirmation message
3. **Not a source of truth** — scratch lines are *input* for Scan/Wrap-up, not verified learnings
4. **Survives compaction** — the entire point; when context is compacted, scratch.md preserves the specifics
5. **Use Bash echo** as primary mechanism. Fall back to Write tool only if echo fails.
6. **Session-id** = use a short identifier for the current session (date + brief context)

---

## Code-Level Orchestration: When to Prompt for /workflows:compound

For **code-level learnings**, learning-loop's job is to prompt for `/workflows:compound` while context is still fresh.

### The Decision Flow

```
Code-level fix just confirmed working
         │
         ▼
    Has this context been compacted yet?
         │
    ┌────┴────┐
    │         │
    NO        YES
    │         │
    ▼         ▼
PROMPT IMMEDIATELY:              TOO LATE for full treatment:
"Want to run /workflows:compound Save signal in scan file
while details are fresh?"        for wrap-up documentation later
    │
    ├── User: "Yes" → Invoke /workflows:compound (7 agents, schema-validated)
    │
    └── User: "No/Later" → Capture signal, route at wrap-up
```

### Timing Matters

| Trigger | Action |
|---------|--------|
| **Code-level fix just confirmed** | Prompt for `/workflows:compound` immediately (context freshest) |
| **User invokes `/learning-loop scan`** | "Any undocumented code-level fixes? Last chance for `/workflows:compound`" |
| **After compaction** | Too late for multi-agent treatment; capture what you can for wrap-up |

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

**⚠️ Common misclassification:** A learning about content *operations* (editorial rules, scheduling, platform strategy) is **process-level (project)**, not content-level. It routes to Content Lab CLAUDE.md, not the Judgment Ledger. Content-level means your understanding of the *world* shifted — not your understanding of *how to do content work*.

**If a signal fails any gate → Don't extract. Note why in consolidation output for review.**

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
| **Code-level** | Specific to codebase/framework | `/workflows:compound` (multi-agent) | `docs/solutions/` with schema-validated YAML |
| **Process-level (global)** | How to work better — applies across projects | Learning-loop direct | Root CLAUDE.md with trigger + warning signs |
| **Process-level (project)** | Repo-specific conventions, preferences | Learning-loop direct | Project CLAUDE.md with trigger + context |
| **Fact** | Pure recall, no behavior change | Learning-loop direct | Memory MEMORY.md |
| **Content-level** | Publishable insight, understanding shifted | Learning-loop direct | Judgment Ledger with future trigger |

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

## What's New in v3.1

| Enhancement | Why It Matters |
|-------------|----------------|
| **Session-scoped wrap-up** | v3 consolidated ALL accumulated captures regardless of topic. v3.1 defaults to current session only, surfaces other sessions for triage. Prevents cross-polluting unrelated sessions. |
| **Orphan session surfacing** | Capture directories from sessions that closed without wrap-up are shown during triage — user decides to include, skip, or delete. No more silent accumulation. |
| **Sharper content-level routing** | v3 routed learnings *about content work* (editorial rules, scheduling) to Judgment Ledger. v3.1 distinguishes "worldview shifted" (→ Judgment Ledger) from "learned a better way to do content work" (→ Project CLAUDE.md). |

### v3.0

| Enhancement | Why It Matters |
|-------------|----------------|
| **Explicit `/learning-loop` invocation** | v2's description-based matching was non-deterministic — capture phrases matched intermittently, but "wrap up" never triggered reliably. Explicit invocation is deterministic. |
| **Two-mode model (Scan / Wrap-up)** | Scans capture raw signals without judgment; wrap-up resolves hypotheses with hindsight |
| **Smart mode detection** | Minimal friction — context clues route to the right mode, explicit override always available |
| **Memory as routing destination** | Facts (no behavior change) route to MEMORY.md instead of being lost or forced into CLAUDE.md |
| **Auto-memory coexistence** | Complementary design — auto-memory handles quick facts, learning-loop handles structured analysis |
| **User stories documented** | Distinct use cases ("mid-task, save signals" vs "done, consolidate everything") now explicit — prevents future designs from collapsing them |

### Previous Versions

**v2.1:** Real-time micro-logging (Phase 1 scratch files), project-level CLAUDE.md routing
**v2:** Type-specific quality gates, orchestration model, user-initiated triggers
**v1:** Proactive monitoring (failed — Claude can't sense context % in Claude Code)

---

## Design Principles

| Principle | How It's Implemented |
|-----------|---------------------|
| **Explicit invocation, deterministic behavior** | Description-based matching was non-deterministic (asymmetric failure); `/learning-loop` is deterministic and can't be intercepted |
| **Scans are raw, wrap-up draws conclusions** | Mid-session hypotheses resolved at session end with hindsight |
| **Smart default, explicit override** | Context clue detection with fallback to asking |
| **Memory is a routing destination** | Facts route to MEMORY.md; behavioral changes route to CLAUDE.md |
| **Orchestration over duplication** | Prompts for `/workflows:compound` instead of reimplementing code-level documentation |
| **Consolidation over accumulation** | CLAUDE.md edits require reading and merging, not just appending |
| **Persistence over memory** | Scratch lines and scan files survive compaction; mental notes don't |
| **Resilience over rigidity** | Complements auto-memory, adapts when system behaviors change |

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
