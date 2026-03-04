# Customer Support RAG SaaS - Engineering Playbook

**A comprehensive guide to building a production-grade AI customer support platform**

---

> Build an end-to-end, multi-tenant AI customer support SaaS from scratch. This playbook covers
> everything from project scaffolding to production deployment -- designed for engineers who learn
> by building, candidates preparing for system-design interviews, and teams bootstrapping a real product.

---

## Architecture Overview

```
                          Customer Support RAG SaaS - High-Level Architecture

    Clients                        API Gateway                     Core Services
 +-----------+               +-------------------+          +------------------------+
 | Chat      |  HTTP/SSE     |                   |          |  Auth Service (JWT)    |
 | Widget    |-------------->|   FastAPI          |--------->|  Tenant Middleware     |
 +-----------+               |   Application      |          |  RBAC Authorization   |
                             |   (Uvicorn)        |          +------------------------+
 +-----------+               |                   |
 | Admin     |  REST API     |   - Routes        |          +------------------------+
 | Dashboard |-------------->|   - Middleware     |--------->|  Chat Service          |
 +-----------+               |   - Auth Guards   |          |    |                   |
                             +-------------------+          |    v                   |
                                      |                     |  Retrieval Pipeline    |
                                      |                     |    |                   |
                                      v                     |    +-> Vector Search   |
                            +-------------------+           |    +-> BM25 Search     |
                            |  Background Jobs  |           |    +-> RRF Fusion      |
                            |                   |           |    |                   |
                            |  Celery Workers   |           |    v                   |
                            |  + Celery Beat    |           |  Generation Pipeline   |
                            |                   |           |    |                   |
                            |  - URL Crawling   |           |    +-> Prompt Builder  |
                            |  - File Parsing   |           |    +-> GPT Client      |
                            |  - Chunking       |           |    +-> Streaming (SSE) |
                            |  - Embedding      |           +------------------------+
                            |  - Analytics Agg  |
                            +-------------------+
                                      |
            +-------------------------+---------------------------+
            |                         |                           |
            v                         v                           v
   +-----------------+      +------------------+        +------------------+
   |    MongoDB      |      |     Qdrant       |        |      Redis       |
   |                 |      |   (Vector DB)    |        |                  |
   | - Organizations |      |                  |        | - Response Cache |
   | - Assistants    |      | - Chunk Vectors  |        | - Embedding Cache|
   | - Knowledge Src |      | - Multi-tenant   |        | - Rate Limits    |
   | - Chat Sessions |      |   Collections    |        | - Celery Broker  |
   | - Tickets       |      |                  |        | - Session Store  |
   | - Analytics     |      +------------------+        +------------------+
   | - Jobs          |
   +-----------------+

   Ingestion Flow:  URL/File --> Crawl/Parse --> Chunk --> Embed (MiniLM) --> Qdrant
   Query Flow:      Question --> Embed --> Hybrid Search --> Rerank --> Prompt --> GPT --> Stream
```

---

## Table of Contents

### Part 1: Foundations (`PART_1_FOUNDATIONS.md` -- Sections 1-8)

| #  | Section                    | Description                                                  |
|----|----------------------------|--------------------------------------------------------------|
| 1  | [Product Overview](#s1)    | What we are building, user personas, feature matrix          |
| 2  | [Multi-Tenant Architecture](#s2) | Tenant isolation strategies, data scoping, middleware    |
| 3  | [System Architecture](#s3) | Component diagram, data flow, technology decisions           |
| 4  | [Project Structure](#s4)   | Directory layout, naming conventions, layered architecture   |
| 5  | [Environment Setup](#s5)   | Docker Compose, env vars, local development workflow         |
| 6  | [Knowledge Ingestion](#s6) | Crawling, parsing, chunking, embedding, vector storage       |
| 7  | [Retrieval Pipeline](#s7)  | Vector search, BM25, hybrid fusion, reranking, filtering     |
| 8  | [Generation Pipeline](#s8) | Prompt engineering, GPT streaming, citation injection         |

### Part 2: Production (`PART_2_PRODUCTION.md` -- Sections 9-17)

| #  | Section                        | Description                                              |
|----|--------------------------------|----------------------------------------------------------|
| 9  | [API Design](#s9)              | RESTful conventions, versioning, pagination, error codes  |
| 10 | [Caching](#s10)                | Redis strategy, cache layers, invalidation patterns       |
| 11 | [Background Jobs](#s11)        | Celery architecture, task design, job tracking            |
| 12 | [Security](#s12)               | Auth, RBAC, prompt injection defense, rate limiting       |
| 13 | [Chat Widget](#s13)            | Embeddable widget, SSE streaming, session management      |
| 14 | [Analytics](#s14)              | Chat logs, resolution rates, dashboard API                |
| 15 | [Scaling](#s15)                | Horizontal scaling, sharding, connection pooling          |
| 16 | [Interview Prep](#s16)         | System design walkthrough, common questions, trade-offs   |
| 17 | [Advanced Improvements](#s17)  | Agentic RAG, cross-encoder reranking, evaluation, CI/CD  |

---

## Quick Start

### One-Command Launch

```bash
# Clone the repository and start all services
git clone <your-repo-url> customer-support-rag-saas
cd customer-support-rag-saas
cp .env.sample .env        # Edit with your OpenAI API key
docker compose up --build
```

This single command spins up:

| Service        | Port  | Purpose                        |
|----------------|-------|--------------------------------|
| FastAPI App    | 8000  | REST API + SSE streaming       |
| MongoDB        | 27017 | Document store                 |
| Qdrant         | 6333  | Vector database                |
| Redis          | 6379  | Cache, rate limits, broker     |
| Celery Worker  | --    | Background ingestion tasks     |
| Celery Beat    | --    | Scheduled job runner           |

After startup, visit:
- API docs: `http://localhost:8000/docs` (Swagger UI)
- Qdrant dashboard: `http://localhost:6333/dashboard`

### Minimal `.env` File

```env
# Required
OPENAI_API_KEY=sk-...
MONGO_URI=mongodb://mongo:27017
QDRANT_HOST=qdrant
QDRANT_PORT=6333
REDIS_URL=redis://redis:6379/0

# Auth
JWT_SECRET=your-secret-key-change-in-production
JWT_ALGORITHM=HS256
JWT_EXPIRY_MINUTES=1440

# Optional tuning
EMBEDDING_MODEL=all-MiniLM-L6-v2
CHUNK_SIZE=512
CHUNK_OVERLAP=50
GPT_MODEL=gpt-4o-mini
GPT_MAX_TOKENS=1024
GPT_TEMPERATURE=0.3
```

---

## Tech Stack

| Component          | Technology                   | Recommended Version | Purpose                                    |
|--------------------|------------------------------|---------------------|--------------------------------------------|
| **Web Framework**  | FastAPI                      | 0.115+              | Async REST API, dependency injection, SSE  |
| **LLM**           | OpenAI GPT API               | gpt-4o / gpt-4o-mini | Answer generation, summarization          |
| **Embeddings**     | sentence-transformers (MiniLM) | all-MiniLM-L6-v2  | Text embedding (384-dim, fast, free)       |
| **Vector DB**      | Qdrant                       | 1.12+               | Vector similarity search, filtering        |
| **Document DB**    | MongoDB                      | 7.0+                | Multi-tenant data, chat history, analytics |
| **ODM**           | Beanie                        | 1.26+               | Async MongoDB ODM with Pydantic v2         |
| **Cache / Broker** | Redis                        | 7.4+                | Caching, rate limiting, Celery broker      |
| **Task Queue**     | Celery                       | 5.4+                | Background ingestion, scheduled analytics  |
| **Containerization** | Docker Compose             | 2.29+               | Local dev, CI, production orchestration    |
| **Auth**          | PyJWT                         | 2.9+                | JWT token generation and validation        |
| **HTML Parsing**   | BeautifulSoup4               | 4.12+               | URL crawling, HTML-to-text extraction      |
| **PDF Parsing**    | PyPDF2                       | 3.0+                | PDF text extraction                        |
| **DOCX Parsing**   | python-docx                  | 1.1+                | Word document text extraction              |
| **Keyword Search** | rank-bm25                    | 0.2+                | BM25 scoring for hybrid retrieval          |
| **Testing**       | pytest + pytest-asyncio       | 8.0+ / 0.24+       | Async test suite with mongomock            |

---

## Who This Playbook Is For

### Engineers Learning RAG

If you want to understand Retrieval-Augmented Generation beyond toy demos, this playbook walks you through
building a real multi-tenant system. You will learn chunking strategies, hybrid search with reciprocal rank
fusion, prompt engineering for grounded answers, and streaming responses -- all within a production-quality
codebase.

### Interview Candidates

Sections 1-3 and 16 are specifically designed for system-design interviews. The architecture diagrams,
trade-off discussions, and scaling strategies map directly to the kind of questions asked in senior/staff
engineering interviews at top companies. Each section includes "why we chose X over Y" reasoning.

### Teams Building a Product

If you are bootstrapping a real customer-support AI product, this playbook provides the complete blueprint.
The task list (`TASK_LIST.md`) breaks the entire build into 28 discrete tasks with time estimates, dependencies,
and learning outcomes. Follow the phases sequentially for a structured build-out.

---

## How to Use This Playbook

### Suggested Reading Order

**First pass (understand the system):**
1. Read **Section 1: Product Overview** to understand what we are building
2. Read **Section 3: System Architecture** for the high-level picture
3. Skim the **Architecture Overview** diagram above
4. Read **Section 4: Project Structure** to understand the codebase layout

**Second pass (build it):**
1. Open `TASK_LIST.md` and start with Phase 1
2. For each task, read the corresponding playbook section for deep context
3. Implement the task, referencing the "Key Files to Create" list
4. Check off completed tasks and move to the next phase

**For interview prep:**
1. Read Sections 1-3 thoroughly (Product, Multi-Tenant, System Architecture)
2. Jump to **Section 16: Interview Prep** for structured practice
3. Practice explaining the architecture diagram from memory
4. Review trade-offs in Sections 7 (Retrieval), 8 (Generation), and 15 (Scaling)

### Reading the Companion Documents

| Document                | What It Contains                                         |
|-------------------------|----------------------------------------------------------|
| `README.md` (this file) | Master index, architecture overview, quick start         |
| `PART_1_FOUNDATIONS.md`  | Sections 1-8: Core concepts and implementation details   |
| `PART_2_PRODUCTION.md`   | Sections 9-17: Production features and advanced topics   |
| `TASK_LIST.md`           | 28 buildable tasks with dependencies and time estimates  |

### Conventions Used

- **Code blocks** contain runnable code or terminal commands
- **Trade-off boxes** highlight architectural decisions and alternatives
- **Interview tips** call out topics frequently asked in system-design rounds
- File paths reference the project structure defined in Section 4
- All code examples use Python 3.12+ syntax with type hints

---

## Section Summaries

### <a id="s1"></a> Section 1: Product Overview
Define the product: a multi-tenant SaaS where businesses upload knowledge bases (URLs, PDFs, docs) and get
an AI-powered chat widget that answers customer questions grounded in their content. Covers user personas
(end-customer, support agent, admin), feature matrix, and the core value proposition.

### <a id="s2"></a> Section 2: Multi-Tenant Architecture
Deep dive into tenant isolation. Every query, every vector search, every cache key is scoped by
`organization_id`. Covers middleware-based tenant extraction from JWT, Qdrant collection-per-tenant vs
payload filtering, MongoDB query scoping, and Redis key namespacing.

### <a id="s3"></a> Section 3: System Architecture
Component-level architecture with data flow diagrams for ingestion and query paths. Covers technology
selection rationale (why Qdrant over Pinecone, why MiniLM over OpenAI embeddings, why Celery over
background threads), latency budget breakdown, and failure mode analysis.

### <a id="s4"></a> Section 4: Project Structure
Full directory tree with explanations. Follows a layered architecture: routes (API) -> services (business
logic) -> repositories (data access). Covers naming conventions, module organization, and dependency flow.

### <a id="s5"></a> Section 5: Environment Setup
Docker Compose configuration for all six services. Covers environment variables, volume mounts, health
checks, dependency ordering, and the local development workflow with hot reloading.

### <a id="s6"></a> Section 6: Knowledge Ingestion
The full ingestion pipeline: URL crawling with BeautifulSoup, PDF/DOCX parsing, section-aware text
chunking with overlap, MiniLM embedding generation, and Qdrant vector upsert. Covers chunking strategies
(fixed-size vs semantic), metadata attachment, and Celery-based async processing.

### <a id="s7"></a> Section 7: Retrieval Pipeline
Hybrid search combining dense vector similarity (Qdrant) with sparse keyword matching (BM25). Covers
reciprocal rank fusion (RRF), score thresholding, multi-tenant filtering, and optional cross-encoder
reranking. Includes latency benchmarks and relevance tuning.

### <a id="s8"></a> Section 8: Generation Pipeline
Prompt engineering for grounded, citation-aware answers. Covers system/user prompt construction, context
window management, OpenAI streaming with Server-Sent Events, hallucination mitigation, and fallback to
"I don't know" when retrieval confidence is low.

### <a id="s9"></a> Section 9: API Design
RESTful API design with FastAPI. Covers route organization, request/response schemas with Pydantic,
pagination, error handling middleware, API versioning strategy, and Swagger documentation.

### <a id="s10"></a> Section 10: Caching
Multi-layer Redis caching: response cache (same question = cached answer), embedding cache (avoid
re-embedding identical text), and search result cache. Covers TTL strategies, cache invalidation on
knowledge base updates, and cache-aside pattern implementation.

### <a id="s11"></a> Section 11: Background Jobs
Celery architecture for ingestion workers and scheduled tasks. Covers task design (idempotent, retryable),
job status tracking in MongoDB, progress reporting, Celery Beat for scheduled analytics aggregation and
stale-job cleanup, and dead-letter handling.

### <a id="s12"></a> Section 12: Security
Comprehensive security: JWT authentication, role-based access control (owner/admin/agent/viewer), prompt
injection detection and sanitization, per-tenant and per-endpoint rate limiting with Redis, input
validation, and CORS configuration.

### <a id="s13"></a> Section 13: Chat Widget
Embeddable JavaScript chat widget that communicates via SSE for streaming responses. Covers widget
configuration (API key, styling), session management, conversation history, typing indicators, and
the SSE protocol implementation in FastAPI.

### <a id="s14"></a> Section 14: Analytics
Analytics pipeline: chat log storage, resolution rate tracking (answered by AI vs escalated), popular
question clustering, response quality metrics, and dashboard API endpoints. Covers aggregation pipelines
in MongoDB and scheduled analytics jobs.

### <a id="s15"></a> Section 15: Scaling
Horizontal scaling strategies: stateless API servers behind a load balancer, Qdrant cluster mode,
MongoDB replica sets, Redis Cluster for cache, Celery worker scaling, connection pooling, and
performance optimization techniques.

### <a id="s16"></a> Section 16: Interview Prep
Structured system-design interview walkthrough for this exact system. Covers requirements gathering,
capacity estimation, high-level design, deep dives (retrieval, generation, multi-tenancy), and common
follow-up questions with model answers.

### <a id="s17"></a> Section 17: Advanced Improvements
Forward-looking improvements: agentic RAG with LangGraph (multi-step reasoning, tool use), cross-encoder
reranking for precision, RAGAS-based evaluation, A/B testing frameworks, fine-tuned embeddings, and
CI/CD pipeline with GitHub Actions.

---

## License

This playbook is provided as an educational resource. See `LICENSE` for details.
