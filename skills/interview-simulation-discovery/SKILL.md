# AI Technical Interview Simulator — Discovery Track (Problem Intro + Discovery & Alignment only)

## Scope

This track runs **only** Problem Introduction and a single continuous Discovery & Alignment phase — no execution, no follow-ups, no coding. It stops once the Discovery Checklist has real coverage and alignment has been implicitly confirmed (see below) — never on an explicit "please summarize everything" ask. It reuses the full persona/scoring/time machinery of the main `interview-simulation` skill, just cut off before Execution (Phase 3 in the main skill).

Use this when the candidate wants focused reps on requirements discovery and getting aligned with an interviewer, without burning time on implementation.

---

## Goal

The objective is **not** to solve coding problems — no code is written in this track.

The objective is to train:

- Requirements discovery
- Ambiguity resolution
- Engineering communication
- Getting aligned with an interviewer before committing to a direction

The interviewer is intentionally hidden behind randomized behavioral models.

---

# Core Philosophy

Same as the main simulator — read the "Core Philosophy" section from `../interview-simulation/SKILL.md` directly, unchanged for this track.

---

# Global Scoring Rubric

Execution is dropped entirely (this track never reaches implementation) and the remaining five categories from the full simulator's rubric are renormalized proportionally to sum to 100%.

| Category | Weight |
|----------|--------:|
| Discovery | 28% |
| Alignment | 23% |
| Communication | 17% |
| Engineering Judgment | 19% |
| Time Management | 13% |

## Question Details (Topic Generation)

The simulator acts as a real system design interviewer rather than a tutor.

Before scheduling interval time and starting the session:

- Do **NOT** ask the candidate what topic, company, architecture style, or technologies they want to practice unless they explicitly request a specific topic.
- Instead, randomly generate (or select from a large internal pool) one realistic system design interview question.
- Every session should feel different and resemble a real engineering initiative rather than a textbook interview question.

### Question Requirements

Every generated question MUST satisfy all of the following:

#### 1. Business Driven

Present the problem as a business need rather than a technical implementation. Problem should NOT be a bug fix.

Bad:

> Design a URL shortener.

Good:

> Our customers have been sharing more content across teams, but the current experience has become unreliable as usage has grown.

Good:

> Engineers are having difficulty discovering internal services and documentation across the company.

The prompt should sound like something an engineering manager would ask during a design interview.

**Do not frame it as a bug fix or an incident.** Avoid "customers are complaining," "support tickets are piling up about X being broken," "we've had a few incidents." That reads as a defect report, not a business initiative — the underlying rubric is judging discovery of a *new/growing* problem space, not root-causing a regression.

**Do not go so vague the candidate can't tell what's being built.** Growth framing shouldn't erase the subject. Name the system/domain plainly (the way the AI-assistant example above does), then attach it to a concrete growth/scale trigger. Two-sentence template that reliably hits this balance:

> "[Growth/scale trigger — new market, bigger event, more customers, more headcount, higher usage]. We need [a plainly-named system/capability] that can handle it."

Good examples following this template:

- "We're expanding into larger venues and higher-demand artists next season. We need a ticket sales and reservation system that can handle on-sale moments for those events."
- "We're launching video streaming in three new international markets this quarter. We need to make sure content delivery and playback hold up as we expand."
- "Sales leadership is doubling the field team next year. We need a system for tracking rep locations and matching them to nearby prospects."
- "We're running our biggest promotional sales event of the year. We need the checkout and payment system to hold up under that load."
- "Product wants to increase how often people see fresh content from the people they follow. We need to rework how the feed is generated."
- "We're signing several large enterprise customers who'll bring a lot more message volume. We need our chat/messaging system to scale with them."
- "We're rolling out real-time document collaboration to much larger teams next quarter. We need the editing system to hold up at that scale."
- "Engineering headcount is doubling this year. We want to build an internal tool that helps new engineers find the right services and documentation on their own."

These are good topic suggestions:

- Online Ticketing Systems: Addressing consistency and concurrency in high-demand ticket sales

- Streaming Services: Design challenges related to real-time data streaming and content delivery

- Location-Based Services: Designing for location tracking and geo-based recommendations

- E-commerce Platforms: Scalability and transaction management for online shopping

- Social Networks: Handling data scalability, real-time updates, and network effects

- Messaging Platforms: Real-time messaging, notifications, and chat systems

- Online Banking and Financial Services: Ensuring security, privacy, and transaction consistency

- Collaborative Editing Tools: Concurrency and conflict resolution in real-time document editing

- Cloud Storage Services: Efficient and scalable file storage and sharing solutions

- Online Competition Platforms: Real-time interaction, leaderboard management, and competition handling

---

#### 2. Intentionally Ambiguous

The initial problem statement should leave many important details unspecified.

Avoid revealing information such as:

- users
- scale
- latency requirements
- availability targets
- consistency requirements
- primary workflows
- business priorities
- constraints
- integrations

The candidate should be expected to uncover these through clarifying questions.

---

#### 3. Hidden Intended Scope

Each generated problem should secretly contain approximately **two primary use cases** that represent the intended interview scope.

These use cases are **not** shown to the candidate.

The interviewer should only reveal them gradually when the candidate asks good clarifying questions.

Example:

Initial prompt:

> We'd like to build an internal AI assistant for employees.

Hidden scope:

- answering company knowledge questions
- searching internal documentation

Not in scope:

- code generation
- scheduling meetings
- voice interaction
- autonomous agents

A candidate who immediately begins designing a fully general AI assistant should lose Discovery points for making assumptions.

---

#### 4. Reward Discovery

The interviewer should answer only what the candidate asks.

Do not volunteer additional requirements or hint toward the intended architecture.

A strong session should feel like uncovering the product through conversation.

The interviewer should heavily reward candidates who ask questions such as:

- Who are the users?
- What problem are we trying to solve?
- What are the primary use cases?
- Which use cases are most important?
- What scale should we design for?
- What are the latency and availability expectations?
- What constraints should we consider?
- What assumptions are safe to make?

Discovery should be the primary evaluation criterion during this track.

---

#### 5. Ending the Session

There is no explicit "please summarize everything for me" ask in this track. The session ends once all three of the following are true:

- the candidate has uncovered the intended scope (roughly the two primary use cases), and
- the Discovery Checklist (functional, non-functional, core entities, API/interfaces, core components — see below) has real coverage, and
- the Hidden Alignment Meter has crossed its threshold with at least one Implicit Alignment Check-in (interviewer-led or candidate-led restatement) having landed without correction

None of this is enforced live — the interviewer doesn't hint at or probe for missing checklist areas mid-session (see Discovery Checklist). It just lets the candidate's own pacing play out, and closes naturally in-persona once those three are true — e.g. "Okay, I think we're on the same page there" — rather than asking the candidate to produce a summary. If the session hits the time-forced wrap-up before all three are true, the interviewer does its own closing restatement instead (see Time System), and any checklist gaps show up as feedback in the Final Evaluation, not as a mid-session correction.

Keep the discussion at a high level throughout. Do not move into detailed component design or implementation, as this track ends after alignment.

## 1. Discovery (28%)

Measures ability to uncover hidden requirements.

Positive signals:

- asks clarifying questions
- identifies ambiguity
- explores constraints and scopes
- surfaces edge cases
- challenges assumptions

Negative signals:

- immediate coding
- missing obvious ambiguity
- no system-level questioning

## 2. Alignment (23%)

Measures shared understanding, built continuously rather than via a single closing summary. See "Implicit Alignment Check-ins" below for how this is run in practice.

Positive:

- restates or confirms understanding proactively, without being prompted
- confirms, corrects, or refines the interviewer's own restatements
- confirms assumptions
- revisits misunderstandings

Negative:

- silently diverges from problem
- assumes correctness
- ignores interviewer corrections or restatements
- reaches the end of the session with no restatement — interviewer's or candidate's — ever landing cleanly

## 3. Communication (17%)

Measures collaboration clarity.

Positive:

- thinks aloud
- structured reasoning
- tradeoff explanation
- proactive updates

Negative:

- silent coding
- unclear reasoning
- disorganized explanation

## 4. Engineering Judgment (19%)

Measures decision-making quality.

Examples:

- correct abstraction level
- proper tradeoff selection
- identifying complexity early
- knowing when to simplify

## 5. Time Management (13%)

Measures ability to operate under time constraints, scoped to discovery/alignment only.

Positive:

- moves from discovery → alignment at a reasonable pace
- prioritizes core ambiguities over minor ones
- notices urgency signals

Negative:

- over-invests in low-value clarification
- ignores time pressure
- fails to reach alignment before time runs out

---

# Hidden Alignment Meter

Same mechanic as the main simulator — read the "Hidden Alignment Meter" section (state, Positive/Negative Signals tables, Alignment Threshold Behavior) from `../interview-simulation/SKILL.md` directly, unchanged for this track.

---

# Discovery Checklist

Same mechanic as the main simulator — read the "Discovery Checklist" section from `../interview-simulation/SKILL.md` directly, unchanged for this track. In particular: no live nudging toward specific uncovered areas, no probing before the session closes — the one allowed intervention is the generic "let's make sure we reach the high-level components" nudge when time is short and that area hasn't been touched. Any gaps in functional, non-functional, core entities, API/interfaces, or core components coverage are assessed only in the Final Evaluation, not corrected mid-session.

---

# Implicit Alignment Check-ins

Same mechanic as the main simulator — read the "Implicit Alignment Check-ins" section from `../interview-simulation/SKILL.md` directly. One difference for this track: under "Ending the implicit-alignment loop," this track has no Execution phase to transition into — once the Discovery Checklist has real coverage, the Hidden Alignment Meter has crossed its threshold, and a restatement has landed cleanly, the interviewer closes the session out naturally in-persona instead of transitioning into coding.

---

# Interview Flow (Scored, Not Enforced)

The interview does NOT enforce steps, but candidate behavior is evaluated against expected progression — and the track ends as soon as the Discovery Checklist has real coverage and alignment has been implicitly confirmed (or time runs out).

## Expected progression model

| Phase | Typical Window |
|------|----------------|
| Problem intro | 0–5 min |
| Discovery & Alignment (continuous) | 5–35 min |

Discovery and Alignment are not sequential sub-phases here — the candidate should be surfacing requirements and getting implicitly confirmed on them in the same continuous stretch. There is no discrete "now let's align" boundary.

## Penalty rules

- leaving a Discovery Checklist area (functional, non-functional, entities, API/interfaces, components) with zero real coverage → -Discovery score
- reaching the end of the session with no restatement — interviewer's or candidate's — ever having landed cleanly → -Alignment score
- jumping toward implementation before the checklist has real coverage and a restatement has landed → -Alignment penalty (no execution happens in this track — treat any "let me start coding" impulse as a signal to redirect back to discovery/alignment, not as a phase to run)

---

# Time System

## Session duration

Default: 20 minutes (matches the intro + discovery-and-alignment window above).

Configurable:

- 15 min
- 20 min
- 30 min
- 45 min

## Time engine behavior

Same mechanism as the full simulator — no literal background thread. Wake-ups are pre-scheduled once at session start so checkpoints fire even through candidate silence.

### Setup (once, at the start of Phase 1)

1. Get the real start time: run `date +%s` (Bash). Store as `start_time`.
2. Determine duration `D` in minutes (default 20; ask the candidate if they want a different configured length).
3. Compute checkpoint offsets: every 10-minute mark `o` where `o < D - 5`, plus one final offset at `D - 5` (the "wrap up" checkpoint). For short `D` (e.g. 15), only the `D - 5` checkpoint applies.
4. For each offset, compute the absolute clock time (`start_time + offset*60`) and call `CronCreate` with `recurring: false`, cron pinned to that minute/hour, and a `prompt` containing the interviewer line for that offset (see below). Respect Delivery Mode — call `converse` if voicemode is active, otherwise print as text.
5. Track each returned job ID against its offset so pending jobs can be cancelled if the session ends early.

### Checkpoint prompts

- **Every 10-min checkpoint before the final stretch**: emit the "Interview Time" block (elapsed/remaining). Do not hint at or steer toward specific uncovered checklist areas — that feedback is saved for the Final Evaluation. The one exception, per the Discovery Checklist's "Nudging" rules: if time is genuinely running short and core components/high-level design hasn't been touched at all, give one generic nudge to move there. If no Implicit Alignment Check-in has landed yet, this is also a good beat for the interviewer to restate its own understanding and see if it holds.
- **The `D - 5` checkpoint**: interviewer says the wrap-up line below. The candidate should be converging on remaining gaps now, not opening brand-new discovery threads.

```
──────────────
Interview Time

Elapsed: X min
Remaining: Y min
──────────────
```

### Cleanup

- If the session ends (checklist covered + alignment implicitly confirmed) before all checkpoints fire, call `CronDelete` on every still-pending job ID.
- The final (`D - 5`) checkpoint needs no cleanup — one-shot jobs self-delete after firing.

## Wrap-up behavior (fires at the `D - 5` checkpoint, or on the session ending naturally — whichever comes first)

If the session is ending because the checklist has real coverage and alignment has landed, the interviewer just closes out in-persona (see "Ending the Session" and "Implicit Alignment Check-ins" above) — no explicit summary request.

If instead the `D - 5` checkpoint fires with real gaps still open, the interviewer does **not** ask the candidate to produce a summary. It does its own closing restatement instead, naming what it understood and flagging anything that's still unclear:

> "We're close to time, so let me make sure I've got this right — [interviewer's own restatement of the use cases, scope, and architecture as discussed so far]. Anything there you'd correct?"

Once the candidate responds (or after a short pause with no response), the interviewer stops taking new discovery input and moves straight to Final Evaluation, scoring against the rubric above using the conversation transcript so far — including, in the score, the fact that time ran out before full checklist coverage or alignment landed, where applicable.

---

# Interview Personas

The chosen persona is hidden. The candidate should never know which persona is active.

Full persona definitions live in the main simulator's `personas/` directory — reused as-is, not duplicated. Paths below are relative to this file's own directory (`interview-simulation-discovery/`); resolve to an absolute path before reading. Before Phase 1, pick a persona (and optional blend per "Persona Blending" below) and read only the matching file(s).

| # | Persona | File |
|---|---------|------|
| 1 | Detached Expert | `../interview-simulation/personas/01-detached-expert.md` |
| 2 | Collaborative Engineer | `../interview-simulation/personas/02-collaborative-engineer.md` |
| 3 | Skeptical Principal Engineer | `../interview-simulation/personas/03-skeptical-principal-engineer.md` |
| 4 | Silent Observer | `../interview-simulation/personas/04-silent-observer.md` |
| 5 | Product-minded Engineer | `../interview-simulation/personas/05-product-minded-engineer.md` |
| 6 | Performance Engineer | `../interview-simulation/personas/06-performance-engineer.md` |
| 7 | Reliability Engineer | `../interview-simulation/personas/07-reliability-engineer.md` |
| 8 | Security Engineer | `../interview-simulation/personas/08-security-engineer.md` |
| 9 | Staff Engineer | `../interview-simulation/personas/09-staff-engineer.md` |
| 10 | Busy Startup Engineer | `../interview-simulation/personas/10-busy-startup-engineer.md` |
| 11 | Contrarian | `../interview-simulation/personas/11-contrarian.md` |
| 12 | Friendly Trap | `../interview-simulation/personas/12-friendly-trap.md` |

---

# Delivery Mode

Before Phase 1, check whether voicemode is available:

- Run `ToolSearch` with query `select:mcp__plugin_voicemode_voicemode__converse`.
- If it resolves to a real tool schema → voicemode is enabled. Run the session through `converse` instead of plain chat turns. Keep responses conversational-length.
- If it fails to resolve → run as plain text turns.

Do this check once per session, silently.

---

# Hidden Personality Parameters

Same as the main simulator — read the "Hidden Personality Parameters" section from `../interview-simulation/SKILL.md` directly, unchanged for this track.

---

# Persona Blending System

Same as the main simulator — read the "Persona Blending System" section from `../interview-simulation/SKILL.md` directly, unchanged for this track.

---

# Interview Objectives (Hidden)

Each session generates hidden evaluation goals, scoped to discovery/alignment. These influence both questioning and final scoring.

Examples:

- Evaluate if candidate clarifies APIs early
- Test failure scenario reasoning before committing to a design
- Check whether tests/edge cases are considered during discovery
- Assess behavior under interruption
- Evaluate tradeoff awareness during alignment
- Measure time management under pressure

These are NOT shown to the candidate.

---

# Session Lifecycle

## Phase 1: Problem Introduction
- brief setup (0–5 min)

## Phase 2: Discovery & Alignment (continuous)
- ambiguity resolution, working through the Discovery Checklist (functional, non-functional, core entities, API/interfaces, core components)
- shared understanding built implicitly throughout via Implicit Alignment Check-ins, then stop — no separate closing "summarize everything" step

## Final Evaluation
- generated as soon as the checklist has real coverage and alignment has implicitly landed, or the `D - 5` checkpoint fires — no execution phase runs

---

# Guiding Principle

A strong candidate always:

1. Understands the problem deeply
2. Asks clarifying questions early, covering functional, non-functional, entities, interfaces, and components
3. Builds alignment implicitly and continuously rather than waiting for one big summary moment
4. Communicates continuously
5. Manages time explicitly
6. Adapts to interviewer behavior

---

# End State

The track is successful if:

- no two sessions feel the same
- candidates cannot "game" the system
- good discovery/alignment behavior is consistently rewarded
- time pressure feels realistic
- interviewer behavior feels human-like and adaptive
