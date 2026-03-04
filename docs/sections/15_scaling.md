# Section 15: Scaling & Performance

## WHY Scaling Matters

Your RAG system may start with 10 customers and 100 documents, but success brings challenges: 1000 customers, millions of documents, 10K queries per minute. Without proper scaling architecture, you'll hit bottlenecks: API timeouts, database contention, memory exhaustion, embedding queue backlogs. Scaling isn't just about handling more load; it's about maintaining sub-2-second response times, keeping costs predictable, and ensuring reliability. A well-architected system scales horizontally (add more machines) rather than vertically (bigger machines), allowing elastic growth and graceful degradation.

## WHAT to Scale

**Horizontal Scaling Strategy**: Every component must be stateless and independently scalable. FastAPI instances behind a load balancer scale based on CPU/memory. Celery workers auto-scale based on queue depth (embedding and ingestion queues). MongoDB replica sets provide read scaling and high availability. Qdrant clusters shard vectors across nodes. Redis clusters distribute cache and session data. The goal: no single point of failure, linear scaling with resources.

**Performance Optimization Areas**:
- **Embedding Performance**: Batch embeddings (process 50-100 chunks at once), GPU acceleration for large ingestion jobs, connection pooling to embedding APIs
- **Database Performance**: Connection pools (10-50 connections per instance), query optimization with proper indexes, async I/O everywhere, read replicas for reporting
- **Vector Search Performance**: HNSW parameter tuning (ef, M), result caching for frequent queries, pre-filtering strategies
- **API Performance**: Response streaming for long answers, async request handling, CDN for static assets, compression (gzip/brotli)

**Cost Optimization**: Costs grow linearly with usage if unchecked. Use GPT-4o-mini for simple queries ($0.15/1M tokens) and GPT-4o for complex ones ($2.50/1M tokens). Cache frequent query results in Redis (TTL 1-24 hours). Batch operations reduce API calls by 10x. Right-size Qdrant nodes (don't over-provision). Monitor per-tenant costs to identify outliers. Implement rate limiting to prevent abuse. Archive old embeddings to cheaper storage.

## HOW to Implement Scaling

### Scaled Deployment Architecture

```
                                 ┌─────────────────┐
                                 │   Load Balancer │
                                 │   (ALB/Nginx)   │
                                 └────────┬────────┘
                                          │
                    ┌─────────────────────┼─────────────────────┐
                    │                     │                     │
              ┌─────▼─────┐         ┌─────▼─────┐       ┌─────▼─────┐
              │  FastAPI  │         │  FastAPI  │       │  FastAPI  │
              │ Instance 1│         │ Instance 2│       │ Instance N│
              │ (Stateless)│        │ (Stateless)│      │ (Stateless)│
              └─────┬─────┘         └─────┬─────┘       └─────┬─────┘
                    │                     │                     │
                    └─────────────────────┼─────────────────────┘
                                          │
                    ┌─────────────────────┼─────────────────────┐
                    │                     │                     │
              ┌─────▼─────┐         ┌─────▼─────┐       ┌─────▼─────┐
              │   Redis   │         │  MongoDB  │       │  Qdrant   │
              │  Cluster  │         │  Replica  │       │  Cluster  │
              │ (Cache/   │         │    Set    │       │ (Sharded  │
              │  Session) │         │ (Primary+ │       │  Vectors) │
              │           │         │  Replicas)│       │           │
              └─────┬─────┘         └─────┬─────┘       └─────┬─────┘
                    │                     │                     │
                    │               ┌─────▼─────────────────────▼─────┐
                    │               │     Celery Worker Pool          │
                    └───────────────►  ┌─────────┬─────────┬─────────┐│
                                    │  │ Worker 1│ Worker 2│ Worker N││
                                    │  │ (Embed) │ (Ingest)│ (Tasks) ││
                                    │  └─────────┴─────────┴─────────┘│
                                    └──────────────────────────────────┘
                                              (Auto-scales by queue depth)
```

### FastAPI Horizontal Scaling

**Stateless API Design**:
```python
# app/main.py - Stateless FastAPI application
from fastapi import FastAPI, Depends
from contextlib import asynccontextmanager
import asyncio

@asynccontextmanager
async def lifespan(app: FastAPI):
    # Startup: Initialize shared resources
    await initialize_mongodb_pool()
    await initialize_redis_pool()
    await initialize_qdrant_client()
    yield
    # Shutdown: Cleanup
    await close_all_connections()

app = FastAPI(lifespan=lifespan)

# NO in-memory state! Use Redis for sessions/cache
# NO file uploads to local disk! Use S3 or database
# NO scheduled jobs in API! Use separate scheduler service

@app.get("/api/v1/query")
async def query_endpoint(
    query: str,
    tenant_id: str = Depends(get_tenant_id),
    db: AsyncIOMotorDatabase = Depends(get_db),
    redis: Redis = Depends(get_redis),
    qdrant: QdrantClient = Depends(get_qdrant)
):
    # Each request gets fresh connections from pool
    # No shared state between requests
    cache_key = f"query:{tenant_id}:{hash(query)}"

    # Check cache
    cached = await redis.get(cache_key)
    if cached:
        return json.loads(cached)

    # Process query
    result = await process_rag_query(query, tenant_id, db, redis, qdrant)

    # Cache for 1 hour
    await redis.setex(cache_key, 3600, json.dumps(result))
    return result
```

**Connection Pooling Configuration**:
```python
# app/database.py - MongoDB connection pool
from motor.motor_asyncio import AsyncIOMotorClient
from pymongo import MongoClient

# Global connection pool (initialized at startup)
_mongodb_client: AsyncIOMotorClient = None

async def initialize_mongodb_pool():
    global _mongodb_client
    _mongodb_client = AsyncIOMotorClient(
        MONGO_URI,
        maxPoolSize=50,  # Max connections per instance
        minPoolSize=10,  # Keep 10 ready
        maxIdleTimeMS=30000,  # Close idle after 30s
        waitQueueTimeoutMS=5000,  # Fail fast if pool exhausted
        serverSelectionTimeoutMS=5000,
        retryWrites=True,
        retryReads=True
    )

async def get_db():
    return _mongodb_client[DB_NAME]

# app/cache.py - Redis connection pool
from redis.asyncio import Redis, ConnectionPool

_redis_pool: ConnectionPool = None

async def initialize_redis_pool():
    global _redis_pool
    _redis_pool = ConnectionPool(
        host=REDIS_HOST,
        port=REDIS_PORT,
        password=REDIS_PASSWORD,
        db=0,
        max_connections=50,
        socket_timeout=5,
        socket_connect_timeout=5,
        decode_responses=True
    )

async def get_redis() -> Redis:
    return Redis(connection_pool=_redis_pool)

# app/vector_store.py - Qdrant connection pool
from qdrant_client import QdrantClient
from qdrant_client.http.api_client import AsyncAPIClient

_qdrant_client: QdrantClient = None

async def initialize_qdrant_client():
    global _qdrant_client
    _qdrant_client = QdrantClient(
        url=QDRANT_URL,
        api_key=QDRANT_API_KEY,
        timeout=10.0,
        # Connection pooling handled by httpx internally
        limits=httpx.Limits(
            max_connections=50,
            max_keepalive_connections=20
        )
    )

async def get_qdrant() -> QdrantClient:
    return _qdrant_client
```

### Celery Auto-Scaling

**Queue-Based Worker Scaling**:
```python
# celery_app.py - Celery with auto-scale configuration
from celery import Celery
from kombu import Queue

celery_app = Celery(
    "rag_workers",
    broker=f"redis://{REDIS_HOST}:{REDIS_PORT}/0",
    backend=f"redis://{REDIS_HOST}:{REDIS_PORT}/1"
)

celery_app.conf.update(
    task_queues=[
        Queue("embedding_queue", routing_key="embedding"),
        Queue("ingestion_queue", routing_key="ingestion"),
        Queue("default_queue", routing_key="default")
    ],
    task_routes={
        "tasks.embed_chunks": {"queue": "embedding_queue"},
        "tasks.ingest_document": {"queue": "ingestion_queue"}
    },
    # Worker auto-scale settings
    worker_prefetch_multiplier=4,  # Prefetch 4 tasks per worker
    worker_max_tasks_per_child=100,  # Restart after 100 tasks (memory leaks)
    task_acks_late=True,  # Ack only after completion (reliability)
    task_reject_on_worker_lost=True,
    task_time_limit=600,  # 10 minute hard limit
    task_soft_time_limit=540  # 9 minute soft limit
)

# Start workers with auto-scale: celery -A celery_app worker --autoscale=10,3
# This runs 3-10 processes per worker node based on queue depth
```

**Batch Embedding for Performance**:
```python
# tasks/embedding.py - Batch embedding task
from celery import group
from typing import List

@celery_app.task(bind=True, max_retries=3)
def embed_chunks(self, chunks: List[dict], tenant_id: str, document_id: str):
    """Batch embed chunks (50-100 at a time) for performance."""
    try:
        batch_size = 50
        embeddings = []

        # Process in batches to avoid API rate limits
        for i in range(0, len(chunks), batch_size):
            batch = chunks[i:i+batch_size]
            texts = [c["text"] for c in batch]

            # Batch API call (10x faster than individual)
            batch_embeddings = embed_batch(texts)
            embeddings.extend(batch_embeddings)

            # Small delay to respect rate limits
            time.sleep(0.1)

        # Bulk upsert to Qdrant (100x faster than individual)
        points = [
            PointStruct(
                id=f"{document_id}_{i}",
                vector=emb,
                payload={**chunks[i], "tenant_id": tenant_id}
            )
            for i, emb in enumerate(embeddings)
        ]

        qdrant_client.upsert(
            collection_name=f"tenant_{tenant_id}",
            points=points,
            wait=True
        )

        return {"status": "success", "embedded": len(chunks)}

    except Exception as exc:
        # Exponential backoff retry
        self.retry(exc=exc, countdown=2 ** self.request.retries)
```

### Database Scaling

**MongoDB Replica Set Configuration**:
```python
# config/mongodb.py - Replica set connection
MONGO_URI = (
    "mongodb://{user}:{password}@"
    "mongo1.example.com:27017,"
    "mongo2.example.com:27017,"
    "mongo3.example.com:27017/"
    "{db_name}?replicaSet=rs0"
    "&readPreference=secondaryPreferred"  # Read from replicas
    "&w=majority"  # Write to majority for safety
    "&retryWrites=true"
)

# For read-heavy operations, use secondaryPreferred
# For critical writes, use primary only
async def get_conversation(conversation_id: str, tenant_id: str):
    # Reads can use replicas
    return await conversations_collection.find_one(
        {"_id": conversation_id, "tenant_id": tenant_id}
    )

async def save_conversation_turn(conversation_id: str, turn: dict):
    # Critical writes go to primary
    return await conversations_collection.update_one(
        {"_id": conversation_id},
        {"$push": {"turns": turn}},
        # Force primary for consistency
        read_preference=ReadPreference.PRIMARY
    )
```

**Qdrant Clustering**:
```yaml
# qdrant-cluster.yaml - Distributed Qdrant configuration
# 3-node cluster with sharding
version: '3.8'
services:
  qdrant-node1:
    image: qdrant/qdrant:latest
    environment:
      - QDRANT__CLUSTER__ENABLED=true
      - QDRANT__CLUSTER__P2P__PORT=6335
      - QDRANT__CLUSTER__CONSENSUS__TICK_PERIOD_MS=100
    volumes:
      - ./qdrant_data1:/qdrant/storage

  qdrant-node2:
    image: qdrant/qdrant:latest
    environment:
      - QDRANT__CLUSTER__ENABLED=true
      - QDRANT__CLUSTER__P2P__PORT=6335
      - QDRANT__CLUSTER__BOOTSTRAP_URL=http://qdrant-node1:6335

  qdrant-node3:
    image: qdrant/qdrant:latest
    environment:
      - QDRANT__CLUSTER__ENABLED=true
      - QDRANT__CLUSTER__P2P__PORT=6335
      - QDRANT__CLUSTER__BOOTSTRAP_URL=http://qdrant-node1:6335

# Create collections with sharding
# curl -X PUT 'http://localhost:6333/collections/tenant_123' \
#   -H 'Content-Type: application/json' \
#   -d '{
#     "vectors": {"size": 384, "distance": "Cosine"},
#     "shard_number": 3,
#     "replication_factor": 2
#   }'
```

### Performance Optimization

**Response Streaming**:
```python
# app/routes/query.py - Stream responses for better UX
from fastapi.responses import StreamingResponse
import asyncio

@app.post("/api/v1/query/stream")
async def query_stream(request: QueryRequest, tenant_id: str = Depends(get_tenant_id)):
    """Stream response chunks as they're generated."""

    async def generate_response():
        # 1. Retrieve context (fast)
        context_chunks = await retrieve_context(request.query, tenant_id)

        # Send context immediately
        yield json.dumps({
            "type": "context",
            "chunks": [c.dict() for c in context_chunks]
        }) + "\n"

        # 2. Stream LLM response token by token
        prompt = build_prompt(request.query, context_chunks)

        async for token in stream_llm_response(prompt):
            yield json.dumps({
                "type": "token",
                "content": token
            }) + "\n"

        # 3. Send final metadata
        yield json.dumps({
            "type": "complete",
            "metadata": {"tokens": 150, "latency_ms": 1200}
        }) + "\n"

    return StreamingResponse(
        generate_response(),
        media_type="application/x-ndjson"
    )

async def stream_llm_response(prompt: str):
    """Stream tokens from OpenAI API."""
    response = await openai_client.chat.completions.create(
        model="gpt-4o",
        messages=[{"role": "user", "content": prompt}],
        stream=True
    )

    async for chunk in response:
        if chunk.choices[0].delta.content:
            yield chunk.choices[0].delta.content
```

**Query Result Caching**:
```python
# app/services/cache_service.py - Intelligent caching
import hashlib
from typing import Optional

async def get_cached_query_result(
    query: str,
    tenant_id: str,
    filters: dict,
    redis: Redis
) -> Optional[dict]:
    """Cache query results with semantic hashing."""
    # Hash query semantically (ignore whitespace, case)
    normalized_query = " ".join(query.lower().split())
    cache_key = f"query:{tenant_id}:{hashlib.md5(normalized_query.encode()).hexdigest()}"

    if filters:
        cache_key += f":{hashlib.md5(json.dumps(filters, sort_keys=True).encode()).hexdigest()}"

    cached = await redis.get(cache_key)
    if cached:
        return json.loads(cached)
    return None

async def cache_query_result(
    query: str,
    tenant_id: str,
    filters: dict,
    result: dict,
    redis: Redis,
    ttl: int = 3600  # 1 hour default
):
    """Cache result with TTL."""
    normalized_query = " ".join(query.lower().split())
    cache_key = f"query:{tenant_id}:{hashlib.md5(normalized_query.encode()).hexdigest()}"

    if filters:
        cache_key += f":{hashlib.md5(json.dumps(filters, sort_keys=True).encode()).hexdigest()}"

    await redis.setex(cache_key, ttl, json.dumps(result))
```

## Production Tips

### Load Testing Strategy

**Gradual Load Testing**:
```bash
# Use Locust or K6 for load testing
# locustfile.py
from locust import HttpUser, task, between

class RAGUser(HttpUser):
    wait_time = between(1, 3)  # 1-3 seconds between requests

    @task(3)
    def query_simple(self):
        self.client.post("/api/v1/query", json={
            "query": "What is your return policy?",
            "tenant_id": "test_tenant"
        })

    @task(1)
    def query_complex(self):
        self.client.post("/api/v1/query", json={
            "query": "Compare shipping options for international orders",
            "tenant_id": "test_tenant"
        })

    @task(1)
    def ingest_document(self):
        self.client.post("/api/v1/ingest", json={
            "url": "https://example.com/docs/page1.html",
            "tenant_id": "test_tenant"
        })

# Run test: locust -f locustfile.py --host=http://localhost:8000
# Test progression:
# 1. 10 users -> baseline latency
# 2. 100 users -> normal load
# 3. 500 users -> peak load
# 4. 1000 users -> stress test
# Monitor: p50, p95, p99 latency, error rate, CPU, memory, DB connections
```

### Capacity Planning

**Resource Estimation**:
```
Per-tenant resource usage:
- Embeddings: 1M chunks × 384 dims × 4 bytes = 1.5 GB vector storage
- MongoDB: 1M documents × 2 KB avg = 2 GB document storage
- Redis cache: 10K frequent queries × 5 KB = 50 MB
- Compute: 1K queries/day × 2s avg = 33 minutes CPU time/day

Scaling thresholds:
- Add FastAPI instance: When avg CPU > 70% or p95 latency > 2s
- Add Celery worker: When embedding queue depth > 100
- Add MongoDB replica: When read latency > 100ms
- Add Qdrant node: When vector count > 10M per node

Cost projection (AWS):
- FastAPI (3× t3.large): $150/month
- Celery workers (5× t3.medium): $125/month
- MongoDB (replica set, 3× m5.large): $300/month
- Qdrant (3× m5.xlarge with SSD): $450/month
- Redis (r6g.large cluster): $100/month
- OpenAI API (1M queries/month, GPT-4o-mini avg): $200/month
Total: ~$1325/month for 100 tenants = $13.25/tenant/month
```

## Interview Questions

**Q1: How would you scale a RAG system from 10 to 10,000 concurrent users?**

A: Start by making the API layer stateless (no local state, use Redis for sessions) and deploy behind a load balancer to add FastAPI instances horizontally. Scale Celery workers auto-scaling based on queue depth for background embedding jobs. Implement MongoDB replica sets for read scaling and use read preference for non-critical reads. Cluster Qdrant with sharding to distribute vector search across nodes. Add connection pooling (50+ connections per instance) for all databases. Implement response caching in Redis for frequent queries with 1-hour TTL. Monitor p95 latency and add capacity when it exceeds 2s. Use response streaming to improve perceived performance for long-running queries.

**Q2: What are the most critical performance optimizations for a RAG system?**

A: Batch embedding operations (50-100 chunks at once) instead of embedding individually to reduce API overhead by 10x. Implement connection pooling for MongoDB, Redis, and Qdrant to avoid connection setup latency (adds 50-100ms per request). Use async I/O everywhere (FastAPI, Motor, asyncio) to handle thousands of concurrent requests with minimal threads. Cache frequent query results in Redis with semantic hashing to serve repeated queries instantly. Stream LLM responses token-by-token for better UX and lower perceived latency. Tune HNSW parameters (ef, M) in Qdrant for optimal recall/speed tradeoff. Pre-filter vectors by tenant_id at query time instead of storing separate collections.

**Q3: How would you optimize costs in a multi-tenant RAG system?**

A: Implement intelligent model selection—use GPT-4o-mini ($0.15/1M tokens) for simple FAQ queries and GPT-4o ($2.50/1M tokens) only for complex reasoning. Cache frequent queries in Redis to avoid redundant LLM calls (30-50% hit rate typical). Batch ingestion jobs to reduce per-document overhead. Right-size Qdrant nodes based on actual vector count (don't over-provision). Use MongoDB compression to reduce storage by 50-70%. Implement per-tenant rate limiting to prevent abuse. Archive inactive tenants' embeddings to cheaper S3 storage and lazy-load on query. Monitor per-tenant costs and flag outliers (top 10% often use 50% of resources). Set query complexity limits (max context chunks, max output tokens) to cap worst-case costs.

**Q4: Why is connection pooling critical, and what happens without it?**

A: Without connection pooling, each request creates a new database connection (TCP handshake, auth, SSL) adding 50-200ms latency per request. Under high load, this causes connection exhaustion (databases have max connection limits like 1000) leading to "too many connections" errors and complete outages. Connection pooling maintains a pool of ready connections (10-50 per instance) that requests reuse. Connections are validated before use (check if stale) and recycled after a timeout. This reduces latency to <5ms for getting a connection and prevents exhaustion. Configure maxPoolSize (50), minPoolSize (10), and maxIdleTimeMS (30000) appropriately. Monitor connection pool metrics (in-use, available, wait time) to detect issues early.

**Q5: How would you design a load testing strategy for a RAG production deployment?**

A: Use Locust or K6 to simulate realistic user patterns (70% simple queries, 20% complex queries, 10% ingestion). Start with baseline load (10 users) to measure ideal latency, then gradually increase to normal load (100 users), peak load (500 users), and stress test (1000+ users) to find breaking points. Test for 30+ minutes at each level to catch memory leaks and connection exhaustion. Monitor key metrics: p50/p95/p99 latency, error rate, CPU/memory per service, database connection pool usage, queue depths. Define SLOs (e.g., p95 latency <2s, error rate <0.1%) and fail tests that violate them. Run load tests weekly in staging and before every major production deployment. Test specific scenarios: thundering herd (all users query at once), large document ingestion, cache cold start.
