# System Architecture Specification: Ticket Triage and Draft Assistant

## 1. Architectural Style: Modular Monolith

The system is implemented as a single deployable unit with explicit internal module boundaries. 

*   **Modules:** Ingestion, Triage, Retrieval, Drafting, Review, and Identity.
*   **Boundaries:** Modules communicate exclusively through defined service interfaces. Cross-module direct imports of database models are strictly prohibited and enforced via `import-linter` in CI.
*   **Exclusion of Microservices:** Microservices are explicitly out of scope. A distributed monolith introduces unnecessary network latency, complex deployment orchestration, and distributed transaction management that is unjustified for the current scale.

## 2. Data Plane Topology

The architecture employs three distinct data stores, each with a strictly defined role.

| Store | Technology | Role | Source of Truth |
| :--- | :--- | :--- | :--- |
| Relational | PostgreSQL | Users, tickets, feedback, audit logs, knowledge base. | Yes (Primary) |
| Ephemeral | Redis | Rate limits, queues, idempotency keys, ephemeral cache. | No |
| Vector Index | Qdrant | Dedicated vector database for high-performance ANN search. | No (Derived) |

**Data Plane Principle:** PostgreSQL is the absolute source of truth. The Qdrant vector index is a derived read-model. If the vector index is destroyed, the system suffers degraded search performance but zero data loss. The index can be rebuilt entirely from PostgreSQL.

## 3. Vector Synchronization Strategy (The Dual-Write Paradigm)

To facilitate learning advanced distributed system concepts without initially introducing the operational overhead of the Transactional Outbox pattern, the system utilizes **Tier 1: Dual-Write** synchronization.

*   **Mechanism:** The application layer writes the ticket to PostgreSQL. Upon successful commit, it generates the embedding and writes it to Qdrant.
*   **The ACID Violation Risk:** Because the writes span two independent systems, ACID guarantees are lost. If the PostgreSQL write succeeds but the Qdrant write fails (e.g., network partition, service restart), the vector index will become desynchronized from the source of truth.
*   **Mitigation:** The system must include a reconciliation script capable of scanning PostgreSQL and rebuilding the Qdrant index to restore parity.
*   **Future State:** The Transactional Outbox pattern (`outbox_events` table) is a documented requirement for production-grade synchronization but is deferred to a later iteration.

## 4. AI Pipeline Architecture (Logical Layers)

The AI integration is not implemented as a distributed system of microservices. It is implemented as an ordered pipeline of functions within the `ai_pipeline` module. 

The 11 logical layers are:

1.  **Model Gateway:** Abstracts the underlying LLM provider (OpenAI/Anthropic). Handles connection pooling and API key injection.
2.  **Prompt Management:** A versioned dictionary of system prompts. Every output is tagged with the executing prompt version.
3.  **Context Builder:** Formats ticket data, user history, and retrieved documents into the final prompt payload.
4.  **Retrieval (RAG):** Queries the vector index for relevant knowledge base articles, strictly scoped by tenant/team identity.
5.  **Orchestration:** The primary execution sequence that coordinates layers 1-4.
6.  **Tools / MCP:** (Deferred) Placeholder for external API calls and function calling.
7.  **Identity & Scoping:** Enforces tenant isolation by injecting metadata filters into the retrieval layer.
8.  **Guardrails:** Pydantic schema validation applied to the LLM output. Malformed JSON is rejected.
9.  **Cache & Router:** Checks Redis for identical recent tickets to bypass the LLM API entirely.
10. **Fallback & Degradation:** A `try/except` boundary. If the LLM layer fails, times out, or breaches the cost budget, the system routes to the deterministic rule-based classifier.
11. **Observability & Evals:** Logs latency, token cost, prompt version, and trace IDs to the `suggestions` table for offline evaluation.

## 5. Phased Implementation Strategy

The architecture is deployed in three distinct iterations to isolate complexity and validate core mechanics before introducing AI dependencies.

### Phase v1: Deterministic Core
*   **Scope:** FastAPI framework, PostgreSQL schema, Redis queue, JWT authentication, RBAC authorization, idempotent webhook ingestion.
*   **AI:** None.
*   **Deployment:** Podman Compose stack with strict container memory limits.

### Phase v2: AI Integration (pgvector)
*   **Scope:** Introduction of the 11-layer AI pipeline. 
*   **Vector Store:** To minimize synchronization overhead during initial development, vectors are stored directly inside PostgreSQL using the `pgvector` extension.
*   **Evaluation:** Implementation of the Eval Harness and CI regression gate.

### Phase v2.5: Dedicated Vector Store Migration
*   **Scope:** Deployment of Qdrant. Migration of all vector data from `pgvector` to Qdrant.
*   **Objective:** Transition the RAG layer to query Qdrant. This phase exposes the developer to dual-write consistency failures, requiring the implementation of the `outbox_events` table and a background synchronization worker (Tier 2 architecture).

## 6. Infrastructure & Hardware Constraints

Development and local testing occur on a host machine with 16GB of RAM. 

*   **Resource Limits:** All Podman containers must enforce explicit memory limits to prevent Out-Of-Memory (OOM) kills.
    *   PostgreSQL: 2GB limit
    *   Qdrant: 512MB limit
    *   Redis: 256MB limit
    *   FastAPI Application: 512MB limit
*   **Volume Mapping:** Named volumes are mandatory for PostgreSQL and Qdrant to ensure data persistence across container lifecycles.

## 7. Security & Tenant Isolation

Security is enforced at the architectural boundary, not appended as an afterthought.

*   **Identity Propagation:** The authenticated user's `team_id` must be passed as a mandatory parameter to the Retrieval layer.
*   **Pre-Filtering:** The vector search must apply the `team_id` metadata filter *before* executing the Approximate Nearest Neighbor (ANN) search. Post-search filtering is an architectural failure and a tenant data leak.
*   **Output Validation:** All LLM outputs must pass Pydantic validation before being returned to the client or persisted.
*   **Human-in-the-Loop:** The AI pipeline can only generate `suggestions`. It cannot mutate ticket state. State transitions require explicit agent action (`accepted`, `edited`, `rejected`).