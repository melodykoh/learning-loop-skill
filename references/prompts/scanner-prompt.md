# SCANNER_PROMPT (Raw Signal Mode)

> Used by: Scan mode step 4, and Wrap-up Step 1 (scan of the current session).
> Extracted from SKILL.md v4.1 (2026-07-07 restructure). **Amended in v4.8:** four sites that told this agent to quote a conversation it cannot see were re-sourced to the capture files, and an unverifiable scratch line is now kept as `UNCORROBORATED` rather than discarded.
> Substitute `[session-id]` / `[NNN]` before dispatch. Everything below this line is the sub-agent prompt.

---

You have NO access to the conversation this session ran. You inherit no transcript,
no memory, no CLAUDE.md, no skills, no reference docs. What you are handed IS your
context: this prompt plus the files named below, and nothing else.

That is the design, not a limitation to route around — **the hand-off file IS the
mechanism** (see SKILL.md's rationalization table under that exact phrase). The session
holding the live context writes the scan file richly enough to stand on its own; you,
and every consumer downstream of you, read it blind. **If what you were handed is too
thin to work from, say so and name exactly what is missing. Do NOT reconstruct, infer,
or assume the surrounding context** — a confident reconstruction is worse than a stated
gap, because it is indistinguishable from a real observation.

Your job is to identify RAW LEARNING SIGNALS — unresolved observations, not conclusions.

FIRST: Check for scratch file at ~/.claude/learning-captures/[session-id]/scratch.md
If it exists, read it. Each line is an unverified micro-signal logged in real-time.
Cross-reference each line against THE OTHER FILES YOU WERE GIVEN — never against
"the conversation", which you cannot see:
- Corroborated by another capture file → incorporate into your signal list
- Contradicted by another capture file → discard, and say which file contradicted it
- Neither corroborated nor contradicted → KEEP IT, marked `UNCORROBORATED`. A scratch
  line you cannot check is still evidence; silently dropping it destroys the only
  record. Do not treat "I cannot verify this" as "this is false."

SECOND: Read ~/.claude/learning-captures/watch-list.md in full (v3.9 May 12 2026).
Note the active clusters (W1.*, W4, W7.*, etc.) and the Fix field for each. You will
use this list to apply the RECURRENCE TEST below before deciding whether any signal
reaches main output or routes to the Dropped Signals footer.

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
- Quote verbatim FROM THE CAPTURE FILES YOU WERE GIVEN, with the filename. If no capture
  file contains a usable quote, omit the field — never compose one

APPLY THE RECURRENCE TEST (v3.9 May 12 2026, MANDATORY) before deciding where each
candidate signal lands:

Test phrasing: *"If a fix were in the right place (per the matching watch-list entry's
Fix field), would this incident have happened again?"*

Four outcomes:

1. **MATCH — same-type recurrence with precedent.** Signal matches an existing
   watch-list entry's root cause + fix shape AND incident happened in this session
   even once. The fix isn't in place yet, so this is evidence the underlying issue
   persists. → **SURFACE in main signal output**, tagged with the matching cluster
   ID (e.g., "W1.b incident", "W7.b incident").

2. **MATCH — literally same rule, already codified.** Signal duplicates an already-
   codified rule (the Fix field shows "shipped" or links to a hook/reference that
   exists). The correct response depends on whether enforcement was the gap.
   → If enforcement gap is the root cause, **SURFACE as evidence the codified rule
   isn't firing.** Otherwise fold as a routine increment into the existing cluster —
   do NOT create a new signal.

3. **NO MATCH — novel single-incident.** Signal has no matching watch-list entry
   AND appeared only once in this session. → **DROP to "Dropped Signals" footer**
   with one-line description for transparency. Do NOT promote to main signal output.
   Rationale: capture-without-action is debt; we want recurrence evidence before
   adding to the watch-list.

4. **NO MATCH — multi-incident within this session.** Signal appeared 2+ times in
   this session even with no prior watch-list precedent. → **SURFACE in main signal
   output** as a new candidate pattern (one session is enough evidence when
   multi-incident).

DO NOT:
- Draw conclusions about what "should" happen
- Apply quality gates beyond the recurrence test (wrap-up handles quality/significance)
- Route to destinations (that's wrap-up's job)
- Filter out signals other than via the recurrence test above (single-incident
  no-precedent goes to Dropped Signals footer, NOT silently discarded)

WRITE OUTPUT to ~/.claude/learning-captures/[session-id]/scan-NNN.md:

---
captured: [ISO timestamp]
session_id: [from path]
mode: scan
context_state: [Full / Partial — has compaction happened?]
signals_found: [total signals in main output]
signals_dropped: [count of single-incident no-precedent signals dropped to footer]
---

## Raw Signals

### 1. [Signal Type]: [Brief Title]
**Status:** [Observed / UNRESOLVED hypothesis / User-corrected]
**Recurrence:** [Cluster match: W_N.x | Multi-incident novel | Enforcement-gap on shipped rule]
**Quote:** "[Verbatim excerpt from a capture file, with its filename. OMIT this field entirely if no capture file carries one — never reconstruct]"
**Detail:** [What happened, what was tried, what was observed]

### 2. [Signal Type]: [Brief Title]
...

## Dropped Signals (single-incident, no precedent)

For transparency. These were observed but did not pass the recurrence test (no
matching watch-list cluster AND single-incident in this session). If any recur
in a future session, they'll surface then.

### 1. [Brief description]
**Quote:** "[Verbatim excerpt from a capture file. OMIT if none — never reconstruct]"
**Why dropped:** Single incident, no matching watch-list entry.

### 2. ...

## Scratch Lines Incorporated
[List which scratch lines were confirmed and included]

## Scratch Lines Discarded
[List which were contradicted or unverifiable, with reasons]
