---
name: learning-loop
description: Two-mode learning system — raw signal scanning before compaction, quality-gated consolidation at session end. Invoke with /learning-loop.
version: 3.3.0
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

# learning-loop Skill v3.3

**Purpose:** Two-mode learning capture — raw signal scanning mid-session, quality-gated consolidation at session end. Ensures `/ce:compound` runs when it should, and nothing valuable is lost to compaction or `/clear`.

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
3. **Wait for user confirmation** before routing to CLAUDE.md, Judgment Ledger, Memory, or running `/ce:compound`

> **Why this exists (Jan 29, 2026):** AI-generated captures can contain hallucinations — wrong names, fabricated premises, misremembered details. A capture once got the user's husband's name wrong and claimed constraints that didn't exist.

**DO NOT** update any destination based on captures without user sign-off.

---

## Core Insight

> **"The human shouldn't need to remember."**

Context compaction and `/clear` destroy details. Files persist. This skill ensures:
1. **Learnings are captured** before compaction erases them
2. **The right tools get invoked** (`/ce:compound` for code-level, direct edits for process/content)
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
7. "Repeated instruction: user gave the same multi-step instruction seen in a prior session" — skill candidate signal. If the user directed the same workflow pattern across 2+ sessions, flag it as a potential skill to codify. Note what the pattern is and why it recurs.

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
| 2 | User pushback | Process-level (behavioral) | Hypothesis testing before fixes |

### Resolved Hypotheses

| # | Original Hypothesis | Resolution | Evidence |
|---|---------------------|------------|----------|
| 1 | "Root cause might be connection pooling" | CONFIRMED — pool exhaustion under load | Fixed with pool size increase |

### Watch List (Recurring Candidates)

Before finalizing the Noted bucket: check `~/.claude/learning-captures/watch-list.md`.

| # | Observation | Prior Count | New Count | Threshold | Action |
|---|-------------|-------------|-----------|-----------|--------|
| [N] | [Brief description] | [existing count] | [+1] | [2] | [Escalate → destination / Still watching] |

- If an observation from this session **matches an existing entry**: increment its count. If count reaches threshold, move it to "Ready for Documentation" and route it.
- If an observation is "Noted" but **could recur** (a gap in a skill, a behavioral pattern that slipped): add it as a new watch-list entry with count=1.
- After user verification, update `watch-list.md` (increment or add entries). Move escalated entries to the Archived section.

### Noted (Passed Quality Gates, Below Persistence Threshold)

| # | Type | Summary | Why Not Persisted |
|---|------|---------|-------------------|
| [N] | [Type] | [Brief description] | [What would NOT go wrong if forgotten] |

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
| **Code-level** (confirmed fixes) | `docs/solutions/` | `/ce:compound` (7 agents, schema-validated) |
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

#### Step 6: Clean Up

After documentation is confirmed, clean up only the **sessions that were consolidated** (this session + any user-approved others):

```bash
# Delete consolidated session directories only — NOT sessions the user skipped
rm -r ~/.claude/learning-captures/[consolidated-session-id]/
```

**Do NOT delete sessions the user chose to "skip for now"** — they remain for future wrap-ups.

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
   └── Confirm: "Committed and pushed: [short hash] [message]"

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

7. RECONCILE Pass A drift (runs AFTER all commits in step 6 succeed):
   Compare Pass B's discovered set against Pass A's hardcoded list.
   For each repo Pass B found that is NOT in Pass A, prompt the user:

   "Pass B discovered 1 repo not in Pass A's hardcoded list:
      - [toplevel path]
        Files touched: [file1, file2, ...]
        Routing context: [which destination added these files]

    Pass B will catch this repo every session, but adding it to Pass A
    documents it as an expected destination and makes future sessions
    faster. Add to Pass A? (Y/skip/one-off)

      Y       → edit SKILL.md to add the repo to Pass A with a dated note
                  (e.g. '- [path] (added [date]: [routing context]')
      skip    → leave as-is; Pass B keeps rediscovering it each session
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

   Why a three-way choice: two-way (Y/skip) trains users to reflexively skip
   when a legitimate one-off appears (temporary worktree, experimental repo).
   The one-off option preserves the signal value of Y by giving drift a
   legitimate non-persistent home.
```

**Why this exists:** Most local repos were set up for git-based safety (backup + rollback) but may lack .gitignore. Without a session-end prompt, operational doc changes accumulate uncommitted across sessions, losing the backup and history benefits. The .gitignore check ensures each repo only needs one-time setup — after that, commits are clean automatically.

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

2. **Root cause check (repeated failures only):** If a signal represents a mistake
   that an existing rule should have caught, or the same type of error occurred
   multiple times in the session, STOP before classifying and ask:
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

6. **Apply Significance Threshold (Gate 6):**
   Ask: "If this were lost after this session, would a future session go WRONG?"
   □ YES — Claude would repeat a mistake, skip a step, or lose needed context → Route to destination
   □ NO — this is an interesting observation but forgettable → Route to "Noted"

   For process-level conclusions that pass significance, apply the behavioral/operational split:
   □ Behavioral (changes what Claude decides) → CLAUDE.md (root or project)
   □ Operational (changes how Claude executes a procedure) → Check: does project have
     dedicated operational docs (playbooks/, etc.)? If yes → route there. If no → CLAUDE.md
     if significant, Memory if marginal.

7. **If PASSES all gates + significance:** Extract with routing recommendation

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
**Route to:** [docs/solutions/ / CLAUDE.md (root or project) / Project operational docs / Memory MEMORY.md / Judgment Ledger / Noted]

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

## Code-Level Orchestration: When to Prompt for /ce:compound

For **code-level learnings**, learning-loop's job is to prompt for `/ce:compound` while context is still fresh.

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
"Want to run /ce:compound Save signal in scan file
while details are fresh?"        for wrap-up documentation later
    │
    ├── User: "Yes" → Invoke /ce:compound (7 agents, schema-validated)
    │
    └── User: "No/Later" → Capture signal, route at wrap-up
```

### Timing Matters

| Trigger | Action |
|---------|--------|
| **Code-level fix just confirmed** | Prompt for `/ce:compound` immediately (context freshest) |
| **User invokes `/learning-loop scan`** | "Any undocumented code-level fixes? Last chance for `/ce:compound`" |
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
| **Code-level** | Specific to codebase/framework | `/ce:compound` (multi-agent) | `docs/solutions/` with schema-validated YAML |
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

## What's New in v3.3

| Enhancement | Why It Matters |
|-------------|----------------|
| **Significance threshold (Gate 5)** | Gates 1-4 are pass/fail on quality. Gate 5 asks "would a future session go WRONG without this?" — separating interesting observations from consequential learnings. Prevents over-documentation of signals that pass quality gates but aren't worth persisting. |
| **"Noted" routing option** | Explicit acknowledgment for signals that pass quality gates but fall below the persistence threshold. Shown in wrap-up summary but not routed anywhere. Prevents the false binary of "document everything" vs. "lose it." |
| **Behavioral vs. operational split** | Process-level learnings now split into behavioral (changes decisions → CLAUDE.md) vs. operational (changes procedures → project operational docs). Prevents CLAUDE.md from accumulating scheduling heuristics and workflow sequences that belong in playbooks or operational docs. |
| **Repo-adaptive operational routing** | Operational learnings route to playbooks/ if the project has them, otherwise to CLAUDE.md or Memory. The skill is global but adapts to each project's documentation infrastructure instead of assuming playbooks exist. |

### v3.2

| Enhancement | Why It Matters |
|-------------|----------------|
| **Content wedge filter** | Judgment Ledger entries must now pass the content wedge test ("where AI capability meets reality"). Prevents accumulation of operationally useful but non-publishable entries. Borderline cases tagged `⚠️ wedge-check` for user decision. |
| **Content-level quality gate** | Added wedge fit checkbox to content-level quality gates in consolidation prompt. Entries that fail get reclassified as process-level. |

### v3.1

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
| **Orchestration over duplication** | Prompts for `/ce:compound` instead of reimplementing code-level documentation |
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
