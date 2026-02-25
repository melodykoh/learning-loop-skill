# Learning-Loop Skill for Claude Code

A two-mode learning system that captures raw signals mid-session and consolidates them with quality gates at session end. Ensures nothing valuable is lost to context compaction or `/clear`.

## The Problem

> "Sometimes I remember to run `/workflows:compound`, and sometimes I forget before context diminishes or I just clear without thinking."

Claude Code sessions accumulate valuable insights — bug fixes, process decisions, failed approaches. But context compaction and `/clear` destroy these details. **Learning-loop ensures nothing valuable is lost.**

## How It Works

### Two Modes

**Scan mode** (mid-session): Captures raw signals — observations, hypotheses, failed attempts — without drawing conclusions. Fast, lightweight, back to work.

**Wrap-up mode** (session end): Reads all capture files across sessions, resolves hypotheses with hindsight, applies quality gates, routes conclusions to the right destinations.

```
Session 1: Working on feature...
  [context getting long]
  /learning-loop scan         → raw signals saved
  [compaction happens]

Session 2: Still working...
  /learning-loop scan         → more raw signals saved

Session 3: Finally done!
  /learning-loop wrap up      → consolidates everything
    1. Scan this session
    2. Read all captures (sessions 1-3)
    3. Resolve hypotheses → conclusions
    4. Quality gates → verify → route → clean up
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
| **Code-level** | `docs/solutions/` via `/workflows:compound` | Specific to codebase/framework |
| **Process-level (global)** | Root `CLAUDE.md` | "Would this apply in a different project?" → Yes |
| **Process-level (project)** | Project `CLAUDE.md` | "Would this apply in a different project?" → No |
| **Fact** | Memory `MEMORY.md` | "Does this change behavior?" → No, it's a fact |
| **Content-level** | Judgment Ledger | Understanding shifted, potentially publishable |

### Quality Gates

Quality gates apply during **wrap-up consolidation**, not during scans. Each conclusion passes through:

1. **Reusability** — Would this help in a future similar situation?
2. **Non-triviality** — Did this require genuine discovery?
3. **Type-specific gates** — Error messages defined? Fix confirmed? Experience validated?

### User Verification

Before any learning is routed to its destination, the system presents a summary for user review. This catches hallucinations and misattributions before they reach persistent documentation.

## Auto-Memory Coexistence

v3 is designed to complement Claude Code's built-in auto-memory, not compete with it:

| Feature | Auto-Memory | Learning-Loop |
|---------|-------------|---------------|
| Invocation | Natural language | Explicit `/learning-loop` |
| Scope | Quick facts | Multi-signal session analysis |
| Quality gates | None | Type-specific + user verification |
| Output | MEMORY.md | 5 destinations based on type |

## Key Design Decisions

See [SESSION_LOG.md](SESSION_LOG.md) for the full reasoning trail. Highlights:

- **Explicit invocation** — Avoids collision with auto-memory. `/learning-loop` can't be intercepted.
- **Two-mode model** — Scans capture raw signals; wrap-up resolves hypotheses with hindsight. Matches how learning actually works.
- **Memory routing** — Facts (no behavior change) route to MEMORY.md instead of being forced into CLAUDE.md or lost.
- **Orchestration over duplication** — Prompts for `/workflows:compound`, doesn't replace it.
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

## The Meta-Principle

> **"The human shouldn't need to remember."**

Files persist. Context doesn't. This skill ensures the right things get written to files before context is lost.

v2 added: *"...and the system should be able to find what it learned."*
v2.1 added: *"...and nothing is lost between the moment of learning and the moment of capture."*
v3 added: *"...and the system adapts to its environment rather than fighting it."*

## Contributing

See [CLAUDE.md](CLAUDE.md) for development conventions when modifying this skill.

## License

MIT
