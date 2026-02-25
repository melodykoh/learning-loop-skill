# Learning-Loop Skill: Session Log

> **Purpose:** This file captures the *reasoning* behind skill evolution — the investigations, failed approaches, and decisions that don't fit in git commits. It prevents re-litigation of settled questions.

---

## Session: Jan 24, 2026 — V2 Redesign & Git Initialization

### Context
V1 was created but had fundamental issues with trigger reliability. This session redesigned the trigger model and established proper versioning.

### Problem 1: Proactive Monitoring Never Fires

**What V1 assumed:**
- Claude could detect when context was ~70% full
- Skill could proactively prompt for capture before compaction

**What we discovered:**
- Claude does NOT receive a system signal when context is low
- There's no `context.percentage` or similar accessible to Claude
- The 70% threshold was arbitrary and unverifiable

**Implication:** Proactive monitoring is fundamentally impossible with current Claude Code architecture.

---

### Problem 2: PreCompact Hooks Can't Spawn Sub-Agents

**Investigation:**
We explored whether PreCompact hooks (in `~/.claude/settings.json`) could solve the trigger problem.

**Hypothesis:** A PreCompact hook could spawn a learning-loop sub-agent before compaction.

**Test approach:**
```json
{
  "hooks": {
    "PreCompact": [{
      "type": "command",
      "command": "..."
    }]
  }
}
```

**Finding:**
- Hooks can run shell commands (bash scripts, file operations)
- Hooks CANNOT spawn Claude sub-agents or invoke the Task tool
- The hook runs in a shell context, not a Claude context

**Conclusion:** PreCompact hooks are useful for file operations (like checking for existing captures) but cannot trigger Claude-driven learning extraction.

---

### Problem 3: Skill Registration Discovery

**Symptom:** Custom SKILL.md files weren't appearing in autocomplete or invocable via `/skill-name`.

**Investigation path:**
1. Checked file syntax — valid YAML frontmatter, correct structure
2. Checked permissions — readable
3. Tried different locations — `skills/`, `.claude/skills/`

**Discovery:** The **dot-prefix matters**. Skills must be in:
- Personal: `~/.claude/skills/<skill-name>/SKILL.md`
- Project: `./.claude/skills/<skill-name>/SKILL.md`

The folder name `skills/` (without dot) is ignored by Claude Code's discovery mechanism.

**Resolution:** Moved skills to `.claude/skills/` — immediately invocable.

---

### Decision: User-Initiated Triggers Are The Path

Given that:
1. Claude can't sense context %
2. PreCompact hooks can't spawn agents
3. Proactive monitoring is impossible

**The reliable trigger model is user-initiated phrases:**
- "wrap up" / "let's wrap up"
- "run a capture" / "capture learnings"
- "before I clear" / "going to clear"

These are phrases the user naturally says when ending a session or seeing context warnings.

**Design principle:** The human sees the context warning in the terminal. The human initiates capture. Claude responds to the phrase.

---

### Decision: CLAUDE.md as Dispatcher, SKILL.md as Source of Truth

**Problem observed:**
Process logic was duplicated between root CLAUDE.md (Section 8) and SKILL.md. Changes in one weren't reflected in the other.

**Resolution:**
- **CLAUDE.md** contains only trigger conditions ("when to invoke /learning-loop")
- **SKILL.md** is the single source of truth for process logic ("what to do")

This follows the DRY principle — CLAUDE.md dispatches, SKILL.md executes.

---

### New in V2: Type-Specific Quality Gates

**Observation:** Not all learnings are equal. Code fixes need different validation than process insights.

**V2 adds type-specific gates:**

| Type | Specificity Gate | Validation Gate |
|------|------------------|-----------------|
| Code-level | Exact error messages, symptoms | Fix confirmed working |
| Process-level | Observable trigger situation | Experienced the failure |
| Content-level | What understanding shifted | Contradicts prior belief |

This prevents capturing vague "lessons" that aren't actionable.

---

### New in V2: Orchestration Model

**Key insight:** Learning-loop doesn't replace `/workflows:compound` — it ensures it gets run.

For code-level learnings, `/workflows:compound` does the heavy lifting (7 parallel agents, schema-validated output). Learning-loop's job is to:
1. Detect when code-level learning occurred
2. Prompt to run `/workflows:compound` while context is fresh
3. Fall back to capture if context is too low

This prevents duplication and leverages existing tools.

---

### Versioning Decision: Git + SESSION_LOG

**Problem:** We deleted SKILL-v1-archived.md during reorganization, losing the reasoning about why PreCompact hooks don't work. Only recovered because a draft post happened to document it.

**Options considered:**
1. Git only — shows what changed, not why
2. SESSION_LOG only — no diffing, no recovery
3. Git + SESSION_LOG — both layers

**Decision:** Both.
- Git provides version control, diffing, distribution
- SESSION_LOG provides reasoning trail, failed approaches, decision rationale

**This file is the implementation of that decision.**

---

### Files Changed This Session

| File | Change |
|------|--------|
| SKILL.md | Complete rewrite from v1 to v2 |
| SESSION_LOG.md | Created (this file) |
| ~/.claude/settings.json | Added SessionStart hook for post-clear recovery |
| Root CLAUDE.md | Refactored Section 8, added "Before Deleting Files" rule |

---

## Session: Jan 24, 2026 (Part 2) — GitHub Publication & Development Conventions

### Context
Following the git initialization, we needed to:
1. Publish to GitHub for shareability
2. Establish conventions so Claude knows to update SESSION_LOG.md on every change

### Decision: Separate CLAUDE.md for Development Conventions

**Problem:** How does Claude know to update SESSION_LOG.md every time SKILL.md changes?

**Options considered:**
1. Add instructions to SKILL.md itself — clutters the skill with meta-instructions
2. Add to root CLAUDE.md — but this is a per-repo convention
3. Create repo-local CLAUDE.md — Claude sees it when working in this directory

**Decision:** Create `CLAUDE.md` in the learning-loop repo with:
- Mandatory SESSION_LOG.md update requirement
- Commit and push workflow
- Quality checklist

This follows Claude Code's convention: CLAUDE.md files at any level are read and followed.

### Decision: SESSION_LOG.md Is a Teaching Artifact

**Question from earlier:** "When publishing to GitHub, should SESSION_LOG be included?"

**Answer:** Yes. SESSION_LOG.md serves dual purposes:
1. **For maintainers:** Prevents re-litigation of decisions
2. **For readers:** Shows *how* to think about building skills, not just final state

The reasoning trail is as valuable as the skill itself.

### Files Created

| File | Purpose |
|------|---------|
| README.md | User-facing documentation, installation, usage |
| CLAUDE.md | Development conventions for Claude when modifying this repo |

### GitHub Repo

- **URL:** https://github.com/melodykoh/learning-loop-skill
- **Visibility:** Public
- **Description:** Claude Code skill for capturing learnings before context compaction

---

## Session: Jan 23, 2026 — V1 Creation (Reconstructed)

> Note: V1 file was deleted. This section is reconstructed from memory and the draft post.

### Original Design Intent

Learning-loop was conceived to solve: "Sometimes I remember to run `/workflows:compound`, and sometimes I forget before context diminishes."

### V1 Approach (What We Tried)

- **Continuous monitoring:** Claude would track context usage
- **Proactive prompting:** At ~70% context, prompt for capture
- **Automatic invocation:** If user agreed, spawn capture sub-agent

### Why V1 Failed

1. **No context signal:** Claude has no access to context percentage
2. **No hook support:** PreCompact can't spawn agents
3. **Arbitrary threshold:** 70% was a guess with no way to verify

### What Survived to V2

- Core purpose: Ensure learnings are captured before compaction
- Quality gates concept (refined with type-specificity)
- The "human shouldn't need to remember" principle
- Integration with `/workflows:compound` for code-level learnings

---

## Session: Feb 11, 2026 — V2.1: Real-time Micro-Logging + Project-Level Routing

### Context

Analysis of the Napkin skill (blader/napkin) during content-lab work exposed two gaps in learning-loop v2. Napkin takes a radically simple approach — write mistakes down immediately, read them at session start, compound over sessions. Cross-analyzing this against our more sophisticated system revealed where the sophistication was hiding a blind spot.

### Problem 1: Phase 1 Was Mental-Only

**What v2 assumed:**
- "Mentally flag for capture" was sufficient
- User would initiate capture before compaction erased details

**What Napkin exposed:**
- If compaction happens before "run a capture", the Phase 3 sub-agent inherits a *summary*, not the specifics
- The sub-agent can't extract what the conversation no longer contains
- "Mental flagging" means details live only in volatile context — exactly the problem the skill exists to solve

**The irony:** Learning-loop's entire purpose is "files persist, context doesn't" — but Phase 1 was relying on context persistence.

### Solution: Scratch File via Bash Echo

Instead of "mentally flag", Phase 1 now appends one-line entries to `scratch.md` via `Bash(echo *)` — already in the global allow list, so no permission prompt.

**Key design decisions:**
- **Bash echo, not Write tool** — `echo` was already permitted globally; Write required adding `Write(~/.claude/learning-captures/**)` to global settings (done as a fallback)
- **One line per signal** — no multi-line entries. Forces concision, keeps the file scannable
- **Source tags** (`[self]`, `[user]`, `[env]`) — allows Phase 3 to prioritize user corrections over Claude's own observations
- **Not a source of truth** — scratch lines are *input* for Phase 3 quality gates, not verified learnings. The user verification step in Phase 4 still catches hallucinations.

### What We Took from Napkin

| Napkin Feature | Adopted? | Rationale |
|----------------|----------|-----------|
| Write mistakes immediately | ✅ Yes | Core of the scratch file mechanism |
| Single-file structure | ❌ No | We need type-specific routing; single file can't distinguish code/process/content |
| No quality gates | ❌ No | Quality gates prevent CLAUDE.md bloat and ensure learnings are actionable |
| Read at session start | Already had | SessionStart hook + CLAUDE.md auto-loading already covers this |

---

### Problem 2: No Route to Project-Level CLAUDE.md

**What v2 assumed:**
- "Process-level → CLAUDE.md" was sufficient
- All process learnings belong in root CLAUDE.md

**What we observed:**
- Root CLAUDE.md has a 550-line budget and global scope
- Repo-specific observations ("this API uses camelCase", "user prefers LinkedIn drafts to open with a question") don't belong there
- Without a route, project-specific learnings were silently dropped — user had to re-teach Claude each session

### Solution: Split Process-Level Routing

Added a decision boundary test: *"Would this apply if I was working in a completely different project?"*

- YES → root CLAUDE.md (global process rule)
- NO → project CLAUDE.md (repo-specific convention)

**Key design decision: Why project CLAUDE.md, not a separate patterns file?**

We considered creating `~/.claude/project-patterns/[repo-name].md` files. Rejected because:
1. Claude Code already auto-loads all CLAUDE.md files in the working directory tree at session start
2. A separate patterns directory would require custom loading logic
3. CLAUDE.md is the canonical location for "instructions Claude should follow in this directory"
4. Session-start review comes for free — no new Phase 0 needed

### Also: Settings Cleanup

The SessionStart hook detected two rogue project-level settings files created by "Always allow" clicks:
- `NextView/.claude/settings.local.json` — migrated `gh repo clone`, `sort`, `pip3`, `pip install`, `source`, `git cherry-pick` to global; discarded one-off loop commands and specific git commit message
- Root `.claude/settings.local.json` — migrated Playwright MCP tools (`browser_navigate`, `browser_wait_for`, `browser_close`), `WebSearch`, and `gh api` to global

Both files deleted after migration. Also added `Write(~/.claude/learning-captures/**)` to global settings for scratch file fallback.

---

### Files Changed This Session

| File | Change |
|------|--------|
| SKILL.md | v2.1 upgrade: Phase 1 scratch logging, Phase 3 scratch awareness, Phase 4 routing split, diagrams/tables updated |
| SESSION_LOG.md | Added this entry |
| ~/.claude/settings.json | Added learning-captures Write permission + migrated rules from 2 rogue project files |
| NextView/.claude/settings.local.json | Deleted (rules migrated to global) |
| .claude/settings.local.json | Deleted (rules migrated to global) |

---

## Session: Feb 24, 2026 — V3: Explicit Invocation, Two-Mode Model, Auto-Memory Coexistence

### Context

Auto-memory collision discovered: Claude Code's built-in auto-memory feature intercepts natural-language phrases like "capture" and "remember this." The skill's YAML `triggers` field used the same phrases, causing unpredictable behavior — sometimes auto-memory handled the request, sometimes the skill did, sometimes neither. Additionally, the YAML frontmatter `triggers` field was causing parsing issues that prevented the skill from loading reliably.

### Problem 1: Auto-Memory Collides with Trigger Words

**What v2.1 assumed:**
- Natural-language triggers ("run a capture", "capture learnings") would reliably invoke the skill
- Claude Code wouldn't intercept these phrases for other purposes

**What happened:**
- Claude Code added auto-memory, which intercepts "capture" and similar words
- The skill's `triggers` YAML field competed with auto-memory for the same phrases
- Users got unpredictable behavior — auto-memory would fire instead of the structured learning-loop

### Problem 2: YAML Triggers Field Causing Load Failures

**Symptom:** Skill wasn't appearing in available skills list or failing to load
**Root cause:** The `triggers` field in YAML frontmatter was causing parsing issues
**Resolution:** Removed `triggers` field entirely — skill uses explicit `/learning-loop` invocation only

### Decision: Explicit Invocation Only

Given that:
1. Auto-memory intercepts natural-language trigger phrases
2. YAML triggers field was unreliable
3. `/learning-loop` as a slash command cannot be intercepted by auto-memory

**The solution is explicit invocation with smart mode detection:**
- User invokes `/learning-loop` directly (optionally with `scan` or `wrap up`)
- Mode detection uses context clues from the user's recent messages
- If ambiguous, the skill asks which mode

This is a **resilience principle** — rather than fighting the platform's built-in behavior, the skill adapts to coexist.

### Decision: Two-Mode Model (Raw vs. Conclusions)

**Previous model:** Single capture phase applied quality gates immediately
**Problem:** Mid-session captures were drawing conclusions too early — hypotheses hadn't been validated yet

**New model:**
- **Scan mode:** Captures raw signals WITHOUT quality gates or conclusions. Hypotheses marked as UNRESOLVED.
- **Wrap-up mode:** Reads ALL capture files, resolves hypotheses with hindsight, THEN applies quality gates and routes.

This matches how learning actually works: observations accumulate, and conclusions emerge with the benefit of hindsight.

### Decision: Memory as a Routing Destination

**Previous model:** 4 destinations (docs/solutions, root CLAUDE.md, project CLAUDE.md, Judgment Ledger)
**Gap:** Facts (pure recall, no behavior change) didn't fit any destination. "User's husband is Ted" shouldn't be a CLAUDE.md rule, and it's not an insight worth a Judgment Ledger entry.

**New routing test:** "Does this change how Claude should behave?"
- Yes → CLAUDE.md (global or project)
- No, it's a fact → Memory MEMORY.md
- No, understanding shifted → Judgment Ledger
- Code fix → docs/solutions/

### Investigation: Multi-Session Capture Consolidation

**Previously a pending question** — "When captures span multiple sessions, what's the best consolidation UX?"

**Resolution:** Wrap-up mode handles this explicitly:
1. Scan current session first (final signals)
2. Read ALL capture files across all session directories
3. Consolidate with hindsight
4. Present unified summary for user verification
5. Route and clean up

The multi-session flow is now documented in SKILL.md with a concrete example (3-session scenario).

### Changes Made

| File | Change |
|------|--------|
| SKILL.md | v3 rewrite: removed triggers YAML, added mode detection, scan mode, wrap-up mode, auto-memory coexistence, memory routing, new scanner/consolidation prompts, v3 changelog |
| SESSION_LOG.md | Added this entry |
| README.md | Updated for v3: explicit invocation, two-mode flow, routing table with Memory, coexistence |
| ~/.claude/reference/learning-and-content.md | Replaced trigger table with /learning-loop invocation instructions |
| Root CLAUDE.md (Section 6) | Updated trigger description for explicit /learning-loop invocation |

---

## Pending Questions / Future Investigation

1. **SessionStart hook reliability:** Does the LEARNING_CAPTURES_EXIST check work consistently across all clear scenarios?

2. **Multi-session capture consolidation:** When captures span multiple sessions, what's the best consolidation UX?

3. ~~**Publication format:** When publishing to GitHub, should SESSION_LOG be included?~~ **RESOLVED:** Yes — it's a teaching artifact. See Jan 24 Part 2 session.

---

## How to Use This Log

**Before making changes:**
1. Check if the question was already investigated
2. If a decision was made, understand the reasoning before reopening

**After making changes:**
1. Document what you investigated, not just what you changed
2. Capture failed approaches — they're as valuable as successes
3. Note the decision and its rationale

**The goal:** Future sessions (or future readers) can understand not just WHAT the skill does, but WHY it's designed this way.
