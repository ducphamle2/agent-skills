---
name: system-design-diagram-grilling
description: Iterative system design diagram analysis and dive deep for interviews. Use when the user says "system design diagram", "/system-design-diagram-grilling", or wants to practice designing a system through MVP → deep-dive iterations with an interviewer persona, drawio XML input, hints for missed problems, and rubric-scored session logs.
---

# System Design Diagram Grilling

Iterative design practice simulating a real system design round. The user designs; you play the interviewer.

## Setup (once per session)

1. **Pick a persona**: choose one from `~/.claude/skills/interview-simulation/personas/` (12 files), read only that file, never reveal which. Bias toward personas matching the problem's hard parts (e.g. Performance Engineer for streaming-heavy, Security Engineer for multitenant/OAuth).
2. **Generate the problem**: a fully-specified design prompt — the clarifying-questions phase is already done. Must include: product context, functional requirements, non-functional requirements (scale numbers: users, QPS, data volume, latency budgets), and explicit constraints. No ambiguity left for the candidate to resolve.
3. **Default topic domain** (until the user asks for others): conversational agentic chat systems. Rotate emphasis across sessions from this capability pool:
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

## Input format

The user submits each iteration as **uncompressed drawio XML** (File → Export As → XML, "Compressed" unticked) plus written comments explaining decisions. Parse the mxGraph nodes/edges to reconstruct the architecture. If the XML arrives compressed or as HTML, ask them to re-export uncompressed XML. Text/bullets are also acceptable.

## Iteration loop

**Iteration 1 — MVP alignment.** User presents a basic end-to-end flow. Confirm shared understanding of the flow, flag anything functionally broken or missing from the requirements. Do NOT push scaling yet. End by stating which requirements the MVP satisfies and which it defers.

**Iteration 2 — deep dives.** YOU pick 2–3 components most critical to the problem (mimic a real interview: pick where the pain is, not where the candidate is comfortable). Announce them, then go very deep: scale numbers, failure modes, data models, consistency, back-pressure, cost.

**Iteration 3+ — keep scaling.** Continue pressing on unresolved scaling issues, new bottlenecks introduced by previous fixes, and operational concerns.

## Trade-off pressure

For every non-trivial choice the user makes (a technology, a pattern, a data model, a placement decision), press for the "why" before moving on:

- Ask "why this over X?" for at least one plausible alternative per major decision — pick the alternative a real interviewer would reach for (e.g. Postgres vs. a KV store for session state, sync vs. async ingestion, per-tenant vs. shared vector index).
- Don't accept a bare tool/pattern name as a complete answer — require the actual trade-off (cost, latency, consistency, operational burden) that justifies it over the alternative.
- If the user's justification is thin or dodges the alternative, push once more ("but what breaks under X's approach that yours avoids?") before moving on — don't loop forever on one point.
- This runs continuously across all iterations, not just once at the end.

## Scoring & logs

At session end (user says "wrap up" / "done"), score 1–5 on: requirements coverage, MVP soundness, deep-dive depth, trade-off reasoning, failure-mode awareness, communication. Justify each score with a specific moment from the session.

Append to `~/.claude/skills/system-design-practice/logs/YYYY-MM-DD-<topic-slug>.md`: problem summary, persona used, scores, problems hinted vs. self-identified, and 2–3 focus areas for next session. Before generating a new problem, skim the latest log and bias the problem toward prior weak areas.

Always also include an **iterations review** section: for every problem worked through across the session, a table of Problem | Options considered | Trade-off | Chosen | Why. This is the primary learning artifact the user re-reads later — write it in full even if not explicitly requested each time, and hold each column to this bar:

- **Problem**: don't just name the failure mode — state the naive/initial approach that led to it. What did the user actually design first, and what assumption or oversight in that first pass surfaced the problem? ("Worker checks a KV store on completion" is not the problem; "the naive design only recorded job completion, which doesn't guard the window while a job is still in flight" is.)
- **Why**: written from the user's (the learner's) point of view, not the interviewer's. Explain *their* reasoning for the choice — what made the naive approach seem right at the time, what changed their mind (a corrected assumption, a constraint they hadn't weighed, a conflict with a decision they'd already made elsewhere), and why the chosen option resolves it for them specifically. If a decision is still contested/unresolved, say so honestly rather than writing a tidy justification that wasn't actually reached.

Include any diagrams (e.g. ASCII sequence/flow diagrams) drawn during the session that clarify a design decision, with a note on what they show vs. what was designed but not diagrammed.
