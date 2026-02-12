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
     ├─► Learning signal detected → one-line scratch entry
     │   written silently to ~/.claude/learning-captures/
     │   (survives compaction — details preserved even if context is lost)
     │
     ▼
User sees context warning OR wants to preserve learnings
     │
     ▼
User says "run a capture" or "wrap up"
     │
     ▼
Sub-agent spawns, scans context + scratch file,
applies quality gates, writes to capture file
     │
     ▼
User verifies captures for accuracy before routing
     │
     ▼
Learnings route to proper destinations
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

Learning-loop detects and classifies signals and routes them by type:

| Type | Example | Destination |
|------|---------|-------------|
| **Code-level** | Bug fixes, error resolutions | `docs/solutions/` via `/workflows:compound` |
| **Process-level (global)** | "Always read full implementation before modifying" | Root `CLAUDE.md` |
| **Process-level (project)** | "This repo's API uses camelCase" | Project `CLAUDE.md` |
| **Content-level** | Insights worth publishing | Judgment Ledger |

Process-level routing uses a simple test: *"Would this apply in a completely different project?"* Yes → global. No → project-specific.

### Quality Gates

Not everything is worth capturing. Each signal passes through:

1. **Reusability** — Would this help in a future similar situation?
2. **Non-triviality** — Did this require genuine discovery?
3. **Type-specific gates** — Error messages defined? Fix confirmed? Experience validated?

### User Verification

Before any learning is routed to its destination, the system presents a summary for user review. This catches hallucinations and misattributions before they reach persistent documentation.

## Key Design Decisions

See [SESSION_LOG.md](SESSION_LOG.md) for the full reasoning trail. Highlights:

- **User-initiated triggers** — Claude cannot sense context % in Claude Code. PreCompact hooks cannot spawn agents. User phrases are the reliable path.
- **Orchestration over duplication** — Learning-loop prompts for `/workflows:compound`, doesn't replace it.
- **Real-time micro-logging (v2.1)** — Phase 1 originally relied on "mentally flagging" signals — but compaction erased details before capture. Now writes one-line scratch entries silently, surviving compaction as input for the scanning sub-agent. Inspired by cross-analyzing [blader/napkin](https://github.com/blader/napkin).
- **Project-level routing (v2.1)** — Process learnings previously defaulted to root `CLAUDE.md` only. Repo-specific observations were silently dropped because they didn't belong in a global doc. Now routes project-specific conventions to the project's own `CLAUDE.md`.
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

v2 added: *"...and the system should be able to find what it learned."*
v2.1 added: *"...and nothing is lost between the moment of learning and the moment of capture."*

## Contributing

See [CLAUDE.md](CLAUDE.md) for development conventions when modifying this skill.

## License

MIT
