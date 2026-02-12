# Learning-Loop Skill: Development Conventions

> **For Claude:** Follow these conventions when modifying this skill.

## Mandatory: Update SESSION_LOG.md

**Every time you modify SKILL.md, you MUST also update SESSION_LOG.md.**

### What to Document

1. **What problem you were solving** — Why was this change needed?
2. **What you investigated** — What approaches did you consider or try?
3. **What you discovered** — Any non-obvious findings, especially failures
4. **What you decided** — The choice made and its rationale
5. **What changed** — Summary of files/sections modified

### SESSION_LOG Entry Template

```markdown
## Session: [Date] — [Brief Title]

### Context
[Why this session happened, what triggered it]

### Problem
[What wasn't working or what needed to change]

### Investigation (if applicable)
[What approaches were tried, what was discovered]

### Decision
[What was decided and why]

### Changes Made
| File | Change |
|------|--------|
| SKILL.md | [What changed] |
| ... | ... |
```

### Why This Matters

Git commits show **what** changed. SESSION_LOG shows **why**.

Without SESSION_LOG, future sessions might:
- Re-investigate already-settled questions
- Revert intentional design decisions
- Lose context about failed approaches

## Mandatory: Commit and Push

After updating both SKILL.md and SESSION_LOG.md:

1. **Stage both files:**
   ```bash
   git add SKILL.md SESSION_LOG.md
   ```

2. **Commit with descriptive message:**
   ```bash
   git commit -m "Brief description of change

   [Optional: Key decision or finding]

   Co-Authored-By: Claude Opus 4.6 <noreply@anthropic.com>"
   ```

3. **Push to origin:**
   ```bash
   git push
   ```

## File Purposes

| File | Purpose | When to Update |
|------|---------|----------------|
| `SKILL.md` | The skill logic itself | When behavior changes |
| `SESSION_LOG.md` | Reasoning trail | Every time SKILL.md changes |
| `README.md` | User-facing docs | When usage/installation changes |
| `CLAUDE.md` | Development conventions | Rarely (this file) |

## Quality Checklist

Before committing changes:

- [ ] SKILL.md change is minimal and focused
- [ ] SESSION_LOG.md explains the reasoning
- [ ] README.md updated if user-facing behavior changed
- [ ] Commit message is descriptive
- [ ] Changes pushed to origin

## The Principle

> **"Decisions without reasoning become mysteries."**

Future sessions (or future contributors) should be able to understand not just what this skill does, but why it's designed this way.
