# Analytics Initiative — Charter

> Foundational context for the Agent Analytics and Optimization work. ADRs reference this document. Last updated: 2026-07-07.

## Vision

See `vision/agent-analytics-and-optimization-vision.md`. In short: make Microsoft Fabric the analytical and optimization layer for agentic applications, with **Azure Cosmos DB as the operational system of record** and Fabric as the analytical/optimization system of record, forming a continuous learning loop (analyze → recommend → govern → apply → measure).

## Scope of this workshop implementation

- **Pillar focus:** Pillar 4 — **Memory Intelligence** (the vision's flagship pillar), built on the existing travel multi-agent app.
- **Maturity target:** advance from Level 1 (Visibility) through **Level 2 (Recommendations)** and **Level 3 (Assisted Optimization)** for memory — i.e., generate memory-optimization recommendations, present them for human review, and apply approved changes back to the operational store.
- **Deliverables:**
  1. A full functional implementation in `02_completed/`.
  2. One or two workshop modules in `01_exercises/` teaching users to build it themselves, with clear learning elements.
  3. This knowledge base (vision, charter, ADRs, deep docs) kept current.

### Lower-risk optimization domains in scope (per the vision's Human-Governed principle)

Memory salience tuning, memory retention/TTL policies, and retrieval weighting — all **human-approved** before apply. Higher-risk domains (prompt/workflow/agent-instruction changes) remain out of scope for autonomous action.

## First principles (governing rules for this effort)

1. **Data-grounded, always.** Every assertion — whether made by the human or by an agent — must be grounded in data and **tested and observed to be correct before it is relied upon**. Structural review is not a test. "Should work" is not "works." When something has not been verified, say so explicitly and mark it as an open item to test.
2. **Record every architectural decision as an ADR.** Include context, options considered, the evidence, the decision, and consequences. Keep ADRs current; supersede rather than silently rewrite.
3. **Produce and maintain deep implementation docs** for both humans and agents, so collaborators and their agents can explore and analyze the build and propose intelligent fixes.

## Known constraints (verified)

- The app's Azure Cosmos DB account is provisioned with `disableLocalAuth: true` and (on the analytics branch) provisioned-throughput + continuous backup to support Fabric Mirroring. (See `02_completed/infra/shared/cosmosdb.bicep`.)
- Fabric Mirroring replicates only: **Memories, Users, Trips, Places**.
- Fabric User Data Functions' managed Cosmos connection targets **Cosmos DB *in Fabric***, not an external Azure Cosmos DB account (see ADR-0001).
- Deployed Azure apps cannot connect to Fabric Lakehouse SQL endpoints via managed identity; Azure Cosmos DB is the operational bridge (see ADR-0001).

## Open questions to resolve with evidence (tracked, not yet decided)

- **Data generation for the workshop:** the current live generator costs ~3 hrs / ~10M tokens (per `analytics/README.md`) — not workshop-viable. Need a tested, low-cost data path that also produces the new signals Level 2/3 requires (memory-retrieval events + outcome linkage).
- **Memory-effectiveness instrumentation:** the app currently records `lastUsedAt` on recall but no retrieval **event stream** and no link between recalled memories and turn outcomes. Level 2/3 recommendations depend on adding this (the vision's `MemoryEvent` primitive). To be designed and validated.
- **Power BI role:** whether/how to keep Power BI as an optional Level-1 visibility surface once the web-app loop exists.
