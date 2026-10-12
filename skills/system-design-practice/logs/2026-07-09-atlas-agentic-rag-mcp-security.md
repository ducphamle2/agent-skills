# Session: 2026-07-09 — Atlas (Enterprise AI Work Assistant)

**Persona used:** Staff Engineer (leadership, tradeoffs, prioritization — hidden during session)

## Problem summary

B2B conversational AI assistant for enterprises. Chat + file upload/RAG + connectors (Drive/Salesforce/Jira) with OAuth + custom MCP servers wrapped as connectors + MCP gateway (3,000+ tools per large tenant) + LLM routing across 3 providers + subagent orchestration. Scale: 4,000 tenants, 150k DAU, 500 msg/sec peak, 1.5PB documents, 40M docs for largest tenant, 300ms p95 retrieval, 99.9% availability. Constraints: tenant isolation everywhere, OAuth tokens are high-value secrets, custom MCP servers must not exfiltrate data or take down the platform, CFO flagged LLM spend + vector storage cost.

## Scores (1–5)

| Axis | Score | Evidence |
|---|---|---|
| Requirements coverage | 4 | Chat, RAG, connectors+OAuth, custom MCP isolation, MCP gateway/tool scale all covered in depth. LLM gateway/model routing and streaming intermediate tool-status events were flagged twice but never actually designed — left open. |
| MVP soundness | 4 | First pass had a real functional break: ingestion pipeline entirely missing (upload → S3, nothing populated the vector store). Fully fixed in the next iteration. KG-vs-vector-store overengineering was self-caught and cut immediately — good instinct. |
| Deep-dive depth | 5 | Ingestion at scale (queue/worker/DLQ/idempotency/autoscaling), two-index tool search with score-merging strategy, and a genuinely correct OAuth token architecture (delegated flow vs. service-to-service, envelope encryption, zero-trust connector boundary) were all pushed to real depth. |
| Trade-off reasoning | 3 | Several first-pass justifications were incorrect or thin and needed correction: claimed S3 event notifications are "unstable" (factually wrong), conflated NL-query difficulty with index-level tenant scoping, placed refresh-token logic in orchestration (inconsistent with the token boundary just designed), initial "take whatever we can take" merge answer was a non-decision. Recovery after each correction was strong, but independent first-pass reasoning needed a lot of propping up. |
| Failure-mode awareness | 4 | Missed the ingestion idempotency race (concurrent workers) until probed, but then independently derived the lease/TTL fix. DLQ, autoscaling-on-queue-depth, and reactive revocation detection were all reasonable. |
| Communication | 4 | Concise, direct answers; owned mistakes readily and iterated well. A few early answers were hand-wavy ("that's all", "as simple as that") until pushed for the actual mechanism. |

## Problems hinted vs. self-identified

**Self-identified (no hint needed — good instinct):**
- KG vs. vector-store overengineering for MVP
- Tool-description drift between the two search indexes + need for an update pipeline
- Lease/TTL fix for the ingestion idempotency race, once the race itself was pointed out
- Refusing to fully solve LLM gateway/presigned-URL in iteration 1 (correctly deferred, though LLM-gateway deferral was later challenged as a functional not scaling concern)

**Hinted / corrected by interviewer:**
- Missing ingestion pipeline (flagged as a broken MVP link, not scaling)
- Ingestion idempotency race condition (concurrent workers) — probed, user found the fix themselves
- S3 event notification reliability — factual correction, not just a hint
- Filter-before vs. filter-after for tenant-scoped tool search — corrected the "NL query makes pre-filtering hard" reasoning
- Refresh-token placement inconsistency (orchestration vs. connector boundary) — corrected
- "Atlas is a third party" — fundamental confusion about what's in-scope to design, corrected directly

**Still open / never resolved this session (carry into next):**
- LLM gateway / model routing by complexity — never designed, only debated whether it's in-scope
- Streaming intermediate tool-status events ("searching Salesforce…") — flagged in iteration 1, never revisited
- Subagent-count dependency-graph scheduler — debated as possible overengineering, no firm resolution
- Revocation notification delivery mechanism (in-chat error vs. persistent admin alert) — left thin

## Focus areas for next session

1. **LLM gateway design** — actually design it this time: routing logic by question complexity, fallback across 3 providers on rate-limit/outage, cost tracking per tenant.
2. **Trade-off reasoning under first contact** — practice justifying a decision correctly the *first* time rather than defaulting to a plausible-sounding but wrong claim (the S3 event-notification and NL-query/index-scoping mix-ups are the pattern to watch for: technical claims stated with more confidence than backing).
3. **Streaming/UX requirements** — this NFR (intermediate tool-status events during multi-tool turns) has been dropped twice; make sure a future session's MVP actually includes it from iteration 1.

## Iterations: problems, options, trade-offs, decisions

Each entry: naive approach that surfaced the problem → options considered → trade-off → what was chosen → why, in the learner's own reasoning.

### 1. MVP gaps (iteration 1)

**1a. Document relationships (knowledge graph vs. vector store)**
- *Naive approach*: the first MVP diagram included both a "document knowledge graph" (relationships between files) and a document vector store, side by side, without evidence that relationship-aware retrieval was actually needed yet — just assumed upfront that documents would have "tight relationships" worth modeling.
- *Options*: KG + vector store together, or vector store alone.
- *Trade-off*: the KG adds real modeling and maintenance cost; dropping it risks needing to retrofit relationship queries later if they turn out to matter.
- *Chosen*: vector store only, for now.
- *Why (learner)*: I realized I'd added the KG speculatively, before any requirement or query pattern actually demanded it. Cutting it removes complexity I can't yet justify, and I can add it back once a real need for relationship queries shows up — building it now would be designing for a problem I don't have evidence of yet.

**1b. Presigned upload vs. proxying through the file-upload service**
- *Naive approach*: the first MVP had the file-upload service sit in the middle of every upload — client → service → S3 via the AWS SDK — without weighing what routing 200MB files through the service itself would cost at scale.
- *Options*: keep proxying through the service, or have the client upload directly to S3 via a presigned URL.
- *Trade-off*: proxying gives a place to inspect/validate before the file lands in storage, but costs the service's own bandwidth and memory; presigned upload removes that cost but means validation has to happen after the fact (or not at the upload boundary at all).
- *Chosen*: deferred in iteration 1, adopted once the ingestion pipeline was designed in iteration 2.
- *Why (learner)*: at MVP scale, direct proxying works fine and isn't worth optimizing yet — I didn't have a reason to add the complexity of presigned URLs before I even had an ingestion pipeline to hand the file off to. Once I was designing that pipeline for 1,000 uploads/min, the cost of a service sitting in the request path for every large file became a real bottleneck, not a hypothetical one, so presigned upload stopped being premature and became the right call.

**1c. Missing ingestion pipeline**
- *Naive approach*: I drew the upload flow and the retrieval flow as two separate diagrams and didn't cross-check them against each other — the upload flow ended at "file lands in S3," while the retrieval flow separately assumed a populated vector store existed to search against.
- *Options*: n/a — this wasn't a design choice, it was an inconsistency between two flows I'd drawn independently.
- *Chosen*: added the missing async ingestion pipeline (queue → worker → chunk → embed) in iteration 2.
- *Why (learner)*: I hadn't actually traced the full path a file takes from upload to becoming searchable — I designed each flow in isolation and assumed the other side would "just work." The gap only became obvious once someone asked directly where content gets from S3 into the vector store, which forced me to notice retrieval had no data source.

**1d. LLM gateway / model routing**
- *Naive approach*: I assumed one hardcoded model provider is fine for the MVP, reasoning that a frontier model can functionally answer both simple and complex questions, so routing felt like a pure performance/cost optimization to bolt on later.
- *Options*: hardcode one provider now, or design cost-based routing across the 3 providers from the start.
- *Trade-off*: routing adds real complexity (classification, fallback across providers) but every trivial question hitting a frontier model burns unnecessary cost regardless of traffic volume.
- *Chosen*: deferred — contested, not resolved this session.
- *Why (learner)*: I conflated "the model works for this question" with "the model is the right choice for this question." My reasoning that low load excuses skipping routing doesn't actually hold, since the requirement is about cost efficiency per request, not about system load — I still haven't picked where this belongs in the architecture or what the routing logic should look like.

### 2. Ingestion pipeline at scale (iteration 2)

**2a. Detecting upload completion**
- *Naive approach*: designed a worker that repeatedly polls a "presigned URL checker" to see whether the client's direct-to-S3 upload had finished, based on an assumption that push-based S3 event notifications weren't reliable enough on their own — I believed they'd need extra infrastructure (like routing through Kafka) to guarantee delivery.
- *Options*: keep polling, or trigger ingestion directly off an S3 event notification into the queue.
- *Trade-off*: polling wastes worker cycles re-checking files that aren't ready and adds latency bounded by the poll interval; event-driven triggers are lower-latency and lower-cost if the delivery guarantee actually holds.
- *Chosen*: started with polling; reasoning corrected, no longer confident polling is the better choice.
- *Why (learner)*: I picked polling because I didn't trust the event system to reliably tell me when an upload finished. That assumption turned out to be wrong — S3-to-SQS-style event notifications already come with a durable, guaranteed-delivery queue built in, so I didn't need to add anything on top for reliability. Once that assumption fell apart, polling lost its justification; I was solving a reliability problem that didn't exist while eating the latency and cost polling actually does have.

**2b. Idempotency**
- *Naive approach*: the first design only wrote a job's ID to a KV store once a worker finished it, and checked that store before starting new work — I believed this was enough to prevent duplicate processing.
- *Options*: keep completion-only marking, or mark a job "processing" at the start and use a TTL to reclaim it if the worker dies.
- *Trade-off*: completion-only marking is simple but leaves a window open while a job is still being worked on; start-time marking closes that window but introduces its own failure mode — a job can get stuck forever if the worker that claimed it dies without ever finishing or releasing it.
- *Chosen*: lease pattern — mark "processing" at start, TTL reverts it to reprocessable if it's stuck.
- *Why (learner)*: my completion-only check does stop a job from being reprocessed after it's already done, which is what I was originally trying to solve. What I hadn't thought through was the in-flight window — if a second worker picks up the same job before the first one finishes, neither the KV store nor my check catches that, since nothing's been written yet. Once that gap was pointed out, I realized I needed a marker at the start of work, not just at the end — and I added the TTL myself because I could see that a raw "processing" flag with no expiry would just trade one failure mode (duplicate processing) for another (a job stuck forever if its worker crashes).

**2c. Failure handling**
- *Naive approach*: none — a dead-letter queue was included from the first version of this component, not discovered as a gap.
- *Options*: retry failed jobs inline against the main queue, or route them to a separate dead-letter queue.
- *Trade-off*: inline retries risk a small number of consistently-failing files blocking or slowing the whole queue behind them; a DLQ isolates them but needs its own investigation/replay process.
- *Chosen*: DLQ.
- *Why (learner)*: I wanted failing jobs to be visible and recoverable without letting them stall healthy ones — isolating them into their own queue means I can investigate or replay failures later on my own schedule, instead of having one bad file degrade ingestion for everyone else uploading at the same time.

**2d. Scaling workers**
- *Naive approach*: the initial diagram didn't specify fleet sizing at all until asked directly about the 1,000-uploads/min Monday-morning peak.
- *Options*: fixed-size worker fleet, or autoscale based on queue depth.
- *Trade-off*: a fixed fleet either overpays for idle capacity most of the time or falls behind during the peak; autoscaling matches capacity to load but adds operational complexity (scaling policy, threshold tuning).
- *Chosen*: autoscale on a queue-depth threshold.
- *Why (learner)*: the load isn't steady — it's bursty around a known peak — so sizing for either the average or the peak alone is wrong in one direction or the other. Scaling off a concrete signal (how deep the queue actually is) means the fleet grows and shrinks with real demand instead of me having to guess a fixed number that's right some of the time.

### 3. MCP gateway / tool RAG / subagents (iteration 2)

**3a. Combining two search results (structured + semantic)**
- *Naive approach*: originally planned for the toolset merger itself to somehow combine the SQL match score and the vector similarity score into one ranked list, without a concrete plan for how two differently-scaled scores would actually be compared.
- *Options*: normalize and blend the two scores into a single ranking, or have each search service pre-rank and select its own top-N, then merge via set operations (intersection, falling back to union).
- *Trade-off*: score blending needs a shared scale that doesn't naturally exist between a categorical SQL match and a cosine similarity value — any conversion factor between them would be arbitrary; set-based merging avoids inventing that scale but risks either being too strict (intersection drops a tool only one search method found) or too loose (union lets weak matches through).
- *Chosen*: each service pre-ranks and selects top-N; merger intersects, falling back to union only when intersection is empty.
- *Why (learner)*: I hadn't actually worked out how to make an SQL structured-match score and a vector similarity score comparable, and once asked, I realized any way I picked to blend them would be a number I made up, not something principled. Letting each side independently decide its own top candidates and then just checking for agreement between them sidesteps that problem entirely — I don't need to compare scores I have no honest way to compare.

**3b. Tenant scoping in tool search**
- *Naive approach*: assumed a single shared index/database across tenants by default, without explicitly deciding a multi-tenancy storage pattern, and planned to have the RBAC/policy engine filter the already-ranked, merged tool list after search rather than restrict the search itself.
- *Options*: filter after ranking (post-filter), or filter the index before searching (pre-filter, e.g. via row-level security or per-tenant index partitioning).
- *Trade-off*: post-filter briefly ranks and computes over tools the requesting user isn't authorized to see at all, which is both an exposure risk and wasted compute at scale; pre-filter requires the index itself to be partitioned or scoped by tenant/role, which is more upfront design work.
- *Chosen*: pre-filter — RLS for the SQL-based search, metadata/namespace filtering for the vector search.
- *Why (learner)*: my original reasoning was that pre-filtering was harder because the input query is unstructured natural language, so I defaulted to filtering the output list instead, which felt more "natural" as just a permission check on results I already had. That reasoning doesn't actually hold up — whether I can partition the searchable index by tenant/role has nothing to do with whether the query itself is structured or free text; those are two separate problems I'd been treating as one. Once I separated them, pre-filtering was clearly better: it means I never rank or expose another tenant's tools in the first place, and I'm not spending compute searching data the user was never going to see anyway.

**3c. Keeping the two indexes in sync**
- *Naive approach*: none specific — this was flagged proactively while designing the two-index search, before it caused any actual failure.
- *Options*: update both indexes manually/ad hoc as tool metadata changes, or build a dedicated pipeline that propagates tool changes to both indexes together.
- *Trade-off*: manual updates are simpler short-term but risk silent staleness if one index gets updated and the other doesn't; a dedicated pipeline avoids that but is more infrastructure to build and operate.
- *Chosen*: flagged as needing a dedicated pipeline; not fully designed this session.
- *Why (learner)*: I noticed on my own that having two independently-maintained indexes for the same underlying tool descriptions creates a synchronization problem the moment a tool's schema or description changes — I don't want to discover that gap later as a bug where one index quietly drifted out of date while the other updated. Calling it out now, even without a full answer yet, means I know it's there and won't be blindsided by it.

**3d. Subagent routing for multi-domain queries**
- *Naive approach*: picked subagents by description/capability match for single-domain queries, then reached for a dependency-graph/knowledge-graph scheduler as the general mechanism for ordering and parallelizing subagent calls when a query spans multiple domains.
- *Options*: a general dependency-graph scheduler that computes ordering/parallelism for any number of subagents, or a much simpler sequential-by-default execution with parallelism hardcoded only for specific cases I already know benefit from it.
- *Trade-off*: the graph scheduler generalizes well and avoids hardcoding assumptions about which subagents can run in parallel, but it's real infrastructure to build and keep correct as subagents and their relationships change; sequential-by-default is far simpler but doesn't scale gracefully if the number of subagents or their interactions grows.
- *Chosen*: sequential-by-default; the graph idea was debated but not committed to.
- *Why (learner)*: I reached for the graph scheduler because it's the general-purpose answer — it would keep working no matter how many subagents I eventually had. But when challenged on it, I remembered I'd already said I don't expect to scale past a handful of subagents (not hundreds), which means I was building a solution sized for a problem I don't actually have. I haven't fully decided against the graph idea, but I recognize the simpler sequential approach probably earns its keep better right now, given the actual scale I'm designing for.

### 4. Connector security (iteration 3)

**4a. OAuth token acquisition**
- *Naive approach*: none — no OAuth flow existed in the design at all before this iteration; this was the actual gap being addressed, not a refinement of an earlier attempt.
- *Options*: exchange the authorization code for tokens client-side (in the browser), or entirely server-side.
- *Trade-off*: client-side exchange would let the browser hold the access/refresh token directly, simpler to wire up but exposes the token to theft via XSS; server-side exchange keeps the token out of the browser entirely but means the backend has to own the whole OAuth dance.
- *Chosen*: delegated authorization-code flow, code-for-token exchange done entirely server-side.
- *Why (learner)*: once I understood that any token reaching the browser is a token an XSS bug could steal, keeping the exchange backend-only was the obvious choice — the browser only ever needs to know "Drive: connected," never the credential itself, so there's nothing there for a frontend vulnerability to steal in the first place.

**4b. Token storage**
- *Naive approach*: none — the design went straight to encryption rather than starting from plaintext storage.
- *Options*: encrypt all tenants' tokens under one shared key, or envelope-encrypt with a separate KMS-managed data key per tenant.
- *Trade-off*: a shared key is simpler to manage but means any single key compromise exposes every tenant's tokens at once; per-tenant keys add key-management overhead but contain a compromise to one tenant.
- *Chosen*: envelope encryption, per-tenant KMS data key.
- *Why (learner)*: given that tenant isolation is a hard constraint for this whole system, I didn't want a single point of failure that could expose everyone at once — bounding a leak to one tenant's ciphertext, which is useless without that specific tenant's key, directly matches the isolation guarantee I'm supposed to be providing everywhere else too.

**4c. Who resolves tenant_id → token**
- *Naive approach*: this followed directly from a principle I'd already committed to — that a connector "should not trust the MCP gateway" — rather than from a failed first attempt.
- *Options*: have the token resolved upstream (orchestration or the MCP gateway) and passed down to the connector, or have the connector's own auth middleware resolve tenant_id into a token independently at the point of use.
- *Trade-off*: resolving upstream means fewer places need secrets-manager access, but every component the token passes through becomes a place it could leak from a bug or compromise; resolving locally at the connector means more components need that access, but the token's exposure surface shrinks to just the one place it's actually used.
- *Chosen*: connector resolves it independently via its own auth middleware.
- *Why (learner)*: this was my own instinct before anyone pushed on it — if the connector doesn't trust the gateway to do authorization correctly, it shouldn't trust it to safely carry a raw token either. Keeping resolution local to where the token is actually used means orchestration and the gateway only ever need to know tenant identity, not the secret, so a bug anywhere upstream of the connector can't leak a token it never had.

**4d. Connector → secrets-manager authentication**
- *Naive approach*: I hadn't actually specified how the connector itself would authenticate to the secrets manager — when asked directly ("is there some sort of credentials needed?"), I realized I'd left that step undesigned.
- *Options*: embed a static credential in the connector's own config/code, or use a platform-issued machine identity (IAM role / workload identity) that's short-lived and managed by the underlying compute platform.
- *Trade-off*: a static credential is simple to reason about but is itself a long-lived secret that has to be stored, rotated, and protected somewhere — it just relocates the problem instead of solving it; a platform-issued identity avoids that but ties the design to whatever identity mechanism the compute platform provides.
- *Chosen*: platform-issued machine identity.
- *Why (learner)*: once it was pointed out, I could see that embedding a static credential just creates a new "secret zero" I'd have to protect with the same rigor as the tokens themselves — I hadn't actually solved anything, just moved the problem one layer down. A short-lived, platform-managed identity means there's no long-lived secret sitting in my own connector code for me to leak or forget to rotate.

**4e. Custom MCP isolation**
- *Naive approach*: my first answer to "how do you isolate custom MCP servers" was vague — "separated as a tenant, and they don't have access to other tenants' storage" — without naming a concrete mechanism or even stating which hosting model I was assuming (a server the tenant runs themselves, vs. code I execute on their behalf).
- *Options*: sandbox and execute tenant-provided code directly (containers/microVMs, no default network access, resource limits), or treat the tenant's custom MCP server as a remote endpoint I call over the network like a webhook.
- *Trade-off*: sandboxed execution is a much heavier lift — I'd be responsible for containment against arbitrary code — but it's the right model if I'm actually running their code; the remote-endpoint model is lighter to build but shifts the threat to the network layer (the tenant's server could try to reach my internal infrastructure, or degrade the platform for other tenants by being slow or malicious).
- *Chosen*: remote endpoint (tenant-hosted), isolated via egress restriction, URL allowlisting, and per-tenant timeouts/circuit breakers.
- *Why (learner)*: once I actually committed to an assumption — that a "custom MCP server" is infrastructure the tenant runs themselves, not code I execute — the isolation problem stopped being about sandboxing arbitrary code and became a network problem instead: stopping their server from reaching things it shouldn't, and stopping one slow or malicious tenant from starving everyone else. That's a much better match for the threat I actually have than building container/microVM sandboxing for a code-execution risk that doesn't apply under this hosting model.

**4f. Refresh-token ownership**
- *Naive approach*: initially placed refresh logic in the orchestration layer, reasoning that orchestration runs the main LLM loop and felt like the natural place to handle session-level concerns like keeping a token fresh.
- *Options*: refresh in orchestration, or refresh at the connector.
- *Trade-off*: orchestration is centrally located and could refresh once for anything downstream, but doing so means orchestration has to know about tenant tokens at all; the connector already touches the token for every call, so adding refresh there doesn't introduce a new place that needs secrets-manager access.
- *Chosen*: connector owns refresh.
- *Why (learner)*: when I said "orchestration," I hadn't connected it back to the boundary I'd already designed a few exchanges earlier — that orchestration should never see tenant tokens at all. Putting refresh there would have quietly broken that guarantee. The connector already has secrets-manager access and tenant context for the normal token-fetch path, so extending it to also handle refresh doesn't cross any boundary I hadn't already accepted.

**4g. Revocation detection**
- *Naive approach*: first description was vague — "when access token hits atlas, they will check and see authorization scopes revoked" — without specifying whether that check was something Atlas proactively polls for or something discovered as a side effect of a failed call.
- *Options*: rely on a proactive signal from the provider (e.g. a revocation webhook), or detect reactively via the error returned the next time the token is used or refreshed.
- *Trade-off*: proactive detection would surface a revocation immediately, but most providers don't reliably offer that signal; reactive detection is guaranteed to work (it's just handling the error you already get back) but means there's a window between revocation and the next actual use where Atlas doesn't know yet.
- *Chosen*: reactive — detect via `invalid_grant`/401 on next use or refresh, then notify the tenant.
- *Why (learner)*: I realized I was implicitly assuming a push signal exists without checking whether providers actually offer one reliably — they generally don't. Designing around a signal I can't count on receiving would leave a gap; reacting to the failure I'm guaranteed to see when I actually try to use the token is the mechanism I can actually build on, even though I haven't yet decided exactly how that gets surfaced to the tenant.

## OAuth token flow (sequence)

Key property: the raw token only exists between the connector's auth middleware and the secrets store (plus briefly in the connector's own process memory to make the outbound call). Orchestration and the MCP gateway only ever carry `tenant_id` / `user_id` — never the token itself.

```
 Client              API Gateway         Orchestration        MCP Gateway              Connector                 Secrets Store           3rd-party
 (Mobile/Web)         (OAuth authN)        service            (search/merge/policy)    (Drive/SF/Jira)           (KMS envelope-encrypted, per-tenant key)   (Drive API etc.)
    |                     |                     |                      |                       |                          |                       |
    |--- chat request --->|                     |                      |                       |                          |                       |
    |   (user session     |                     |                      |                       |                          |                       |
    |    cookie/JWT)       |                     |                      |                       |                       |                       |
    |                     |-- authenticate,     |                      |                       |                          |                       |
    |                     |   extract           |                      |                       |                          |                       |
    |                     |   {user_id,         |                      |                       |                          |                       |
    |                     |    tenant_id} ------>|                      |                       |                          |                       |
    |                     |                     |                      |                       |                          |                       |
    |                     |                     |-- tool call needed,  |                       |                          |                       |
    |                     |                     |   forward             |                      |                          |                       |
    |                     |                     |   {tenant_id,        |                       |                          |                       |
    |                     |                     |    user_id, intent}->|                       |                          |                       |
    |                     |                     |                      |                       |                          |                       |
    |                     |                     |                      |-- search+merge+RBAC   |                          |                       |
    |                     |                     |                      |   filter (still no    |                          |                       |
    |                     |                     |                      |   token anywhere) --->|                          |                       |
    |                     |                     |                      |   {tenant_id,         |                          |                       |
    |                     |                     |                      |    tool="drive.search"}                         |                       |
    |                     |                     |                      |                       |                          |                       |
    |                     |                     |                      |            [Auth middleware, connector-local]    |                       |
    |                     |                     |                      |                       |-- fetch token for -----> |                       |
    |                     |                     |                      |                       |   tenant_id (NOT         |                       |
    |                     |                     |                      |                       |   forwarded from         |                       |
    |                     |                     |                      |                       |   upstream — resolved    |                       |
    |                     |                     |                      |                       |   independently here)    |                       |
    |                     |                     |                      |                       |                          |                       |
    |                     |                     |                      |                       |<-- decrypt w/ tenant's --|                       |
    |                     |                     |                      |                       |   data key (KMS),        |                       |
    |                     |                     |                      |                       |   return short-lived     |                       |
    |                     |                     |                      |                       |   scoped token           |                       |
    |                     |                     |                      |                       |                          |                       |
    |                     |                     |                      |                       |-- RBAC check: does ----->|  (policy check,       |
    |                     |                     |                      |                       |   user_id have scope     |   in-memory, no       |
    |                     |                     |                      |                       |   for drive.search?      |   token stored here)  |
    |                     |                     |                      |                       |                          |                       |
    |                     |                     |                      |                       |-- call w/ token -------------------------------->|
    |                     |                     |                      |                       |                          |   (token used, held   |
    |                     |                     |                      |                       |                          |    only in connector  |
    |                     |                     |                      |                       |                          |    process memory,    |
    |                     |                     |                      |                       |                          |    never logged/      |
    |                     |                     |                      |                       |                          |    persisted here)    |
    |                     |                     |                      |                       |<-- results --------------------------------------|
    |                     |                     |<-- tool result ------|<----------------------|                          |                       |
    |<-- streamed answer -|<--------------------|                      |                       |                          |                       |
```

**Not shown above but designed in this session:** the delegated OAuth authorization-code flow that populates the secrets store in the first place (admin → OAuth consent → callback → server-side code-for-token exchange → envelope-encrypt → write to `tenants/{tenant_id}/connectors/{connector}`); refresh-token handling (owned by the connector, using its existing secrets-manager access); and reactive revocation detection (`invalid_grant`/401 on use → notify tenant).
