# Customer Support RAG SaaS - Task List

**Pickable task list for building the entire SaaS from scratch.**

Work through these 28 tasks in order. Each task lists its prerequisites, so you can also skip ahead
if you already have experience with a particular area. Check off tasks as you complete them.

---

## Progress Summary

| Phase                          | Tasks   | Estimated Hours | Status     |
|--------------------------------|---------|-----------------|------------|
| Phase 1: Foundation            | 1-5     | 16h             | Not Started |
| Phase 2: Knowledge Ingestion   | 6-10    | 15h             | Not Started |
| Phase 3: RAG Pipeline          | 11-14   | 18h             | Not Started |
| Phase 4: Background Processing | 15-17   | 7h              | Not Started |
| Phase 5: Production Features   | 18-22   | 17h             | Not Started |
| Phase 6: Security & Polish     | 23-25   | 8h              | Not Started |
| Bonus: Advanced                | 26-28   | 14h             | Not Started |
| **Total**                      | **28**  | **~95h**        |             |

---

## Phase 1: Foundation

> Set up the project skeleton, database models, authentication, multi-tenancy, and basic CRUD.
> After this phase you will have a running API server with user auth and tenant-scoped data access.

---

### Task 1: Project Setup & Docker Compose

| Field              | Value                                                                 |
|--------------------|-----------------------------------------------------------------------|
| **Difficulty**     | Easy                                                                  |
| **Estimated Time** | 2 hours                                                               |
| **Prerequisites**  | None                                                                  |
| **Status**         | [ ] Not Started                                                       |

**Learning Outcomes:**
- Understand the layered project structure (routes / services / models / database)
- Configure Docker Compose to orchestrate multiple services with health checks
- Set up FastAPI with Uvicorn, lifespan events, and CORS middleware
- Manage environment variables with Pydantic Settings

**Key Files to Create:**
```
docker-compose.yml
Dockerfile
.env.sample
requirements.txt
app.py                          # FastAPI app with lifespan
main.py                         # Uvicorn entry point
config/
  config.py                     # Pydantic BaseSettings
  __init__.py
database/
  mongodb.py                    # Beanie init, MongoDB connection
  redis.py                      # Redis connection pool
  __init__.py
```

**Acceptance Criteria:**
- `docker compose up --build` starts FastAPI (8000), MongoDB (27017), Qdrant (6333), Redis (6379)
- `GET /health` returns `{"status": "ok"}`
- Hot reload works for local development

---

### Task 2: MongoDB Models & Database Setup

| Field              | Value                                                                 |
|--------------------|-----------------------------------------------------------------------|
| **Difficulty**     | Easy                                                                  |
| **Estimated Time** | 3 hours                                                               |
| **Prerequisites**  | Task 1                                                                |
| **Status**         | [ ] Not Started                                                       |

**Learning Outcomes:**
- Design Beanie Document models with proper indexes and audit fields
- Understand multi-tenant data modeling (every document scoped by `organization_id`)
- Create Pydantic schemas for request/response validation
- Initialize Beanie with all document models on app startup

**Key Files to Create:**
```
models/
  organization.py               # OrganizationDB document
  user.py                       # UserDB document
  assistant.py                  # AssistantDB document (per-tenant AI config)
  knowledge_source.py           # KnowledgeSourceDB (URL, PDF, DOCX metadata)
  chunk.py                      # ChunkDB (text chunk + metadata, optional)
  chat_session.py               # ChatSessionDB (conversation history)
  chat_message.py               # ChatMessageDB (individual messages)
  ticket.py                     # TicketDB (escalation tickets)
  job.py                        # JobDB (background job tracking)
  analytics.py                  # AnalyticsDB (aggregated metrics)
  __init__.py
schemas/
  organization_schema.py
  user_schema.py
  assistant_schema.py
  knowledge_source_schema.py
  chat_schema.py
  __init__.py
```

**Acceptance Criteria:**
- All models have `organization_id` field with index
- All models include `created_at`, `updated_at` audit fields
- Beanie initializes successfully with all models on app startup
- Pydantic schemas validate request/response payloads

---

### Task 3: Authentication System

| Field              | Value                                                                 |
|--------------------|-----------------------------------------------------------------------|
| **Difficulty**     | Medium                                                                |
| **Estimated Time** | 4 hours                                                               |
| **Prerequisites**  | Task 2                                                                |
| **Status**         | [ ] Not Started                                                       |

**Learning Outcomes:**
- Implement JWT token generation and validation with PyJWT
- Build signup and login endpoints with password hashing (bcrypt)
- Create a FastAPI dependency for extracting the current user from the token
- Understand the difference between API key auth (widget) and JWT auth (dashboard)

**Key Files to Create:**
```
auth/
  jwt_handler.py                # Token creation, validation, decode
  password.py                   # Bcrypt hash and verify
  dependencies.py               # get_current_user dependency
  __init__.py
routes/
  auth_routes.py                # POST /auth/signup, POST /auth/login
  __init__.py
services/
  auth_service.py               # Signup/login business logic
  __init__.py
```

**Acceptance Criteria:**
- `POST /auth/signup` creates user, returns JWT
- `POST /auth/login` validates credentials, returns JWT
- Protected routes return 401 without a valid token
- JWT payload contains `user_id`, `organization_id`, `role`

---

### Task 4: Multi-Tenant Middleware

| Field              | Value                                                                 |
|--------------------|-----------------------------------------------------------------------|
| **Difficulty**     | Medium                                                                |
| **Estimated Time** | 3 hours                                                               |
| **Prerequisites**  | Task 3                                                                |
| **Status**         | [ ] Not Started                                                       |

**Learning Outcomes:**
- Build FastAPI middleware that extracts `organization_id` from the JWT
- Use Python contextvars to make tenant ID available throughout the request lifecycle
- Scope all database queries by `organization_id` automatically
- Understand tenant isolation patterns and why they matter

**Key Files to Create:**
```
middleware/
  tenant_middleware.py           # Extract org_id, set context var
  context.py                    # contextvars for tenant_id, user_id
  __init__.py
services/
  common/
    base_service.py             # Base service with tenant-scoped queries
    __init__.py
```

**Acceptance Criteria:**
- Every request has `organization_id` available via context
- Database queries automatically filter by the current tenant
- A user from Org A cannot access data belonging to Org B
- Public endpoints (health, widget with API key) bypass tenant middleware

---

### Task 5: Basic CRUD APIs

| Field              | Value                                                                 |
|--------------------|-----------------------------------------------------------------------|
| **Difficulty**     | Easy                                                                  |
| **Estimated Time** | 4 hours                                                               |
| **Prerequisites**  | Task 4                                                                |
| **Status**         | [ ] Not Started                                                       |

**Learning Outcomes:**
- Implement full CRUD for Organization, Assistant, and Knowledge Source entities
- Use FastAPI routers with proper tags, prefixes, and status codes
- Apply pagination pattern (skip/limit) for list endpoints
- Handle 404 and validation errors consistently

**Key Files to Create:**
```
routes/
  organization_routes.py        # CRUD for organizations
  assistant_routes.py           # CRUD for assistants
  knowledge_source_routes.py    # CRUD for knowledge sources
services/
  organization_service.py
  assistant_service.py
  knowledge_source_service.py
```

**Acceptance Criteria:**
- Full CRUD (Create, Read, Update, Delete, List) for all three entities
- List endpoints support `skip` and `limit` query parameters
- All endpoints are tenant-scoped
- Swagger docs show organized routes with tags

---

## Phase 2: Knowledge Ingestion

> Build the pipeline that converts raw content (URLs, PDFs, DOCX files) into searchable vector embeddings.
> After this phase you can ingest a knowledge base and store it in Qdrant.

---

### Task 6: Text Chunking Service

| Field              | Value                                                                 |
|--------------------|-----------------------------------------------------------------------|
| **Difficulty**     | Medium                                                                |
| **Estimated Time** | 3 hours                                                               |
| **Prerequisites**  | Task 5                                                                |
| **Status**         | [ ] Not Started                                                       |

**Learning Outcomes:**
- Understand why chunking matters for RAG quality (context window, relevance)
- Implement section-aware chunking that respects paragraph/heading boundaries
- Add configurable chunk size and overlap parameters
- Attach metadata to each chunk (source URL, page number, position, title)

**Key Files to Create:**
```
services/
  ingestion/
    chunking_service.py         # Text chunking with overlap
    __init__.py
tests/
  test_chunking_service.py
```

**Acceptance Criteria:**
- Chunks respect sentence boundaries (no mid-sentence splits)
- Overlap between consecutive chunks is configurable (default 50 tokens)
- Each chunk carries metadata: `source_id`, `source_type`, `position`, `title`
- Unit tests cover edge cases: empty input, very short text, single long paragraph

---

### Task 7: Embedding Service (MiniLM)

| Field              | Value                                                                 |
|--------------------|-----------------------------------------------------------------------|
| **Difficulty**     | Medium                                                                |
| **Estimated Time** | 2 hours                                                               |
| **Prerequisites**  | Task 6                                                                |
| **Status**         | [ ] Not Started                                                       |

**Learning Outcomes:**
- Load and use sentence-transformers models for text embedding
- Understand embedding dimensions, normalization, and batch processing
- Build an abstraction layer that could swap MiniLM for OpenAI embeddings later
- Optimize batch embedding for throughput

**Key Files to Create:**
```
services/
  ingestion/
    embedding_service.py        # MiniLM embedding generation
tests/
  test_embedding_service.py
```

**Acceptance Criteria:**
- `embed_text(text) -> List[float]` returns a 384-dimensional vector
- `embed_batch(texts) -> List[List[float]]` processes multiple texts efficiently
- Model is loaded once and reused across requests
- Unit tests verify vector dimensionality and deterministic output

---

### Task 8: Qdrant Vector Storage

| Field              | Value                                                                 |
|--------------------|-----------------------------------------------------------------------|
| **Difficulty**     | Medium                                                                |
| **Estimated Time** | 3 hours                                                               |
| **Prerequisites**  | Task 7                                                                |
| **Status**         | [ ] Not Started                                                       |

**Learning Outcomes:**
- Create and manage Qdrant collections with proper vector configuration
- Implement multi-tenant isolation via payload filtering or collection-per-tenant
- Upsert vectors with metadata payloads
- Perform filtered similarity search with score thresholds

**Key Files to Create:**
```
services/
  ingestion/
    vector_store_service.py     # Qdrant collection management, upsert, search
database/
  qdrant.py                     # Qdrant client connection
tests/
  test_vector_store_service.py
```

**Acceptance Criteria:**
- Collections are created with cosine distance and 384 dimensions
- Vectors are upserted with `organization_id` and `source_id` in the payload
- Search filters by `organization_id` to enforce tenant isolation
- Deletion by `source_id` removes all chunks for a knowledge source

---

### Task 9: URL Crawler & HTML Parser

| Field              | Value                                                                 |
|--------------------|-----------------------------------------------------------------------|
| **Difficulty**     | Medium                                                                |
| **Estimated Time** | 4 hours                                                               |
| **Prerequisites**  | Task 6                                                                |
| **Status**         | [ ] Not Started                                                       |

**Learning Outcomes:**
- Crawl web pages with httpx (async HTTP client)
- Extract clean text from HTML using BeautifulSoup
- Handle common crawling challenges: redirects, timeouts, robots.txt respect
- Build a pipeline: fetch -> parse -> clean -> chunk

**Key Files to Create:**
```
services/
  ingestion/
    url_crawler_service.py      # Async URL fetching
    html_parser_service.py      # BeautifulSoup text extraction
tests/
  test_url_crawler_service.py
  test_html_parser_service.py
```

**Acceptance Criteria:**
- Fetches a URL and extracts meaningful text (strips nav, footer, scripts, styles)
- Handles common errors gracefully (404, timeout, SSL errors)
- Respects a configurable timeout and max content length
- Returns structured output: `{title, text, url, fetched_at}`

---

### Task 10: File Parser (PDF/DOCX)

| Field              | Value                                                                 |
|--------------------|-----------------------------------------------------------------------|
| **Difficulty**     | Medium                                                                |
| **Estimated Time** | 3 hours                                                               |
| **Prerequisites**  | Task 6                                                                |
| **Status**         | [ ] Not Started                                                       |

**Learning Outcomes:**
- Extract text from PDF files using PyPDF2
- Extract text from DOCX files using python-docx
- Handle file upload via FastAPI's `UploadFile`
- Build a unified parser interface for multiple file types

**Key Files to Create:**
```
services/
  ingestion/
    pdf_parser_service.py       # PyPDF2 text extraction
    docx_parser_service.py      # python-docx text extraction
    file_parser_service.py      # Unified parser (dispatches by file type)
routes/
  upload_routes.py              # File upload endpoints
tests/
  test_file_parser_service.py
```

**Acceptance Criteria:**
- PDF parser extracts text from all pages with page number metadata
- DOCX parser extracts text preserving heading hierarchy
- File upload endpoint accepts PDF and DOCX, rejects other types
- Unified parser dispatches to the correct parser based on file extension

---

## Phase 3: RAG Pipeline

> Build the core retrieval-augmented generation pipeline. After this phase you can ask questions
> and get AI-generated answers grounded in the ingested knowledge base.

---

### Task 11: Vector Search Service

| Field              | Value                                                                 |
|--------------------|-----------------------------------------------------------------------|
| **Difficulty**     | Hard                                                                  |
| **Estimated Time** | 4 hours                                                               |
| **Prerequisites**  | Task 8                                                                |
| **Status**         | [ ] Not Started                                                       |

**Learning Outcomes:**
- Implement semantic search using Qdrant's similarity search API
- Apply score thresholds to filter low-confidence results
- Understand the impact of `top_k` on retrieval quality and latency
- Build tenant-scoped search with metadata filtering

**Key Files to Create:**
```
services/
  retrieval/
    vector_search_service.py    # Qdrant similarity search with filtering
    __init__.py
tests/
  test_vector_search_service.py
```

**Acceptance Criteria:**
- Search returns top-k results with scores, filtered by `organization_id`
- Results below a configurable score threshold are excluded
- Search accepts optional `source_id` filter to restrict to specific knowledge sources
- Response includes chunk text, metadata, and similarity score

---

### Task 12: Hybrid Search (BM25 + Vector)

| Field              | Value                                                                 |
|--------------------|-----------------------------------------------------------------------|
| **Difficulty**     | Hard                                                                  |
| **Estimated Time** | 5 hours                                                               |
| **Prerequisites**  | Task 11                                                               |
| **Status**         | [ ] Not Started                                                       |

**Learning Outcomes:**
- Implement BM25 keyword search over stored chunks
- Understand reciprocal rank fusion (RRF) for combining ranked lists
- Compare and tune the balance between semantic and keyword search
- Handle edge cases: no vector results, no BM25 results, complete overlap

**Key Files to Create:**
```
services/
  retrieval/
    bm25_search_service.py      # BM25 keyword search
    hybrid_search_service.py    # RRF fusion of vector + BM25 results
tests/
  test_bm25_search_service.py
  test_hybrid_search_service.py
```

**Acceptance Criteria:**
- BM25 search tokenizes query and scores chunks using BM25 algorithm
- RRF fusion merges vector and BM25 ranked lists with configurable weight (`k=60`)
- Hybrid search returns deduplicated results with combined scores
- Unit tests verify fusion correctness with known inputs

---

### Task 13: Prompt Builder & GPT Client

| Field              | Value                                                                 |
|--------------------|-----------------------------------------------------------------------|
| **Difficulty**     | Hard                                                                  |
| **Estimated Time** | 4 hours                                                               |
| **Prerequisites**  | Task 11                                                               |
| **Status**         | [ ] Not Started                                                       |

**Learning Outcomes:**
- Design system and user prompts for grounded, citation-aware answers
- Integrate the OpenAI API with streaming responses
- Manage context window budget (system prompt + context chunks + history + question)
- Implement fallback behavior when retrieval confidence is low

**Key Files to Create:**
```
services/
  generation/
    prompt_builder_service.py   # System/user prompt construction
    gpt_client_service.py       # OpenAI API client with streaming
    __init__.py
resources/
  prompts/
    system_prompt.txt           # System prompt template
    no_context_prompt.txt       # Fallback when no relevant chunks found
tests/
  test_prompt_builder_service.py
  test_gpt_client_service.py
```

**Acceptance Criteria:**
- System prompt instructs GPT to answer only from provided context
- Context chunks are formatted with source attribution markers
- Streaming responses yield tokens via an async generator
- When no relevant chunks are found, GPT responds with a polite "I don't know" variant

---

### Task 14: Chat Service (End-to-End)

| Field              | Value                                                                 |
|--------------------|-----------------------------------------------------------------------|
| **Difficulty**     | Hard                                                                  |
| **Estimated Time** | 5 hours                                                               |
| **Prerequisites**  | Tasks 12, 13                                                          |
| **Status**         | [ ] Not Started                                                       |

**Learning Outcomes:**
- Orchestrate the full RAG pipeline: embed query -> hybrid search -> build prompt -> generate
- Manage chat sessions and conversation history
- Implement Server-Sent Events (SSE) for streaming responses in FastAPI
- Store chat messages with retrieved sources for audit and analytics

**Key Files to Create:**
```
services/
  chat_service.py               # Orchestrates retrieval + generation + history
routes/
  chat_routes.py                # POST /chat, GET /chat/{session_id}/stream (SSE)
tests/
  test_chat_service.py
  test_chat_routes.py
```

**Acceptance Criteria:**
- `POST /chat` accepts a question, returns a streaming SSE response
- Chat history is stored in MongoDB and included in the prompt (last N turns)
- Retrieved sources are stored alongside the AI response
- New sessions are created automatically; existing sessions are continued

---

## Phase 4: Background Processing

> Move heavy ingestion work to background workers. After this phase, knowledge ingestion
> happens asynchronously with progress tracking.

---

### Task 15: Celery Setup & Ingestion Tasks

| Field              | Value                                                                 |
|--------------------|-----------------------------------------------------------------------|
| **Difficulty**     | Medium                                                                |
| **Estimated Time** | 3 hours                                                               |
| **Prerequisites**  | Tasks 8, 9, 10                                                        |
| **Status**         | [ ] Not Started                                                       |

**Learning Outcomes:**
- Configure Celery with Redis as broker and MongoDB for result backend
- Convert synchronous ingestion pipeline into Celery tasks
- Design idempotent, retryable tasks with proper error handling
- Understand Celery worker concurrency and prefetch settings

**Key Files to Create:**
```
workers/
  celery_app.py                 # Celery application configuration
  ingestion_tasks.py            # Celery tasks: ingest_url, ingest_file
  __init__.py
```

**Acceptance Criteria:**
- `docker compose up` starts a Celery worker alongside the API
- `ingest_url.delay(source_id)` crawls, chunks, embeds, and stores asynchronously
- `ingest_file.delay(source_id, file_path)` processes uploaded files
- Failed tasks retry up to 3 times with exponential backoff

---

### Task 16: Job Status Tracking

| Field              | Value                                                                 |
|--------------------|-----------------------------------------------------------------------|
| **Difficulty**     | Medium                                                                |
| **Estimated Time** | 2 hours                                                               |
| **Prerequisites**  | Task 15                                                               |
| **Status**         | [ ] Not Started                                                       |

**Learning Outcomes:**
- Track background job status in MongoDB (pending, processing, completed, failed)
- Report progress updates from within Celery tasks
- Build API endpoints for checking job status
- Handle job cancellation and cleanup

**Key Files to Create:**
```
services/
  job_service.py                # Job creation, status updates, queries
routes/
  job_routes.py                 # GET /jobs/{id}, GET /jobs (list)
```

**Acceptance Criteria:**
- Knowledge source ingestion creates a JobDB document with status "pending"
- Celery task updates status to "processing" with progress percentage
- On completion, status becomes "completed" with chunk count
- On failure, status becomes "failed" with error message
- `GET /jobs/{id}` returns current job status

---

### Task 17: Scheduled Tasks (Celery Beat)

| Field              | Value                                                                 |
|--------------------|-----------------------------------------------------------------------|
| **Difficulty**     | Medium                                                                |
| **Estimated Time** | 2 hours                                                               |
| **Prerequisites**  | Task 16                                                               |
| **Status**         | [ ] Not Started                                                       |

**Learning Outcomes:**
- Configure Celery Beat for periodic task scheduling
- Implement analytics aggregation as a scheduled job
- Build cleanup tasks for stale jobs and expired cache entries
- Understand Celery Beat's schedule storage and persistence

**Key Files to Create:**
```
workers/
  scheduled_tasks.py            # Periodic task definitions
  celery_app.py                 # Update with beat_schedule config
```

**Acceptance Criteria:**
- Analytics aggregation runs every hour (configurable)
- Stale job cleanup runs every 30 minutes
- Celery Beat starts as a separate container in Docker Compose
- Scheduled tasks are idempotent (safe to run multiple times)

---

## Phase 5: Production Features

> Add caching, rate limiting, ticketing, analytics, and the chat widget.
> After this phase the system is feature-complete for an MVP launch.

---

### Task 18: Redis Caching Layer

| Field              | Value                                                                 |
|--------------------|-----------------------------------------------------------------------|
| **Difficulty**     | Medium                                                                |
| **Estimated Time** | 3 hours                                                               |
| **Prerequisites**  | Task 14                                                               |
| **Status**         | [ ] Not Started                                                       |

**Learning Outcomes:**
- Implement multi-layer caching: response cache, embedding cache, search cache
- Design cache keys with tenant scoping (`org:{id}:query:{hash}`)
- Build cache invalidation logic triggered by knowledge base updates
- Understand TTL strategies and cache-aside pattern

**Key Files to Create:**
```
services/
  cache_service.py              # Redis cache get/set/invalidate
  decorators/
    cache_decorator.py          # @cached decorator for service methods
    __init__.py
```

**Acceptance Criteria:**
- Identical questions from the same tenant return cached responses
- Cache keys include `organization_id` for tenant isolation
- Knowledge source re-ingestion invalidates relevant cached responses
- TTLs are configurable per cache layer (default: response=1h, embedding=24h)

---

### Task 19: Rate Limiting

| Field              | Value                                                                 |
|--------------------|-----------------------------------------------------------------------|
| **Difficulty**     | Medium                                                                |
| **Estimated Time** | 2 hours                                                               |
| **Prerequisites**  | Task 18                                                               |
| **Status**         | [ ] Not Started                                                       |

**Learning Outcomes:**
- Implement sliding window rate limiting with Redis
- Apply per-tenant and per-endpoint rate limits
- Return proper HTTP 429 responses with `Retry-After` header
- Understand rate limiting algorithms: fixed window vs sliding window vs token bucket

**Key Files to Create:**
```
middleware/
  rate_limit_middleware.py       # Redis-based rate limiting
services/
  rate_limit_service.py         # Rate limit check and tracking
```

**Acceptance Criteria:**
- Chat endpoint limited to 100 requests/minute per tenant (configurable)
- Ingestion endpoint limited to 10 requests/minute per tenant
- 429 response includes `Retry-After` header with seconds until limit resets
- Rate limit state is stored in Redis with automatic expiry

---

### Task 20: Ticketing System

| Field              | Value                                                                 |
|--------------------|-----------------------------------------------------------------------|
| **Difficulty**     | Easy                                                                  |
| **Estimated Time** | 3 hours                                                               |
| **Prerequisites**  | Task 14                                                               |
| **Status**         | [ ] Not Started                                                       |

**Learning Outcomes:**
- Build a simple escalation system from AI chat to human support
- Implement ticket creation, assignment, and status management
- Attach chat context (conversation history, retrieved sources) to tickets
- Understand escalation triggers: user request, low confidence, repeated failures

**Key Files to Create:**
```
services/
  ticket_service.py             # Ticket CRUD, assignment, status transitions
routes/
  ticket_routes.py              # Ticket API endpoints
```

**Acceptance Criteria:**
- Users can escalate a chat to a ticket with one click
- Ticket includes full conversation history and retrieved sources
- Tickets have statuses: open, assigned, in_progress, resolved, closed
- List endpoint supports filtering by status and assignee

---

### Task 21: Analytics Dashboard API

| Field              | Value                                                                 |
|--------------------|-----------------------------------------------------------------------|
| **Difficulty**     | Medium                                                                |
| **Estimated Time** | 4 hours                                                               |
| **Prerequisites**  | Task 14, Task 17                                                      |
| **Status**         | [ ] Not Started                                                       |

**Learning Outcomes:**
- Build MongoDB aggregation pipelines for analytics queries
- Track key metrics: chat volume, resolution rates, popular questions, response times
- Implement time-series bucketing for trend visualization
- Design an analytics API that powers a dashboard frontend

**Key Files to Create:**
```
services/
  analytics_service.py          # Aggregation queries, metric computation
routes/
  analytics_routes.py           # Analytics API endpoints
workers/
  analytics_tasks.py            # Scheduled aggregation tasks
```

**Acceptance Criteria:**
- `GET /analytics/overview` returns total chats, resolution rate, avg response time
- `GET /analytics/trends` returns daily/weekly/monthly chat volume
- `GET /analytics/top-questions` returns most asked questions (clustered)
- All analytics are scoped by `organization_id`

---

### Task 22: Chat Widget (SSE Streaming)

| Field              | Value                                                                 |
|--------------------|-----------------------------------------------------------------------|
| **Difficulty**     | Hard                                                                  |
| **Estimated Time** | 5 hours                                                               |
| **Prerequisites**  | Task 14                                                               |
| **Status**         | [ ] Not Started                                                       |

**Learning Outcomes:**
- Implement Server-Sent Events (SSE) for real-time streaming in FastAPI
- Build an embeddable chat widget (HTML/JS snippet)
- Manage widget authentication via API keys (not JWT)
- Handle connection lifecycle: open, message, error, close

**Key Files to Create:**
```
routes/
  widget_routes.py              # SSE streaming endpoint, widget config
services/
  widget_service.py             # Widget auth, session management
static/
  widget/
    widget.js                   # Embeddable chat widget
    widget.css                  # Widget styles
    widget.html                 # Demo page
```

**Acceptance Criteria:**
- Widget is embeddable via a single `<script>` tag with an API key
- Messages stream in real-time using SSE (not WebSocket)
- Widget maintains conversation history within a session
- Widget is customizable: colors, position, welcome message

---

## Phase 6: Security & Polish

> Harden the application with prompt injection defense, RBAC, and structured error handling.
> After this phase the system is secure and production-ready.

---

### Task 23: Prompt Injection Defense

| Field              | Value                                                                 |
|--------------------|-----------------------------------------------------------------------|
| **Difficulty**     | Medium                                                                |
| **Estimated Time** | 2 hours                                                               |
| **Prerequisites**  | Task 13                                                               |
| **Status**         | [ ] Not Started                                                       |

**Learning Outcomes:**
- Understand common prompt injection attacks and their risks
- Implement input sanitization (strip control characters, suspicious patterns)
- Build a detection layer that flags potential injection attempts
- Design defense-in-depth: input validation + prompt structure + output filtering

**Key Files to Create:**
```
services/
  security/
    prompt_guard_service.py     # Input sanitization, injection detection
    __init__.py
tests/
  test_prompt_guard_service.py
```

**Acceptance Criteria:**
- Known injection patterns are detected and blocked (e.g., "ignore previous instructions")
- Control characters and Unicode tricks are sanitized from user input
- Detected injections are logged with full context for review
- Legitimate questions containing flagged keywords are not blocked (low false positive rate)

---

### Task 24: RBAC Authorization

| Field              | Value                                                                 |
|--------------------|-----------------------------------------------------------------------|
| **Difficulty**     | Medium                                                                |
| **Estimated Time** | 3 hours                                                               |
| **Prerequisites**  | Task 3                                                                |
| **Status**         | [ ] Not Started                                                       |

**Learning Outcomes:**
- Design a role hierarchy: owner > admin > agent > viewer
- Implement a permission system mapping roles to allowed actions
- Build a FastAPI dependency that checks permissions on each endpoint
- Understand the principle of least privilege in multi-tenant systems

**Key Files to Create:**
```
auth/
  rbac.py                       # Role definitions, permission mappings
  permissions.py                # Permission check dependency
services/
  user_service.py               # User management, role assignment
routes/
  user_routes.py                # User management endpoints
```

**Acceptance Criteria:**
- Four roles: owner (full access), admin (manage team + settings), agent (chat + tickets), viewer (read-only)
- Permission check dependency can be applied to any route
- Unauthorized actions return 403 with a clear error message
- Owners can invite users and assign roles within their organization

---

### Task 25: Error Handling & Logging

| Field              | Value                                                                 |
|--------------------|-----------------------------------------------------------------------|
| **Difficulty**     | Medium                                                                |
| **Estimated Time** | 3 hours                                                               |
| **Prerequisites**  | Task 5                                                                |
| **Status**         | [ ] Not Started                                                       |

**Learning Outcomes:**
- Build structured logging with Loguru (JSON format, trace IDs, context)
- Implement global error handling middleware for consistent error responses
- Create custom exception classes for business logic errors
- Add request/response logging for debugging and audit

**Key Files to Create:**
```
middleware/
  error_handler_middleware.py    # Global exception handler
  request_logging_middleware.py  # Request/response logging
utils/
  logger.py                     # Loguru configuration
  exceptions.py                 # Custom exception classes
```

**Acceptance Criteria:**
- All errors return a consistent JSON format: `{error, message, trace_id, status_code}`
- Every request gets a unique `trace_id` that appears in all related log entries
- Unhandled exceptions return 500 with the trace ID (no stack trace in response)
- Logs are structured JSON with timestamp, level, trace_id, user_id, org_id

---

## Bonus: Advanced

> These tasks push the system beyond MVP into a sophisticated, production-grade platform.
> Tackle them after completing all prior phases, or selectively based on interest.

---

### Task 26: Reranking with Cross-Encoder

| Field              | Value                                                                 |
|--------------------|-----------------------------------------------------------------------|
| **Difficulty**     | Hard                                                                  |
| **Estimated Time** | 4 hours                                                               |
| **Prerequisites**  | Task 12                                                               |
| **Status**         | [ ] Not Started                                                       |

**Learning Outcomes:**
- Understand the difference between bi-encoders (embedding) and cross-encoders (reranking)
- Integrate a cross-encoder model (e.g., `cross-encoder/ms-marco-MiniLM-L-6-v2`)
- Build a two-stage retrieval pipeline: fast recall (hybrid) -> precise reranking (cross-encoder)
- Benchmark retrieval quality improvement with and without reranking

**Key Files to Create:**
```
services/
  retrieval/
    reranking_service.py        # Cross-encoder reranking
tests/
  test_reranking_service.py
```

**Acceptance Criteria:**
- Reranker scores each (query, chunk) pair and reorders the result list
- Reranking is optional and configurable per assistant
- Latency overhead is under 200ms for top-20 candidates
- A/B comparison shows measurable improvement in answer relevance

---

### Task 27: Agentic RAG with LangGraph

| Field              | Value                                                                 |
|--------------------|-----------------------------------------------------------------------|
| **Difficulty**     | Hard                                                                  |
| **Estimated Time** | 6 hours                                                               |
| **Prerequisites**  | Task 14                                                               |
| **Status**         | [ ] Not Started                                                       |

**Learning Outcomes:**
- Understand agentic RAG: the LLM decides whether to search, clarify, or answer
- Build a LangGraph state machine with nodes: route, retrieve, grade, generate, clarify
- Implement tool use: the agent can search knowledge base, check ticket status, etc.
- Handle multi-step reasoning for complex customer queries

**Key Files to Create:**
```
services/
  agentic/
    graph.py                    # LangGraph state machine definition
    nodes.py                    # Individual node implementations
    tools.py                    # Tool definitions for the agent
    state.py                    # State schema
    __init__.py
tests/
  test_agentic_rag.py
```

**Acceptance Criteria:**
- Agent can handle multi-step queries (e.g., "What is your refund policy and how do I start one?")
- Agent asks clarifying questions when the query is ambiguous
- Agent uses tools: search knowledge base, check order status, create ticket
- Fallback to standard RAG when agentic mode is disabled

---

### Task 28: Deployment (Docker + CI/CD)

| Field              | Value                                                                 |
|--------------------|-----------------------------------------------------------------------|
| **Difficulty**     | Medium                                                                |
| **Estimated Time** | 4 hours                                                               |
| **Prerequisites**  | All previous tasks                                                    |
| **Status**         | [ ] Not Started                                                       |

**Learning Outcomes:**
- Create a production-optimized multi-stage Dockerfile
- Configure GitHub Actions for CI (lint, test, build, push)
- Set up health checks and graceful shutdown
- Understand environment-specific configuration (dev, staging, production)

**Key Files to Create:**
```
Dockerfile.prod                 # Multi-stage production build
.github/
  workflows/
    ci.yml                      # Lint + test + build pipeline
    deploy.yml                  # Deployment pipeline
docker-compose.prod.yml         # Production compose (no volume mounts, resource limits)
scripts/
  healthcheck.py                # Health check script for Docker
```

**Acceptance Criteria:**
- Production Docker image is under 500MB
- CI pipeline runs on every PR: lint (ruff), test (pytest), build (docker)
- Health check endpoint verifies all dependencies (MongoDB, Qdrant, Redis)
- Graceful shutdown completes in-flight requests before stopping

---

## Quick Reference: All Tasks

| #  | Task                              | Difficulty | Time | Prerequisites       |
|----|-----------------------------------|------------|------|---------------------|
| 1  | Project Setup & Docker Compose    | Easy       | 2h   | --                  |
| 2  | MongoDB Models & Database Setup   | Easy       | 3h   | 1                   |
| 3  | Authentication System             | Medium     | 4h   | 2                   |
| 4  | Multi-Tenant Middleware           | Medium     | 3h   | 3                   |
| 5  | Basic CRUD APIs                   | Easy       | 4h   | 4                   |
| 6  | Text Chunking Service             | Medium     | 3h   | 5                   |
| 7  | Embedding Service (MiniLM)        | Medium     | 2h   | 6                   |
| 8  | Qdrant Vector Storage             | Medium     | 3h   | 7                   |
| 9  | URL Crawler & HTML Parser         | Medium     | 4h   | 6                   |
| 10 | File Parser (PDF/DOCX)            | Medium     | 3h   | 6                   |
| 11 | Vector Search Service             | Hard       | 4h   | 8                   |
| 12 | Hybrid Search (BM25 + Vector)     | Hard       | 5h   | 11                  |
| 13 | Prompt Builder & GPT Client       | Hard       | 4h   | 11                  |
| 14 | Chat Service (End-to-End)         | Hard       | 5h   | 12, 13              |
| 15 | Celery Setup & Ingestion Tasks    | Medium     | 3h   | 8, 9, 10            |
| 16 | Job Status Tracking               | Medium     | 2h   | 15                  |
| 17 | Scheduled Tasks (Celery Beat)     | Medium     | 2h   | 16                  |
| 18 | Redis Caching Layer               | Medium     | 3h   | 14                  |
| 19 | Rate Limiting                     | Medium     | 2h   | 18                  |
| 20 | Ticketing System                  | Easy       | 3h   | 14                  |
| 21 | Analytics Dashboard API           | Medium     | 4h   | 14, 17              |
| 22 | Chat Widget (SSE Streaming)       | Hard       | 5h   | 14                  |
| 23 | Prompt Injection Defense          | Medium     | 2h   | 13                  |
| 24 | RBAC Authorization                | Medium     | 3h   | 3                   |
| 25 | Error Handling & Logging          | Medium     | 3h   | 5                   |
| 26 | Reranking with Cross-Encoder      | Hard       | 4h   | 12                  |
| 27 | Agentic RAG with LangGraph        | Hard       | 6h   | 14                  |
| 28 | Deployment (Docker + CI/CD)       | Medium     | 4h   | All                 |

**Difficulty Distribution:** 8 Easy/Medium-Easy, 13 Medium, 7 Hard

**Total Estimated Time:** ~95 hours
