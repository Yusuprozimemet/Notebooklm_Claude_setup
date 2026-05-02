# System Design Agent — Implementation Tracker

> Built from: NotebookLM knowledge base (Senior System Design & API Architecture) + Claude synthesis  
> Purpose: An AI agent that guides engineers through a full, structured system design process — from vague problem statement to a reviewable architecture blueprint.

---

## Agent Overview

The agent operates as an **interactive co-designer**. It does not generate a blueprint immediately. Instead, it leads the user through 5 structured phases, asking targeted questions, surfacing trade-offs, and recommending specific tools — producing a living document at the end.

**Core behavior rules:**
- Never jump to solutions before completing requirements phase
- Always propose ≥2 options per major decision, with explicit trade-offs
- Anchor every recommendation to stated non-functional requirements
- Flag when a choice conflicts with CAP theorem, scale, or cost constraints

---

## Phase 0 — Agent Bootstrap

> Goal: Understand what kind of system is being designed before anything else.

### Implementation Tasks

- [ ] Accept a free-text system description from user
- [ ] Classify the system type:
  - `read-heavy` / `write-heavy` / `balanced`
  - `latency-sensitive` / `throughput-sensitive`
  - `consistency-critical` / `availability-critical`
  - `user-facing` / `internal service` / `data pipeline`
- [ ] Set a working `system_profile` object used in all downstream phases
- [ ] Output: confirmation summary of what the agent understood

### Prompt Behavior
```
"Before we design anything, help me understand what you're building.
Describe the system in plain English — who uses it, what it does,
and what would make it fail. I'll ask follow-up questions to nail
down the scope."
```

---

## Phase 1 — Requirements Clarification

> Source: Notebook §1 — Jumping directly into a solution is a red flag.

### Implementation Tasks

- [ ] **Functional requirements extraction**
  - [ ] Ask: who are the users? (consumers, internal services, both)
  - [ ] Ask: what are the core features? (enumerate, prioritize)
  - [ ] Ask: what is explicitly out of scope for v1?
  - [ ] Output: ordered feature list with `in-scope` / `deferred` tags

- [ ] **Non-functional requirements extraction**
  - [ ] Ask: expected number of users (DAU/MAU)?
  - [ ] Ask: target latency (p50 / p99)?
  - [ ] Ask: availability target (99.9% / 99.99%)?
  - [ ] Ask: consistency requirement (strong / eventual)?
  - [ ] Ask: geographic distribution (single region / multi-region / global)?

- [ ] **Capacity planning (back-of-envelope)**
  - [ ] Calculate: estimated read QPS
  - [ ] Calculate: estimated write QPS
  - [ ] Calculate: storage requirement (raw + metadata separately)
  - [ ] Calculate: bandwidth requirement
  - [ ] Flag: which resource dominates (storage? compute? network?)
  - [ ] Output: capacity summary table

### Decision Triggers
| Finding | Agent Action |
|---|---|
| QPS > 100k | Flag: single DB will not scale — sharding discussion required |
| Global users | Flag: CDN + multi-region needed |
| Availability ≥ 99.99% | Flag: redundancy + consensus algorithm required |
| Strong consistency required | Flag: CAP — availability trade-off must be discussed |

---

## Phase 2 — High-Level Design & API Definition

> Source: Notebook §2 — Establish client-backend contract before drawing architecture.

### Implementation Tasks

- [ ] **API style selection**
  - [ ] Present decision tree:
    - Standard CRUD + external clients → **REST**
    - Complex UI, multiple clients, flexible queries → **GraphQL**
    - Internal microservices, high performance, streaming → **gRPC**
  - [ ] Ask about real-time requirements:
    - Fire-and-forget → HTTP
    - Real-time push (chat, live feed) → **WebSockets**
    - Async background jobs → **AMQP / message queue**
  - [ ] Output: API contract skeleton (endpoints or schema stub)

- [ ] **Architecture diagram (textual)**
  - [ ] Place entry point: Load Balancer → API Gateway
  - [ ] List services satisfying each functional requirement
  - [ ] Decide: monolith vs microservices
    - Small team / early stage / simple domain → **Monolith**
    - Independent scaling needs / separate deploy cycles → **Microservices**
  - [ ] Map the read flow and write flow for each core operation
  - [ ] Output: component list with connections

- [ ] **Infrastructure entry points**
  - [ ] API Gateway responsibilities: auth, rate limiting, routing, logging
  - [ ] Load balancer algorithm selection:
    - Stateless services → Round Robin / Least Connections
    - Session affinity needed → IP Hashing
    - Geo-distributed → Consistent Hashing (e.g. Google LB)

### Common Pitfall Guard
```
Agent checks: "Have we gone more than 2 levels deep on any single component
before finishing the full high-level map?" → If yes, surface warning:
"We may be digging too deep too early. Let's finish the blueprint first."
```

---

## Phase 3 — Data Modeling & Storage Selection

> Source: Notebook §3 — Storage selection must follow access patterns, not habit.

### Implementation Tasks

- [ ] **Schema design**
  - [ ] Ask: what are the primary entities?
  - [ ] Ask: what are the most frequent read queries?
  - [ ] Ask: what is the read/write ratio?
  - [ ] Decide: normalized (less redundancy, complex joins) vs denormalized (fast reads, complex writes)

- [ ] **Database selection**
  - [ ] Present decision matrix:

    | Requirement | Recommended |
    |---|---|
    | Structured data + ACID + complex joins | **PostgreSQL / MySQL** |
    | High write throughput + flexible schema | **MongoDB / DynamoDB** |
    | Key-value lookups at massive scale | **DynamoDB / Redis** |
    | Wide-column, time-series or IoT | **Cassandra / InfluxDB** |
    | Full-text search | **Elasticsearch** |
    | Graph relationships | **Neo4j** |

- [ ] **Indexing**
  - [ ] Identify all columns used in WHERE / ORDER BY / JOIN
  - [ ] Add index for each — flag: missing indexes = slow searches at scale
  - [ ] Warn about over-indexing (slows writes)

- [ ] **Scaling strategy**
  - [ ] If read QPS dominates → **Leader-Follower replication** (primary writes, replicas serve reads)
  - [ ] If write QPS dominates or data > single node capacity → **Sharding**
    - Geo-based: user location partitioning (reduces latency)
    - Range-based: key range splits (risk: hotspots)
    - Hash-based: even distribution (risk: range queries break)
    - Consistent hashing: add/remove nodes with minimal remapping (used by DynamoDB, Cassandra)
  - [ ] Output: storage architecture diagram

---

## Phase 4 — Design Deep Dive

> Source: Notebook §4 — Focus on non-functional bottlenecks. Always offer ≥2 solutions per problem.

### Implementation Tasks

- [ ] **Bottleneck identification**
  - [ ] Cross-reference capacity plan (Phase 1) with architecture (Phase 2–3)
  - [ ] For each component: does it meet the QPS / latency / availability target?
  - [ ] Flag each bottleneck for resolution

- [ ] **Caching layer**
  - [ ] Pattern: Cache Aside (check cache → miss → fetch DB → populate cache)
  - [ ] Eviction: LRU for large-scale content (Spotify model)
  - [ ] TTL: set on all cache entries to prevent stale data
  - [ ] Tool: **Redis** for in-memory cache
  - [ ] Decide: where to cache (client / CDN edge / app layer / DB query cache)

- [ ] **CDN**
  - [ ] Use when: serving static assets (HTML, JS, images, video chunks) globally
  - [ ] Uses consistent hashing internally to route to nearest edge (e.g. Akamai)
  - [ ] Example: Netflix streams from nearest CDN edge, not origin

- [ ] **Message queue selection**
  - [ ] Present decision tree:

    | Use case | Tool |
    |---|---|
    | Background jobs (email, resize, notify) — processed once | **RabbitMQ** |
    | Event stream — multiple consumers, replayable, durable | **Kafka** |
    | Millions of messages/sec, analytics, fraud detection | **Kafka** |
    | Low-latency task queue with complex routing | **RabbitMQ** |

- [ ] **Consistency & consensus**
  - [ ] CAP Theorem decision:
    - Network partition is inevitable — must choose: Consistency **or** Availability
    - PACELC: even without partition — choose: Latency **or** Consistency
  - [ ] If strong consistency required → **Raft** (leader election + quorum majority)
    - Prevents split-brain (two nodes believing they are leader)
    - Used in etcd, CockroachDB
  - [ ] If eventual consistency acceptable → simpler replication, higher availability
  - [ ] Google Spanner uses **Paxos** for cross-shard consensus

- [ ] **Security layer**
  - [ ] Authentication: **OAuth2** (delegated) + **JWT** (stateless, scalable)
  - [ ] Authorization model:
    - Role-based permissions → **RBAC**
    - Attribute/context-based → **ABAC**
    - Resource-level → **ACL** (e.g. Google Drive)
  - [ ] Threat protection checklist:
    - [ ] Parameterized queries → prevents SQL injection
    - [ ] CORS headers → controls cross-origin API calls
    - [ ] CSRF tokens → session hijack protection
    - [ ] Input validation → XSS prevention
    - [ ] Rate limiting at API Gateway

- [ ] **Trade-off log** ← agent maintains this throughout
  - Every major decision gets logged as: `decision | option chosen | option rejected | reason`

---

## Phase 5 — Wrap-Up & Blueprint Output

> Source: Notebook §5 — Summarize unique parts. Reiterate every trade-off made.

### Implementation Tasks

- [ ] **End-to-end feature check**
  - [ ] For each functional requirement from Phase 1: is it fully covered?
  - [ ] For each non-functional requirement: does the architecture meet the target?

- [ ] **Generate final blueprint document** containing:
  - [ ] System classification (type, scale tier)
  - [ ] Capacity plan table
  - [ ] API contract summary (style + key endpoints)
  - [ ] Architecture component list with responsibilities
  - [ ] Data model + storage selection rationale
  - [ ] Scaling strategy (replication / sharding method)
  - [ ] Cache strategy
  - [ ] Message queue usage
  - [ ] Security model
  - [ ] Trade-off log (every major decision)
  - [ ] Open questions / deferred items

- [ ] **Review prompts**
  - "Is there a component that handles [feature X] end-to-end?"
  - "If [primary DB] goes down, what happens?"
  - "If traffic spikes 10x, which component breaks first?"

---

## Agent Capability Requirements

| Capability | Purpose |
|---|---|
| Multi-turn memory | Carry `system_profile` and decisions across all phases |
| Structured output | Emit capacity tables, component lists, trade-off logs |
| Tool use | Query NotebookLM for domain-specific answers |
| Decision tree execution | Route recommendations based on collected requirements |
| Document generation | Produce the final blueprint MD at Phase 5 |

---

## Implementation Stack (Recommended)

| Layer | Choice | Reason |
|---|---|---|
| LLM | Claude Sonnet / Opus | Multi-turn reasoning, structured output |
| Memory | In-context `system_profile` object | Lightweight, sufficient for one session |
| Knowledge retrieval | NotebookLM via notebooklm-skill | Source-grounded, citation-backed answers |
| Output format | Markdown document | Human-readable, versionable, shareable |
| Orchestration | Claude Code Agent SDK | Tool use + multi-step execution |

---

## Progress Tracker

| Phase | Status | Notes |
|---|---|---|
| Phase 0 — Bootstrap | `[ ] not started` | |
| Phase 1 — Requirements | `[ ] not started` | |
| Phase 2 — High-Level Design | `[ ] not started` | |
| Phase 3 — Data Modeling | `[ ] not started` | |
| Phase 4 — Deep Dive | `[ ] not started` | |
| Phase 5 — Wrap-Up | `[ ] not started` | |
| Agent implementation | `[ ] not started` | |
| NotebookLM integration | `[x] done` | Skill installed, notebook added |

---

## Open Questions

- [ ] Should the agent support multi-session (persist `system_profile` across days)?
- [ ] Should trade-off log be exported as a separate review artifact?
- [ ] Add a "challenge mode" where agent stress-tests the design (failure injection questions)?
- [ ] Integrate diagram generation (Mermaid / Excalidraw) for Phase 2 output?
