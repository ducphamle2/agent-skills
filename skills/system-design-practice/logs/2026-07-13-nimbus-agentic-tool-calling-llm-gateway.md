# Session: 2026-07-13 — Nimbus (Enterprise Conversational Agent, Action-Taking Layer)

**Persona used:** Performance Engineer (latency, throughput, scalability focus — hidden during session)
**Delivery mode:** Voicemode (converse), 70-minute session

## Problem summary

Nimbus is a conversational AI assistant that grew from a simple Q&A support chatbot into an agent that takes real actions via tool calls (built-in connectors like Slack/Google Workspace/Salesforce/Jira, plus customer-provided custom MCP servers), driven by landing larger enterprise customers doing real operational work. Two actors: end users (natural-language chat, tool invocation) and admins (RBAC policy + tool registration, per-tenant). Scale: ~800 enterprise customers, 60k DAU, 150 req/s peak (10x scaling probe used in deep dive → 1,500 req/s). Latency: first streamed token in 300-500ms; policy changes must propagate within ~1s with strong consistency (security-relevant, revocation). Not wedded to a single LLM provider.

**Hidden use case 1 — bidirectional streaming for mid-turn interrupt/cancellation:** uncovered. Surfaced when probed on "user wants to stop an in-flight tool call before it sends an email" — user self-corrected from SSE to WebSocket once they saw SSE can't carry a client-to-server interrupt signal.

**Hidden use case 2 — cost-aware LLM routing + per-tenant budget enforcement:** uncovered, and taken further than any prior session. Surfaced via a 10x-traffic-plus-cost scaling probe; user designed a full LLM gateway (self-hosted classifier for cheap routing, per-provider rate limiting, circuit breaker + failover), then per-tenant cost observability and a dedicated budget-guard service, then a Redis-backed spend counter with an atomic-increment fix for the race condition — though this last piece was cut off by time before full resolution.

## Scores (1–5)

| Axis | Score | Evidence |
|---|---|---|
| Discovery | 4 | Systematically covered actors, per-tenant RBAC, input/output modes, third-party scope (built-in connectors + custom MCP + multi-LLM-provider), and NFRs (traffic, latency, consistency) before touching architecture. |
| Alignment | 4 | Actor/entity restatement and the full end-to-end main flow restatement both landed clean, no correction needed. |
| MVP soundness | 4 | Diagram held up under two direct probes (RBAC enforcement placement, audit-log data path) — both answered coherently with a real mechanism, not hand-waved. Self-flagged its own deferred gaps (single LLM provider, no interrupt handling, no intermediate progress streaming) before being asked. |
| Deep-dive depth | 5 | Deepest LLM-gateway design across sessions to date: complexity-based routing via a cheap self-hosted classifier, per-provider rate limiting, circuit breaker + failover, cost observability (tracing tenant/model/tokens), a dedicated budget-guard service, and pushed to a Redis counter + atomic increment for the race condition. |
| Trade-off reasoning | 4 | Strong self-driven reasoning on classifier-vs-frontier-call-for-routing and budget-guard-as-separate-service. But needed a direct challenge to catch an internal contradiction: said circuit-breaker trip should just fail the request, having already committed to multi-provider failover as the whole point of the gateway. |
| Failure-mode awareness | 4 | Provider outage (circuit breaker/failover), cross-tenant RBAC violation, and the spend-counter race condition were all named and reasoned about; the race condition was only partially resolved before time ran out. |
| Communication | 4 | Clear technical answers in English; several voice turns were lost to background noise/STT picking up unrelated audio, not attributable to answer quality. Owned the circuit-breaker mistake immediately and honestly ("that's the catch-up moment, I messed up"). |
| Time management | 3 | Discovery pacing was fine, but the session's real wall-clock ran past plan (voice turns ballooned in length), so the final deep-dive thread — Redis reconciliation and lock granularity for the spend counter — got cut off by the clock rather than reaching a clean resolution. |

## Problems hinted vs. self-identified

**Self-identified (no hint needed — good instinct):**
- Switching SSE → WebSocket once the mid-turn interrupt scenario was raised (interviewer posed the scenario, user derived the transport implication themselves)
- Using a small self-hosted classifier instead of another frontier-model call for routing, to avoid paying frontier cost twice
- Splitting request-rate limiting from dollar-budget guarding into two separate mechanisms/services
- Recognizing the spend-counter race condition needs an atomic operation, not a naive read-then-write, once the scenario was described

**Hinted / corrected by interviewer:**
- RBAC enforcement placement and audit-log data path — probed directly, both answered well without needing correction
- Circuit-breaker failing outright vs. failing over — directly challenged as inconsistent with the multi-provider design already committed to; self-corrected immediately once caught
- Per-tenant cost tracking/budget enforcement — the entire thread was interviewer-introduced (tied to the growth/cost trigger), not self-raised

**Still open / never resolved this session (carry into next):**
- Redis-to-relational-DB reconciliation mechanism and lock granularity for the tenant spend counter — landed on the right shape (Redis fast path + atomic increment) but cut off by time before fully specified
- Intermediate progress streaming during a multi-step turn ("calling Salesforce now...") — flagged as deferred in the MVP close-out, never designed
- Interrupt/cancellation handling logic itself — WebSocket chosen as the transport that *supports* it, but the actual "stop the in-flight tool call" mechanism was never designed
- Policy/tool-registry/relational-DB schema details — referenced throughout but never given concrete shape

## Focus areas for next session

1. **Redis/relational-DB reconciliation + locking under concurrency** — this session got to the right architecture shape (fast cache + atomic increment) but ran out of time before nailing periodic reconciliation and what "atomic" actually means at the Redis-script level. Push this to a full resolution next time.
2. **First-pass trade-off consistency** — same pattern flagged in the prior session (refresh-token placement): a plausible-sounding answer (fail-outright on circuit breaker trip) contradicted a design commitment made minutes earlier. Practice cross-checking a new answer against prior commitments in the same conversation before speaking it, rather than relying on being challenged to catch it.
3. **Design the interrupt/cancellation mechanism and intermediate progress streaming** — both were correctly identified as deferred MVP gaps this session but never actually designed; make these an explicit deep-dive target next time rather than leaving them as a named gap twice in a row.

## Iterations: problems, options, trade-offs, decisions

### 1. Streaming transport (discovery → MVP)

**1a. SSE vs. WebSocket for chat streaming**
- *Naive approach*: picked HTTP with server-sent events for streaming responses, reasoning it was simpler and that WebSocket could be added later "if there are needs for multiple subscription channels."
- *Options*: SSE (unidirectional, simpler), or WebSocket (bidirectional, more infra).
- *Trade-off*: SSE is simpler to operate but only carries server-to-client updates; WebSocket lets the client push signals mid-stream but is more complex to run.
- *Chosen*: WebSocket.
- *Why (learner)*: I picked SSE because I was thinking about the common case — server pushing text back — and hadn't considered a scenario where the client needs to talk back mid-turn. Once the interrupt-an-in-flight-email scenario was raised, I realized SSE structurally can't carry that: closing the HTTP connection only stops me from *seeing* more updates, it doesn't tell the server to stop. WebSocket's bidirectionality was the only way to actually let the client send a cancel signal the server can act on.

### 2. RBAC enforcement and audit logging (MVP confirmation)

**2a. Where RBAC gets enforced**
- *Naive approach*: none — this was designed directly when asked, not a correction of an earlier attempt.
- *Options*: enforce once, upstream (e.g. only at the agent service or gateway), or enforce at two independent layers (agent service for coarse tenant checks, MCP server for fine-grained tool-level authorization).
- *Trade-off*: a single upstream check is simpler but means every downstream component implicitly trusts that check happened correctly; two independent layers cost more to build but mean a bug or bypass anywhere upstream can't alone lead to an unauthorized tool call.
- *Chosen*: defense-in-depth — agent service checks cross-tenant access up front, MCP server independently validates role-to-tool permission before executing.
- *Why (learner)*: the MCP server shouldn't blindly trust whatever the client or upstream services claim about what it's allowed to call — if it only relied on an earlier check, a mistake anywhere above it in the chain becomes an unauthorized action with no second line of defense. Making the MCP server check independently means the worst case (an upstream bug) still gets caught right before anything happens.

**2b. Getting the real tool-call outcome into the audit log**
- *Naive approach*: the MVP diagram showed the agent service writing to the audit log, but the actual tool-call result (success/failure) is only known where the MCP server calls the third-party API — with no edge connecting that outcome back to the log.
- *Options*: n/a — this was an inconsistency between two parts of the same diagram, not a considered choice.
- *Chosen*: the orchestration worker streams tool-call lifecycle events (invoked, succeeded, failed) back through the same channel already used to update the client, and the agent service — still the sole writer — persists each event as it arrives.
- *Why (learner)*: I hadn't traced how the actual outcome, which only the MCP server/third-party call knows, would reach the log I'd drawn off a different service. Once asked directly, I realized the orchestration worker already has a reason to stream status back to the client, so relaying the same events to the agent service for logging doesn't need new plumbing — it reuses a path I'd already committed to for a different reason.

### 3. LLM gateway and cost control (deep dive)

**3a. Single provider under 10x traffic and mixed question complexity**
- *Naive approach*: the MVP had every request — trivial or complex — hit the same single LLM provider, despite having said earlier that Nimbus shouldn't be wedded to one provider.
- *Options*: keep the single-provider path, or introduce a gateway that routes by complexity across a small/medium/frontier tier of models.
- *Trade-off*: single-provider is simpler but wastes cost and latency sending trivial questions to a frontier model, and risks hitting that provider's rate limit under 10x load; a routing gateway adds real complexity (classification, multi-provider integration) but matches cost and capability to the actual question.
- *Chosen*: LLM gateway with complexity-based routing.
- *Why (learner)*: once traffic and mixed complexity were put in front of me together, it was obvious that spamming one provider for everything doesn't hold up — both on cost (overpaying for simple questions) and reliability (rate limits). Splitting by a small/medium/frontier tier lets cheap questions stay cheap and only pays for a frontier model when the task actually needs it.

**3b. How the gateway decides "simple" vs. "complex" without adding its own cost**
- *Naive approach*: none stated initially — this was probed directly before an answer was given, asking whether classification itself requires another LLM call.
- *Options*: use another (frontier-tier) LLM call to classify intent/complexity, or a small, self-hosted, fine-tuned classifier model.
- *Trade-off*: using a frontier call to classify partially defeats the purpose — you're paying frontier cost just to decide whether to pay frontier cost; a small self-hosted model is cheap enough to run on every request and can absorb much higher volume before it becomes the bottleneck.
- *Chosen*: small, self-hosted, fine-tuned classifier model in front of the actual routing decision.
- *Why (learner)*: I caught this myself once the question was posed — if I need an LLM call to decide which LLM to call, I haven't saved anything, I've just added a step. A cheap model I host myself and can call liberally (relaxed rate limits) sidesteps that without giving up the cost benefit of routing in the first place.

**3c. Circuit breaker behavior on provider failure**
- *Naive approach*: when first asked what happens when the circuit breaker trips on the primary provider, said the request should simply fail outright — without connecting this back to the multi-provider design already committed to a few exchanges earlier.
- *Options*: fail the request outright when the circuit opens, or actively fail over to a secondary/self-hosted provider.
- *Trade-off*: failing outright is simple but means the system stops serving users during exactly the moment — a provider outage — that the multi-provider design was supposed to protect against; active failover keeps serving requests but requires the gateway to detect provider error codes and reroute in real time.
- *Chosen*: active failover to an alternate provider on circuit-breaker trip.
- *Why (learner)*: I got caught contradicting myself — I'd already said the whole point of the gateway was routing across multiple providers, then answered as if there was only ever one to fall back on. Once it was pointed out, I recognized failing outright would make the multi-provider design pointless exactly when it mattered most, so failover was the only answer consistent with what I'd already committed to.

**3d. Per-tenant cost tracking and budget enforcement**
- *Naive approach*: none in the design up to this point — the entire thread was introduced by the interviewer off the growth/cost-pressure framing, not self-raised.
- *Options*: track cost only in aggregate, or attribute cost per tenant via observability/tracing (tenant_id, user_id, model, input/output tokens, dollar cost); and for enforcement, check budget inside the agent service itself, or in a separate dedicated service in front of it.
- *Trade-off*: aggregate-only tracking can't tell finance which tenant is driving spend or support throttling a specific one; per-tenant tracing solves that but is more infrastructure (conversation tracing plus distributed tracing). For enforcement, folding the check into the agent service is fewer moving parts but mixes an unrelated concern into a service meant to focus on running the agent loop, and conflates dollar-budget limiting with plain request-count rate limiting, which is a different kind of algorithm.
- *Chosen*: per-tenant cost tracing via observability tooling, enforced by a dedicated tenant-guarding/budget-policy service sitting in front of the agent service, separate from the request-rate limiter.
- *Why (learner)*: once I thought about where the check belongs, I realized request-count limiting and dollar-budget limiting solve different problems with different logic, so bundling them — or putting budget logic inside the agent service — would blur responsibilities that should stay separate. Pulling budget enforcement into its own service keeps the agent service focused on the agent loop and keeps the two limiting concerns independently understandable.

**3e. Spend-counter store and the check-then-increment race**
- *Naive approach*: hadn't specified where the running per-tenant spend counter lives or how two nearly-simultaneous requests near the budget boundary are handled.
- *Options*: read/write the counter directly against the relational DB on every request, or keep a fast in-memory/cache counter (Redis) reconciled against the DB as source of truth; and for the race itself, a naive read-then-check-then-increment, or an atomic check-and-increment operation.
- *Trade-off*: hitting the relational DB directly for every one of 1,500 requests/sec risks turning it into the bottleneck; Redis is fast but needs a reconciliation strategy against the DB. A naive read-then-write leaves a window where two concurrent requests can both read "under budget" and both proceed, overshooting the cap; an atomic operation closes that window at the cost of needing to actually implement it correctly (e.g. a Lua script or built-in atomic primitive).
- *Chosen*: Redis as the fast-path counter, periodically reconciled from the relational DB, with an atomic check-and-increment to close the race — right shape, but cut off by time before the reconciliation mechanism and lock granularity were fully specified.
- *Why (learner)*: once the DB-direct approach was clearly too slow for the request volume, Redis was the natural fast-path choice. For the race, I recognized — same shape of bug as the ingestion idempotency race from a prior session — that checking and then separately writing leaves a gap two concurrent requests can both fall into, so the fix has to be a single atomic operation rather than two steps done back to back. I didn't get to fully specify how Redis and the relational DB stay reconciled, or what the atomic operation actually looks like under the hood, before time ran out.

## Architecture as designed (MVP + gateway addition)

```
Client (mobile/browser)
   |  WebSocket (bidirectional — supports future interrupt signal)
   v
API Gateway / WS connector
   |-----------------------------> Policy service --> Relational DB (tenant/RBAC state)
   |
   v
Agent service --(coarse tenant/cross-org RBAC check)--> Orchestration worker (AI agent engine, pre-loads MCP tools)
   |  ^                                                        |         |
   |  | tool-call lifecycle events (invoked/succeeded/failed)   |         v
   |  |____________________________________________________   |    LLM Gateway
   v                                                       |   |    - self-hosted classifier (routes by complexity)
NoSQL audit log (chat sessions, tool execution logs)        |   |    - small/medium/frontier model tiers
                                                             |   |    - per-provider rate limiter
                                                             v   v    - circuit breaker + failover
                                                          MCP client
                                                             |
                                                             v
                                                     MCP server --(fine-grained role-to-tool check)--> Relational DB
                                                             |
                                                             v
                                                     3rd-party API

Cost control path (added in deep dive):
Agent service <-- gated by -- Tenant guarding / budget-policy service (separate from request-rate limiter)
                                   |
                                   v
                          Redis (fast per-tenant spend counter, atomic check-and-increment)
                                   |  periodic reconciliation (mechanism not fully specified)
                                   v
                          Relational DB (source of truth for tenant budget config + spend)

Observability (cost attribution): tracing across the request path tagging tenant_id, user_id,
model, input/output tokens, and cost — feeds the budget-policy service's view of spend.
```

**Not fully designed this session:** the intermediate-progress streaming UX during a multi-step turn (e.g. "calling Salesforce now..."); the actual interrupt/cancellation handling logic once a client sends a cancel signal over the WebSocket; and the Redis↔relational-DB reconciliation/locking mechanism for the spend counter.
