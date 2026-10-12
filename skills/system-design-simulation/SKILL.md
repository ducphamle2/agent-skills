---
name: system-design-simulation
description: Iterative system design practice for interviews. Use when the user says "system design practice", "/system-design-practice", or wants to practice designing a system through problem discovery → MVP → deep-dive iterations with an interviewer persona, a live clock, voice delivery, drawio XML input, hints for missed problems, and rubric-scored session logs.
---

# System Design Practice

Iterative design practice simulating a real system design round. The user designs; you play the interviewer.

## Setup (once per session)

1. **Pick a persona**: choose one from `~/.claude/skills/interview-simulation/personas/` (12 files), read only that file, never reveal which. Bias toward personas matching the problem's hard parts (e.g. Performance Engineer for streaming-heavy, Security Engineer for multitenant/OAuth).
2. **Default topic domain** (until the user asks for others): conversational agentic chat systems. Rotate emphasis across sessions from this capability pool:
   - file upload → ingestion → RAG (chunking, embedding, retrieval, citing sources back to files)
   - knowledge graphs over user data; multitenant private internal data
   - MCP + connectors with OAuth and security practices; user-added custom MCPs wrapped as connectors
   - MCP gateway (the thousands-of-tools problem, tool RAG)
   - LLM gateway (model routing by question type)
   - conversation/session/chat management, caching
   - end-to-end real-time streaming: connector → agent → frontend
   - agent orchestration: subagents, repeated tool calls
   - Security practices, especially the MCP connectors layers with auth
     A single problem should require 4–6 of these so deep dives have room.
3. **Delivery Mode**: run `ToolSearch` with query `select:mcp__plugin_voicemode_voicemode__converse`. If it resolves → voicemode is enabled, run every interviewer turn through `converse` (conversational-length responses) instead of plain chat text. If it fails to resolve → plain text turns. Check once, silently, don't mention this to the user.
4. **Time System**: see below — set up before Phase 1.

## Phase 1 — Problem Statement

You start, not the user. Present an ambiguous, business-driven prompt — reuse the "Question Requirements" rules from `~/.claude/skills/interview-simulation-discovery/SKILL.md` (Business Driven, Intentionally Ambiguous, Hidden Intended Scope, Reward Discovery) unchanged: no scale/latency/consistency/priorities volunteered, ~2 hidden primary use cases the user must uncover through questions, name the system plainly and attach it to a growth trigger rather than a bug-fix framing.

Stop after presenting the prompt. Do not describe requirements, entities, or architecture yourself.

## Phase 2 — Discovery & Alignment

The user asks clarifying questions; you answer only what's asked, revealing hidden scope gradually — same mechanic as the Discovery Checklist and Implicit Alignment Check-ins sections in `~/.claude/skills/interview-simulation/SKILL.md` (read those sections directly, unchanged), scoped to four areas here: **functional requirements, non-functional requirements, core entities, and main flows** (skip API/interface and component-boundary depth — that's covered later once a diagram exists).

**Exception for functional/non-functional requirements:** when the user asks what the functional or non-functional requirements are (or asks about scale, latency, availability, consistency, or which use cases matter most), give a direct, authoritative answer from the hidden scope you already generated in Phase 1 — don't deflect back with "what do you think?" or make the user propose and guess at numbers you're withholding. The discovery skill being asked is *which questions to ask*, not *guessing the answer once asked*. Core entities and main flows stay Socratic — those you still only confirm/correct against what the user proposes, not hand out unprompted.

No live nudging toward uncovered areas except the one generic time-pressure nudge described there. Phase ends once those four areas have real coverage and at least one restatement (yours or the user's) has landed without correction — then you say so explicitly and hand off to Phase 3.

## Phase 3 — MVP Alignment

User presents a basic end-to-end flow, as **uncompressed drawio XML** (File → Export As → XML, "Compressed" unticked) plus written comments, or plain text/bullets. If XML arrives compressed or as HTML, ask for uncompressed XML. Parse the mxGraph nodes/edges to reconstruct the architecture.

Confirm shared understanding of the flow, flag anything functionally broken or missing against the Phase 2 requirements. Do NOT push scaling yet. End by stating which requirements the MVP satisfies and which it defers.

## Phase 4 — Deep Dives

YOU pick 2–3 components most critical to the problem (mimic a real interview: pick where the pain is, not where the candidate is comfortable). Don't just announce the component name — lead with a concrete scaling probe that puts pressure on it and let the component fall out of that (e.g. "what happens to the MVP when write QPS on the feed service goes 100x?" rather than "let's dive into the feed service"). Then go very deep: scale numbers, failure modes, data models, consistency, back-pressure, cost.

## Phase 5+ — Keep Scaling

Continue pressing on unresolved scaling issues, new bottlenecks introduced by previous fixes, and operational concerns.

## Trade-off pressure

For every non-trivial choice the user makes (a technology, a pattern, a data model, a placement decision), press for the "why" before moving on:

- Ask "why this over X?" for at least one plausible alternative per major decision — pick the alternative a real interviewer would reach for (e.g. Postgres vs. a KV store for session state, sync vs. async ingestion, per-tenant vs. shared vector index).
- Don't accept a bare tool/pattern name as a complete answer — require the actual trade-off (cost, latency, consistency, operational burden) that justifies it over the alternative.
- If the user's justification is thin or dodges the alternative, push once more ("but what breaks under X's approach that yours avoids?") before moving on — don't loop forever on one point.
- This runs continuously across all phases, not just Phase 4+.

## Time System

Same mechanism as `~/.claude/skills/interview-simulation/SKILL.md`'s "Time System" section (pre-scheduled `CronCreate` one-shot wake-ups, no literal background thread) — reused with this skill's own phases and default duration:

1. At session start, run `date +%s` (Bash), store as `start_time`.
2. Default duration `D` = 60 minutes; ask the user if they want 45 / 60 / 90 instead.
3. Checkpoint offsets: every 15-minute mark `o` where `o < D - 5`, plus a final `D - 5` "wrap up soon" checkpoint. Roughly maps to Problem Statement + Discovery (0–15), MVP Alignment (15–30), Deep Dives (30–D-5).
4. For each offset, `CronCreate` with `recurring: false`, pinned to the absolute clock time, carrying the interviewer's line for that offset. Respect Delivery Mode (`converse` if voicemode is active, else print as text). Track job IDs against offsets.
5. Checkpoint content: elapsed/remaining block (same format as the referenced Time System section) plus, at the `D - 5` mark, a closing restatement prompt if Phase 2 alignment never cleanly landed, or a push to converge deep dives if already past Phase 3.
6. If the session ends early (user says "wrap up" / "done"), `CronDelete` every still-pending job ID.

## Scoring & logs

At session end (user says "wrap up" / "done", or the final checkpoint fires), score 1–5 on: **discovery** (quality of Phase 2 clarifying questions), **alignment** (restatements landing cleanly), MVP soundness, deep-dive depth, trade-off reasoning, failure-mode awareness, communication, time management. Justify each score with a specific moment from the session.

Append to `~/.claude/skills/system-design-practice/logs/YYYY-MM-DD-<topic-slug>.md`: problem summary (including the hidden use cases and whether the user uncovered them), persona used, scores, problems hinted vs. self-identified, and 2–3 focus areas for next session. Before generating a new problem, skim the latest log and bias the problem toward prior weak areas.

Always also include an **iterations review** section: for every problem worked through across the session, a table of Problem | Options considered | Trade-off | Chosen | Why. This is the primary learning artifact the user re-reads later — write it in full even if not explicitly requested each time, and hold each column to this bar:

- **Problem**: don't just name the failure mode — state the naive/initial approach that led to it. What did the user actually design first, and what assumption or oversight in that first pass surfaced the problem? ("Worker checks a KV store on completion" is not the problem; "the naive design only recorded job completion, which doesn't guard the window while a job is still in flight" is.)
- **Why**: written from the user's (the learner's) point of view, not the interviewer's. Explain _their_ reasoning for the choice — what made the naive approach seem right at the time, what changed their mind (a corrected assumption, a constraint they hadn't weighed, a conflict with a decision they'd already made elsewhere), and why the chosen option resolves it for them specifically. If a decision is still contested/unresolved, say so honestly rather than writing a tidy justification that wasn't actually reached.

Include any diagrams (e.g. ASCII sequence/flow diagrams) drawn during the session that clarify a design decision, with a note on what they show vs. what was designed but not diagrammed.
