# Learning-Loop Skill for Claude Code

A two-mode Claude Code skill that catches what your sessions teach you — failed attempts, user corrections, recurring failure modes, judgment shifts — and writes them to the right files before `/clear` or context compaction destroys the details.

> **Heads up: this is a personal skill, published in case it's useful.** It's shaped by one workflow — a root `~/.claude/CLAUDE.md`, per-project `CLAUDE.md` files, a `MEMORY.md`, a personal Judgment Ledger, and Every's [`/ce:compound`](#what-is-cecompound) for code-level capture. If your setup looks roughly like that, the routing will land where you'd expect. If it doesn't, you'll want to read [SKILL.md](SKILL.md) and adapt the destinations.

## The Problem

Claude Code sessions accumulate valuable signal — a hypothesis that turned out to be right, a fix you confirmed, a workflow rule you broke and want to encode, a corrected assumption, a recurring mistake you keep making across projects. Most of that gets lost the moment context compacts or you hit `/clear`.

Claude Code's built-in auto-memory captures quick facts, but it doesn't:
- Distinguish a one-off observation from a pattern you've now hit five times
- Apply quality gates (would this help next time? did you actually verify the fix?)
- Route a process-level rule into `CLAUDE.md` vs. a factual recall into `MEMORY.md` vs. a judgment shift into a content ledger
- Surface recurring failure modes that need a structural fix rather than another note

Learning-loop is the structured pass on top of that — invoked explicitly, run by sub-agents so it doesn't eat your main context, and gated so it doesn't pollute your docs with noise.

## What It Does

| Mode | When you run it | What it does |
|---|---|---|
| **`/learning-loop scan`** | Mid-session, before compaction or `/clear` | Spawns a sub-agent that reads your conversation, extracts raw signals (failed attempts, hypotheses, user corrections, process observations), and writes them to `~/.claude/learning-captures/<session-id>/scan-NNN.md`. No conclusions, no routing — fast, lightweight, back to work. Filters out single-incident-no-precedent noise via a recurrence test against the watch-list. |
| **`/learning-loop wrap up`** | At session end | Runs one final scan, surfaces any orphaned captures from other sessions, resolves hypotheses with hindsight, runs every conclusion through quality gates and a significance threshold, sends them through an adversarial persona review (shadow mode), and routes the survivors to the right destination. Then asks you to verify before anything is written. |

```
Session 1: working on feature X...
  [context getting long]
  /learning-loop scan          → raw signals saved to scan-001.md

Session 2: still on feature X...
  /learning-loop scan          → more signals saved

Session 3: finally done!
  /learning-loop wrap up
    1. Scan THIS session
    2. Triage: confirm session 1/2 captures should be included
    3. Consolidate hypotheses → conclusions
    4. Quality gates + significance threshold → user verification → route → clean up
```

### Routing Destinations

After quality gates pass, conclusions route by type:

| Type | Destination | Decision test |
|---|---|---|
| **Code-level** (bug + fix in a specific codebase) | `docs/solutions/` via user-invoked [`/ce:compound`](#what-is-cecompound) — **not** orchestrated by learning-loop | "Is this tied to a specific codebase/framework?" |
| **Process-level (behavioral)** | Root or project `CLAUDE.md` | "Does this change how Claude decides? Would it apply in a different project?" |
| **Process-level (operational)** | Project operational docs (playbooks, ops guides) or `CLAUDE.md` | "Does this change a procedure? Tied to this repo's infrastructure?" |
| **Skills-level** | The skill's own repo | "Is this about skill authoring, structure, or deployment?" |
| **Fact** | `MEMORY.md` | "Does this change behavior?" → no, just a fact worth recalling |
| **Content-level** | Judgment Ledger | "Did my understanding of the world shift? Could this become published content?" |
| **Below threshold** | Noted (acknowledged in summary, not persisted) | Interesting but a future session wouldn't go wrong without it |

### Watch-List: Recurring Failure Modes

Some signals don't deserve a rule on first sight — but if you hit the same root cause four times in two weeks, that's structural. The watch-list (`~/.claude/learning-captures/watch-list.md`) clusters incidents by **root cause + fix**, not by surface similarity. When a cluster hits the maturation threshold (≥5 sub-IDs and no active plan), wrap-up auto-drafts an execution plan with every historical incident as a Success Criteria checkbox so the fix author has the full test-case set.

When a cluster's fix actually ships, an entry is written to `graduation-log.md` (the mandatory counterpart to the watch-list) so resolved patterns don't just linger as one-way archives.

### Quality Gates

Quality gates apply during wrap-up only — scans capture freely. Each conclusion passes through:

1. **Reusability** — Would this help in a future similar situation?
2. **Non-triviality** — Did this require genuine discovery (not just an obvious docs lookup)?
3. **Type-specific** — Process-level: can you describe the observable trigger? Content-level: did your worldview shift, or just your workflow? Fact: verifiable against the conversation?
4. **Validation** — Did you confirm the fix? Did you experience the consequence of *not* following the rule?
5. **Significance threshold** — "If lost, would a future session go WRONG?" Quality ≠ persistence value. Plenty of true observations are just noted.

If a process-level rule fails repeatedly, a **root-cause check** fires before routing: *"Why did the existing rule fail to prevent this?"* The answer determines the destination — not topical similarity. Rules that fire at the wrong workflow moment get redesigned, not reinforced.

### Adversarial Review (Shadow Mode)

Before user verification, two sub-agent personas critique the consolidation:

- **Trigger-Moment Auditor** — flags rules framed at symptoms instead of mechanisms ("don't make the same mistake X" vs. "the upstream check that would have caught it")
- **Workflow-Step Router** — flags destinations that store recall-facts where decision-changers go, or vice versa

They report; you decide. The skill catches roughly 70% of framing/routing errors at zero workflow cost — surfaced inline in the verification view, not blocking. Each user decision feeds back into a `persona-eval-runs.txt` log used to evaluate whether the personas should ever be promoted to a gating role.

### Zoned Verification

The verification view scales attention to risk, not to volume:

- **Zone 1** — Decisions required. Full detail floor: what happened, what's wrong, what the fix does, why this destination, persona challenges. Capped at 5 items per wrap-up — overflow forces triage.
- **Zone 2** — Routine confirmations. One-line summaries leading with a concrete incident or quote from this session; cluster IDs and destinations at the end as routing metadata. Accept the batch with `y` or list items to expand.
- **Zone 3** — Auto-routed. Informational summary, promote anything to Zone 1 if needed.

### User Verification (Always)

Before any learning is written, the wrap-up summary is presented for review. This catches the AI's most common failure mode in this workflow — hallucinated names, fabricated premises, misremembered details. Nothing reaches `CLAUDE.md`, the Judgment Ledger, or Memory until you sign off.

## Installation

### Personal (recommended)

```bash
mkdir -p ~/.claude/skills
cd ~/.claude/skills
git clone https://github.com/melodykoh/learning-loop-skill.git learning-loop
```

`/learning-loop` is now available in every Claude Code session.

### Project-scoped

```bash
mkdir -p .claude/skills
cd .claude/skills
git clone https://github.com/melodykoh/learning-loop-skill.git learning-loop
```

### Post-Clear Recovery Hook (optional, recommended)

Add to `~/.claude/settings.json` so unconsolidated captures are surfaced after a `/clear`:

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

## Usage

Invoke explicitly. There are no natural-language triggers — this is deliberate (see below).

| Invocation | Mode |
|---|---|
| `/learning-loop scan` | Scan (explicit) |
| `/learning-loop wrap up` | Wrap-up (explicit) |
| `/learning-loop` + "before I clear" / "context getting long" | Scan (context clue) |
| `/learning-loop` + "done for now" / "wrap up" | Wrap-up (context clue) |
| `/learning-loop` (ambiguous) | Asks which mode |

## Why Explicit Invocation

v2 tried `triggers` in YAML frontmatter. `triggers` isn't a supported SKILL.md field. The only auto-invocation mechanism — description-based matching — is non-deterministic: distinctive phrases like "run a capture" matched intermittently, but common phrases like "wrap up" never reliably fired. Captures accumulated with no consolidation step. v3 fixed this with deterministic `/learning-loop` invocation.

Explicit invocation also avoids collision with Claude Code's built-in auto-memory, which intercepts phrases like "remember this":

| Feature | Auto-memory | Learning-loop |
|---|---|---|
| Invocation | Natural language | Explicit `/learning-loop` |
| Scope | Quick facts | Multi-signal session analysis |
| Quality gates | None | 5 gates + significance threshold + adversarial persona review + user verification |
| Output | `MEMORY.md` | 6 destinations based on type and significance |

They're complementary: auto-memory handles quick "remember X" requests; learning-loop handles structured session analysis. `MEMORY.md` is one of learning-loop's routing destinations.

## What Is `/ce:compound`?

`/ce:compound` is the **Compound** skill from Every's [Compound Engineering plugin](https://github.com/EveryInc/compound-engineering-plugin) — a Claude Code plugin that documents recently solved problems via 7 parallel agents into schema-validated `docs/solutions/` entries (error messages, root cause, fix, prevention steps).

**Relationship to learning-loop:** code-level capture is **user-invoked, not orchestrated.** When you confirm a code-level fix during a coding session, invoke `/ce:compound` directly — peak-fresh context produces the best documentation. Learning-loop's wrap-up will surface a one-line nudge if a code-level fix appears in scan signals but `/ce:compound` was never invoked (*"Worth a delayed `/ce:compound` while context is still warm?"*) — but it does not auto-invoke the skill.

> **Note:** Earlier versions referenced `/workflows:compound` — the old name before the skill moved to Every's plugin.

## Key Design Decisions

See [SESSION_LOG.md](SESSION_LOG.md) for the full reasoning trail. Highlights:

- **Explicit invocation** — description-based matching was asymmetrically unreliable. `/learning-loop` is deterministic.
- **Two-mode model** — scans capture raw signals; wrap-up resolves hypotheses with hindsight. Matches how learning actually works.
- **Session-scoped wrap-up** — defaults to the current session; other sessions surface for triage, not auto-included.
- **Quality vs. persistence are separate** — gates 1–4 measure signal quality; gate 5 (significance) measures whether to persist. "Noted" exists for signals that pass quality but aren't worth a future-session correction.
- **Behavioral vs. operational split** — process-level learnings split by what they change: decisions (→ `CLAUDE.md`) vs. procedures (→ operational docs or playbooks).
- **Content wedge filter** — Judgment Ledger entries must fit "where AI capability meets reality" positioning. Content *operations* insights route to operational docs, not the ledger.
- **Root-cause matching on the watch-list** — clusters by underlying fix, not surface text. Two superficially different observations sharing a mechanism are the same entry.
- **Granularity ceiling + graduation ledger (v4.0)** — capture-without-action is debt. Mature clusters auto-draft plans; shipped fixes get atomic graduation entries instead of accumulating as a one-way archive.
- **Adversarial persona review (shadow)** — Trigger-Moment Auditor + Workflow-Step Router catch symptom-vs-mechanism framing and recall-vs-decision destination errors at zero workflow cost.
- **Zoned verification (v3.8)** — cognitive load scales with materiality. Zone 1 caps at 5 items per wrap-up.
- **User-invoked `/ce:compound`** — peak-fresh context beats deferred reconstruction.
- **User verification before any write** — AI captures hallucinate. Names, premises, and constraints get fact-checked before persistence.
- **Git + SESSION_LOG** — git shows *what* changed; SESSION_LOG shows *why*.

## File Map

| File | Purpose |
|---|---|
| `SKILL.md` | The skill logic — invoked by Claude Code. Contains the SCANNER_PROMPT, CONSOLIDATION_PROMPT, persona prompts, and full wrap-up step sequence. |
| `SESSION_LOG.md` | Reverse-chronological reasoning trail. Why each version exists, what was tried, what was discovered. |
| `README.md` | This file. |
| `CLAUDE.md` | Development conventions for modifying the skill. |
| `~/.claude/learning-captures/` (runtime) | Session capture dirs, `watch-list.md`, `graduation-log.md`, `phase-1-decision-log.md`, persona eval logs. Not in this repo — created at runtime per user. |

## Version History

| Version | Date | Key changes |
|---|---|---|
| **v4.0** | May 20, 2026 | Watch-list hygiene pass — Mods 6–10: codify-now overrides recurrence threshold when mechanism + destination + ≥1 incident are all named; granularity ceiling rule (collapse ≥3 sub-entries sharing fix); stalled-deliverable separation (plan-stale ≠ learning-loop-stale); graduation ledger mandatory counterpart to watch-list; path-drift detection at cluster audit. Phase 2 persona gatekeeper officially retired — shadow mode is the permanent active state. |
| v3.11 | May 12, 2026 | Watch-list auto-promotion refined: maturation gate (≥5 sub-IDs) + no-active-plan gate + Fix-field auto-routing + Open Questions handling. Plan drafting offloaded to a child sub-agent (PLAN_DRAFTER_PROMPT) so main wrap-up context stays clean. |
| v3.10 | May 12, 2026 | SCANNER_PROMPT recurrence test — scanner reads watch-list before flagging, drops single-incident-no-precedent signals to a Dropped Signals footer, surfaces same-type recurrences tagged with cluster ID. Raises capture bar upstream. |
| v3.9 | May 12, 2026 | Removed false `/ce:compound` orchestration claims; added per-conclusion wedge-test recording (makes Judgment Ledger screening auditable). |
| v3.8 | May 2, 2026 | Tiered verification — Zone Classification + Zone-1 cap rule; scales floor rigor by zone. |
| v3.7 | Apr 29, 2026 | Verification Detail Floor + Same-Root-Cause Collapse Check in CONSOLIDATION_PROMPT. |
| v3.6 | Apr 29, 2026 | Enforcement-Gap Check + Skill Version Ship Verification STOP. |
| v3.5 | Apr 28, 2026 | Phase 1 Persona Panel — Trigger-Moment Auditor + Workflow-Step Router in shadow mode. |
| v3.4 | Apr 28, 2026 | Watch-list root-cause routing rewrite (Mods 1–5) — cluster by root cause + fix; threshold = auto-draft plan. |
| v3.3 | Mar 3, 2026 | Significance threshold (Gate 5), "Noted" routing, behavioral/operational split. |
| v3.2 | Mar 2, 2026 | Content wedge filter for Judgment Ledger. |
| v3.1 | Mar 2, 2026 | Session-scoped wrap-up, orphan surfacing, sharper content routing. |
| v3.0 | Feb 25, 2026 | Explicit invocation, two-mode model, auto-memory coexistence. |
| v2.1 | Feb 11, 2026 | Real-time micro-logging (scratch files), project-level `CLAUDE.md` routing. |
| v2.0 | Jan 24, 2026 | Type-specific quality gates, orchestration model. |
| v1.0 | Jan 23, 2026 | Proactive monitoring (failed — Claude can't sense context % in Claude Code). |

Pre-v3.4 micro-enhancements not in the table: ideas cross-reference (Mar 6), skills-level routing (Mar 18), root cause check in consolidation (Apr 2), Step 1b Deferred Methodology Check (Apr 22), Step 5b reverse-check consumers gate (Apr 18), Resolution-vs-Increment STOP (May 7).

## The Meta-Principle

> **"The human shouldn't need to remember."**

Files persist. Context doesn't. This skill writes the right things to the right files before context is lost.

- v2 added: *"...and the system should be able to find what it learned."*
- v2.1 added: *"...and nothing is lost between the moment of learning and the moment of capture."*
- v3 added: *"...and the system adapts to its environment rather than fighting it."*
- v4 added: *"...and capture-without-action is debt, not safety."*

## Contributing

See [CLAUDE.md](CLAUDE.md) for development conventions when modifying this skill.

## License

MIT
