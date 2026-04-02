# Learning-Loop Skill for Claude Code

A two-mode learning system that captures raw signals mid-session and consolidates them with quality gates at session end. Ensures nothing valuable is lost to context compaction or `/clear`.

## The Problem

> "Sometimes I remember to run `/ce:compound`, and sometimes I forget before context diminishes or I just clear without thinking."

Claude Code sessions accumulate valuable insights — bug fixes, process decisions, failed approaches. But context compaction and `/clear` destroy these details. **Learning-loop ensures nothing valuable is lost.**

## How It Works

### Two Modes

**Scan mode** (mid-session): Captures raw signals — observations, hypotheses, failed attempts — without drawing conclusions. Fast, lightweight, back to work.

**Wrap-up mode** (session end): Reads current session's captures (and optionally surfaces orphaned captures from other sessions), resolves hypotheses with hindsight, applies quality gates and significance thresholds, routes conclusions to the right destinations.

```
Session 1: Working on feature...
  [context getting long]
  /learning-loop scan         → raw signals saved

Session 2: Still working...
  /learning-loop scan         → more raw signals saved

Session 3: Finally done!
  /learning-loop wrap up      → consolidates everything
    1. Scan this session
    2. Triage captures (this session + surface orphans)
    3. Resolve hypotheses → conclusions
    4. Quality gates + significance threshold → verify → route → clean up
```

### Smart Mode Detection

| Invocation | Mode |
|------------|------|
| `/learning-loop scan` | Scan (explicit) |
| `/learning-loop wrap up` | Wrap-up (explicit) |
| `/learning-loop` + "before I clear" | Scan (context clue) |
| `/learning-loop` + "done for now" | Wrap-up (context clue) |
| `/learning-loop` (ambiguous) | Asks which mode |

## Installation

### Personal Installation (Recommended)

```bash
mkdir -p ~/.claude/skills
cd ~/.claude/skills
git clone https://github.com/melodykoh/learning-loop-skill.git learning-loop
```

The skill is now available as `/learning-loop` in all your Claude Code sessions.

### Project Installation

```bash
mkdir -p .claude/skills
cd .claude/skills
git clone https://github.com/melodykoh/learning-loop-skill.git learning-loop
```

## Usage

Invoke explicitly with `/learning-loop`. No natural-language triggers — this avoids collision with Claude Code's built-in auto-memory feature.

### Routing

Learning-loop classifies and routes signals by type:

| Type | Destination | Decision Test |
|------|-------------|---------------|
| **Code-level** | `docs/solutions/` via [`/ce:compound`](#what-is-cecompound) | Specific to codebase/framework |
| **Process-level (behavioral)** | Root or project `CLAUDE.md` | Changes decisions — "Would this apply in a different project?" |
| **Process-level (operational)** | Project operational docs or `CLAUDE.md` | Changes procedures — adapts to repo infrastructure |
| **Skills-level** | `claude-skills` repo | About skill authoring, structure, deployment |
| **Fact** | Memory `MEMORY.md` | "Does this change behavior?" → No, it's a fact |
| **Content-level** | Judgment Ledger | Understanding shifted + fits content wedge |
| **Below threshold** | Noted (acknowledged, not persisted) | Interesting but forgettable |

### Quality Gates

Quality gates apply during **wrap-up consolidation**, not during scans. Each conclusion passes through:

1. **Reusability** — Would this help in a future similar situation?
2. **Non-triviality** — Did this require genuine discovery?
3. **Type-specific gates** — Error messages defined? Fix confirmed? Experience validated? Content wedge fit?
4. **Significance threshold** — "If lost, would a future session go WRONG?" Prevents over-documentation of signals that pass quality gates but aren't worth persisting.

For repeated failures, a **root cause check** fires before routing: "Why did the existing rule fail to prevent this?" The answer determines the destination — not topical similarity.

### User Verification

Before any learning is routed to its destination, the system presents a summary for user review. This catches hallucinations and misattributions before they reach persistent documentation.

## Why Explicit Invocation

v2 relied on a `triggers` YAML field, but `triggers` is not a supported field in Claude Code's SKILL.md spec. The only auto-invocation mechanism — description-based matching — is non-deterministic: distinctive phrases like "run a capture" matched often enough to produce capture files, but common phrases like "wrap up" consistently failed. Captures accumulated without consolidation. v3 fixes this with deterministic `/learning-loop` invocation.

This also avoids collision with Claude Code's built-in auto-memory, which intercepts natural-language phrases like "capture":

| Feature | Auto-Memory | Learning-Loop |
|---------|-------------|---------------|
| Invocation | Natural language | Explicit `/learning-loop` |
| Scope | Quick facts | Multi-signal session analysis |
| Quality gates | None | Type-specific + significance threshold + user verification |
| Output | MEMORY.md | 7 destinations based on type and significance |

## Key Design Decisions

See [SESSION_LOG.md](SESSION_LOG.md) for the full reasoning trail. Highlights:

- **Explicit invocation** — Description-based matching was non-deterministic (capture phrases matched intermittently, "wrap up" never triggered reliably). `/learning-loop` is deterministic.
- **Two-mode model** — Scans capture raw signals; wrap-up resolves hypotheses with hindsight. Matches how learning actually works.
- **Session-scoped wrap-up** — Defaults to current session's captures. Other sessions surfaced for triage, not auto-included.
- **Significance threshold** — Quality gates measure signal quality; significance threshold measures persistence value. "Noted" acknowledges without persisting.
- **Behavioral vs. operational split** — Process-level learnings split by what they change: decisions (→ CLAUDE.md) vs. procedures (→ operational docs).
- **Content wedge filter** — Judgment Ledger entries must fit "where AI capability meets reality" positioning.
- **Root cause check** — Repeated failures diagnosed before routing. Rules that fire at the wrong workflow moment are redesigned, not reinforced.
- **Memory routing** — Facts (no behavior change) route to MEMORY.md instead of being forced into CLAUDE.md or lost.
- **Ideas cross-reference** — Frustration/bottleneck signals matched against parked ideas at wrap-up.
- **Orchestration over duplication** — Prompts for `/ce:compound`, doesn't replace it.
- **Resilience principle** — Adapts to platform changes rather than fighting them.
- **Git + SESSION_LOG** — Git shows what changed. SESSION_LOG shows why.

## Configuration

Add this to `~/.claude/settings.json` for post-clear recovery:

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

## What Is `/ce:compound`?

Learning-loop references `/ce:compound` as the handler for code-level learnings. This is the **Compound** skill from [Every's Compound Engineering plugin](https://github.com/EveryInc/compound-engineering-plugin) — a Claude Code plugin that documents recently solved problems so your team's knowledge compounds over time.

When learning-loop detects a code-level fix during wrap-up, it prompts you to run `/ce:compound` while context is fresh. `/ce:compound` spawns multiple agents to extract the solution into a structured, schema-validated entry in `docs/solutions/` — complete with error messages, root cause, fix, and prevention steps. Learning-loop orchestrates *when* to invoke it; `/ce:compound` handles *how* to document it.

**To install:** Follow the setup instructions at [EveryInc/compound-engineering-plugin](https://github.com/EveryInc/compound-engineering-plugin). The plugin provides `/ce:compound` along with other engineering workflow skills.

> **Note:** Earlier versions of this skill referenced `/workflows:compound` — the old name before the skill moved to Every's plugin.

## The Meta-Principle

> **"The human shouldn't need to remember."**

Files persist. Context doesn't. This skill ensures the right things get written to files before context is lost.

v2 added: *"...and the system should be able to find what it learned."*
v2.1 added: *"...and nothing is lost between the moment of learning and the moment of capture."*
v3 added: *"...and the system adapts to its environment rather than fighting it."*

## Version History

| Version | Date | Key Changes |
|---------|------|-------------|
| **v3.3** | Mar 3, 2026 | Significance thresholds (Gate 5), "Noted" routing, behavioral/operational split |
| v3.2 | Mar 2, 2026 | Content wedge filter for Judgment Ledger |
| v3.1 | Mar 2, 2026 | Session-scoped wrap-up, orphan surfacing, sharper content routing |
| v3.0 | Feb 25, 2026 | Explicit invocation, two-mode model, auto-memory coexistence |
| v2.1 | Feb 11, 2026 | Real-time micro-logging (scratch files), project-level CLAUDE.md routing |
| v2.0 | Jan 24, 2026 | Type-specific quality gates, orchestration model |
| v1.0 | Jan 23, 2026 | Proactive monitoring (failed — Claude can't sense context %) |

Post-v3.3 enhancements: ideas cross-reference (Mar 6), skills-level routing (Mar 18), root cause check in consolidation (Apr 2).

## Contributing

See [CLAUDE.md](CLAUDE.md) for development conventions when modifying this skill.

## License

MIT
