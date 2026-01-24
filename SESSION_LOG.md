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

## Pending Questions / Future Investigation

1. **SessionStart hook reliability:** Does the LEARNING_CAPTURES_EXIST check work consistently across all clear scenarios?

2. **Multi-session capture consolidation:** When captures span multiple sessions, what's the best consolidation UX?

3. **Publication format:** When publishing to GitHub, should SESSION_LOG be included? (Probably yes — it's a teaching artifact)

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
