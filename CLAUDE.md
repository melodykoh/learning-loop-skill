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
- [ ] SESSION_LOG.md explains the reasoning (newest entry first — reverse chronological)
- [ ] README.md updated if user-facing behavior changed (routing table, quality gates, design decisions, version history)
- [ ] Commit message is descriptive (include what, why, born-from context)
- [ ] Changes pushed to origin

### Version-Sensitive vs. Stable Files

Not all files need updating on every change:

| File | Updates when... | Version-sensitive? |
|------|----------------|-------------------|
| **SKILL.md** | Behavior changes | Yes — the "code" |
| **SESSION_LOG.md** | Any SKILL.md change | Yes — the reasoning trail |
| **README.md** | User-facing behavior changes | Yes — the public face |
| **CLAUDE.md** | Development process changes | No — structural conventions |

The pre-push hook (`doc-companion-check.sh`) enforces this via `.claude/doc-map`. If SKILL.md changes but SESSION_LOG.md or README.md don't, the hook warns before push.

## Gate-state enumeration — a standing check when adding or editing ANY gate

> Moved here from `SKILL.md` in v4.7. It fires when someone **edits** this skill, never when `/learning-loop` runs, so it does not belong in a file loaded on every invocation.

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

## The Principle

> **"Decisions without reasoning become mysteries."**

Future sessions (or future contributors) should be able to understand not just what this skill does, but why it's designed this way.
