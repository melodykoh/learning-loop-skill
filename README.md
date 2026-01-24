# Learning-Loop Skill for Claude Code

An orchestration layer that ensures valuable learnings are captured before context compaction or `/clear` destroys them.

## The Problem

> "Sometimes I remember to run `/workflows:compound`, and sometimes I forget before context diminishes or I just clear without thinking."

Claude Code sessions accumulate valuable insights:
- Why a bug occurred and how it was fixed
- Process decisions and their rationale
- Failed approaches that shouldn't be repeated

But context compaction and `/clear` destroy these details. **Learning-loop ensures nothing valuable is lost.**

## How It Works

```
Session in progress
     │
     ├─► Significant debugging/investigation happens
     │
     ▼
User sees context warning OR wants to preserve learnings
     │
     ▼
User says "run a capture" or "wrap up"
     │
     ▼
Learning-loop scans for signals, applies quality gates,
writes to capture file, routes to proper destinations
     │
     ▼
Learnings survive compaction/clear
```

## Installation

### Personal Installation (Recommended)

```bash
# Clone to your personal skills directory
mkdir -p ~/.claude/skills
cd ~/.claude/skills
git clone https://github.com/melodykoh/learning-loop-skill.git learning-loop
```

The skill is now available as `/learning-loop` in all your Claude Code sessions.

### Project Installation

```bash
# Clone to project skills directory
mkdir -p .claude/skills
cd .claude/skills
git clone https://github.com/melodykoh/learning-loop-skill.git learning-loop
```

## Usage

### Trigger Phrases

| Phrase | When to Use |
|--------|-------------|
| `"wrap up"` / `"let's wrap up"` | End of session |
| `"run a capture"` / `"capture learnings"` | Mid-session when you see context warning |
| `"before I clear"` | Before running `/clear` |

### What Gets Captured

Learning-loop detects and classifies three types of signals:

| Type | Example | Destination |
|------|---------|-------------|
| **Code-level** | Bug fixes, error resolutions | `docs/solutions/` via `/workflows:compound` |
| **Process-level** | "Always test before committing" | Root `CLAUDE.md` |
| **Content-level** | Insights worth publishing | Judgment Ledger |

### Quality Gates

Not everything is worth capturing. Each signal passes through:

1. **Reusability** — Would this help in a future similar situation?
2. **Non-triviality** — Did this require genuine discovery?
3. **Type-specific gates** — Error messages defined? Fix confirmed? Experience validated?

## Key Design Decisions

See [SESSION_LOG.md](SESSION_LOG.md) for the full reasoning trail. Highlights:

- **User-initiated triggers** — Claude cannot sense context %. PreCompact hooks cannot spawn agents. User phrases are the reliable path.
- **Orchestration over duplication** — Learning-loop prompts for `/workflows:compound`, doesn't replace it.
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
            "command": "if [ -d ~/.claude/learning-captures ] && [ \"$(ls -A ~/.claude/learning-captures 2>/dev/null)\" ]; then echo 'LEARNING_CAPTURES_EXIST: Found learning captures. Ask user if they want to review/consolidate.'; fi"
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

## Contributing

See [CLAUDE.md](CLAUDE.md) for development conventions when modifying this skill.

## License

MIT
