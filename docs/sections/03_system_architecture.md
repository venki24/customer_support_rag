# Section 3: System Architecture & Design Patterns

## Overview

This section explains the complete system architecture of the Customer Support RAG SaaS platform, breaking down how components interact, why specific design patterns were chosen, and how the system handles real-world production challenges like API failures, concurrent requests, and multi-tenancy.

**WHY this architecture matters:**
- Separation of concerns enables independent scaling (API vs workers vs storage)
- Async processing prevents blocking operations from degrading user experience
- Strategic use of caching and circuit breakers ensures system resilience
- Multi-tenancy isolation guarantees data security and performance isolation

---

## C4 Container Diagram: System Overview

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           Customer Support RAG SaaS                          │
│                              [Software System]                               │
└─────────────────────────────────────────────────────────────────────────────┘
                                       │
                    ┌──────────────────┼──────────────────┐
                    │                  │                  │
              ┌─────▼─────┐      ┌────▼────┐      ┌──────▼──────┐
              │  Web App  │      │   API   │      │   Mobile    │
              │ [Browser] │      │ Client  │      │     App     │
              └─────┬─────┘      └────┬────┘      └──────┬──────┘
                    │                 │                  │
                    └─────────────────┼──────────────────┘
                                      │ HTTPS/REST
                    ┌─────────────────▼──────────────────┐
                    │      API Gateway (FastAPI)         │
                    │  [Python/FastAPI Application]      │
                    │  - Auth Middleware (JWT)           │
                    │  - Tenant Isolation Middleware     │
                    │  - Rate Limiting Middleware        │
                    │  - Request Validation              │
                    └────┬────────┬────────┬─────────┬───┘
                         │        │        │         │
         ┌───────────────┘        │        │         └─────────────┐
         │                        │        │                       │
┌────────▼────────┐      ┌────────▼────────▼────────┐      ┌──────▼──────┐
│  Chat Service   │      │  Knowledge Service        │      │   Ticket    │
│  [Core Logic]   │      │  [Core Logic]             │      │  Service    │
│  - Chat Flow    │      │  - Ingestion Orchestration│      │             │
│  - RAG Pipeline │      │  - Chunk Management       │      │             │
│  - Response Gen │      │  - Source Validation      │      │             │
└────┬────────┬───┘      └────────┬──────────────┬───┘      └─────────────┘
     │        │                   │              │
     │        │                   │              │
     │    ┌───▼────────────────┐  │              │
     │    │  Generation        │  │              │
     │    │  Pipeline          │  │              │
     │    │  - Prompt Builder  │  │              │
     │    │  - GPT Call        │  │              │
     │    │  - Circuit Breaker │  │              │
     │    └────────────────────┘  │              │
     │                             │              │
     │    ┌────────────────────┐  │              │
     │    │  Retrieval         │  │              │
     └────►  Pipeline          ◄──┘              │
          │  - Query Embedding │                 │
          │  - Vector Search   │                 │
          │  - Re-ranking      │                 │
          │  - Context Builder │                 │
          └──┬──────────────┬──┘                 │
             │              │                    │
             │              │    ┌───────────────▼────────────────┐
             │              │    │   Background Workers (Celery)  │
             │              │    │   [Async Task Processors]       │
             │              │    │   - Document Ingestion          │
             │              │    │   - Batch Embedding             │
             │              │    │   - Scheduled Sync              │
             │              │    │   - Analytics Aggregation       │
             │              │    └────┬────────────────────────┬───┘
             │              │         │                        │
┌────────────▼──────────┐   │    ┌────▼─────┐          ┌──────▼───────┐
│   Qdrant Vector DB    │   │    │  Redis   │          │   MongoDB    │
│   [Vector Storage]    │   │    │  [Cache/ │          │ [Document DB]│
│   - Embeddings        │   │    │  Queue]  │          │ - Orgs       │
│   - Multi-tenant      │   │    │  - Cache │          │ - Assistants │
│   - Collections       │   └────►  - Celery│          │ - Knowledge  │
│   - Metadata Filter   │         │  - Rate  │          │ - Chats      │
└───────────────────────┘         │  Limits  │          │ - Tickets    │
                                  └──────────┘          └──────────────┘
                                        │
                    ┌───────────────────┼───────────────────┐
                    │                   │                   │
          ┌─────────▼──────────┐  ┌─────▼─────┐  ┌─────────▼─────────┐
          │  MiniLM Embedder   │  │  GPT-4    │  │   External APIs   │
          │  [Local Service]   │  │  [OpenAI] │  │  - Zendesk        │
          │  - Fast            │  │  - Chat   │  │  - Slack          │
          │  - Cost Effective  │  │  - Review │  │  - Email          │
          └────────────────────┘  └───────────┘  └───────────────────┘
```

---

## Component Breakdown

### 1. API Gateway (FastAPI)

**WHY:**
- Single entry point simplifies auth, rate limiting, and monitoring
- FastAPI provides automatic OpenAPI docs, request validation, and async support
- Middleware chain enforces cross-cutting concerns before business logic

**WHAT:**
- JWT authentication middleware validates tokens and extracts tenant context
- Tenant isolation middleware ensures all queries are scoped to organization_id
- Rate limiting middleware prevents abuse and ensures fair resource allocation
- Request/response validation via Pydantic schemas

**HOW:**
```python
# app/main.py
from fastapi import FastAPI, Request
from fastapi.middleware.cors import CORSMiddleware
from contextlib import asynccontextmanager

@asynccontextmanager
async def lifespan(app: FastAPI):
    # Startup: Initialize connections
    await init_mongodb()
    await init_qdrant()
    await init_redis()
    logger.info("Application started")

    yield

    # Shutdown: Cleanup
    await close_mongodb()
    await close_redis()
    logger.info("Application shutdown")

app = FastAPI(
    title="Customer Support RAG API",
    version="1.0.0",
    lifespan=lifespan
)

# Middleware chain (executes in reverse order)
app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_methods=["*"],
)
app.middleware("http")(rate_limit_middleware)
app.middleware("http")(tenant_isolation_middleware)
app.middleware("http")(auth_middleware)

# Register routes
app.include_router(auth_router, prefix="/api/v1/auth", tags=["auth"])
app.include_router(chat_router, prefix="/api/v1/chat", tags=["chat"])
app.include_router(knowledge_router, prefix="/api/v1/knowledge", tags=["knowledge"])
```

### 2. Knowledge Ingestion Pipeline

**WHY:**
- Async processing prevents blocking the API during large document uploads
- Chunking strategy balances context size vs retrieval precision
- Validation ensures only quality content enters the knowledge base

**WHAT:**
- Strategy Pattern for different ingestion types (file, URL, text)
- Chunking with overlap preserves context across boundaries
- Metadata extraction for filtering (source, timestamp, tags)
- Batch embedding generation for efficiency

**HOW - Ingestion Flow:**

```
┌──────────────────────────────────────────────────────────────────┐
│              Knowledge Ingestion Pipeline                        │
└──────────────────────────────────────────────────────────────────┘

User Upload              API Layer              Worker Layer
    │                        │                       │
    │  POST /knowledge       │                       │
    ├───────────────────────►│                       │
    │  {type: "file",        │                       │
    │   content: "..."}      │                       │
    │                        │                       │
    │                        │  1. Validate         │
    │                        │  2. Create Record    │
    │                        │  3. Queue Task       │
    │                        │                       │
    │                        ├──────────────────────►│
    │                        │  Celery Task:         │
    │  202 Accepted          │  process_document()   │
    │◄───────────────────────┤                       │
    │  {task_id: "..."}      │                       │
    │                        │                       │
    │                        │                   ┌───▼────┐
    │                        │                   │ STEP 1 │
    │                        │                   │ Parse  │
    │                        │                   │ Content│
    │                        │                   └───┬────┘
    │                        │                       │
    │                        │                   ┌───▼────┐
    │                        │                   │ STEP 2 │
    │                        │                   │ Clean  │
    │                        │                   │  Text  │
    │                        │                   └───┬────┘
    │                        │                       │
    │                        │                   ┌───▼────┐
    │                        │                   │ STEP 3 │
    │                        │                   │ Chunk  │
    │                        │                   │ (512t) │
    │                        │                   │Overlap │
    │                        │                   │ (50t)  │
    │                        │                   └───┬────┘
    │                        │                       │
    │                        │                   ┌───▼────┐
    │                        │                   │ STEP 4 │
    │                        │                   │ Embed  │
    │                        │                   │(MiniLM)│
    │                        │                   │ Batch  │
    │                        │                   └───┬────┘
    │                        │                       │
    │                        │                   ┌───▼────┐
    │                        │                   │ STEP 5 │
    │                        │                   │ Store  │
    │                        │                   │ Qdrant │
    │                        │                   │+MongoDB│
    │                        │                   └───┬────┘
    │                        │                       │
    │                        │   Task Complete       │
    │                        │◄──────────────────────┤
    │                        │   Update Status       │
    │                        │                       │
    │  GET /knowledge/123    │                       │
    ├───────────────────────►│                       │
    │                        │                       │
    │  200 OK                │                       │
    │◄───────────────────────┤                       │
    │  {status: "ready",     │                       │
    │   chunks: 45}          │                       │
```

### 3. Retrieval Pipeline

**WHY:**
- Hybrid search combines semantic understanding (vectors) with exact matching (metadata)
- Re-ranking improves relevance of top results
- Context window management prevents token limit errors

**WHAT:**
- Query embedding generation using MiniLM
- Vector similarity search in Qdrant (cosine similarity)
- Metadata filtering (organization_id, assistant_id, tags)
- Re-ranking top-K results using cross-encoder
- Context assembly with source attribution

**HOW:**
```python
# app/services/retrieval/retrieval_pipeline.py
from typing import List
from app.models.conversation import Message
from app.utils.embeddings import EmbeddingService
from app.utils.token_counter import count_tokens

class RetrievalPipeline:
    def __init__(
        self,
        embedding_service: EmbeddingService,
        vector_db,
        max_context_tokens: int = 3000
    ):
        self.embedding_service = embedding_service
        self.vector_db = vector_db
        self.max_context_tokens = max_context_tokens

    async def retrieve(
        self,
        query: str,
        organization_id: str,
        assistant_id: str,
        top_k: int = 20,
        re_rank_top_k: int = 5
    ) -> List[dict]:
        # Step 1: Embed query
        query_vector = await self.embedding_service.embed_text(query)

        # Step 2: Vector search with metadata filter
        results = await self.vector_db.search(
            collection_name=f"org_{organization_id}",
            query_vector=query_vector,
            filter={
                "assistant_id": assistant_id,
                "is_active": True
            },
            limit=top_k
        )

        # Step 3: Re-rank top results (optional, adds latency)
        if len(results) > re_rank_top_k:
            results = await self._re_rank(query, results, re_rank_top_k)

        # Step 4: Build context within token limit
        context_chunks = []
        current_tokens = 0

        for result in results:
            chunk_tokens = count_tokens(result["text"])
            if current_tokens + chunk_tokens <= self.max_context_tokens:
                context_chunks.append(result)
                current_tokens += chunk_tokens
            else:
                break

        return context_chunks

    async def _re_rank(self, query: str, results: List[dict], top_k: int):
        # Use cross-encoder model for re-ranking
        # (More compute intensive but better relevance)
        pass
```

### 4. Generation Pipeline

**WHY:**
- Circuit breaker prevents cascading failures when GPT API is down
- Prompt engineering ensures consistent, high-quality responses
- Streaming enables real-time user feedback for long responses

**WHAT:**
- Prompt builder assembles system context, retrieved chunks, and user query
- Circuit breaker wraps GPT API calls with failure detection
- Retry logic with exponential backoff
- Response streaming via Server-Sent Events (SSE)

**HOW - Circuit Breaker Pattern:**

```python
# app/services/generation/circuit_breaker.py
import asyncio
from enum import Enum
from datetime import datetime, timedelta
from typing import Callable, Any

class CircuitState(Enum):
    CLOSED = "closed"      # Normal operation
    OPEN = "open"          # Failing, reject requests
    HALF_OPEN = "half_open"  # Testing if recovered

class CircuitBreaker:
    def __init__(
        self,
        failure_threshold: int = 5,
        recovery_timeout: int = 60,
        expected_exception: type = Exception
    ):
        self.failure_threshold = failure_threshold
        self.recovery_timeout = recovery_timeout
        self.expected_exception = expected_exception

        self.failure_count = 0
        self.last_failure_time = None
        self.state = CircuitState.CLOSED

    async def call(self, func: Callable, *args, **kwargs) -> Any:
        if self.state == CircuitState.OPEN:
            if self._should_attempt_reset():
                self.state = CircuitState.HALF_OPEN
            else:
                raise Exception("Circuit breaker is OPEN")

        try:
            result = await func(*args, **kwargs)
            self._on_success()
            return result
        except self.expected_exception as e:
            self._on_failure()
            raise

    def _on_success(self):
        self.failure_count = 0
        self.state = CircuitState.CLOSED

    def _on_failure(self):
        self.failure_count += 1
        self.last_failure_time = datetime.now()

        if self.failure_count >= self.failure_threshold:
            self.state = CircuitState.OPEN

    def _should_attempt_reset(self) -> bool:
        return (
            self.last_failure_time and
            datetime.now() - self.last_failure_time >= timedelta(seconds=self.recovery_timeout)
        )

# Usage in GPT client
class GPTClient:
    def __init__(self):
        self.circuit_breaker = CircuitBreaker(
            failure_threshold=5,
            recovery_timeout=60
        )

    async def generate(self, messages: List[dict]) -> str:
        return await self.circuit_breaker.call(
            self._call_gpt_api,
            messages
        )
```

### 5. Background Workers (Celery)

**WHY:**
- Offload long-running tasks from API servers
- Independent scaling (more workers during ingestion spikes)
- Task retry and failure handling

**WHAT:**
- Document ingestion tasks (parsing, chunking, embedding)
- Scheduled tasks (sync external systems, aggregate analytics)
- Batch operations (bulk re-embedding, cleanup)

**HOW:**
```python
# app/workers/celery_app.py
from celery import Celery
from app.config import settings

celery_app = Celery(
    "customer_support_rag",
    broker=settings.CELERY_BROKER_URL,  # Redis
    backend=settings.CELERY_RESULT_BACKEND  # Redis
)

celery_app.conf.update(
    task_serializer="json",
    result_serializer="json",
    accept_content=["json"],
    timezone="UTC",
    enable_utc=True,
    task_routes={
        "app.workers.ingestion_tasks.*": {"queue": "ingestion"},
        "app.workers.scheduled_tasks.*": {"queue": "scheduled"},
    }
)

# app/workers/ingestion_tasks.py
@celery_app.task(bind=True, max_retries=3)
async def process_document(self, knowledge_id: str, organization_id: str):
    try:
        # Full ingestion pipeline
        document = await knowledge_service.get_by_id(knowledge_id, organization_id)

        # Parse content
        parsed_content = await parser.parse(document.content, document.content_type)

        # Chunk text
        chunks = await chunker.chunk_text(
            parsed_content,
            chunk_size=512,
            overlap=50
        )

        # Generate embeddings
        embeddings = await embedding_service.batch_embed([c.text for c in chunks])

        # Store in Qdrant
        await vector_db.upsert(
            collection_name=f"org_{organization_id}",
            points=[
                {
                    "id": chunk.id,
                    "vector": embedding,
                    "payload": {
                        "knowledge_id": knowledge_id,
                        "assistant_id": document.assistant_id,
                        "text": chunk.text,
                        "metadata": chunk.metadata
                    }
                }
                for chunk, embedding in zip(chunks, embeddings)
            ]
        )

        # Update status
        await knowledge_service.update_status(knowledge_id, "ready")

    except Exception as e:
        # Retry with exponential backoff
        raise self.retry(exc=e, countdown=2 ** self.request.retries)
```

### 6. Storage Layer

**WHY:**
- MongoDB for flexible document schema and rich queries
- Qdrant for fast vector similarity search
- Redis for caching and rate limiting

**WHAT:**
- **MongoDB**: Organizations, Assistants, Knowledge docs, Conversations, Tickets
- **Qdrant**: Vector embeddings with metadata, multi-tenant collections
- **Redis**: Cache (assistant configs, user sessions), Rate limit counters, Celery queue

---

## Full Request Flow: Chat Query to Response

```
┌────────────────────────────────────────────────────────────────────────────┐
│                     RAG Chat Request Flow                                  │
└────────────────────────────────────────────────────────────────────────────┘

Client               API Gateway         Chat Service      Retrieval       Generation
  │                       │                    │               │                │
  │  POST /chat          │                    │               │                │
  ├──────────────────────►│                    │               │                │
  │  {message: "..."}    │                    │               │                │
  │  Authorization: JWT  │                    │               │                │
  │                      │                    │               │                │
  │                  ┌───▼────┐               │               │                │
  │                  │ Auth   │               │               │                │
  │                  │Verify  │               │               │                │
  │                  │ JWT    │               │               │                │
  │                  └───┬────┘               │               │                │
  │                      │                    │               │                │
  │                  ┌───▼────┐               │               │                │
  │                  │Tenant  │               │               │                │
  │                  │Extract │               │               │                │
  │                  │org_id  │               │               │                │
  │                  └───┬────┘               │               │                │
  │                      │                    │               │                │
  │                  ┌───▼────┐               │               │                │
  │                  │ Rate   │               │               │                │
  │                  │ Limit  │               │               │                │
  │                  │ Check  │               │               │                │
  │                  └───┬────┘               │               │                │
  │                      │                    │               │                │
  │                      ├────────────────────►│               │                │
  │                      │  handle_chat()     │               │                │
  │                      │                    │               │                │
  │                      │                ┌───▼────┐          │                │
  │                      │                │ Check  │          │                │
  │                      │                │ Cache  │          │                │
  │                      │                │ (Redis)│          │                │
  │                      │                └───┬────┘          │                │
  │                      │                    │ MISS          │                │
  │                      │                    │               │                │
  │                      │                    ├───────────────►│                │
  │                      │                    │  retrieve()   │                │
  │                      │                    │               │                │
  │                      │                    │           ┌───▼────┐           │
  │                      │                    │           │ Embed  │           │
  │                      │                    │           │ Query  │           │
  │                      │                    │           └───┬────┘           │
  │                      │                    │               │                │
  │                      │                    │           ┌───▼────┐           │
  │                      │                    │           │Vector  │           │
  │                      │                    │           │Search  │           │
  │                      │                    │           │Qdrant  │           │
  │                      │                    │           └───┬────┘           │
  │                      │                    │               │                │
  │                      │                    │           ┌───▼────┐           │
  │                      │                    │           │Re-rank │           │
  │                      │                    │           │Top-K   │           │
  │                      │                    │           └───┬────┘           │
  │                      │                    │               │                │
  │                      │                    │◄──────────────┤                │
  │                      │                    │  context[]    │                │
  │                      │                    │               │                │
  │                      │                    ├───────────────────────────────►│
  │                      │                    │  generate(query, context)      │
  │                      │                    │                                │
  │                      │                    │                            ┌───▼────┐
  │                      │                    │                            │ Build  │
  │                      │                    │                            │ Prompt │
  │                      │                    │                            └───┬────┘
  │                      │                    │                                │
  │                      │                    │                            ┌───▼────┐
  │                      │                    │                            │Circuit │
  │                      │                    │                            │Breaker │
  │                      │                    │                            │ Check  │
  │                      │                    │                            └───┬────┘
  │                      │                    │                                │
  │                      │                    │                            ┌───▼────┐
  │                      │                    │                            │  GPT   │
  │                      │                    │                            │  Call  │
  │                      │                    │                            └───┬────┘
  │                      │                    │                                │
  │                      │                    │◄───────────────────────────────┤
  │                      │                    │  response                      │
  │                      │                    │                                │
  │                      │                ┌───▼────┐                           │
  │                      │                │ Save   │                           │
  │                      │                │ to DB  │                           │
  │                      │                │MongoDB │                           │
  │                      │                └───┬────┘                           │
  │                      │                    │                                │
  │                      │                ┌───▼────┐                           │
  │                      │                │ Cache  │                           │
  │                      │                │Response│                           │
  │                      │                │ Redis  │                           │
  │                      │                └───┬────┘                           │
  │                      │                    │                                │
  │                      │◄───────────────────┤                                │
  │                      │  response          │                                │
  │                      │                    │                                │
  │  200 OK              │                    │                                │
  │◄─────────────────────┤                    │                                │
  │  {response: "...",   │                    │                                │
  │   sources: [...]}    │                    │                                │
```

---

## Design Patterns in Action

### Repository Pattern (Data Access)

**WHY:**
- Abstracts database implementation details from business logic
- Enables easy testing with mock repositories
- Centralizes query logic for consistency

**WHAT:**
```python
# app/repositories/knowledge_repository.py
from typing import List, Optional
from motor.motor_asyncio import AsyncIOMotorDatabase
from app.models.knowledge import KnowledgeBase

class KnowledgeRepository:
    def __init__(self, db: AsyncIOMotorDatabase):
        self.collection = db.knowledge_bases

    async def create(self, knowledge: KnowledgeBase) -> KnowledgeBase:
        result = await self.collection.insert_one(knowledge.dict())
        knowledge.id = str(result.inserted_id)
        return knowledge

    async def find_by_organization(
        self,
        organization_id: str,
        skip: int = 0,
        limit: int = 20
    ) -> List[KnowledgeBase]:
        cursor = self.collection.find(
            {"organization_id": organization_id, "is_deleted": False}
        ).skip(skip).limit(limit)

        return [KnowledgeBase(**doc) async for doc in cursor]

    async def update_status(
        self,
        knowledge_id: str,
        status: str
    ) -> bool:
        result = await self.collection.update_one(
            {"_id": knowledge_id},
            {"$set": {"status": status, "updated_at": datetime.utcnow()}}
        )
        return result.modified_count > 0
```

### Strategy Pattern (Ingestion Types)

**WHY:**
- Different content sources require different parsing logic
- New ingestion types can be added without modifying existing code
- Consistent interface for all ingestion strategies

**WHAT:**
```python
# app/services/ingestion/strategies.py
from abc import ABC, abstractmethod
from typing import List

class IngestionStrategy(ABC):
    @abstractmethod
    async def extract_content(self, source: str) -> str:
        pass

class FileIngestionStrategy(IngestionStrategy):
    async def extract_content(self, file_path: str) -> str:
        # Handle PDF, DOCX, TXT, etc.
        if file_path.endswith(".pdf"):
            return await self._extract_from_pdf(file_path)
        elif file_path.endswith(".docx"):
            return await self._extract_from_docx(file_path)
        else:
            async with aiofiles.open(file_path, 'r') as f:
                return await f.read()

class URLIngestionStrategy(IngestionStrategy):
    async def extract_content(self, url: str) -> str:
        # Fetch and parse HTML
        async with aiohttp.ClientSession() as session:
            async with session.get(url) as response:
                html = await response.text()
                return self._extract_text_from_html(html)

class TextIngestionStrategy(IngestionStrategy):
    async def extract_content(self, text: str) -> str:
        return text

# Usage
class IngestionService:
    def __init__(self):
        self.strategies = {
            "file": FileIngestionStrategy(),
            "url": URLIngestionStrategy(),
            "text": TextIngestionStrategy()
        }

    async def ingest(self, content_type: str, source: str):
        strategy = self.strategies.get(content_type)
        if not strategy:
            raise ValueError(f"Unknown content type: {content_type}")

        return await strategy.extract_content(source)
```

---

## Interview Questions & Answers

### Q1: Walk me through your RAG architecture from end to end.

**A:** Our RAG architecture has three main pipelines:

1. **Ingestion Pipeline**: When a customer uploads knowledge (docs, URLs), the API validates and creates a MongoDB record, then queues a Celery task. The worker parses content, chunks it (512 tokens with 50 token overlap), generates embeddings using MiniLM, and stores vectors in Qdrant with metadata filtering for multi-tenancy.

2. **Retrieval Pipeline**: When a user asks a question, we embed the query, search Qdrant with organization_id filtering, re-rank the top 20 results to get the best 5, and build context within a 3000 token limit to avoid GPT errors.

3. **Generation Pipeline**: We build a prompt with system context + retrieved chunks + user query, then call GPT-4 through a circuit breaker that prevents cascading failures. The response is streamed back via SSE for real-time feedback and cached in Redis for repeated queries.

All components are containerized with Docker Compose, and we use FastAPI middleware for auth, tenant isolation, and rate limiting at the gateway level.

### Q2: Why did you choose to implement a circuit breaker for LLM calls?

**A:** LLM APIs like OpenAI can fail for multiple reasons: rate limits, service outages, timeout issues. Without a circuit breaker, our entire application would keep making failing requests, wasting resources and money while degrading user experience.

The circuit breaker has three states:
- **CLOSED** (normal): Requests go through normally
- **OPEN** (failing): After 5 consecutive failures, we reject requests immediately and return cached responses or graceful errors
- **HALF-OPEN** (testing): After 60 seconds, we allow one test request to check if the service recovered

This prevents cascading failures, reduces costs during outages, and provides faster error responses to users. We also implement exponential backoff retries for transient failures before opening the circuit.

### Q3: How do you handle multi-tenancy at the vector database level?

**A:** We use a hybrid approach:

1. **Collection-per-tenant**: Each organization gets its own Qdrant collection (`org_{organization_id}`). This provides strong isolation and independent scaling.

2. **Metadata filtering**: Within each collection, we filter by `assistant_id` so customers can have multiple assistants with different knowledge bases.

3. **API Gateway enforcement**: Our tenant isolation middleware extracts `organization_id` from the JWT token and injects it into the request context. Every downstream service automatically scopes queries to that organization.

This prevents data leakage, allows per-tenant resource quotas, and enables independent backup/restore. The tradeoff is more complexity in collection management, but the security and isolation benefits outweigh this cost.

### Q4: Explain your chunking strategy and why you chose those parameters.

**A:** We use **512 token chunks with 50 token overlap**. Here's why:

- **512 tokens**: Balances context size vs retrieval precision. Smaller chunks (256) are too granular and lose context. Larger chunks (1024) reduce retrieval accuracy because they contain too many topics.

- **50 token overlap**: Ensures context continuity across chunk boundaries. If a key concept is split between chunks, the overlap captures it. Without overlap, we'd lose important information at boundaries.

- **Semantic boundaries**: We also respect paragraph and sentence boundaries when possible, so we don't split mid-sentence.

We tested multiple configurations and found this gave the best trade-off between retrieval precision (finding the right chunks) and recall (not missing relevant information).

### Q5: How does your system handle failures during document ingestion?

**A:** We use Celery's built-in retry mechanism with exponential backoff:

1. **Task retry**: If parsing, embedding, or Qdrant storage fails, Celery automatically retries up to 3 times with delays of 2s, 4s, 8s.

2. **Partial progress tracking**: We save chunks to MongoDB before embedding, so if embedding fails, we can resume without re-parsing.

3. **Dead letter queue**: After 3 failures, the task goes to a DLQ for manual investigation. We alert the ops team.

4. **User feedback**: The knowledge status field shows "processing", "ready", or "failed". Users can retry failed ingestions or contact support.

5. **Idempotency**: All ingestion operations are idempotent using knowledge_id, so retries don't create duplicates.

This ensures reliable ingestion while providing visibility into failures.

---

## Summary

This architecture enables:
- **Scalability**: Independent scaling of API, workers, and databases
- **Resilience**: Circuit breakers, retries, and graceful degradation
- **Multi-tenancy**: Strong isolation at all layers
- **Performance**: Caching, async processing, and efficient embeddings
- **Maintainability**: Clear separation of concerns and design patterns

The next section covers the detailed project structure and how these components are organized in code.
