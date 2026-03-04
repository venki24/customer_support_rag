# Section 2: Multi-Tenant Architecture

## WHY: Multi-Tenancy is Non-Negotiable for SaaS

### The SaaS Economics Problem

**Without multi-tenancy** (dedicated instances per customer):
- Customer A needs 10GB storage → Deploy dedicated server ($50/month)
- Customer B needs 50GB storage → Deploy dedicated server ($200/month)
- 1000 customers → 1000 servers to manage
- **Cost**: $125,000/month infrastructure + operational nightmare

**With multi-tenancy** (shared infrastructure):
- All customers share same database/vector store
- Resource pooling: Total 60TB storage across 10 servers ($10,000/month)
- **Cost savings**: 92% reduction in infrastructure costs
- **Operational efficiency**: Single deployment, unified monitoring

### The Security Challenge

**Critical requirement**: Complete data isolation between tenants.

**Nightmare scenarios**:
- Acme Inc sees BigCorp's customer conversations
- TechStartup's knowledge base leaks to Competitor Inc
- Bulk query accidentally returns data from all tenants

**Our mandate**: **Zero data leakage** while maximizing resource sharing.

---

## WHAT: Multi-Tenancy Isolation Strategies

### Strategy 1: Database-Per-Tenant (Highest Isolation)

```
┌─────────────┐  ┌─────────────┐  ┌─────────────┐
│  MongoDB    │  │  MongoDB    │  │  MongoDB    │
│  (Acme Inc) │  │  (BigCorp)  │  │ (TechStart) │
└─────────────┘  └─────────────┘  └─────────────┘
```

**Pros**:
- Perfect isolation: Physical separation
- Easy to backup/restore individual tenants
- Regulatory compliance (GDPR, HIPAA)

**Cons**:
- **Expensive**: N databases = N server costs
- **Operational overhead**: N backups, N migrations
- **Not scalable**: 10,000 tenants = 10,000 databases

**When to use**: Enterprise SaaS with large customers paying $10K+/month.

---

### Strategy 2: Collection-Per-Tenant (Shared Database)

```
MongoDB (Shared Instance)
├── collections/
│   ├── acme_conversations
│   ├── acme_knowledge
│   ├── bigcorp_conversations
│   ├── bigcorp_knowledge
│   └── techstart_conversations
```

**Pros**:
- Better than DB-per-tenant: Easier management
- Good isolation: Queries naturally scoped to collection
- Tenant-level backups possible

**Cons**:
- **Connection limit**: MongoDB max 10,000 collections
- **Query complexity**: Must dynamically switch collections
- **Cross-tenant queries hard**: Analytics across all tenants = N queries

**When to use**: 100-1000 tenants with moderate data volumes.

---

### Strategy 3: Document-Per-Tenant (Shared Collection) ✅ **Our Choice**

```
MongoDB: conversations (Shared Collection)
┌────────────────────────────────────────────┐
│ { _id: "conv1", tenant_id: "acme",         │
│   messages: [...] }                        │
├────────────────────────────────────────────┤
│ { _id: "conv2", tenant_id: "bigcorp",      │
│   messages: [...] }                        │
├────────────────────────────────────────────┤
│ { _id: "conv3", tenant_id: "acme",         │
│   messages: [...] }                        │
└────────────────────────────────────────────┘

Index: { tenant_id: 1, created_at: -1 }
```

**Pros**:
- **Maximum resource sharing**: Single collection, no limits
- **Simple queries**: Add `tenant_id` to every filter
- **Cross-tenant analytics**: Aggregate across all tenants easily
- **Cost-effective**: Optimal for SMB SaaS (thousands of small tenants)

**Cons**:
- **Requires discipline**: Every query MUST include `tenant_id`
- **Noisy neighbor risk**: Large tenant can slow others (mitigate with sharding)
- **Index bloat**: Large multi-tenant indexes (mitigate with compound indexes)

**When to use**: B2B SaaS with 1,000-100,000 small-medium tenants (our sweet spot).

---

## WHAT: Our Multi-Tenant Design

### Decision Matrix

| Layer | Isolation Strategy | Rationale |
|-------|-------------------|-----------|
| **MongoDB** | Document-level (`tenant_id` field) | Max scalability for 10K+ tenants |
| **Qdrant (Vectors)** | Collection-per-tenant | Easier to manage, better isolation for embeddings |
| **Redis** | Key-prefix-per-tenant (`tenant:{org_id}:*`) | Shared instance, logical separation |
| **Celery** | Task metadata includes `tenant_id` | Shared workers, filtered queues |

### Qdrant: Why Collection-Per-Tenant?

**Qdrant peculiarity**: Vector databases optimize for similarity search, not filtering.

**Problem with document-level in Qdrant**:
```python
# BAD: Filter vectors by tenant_id
results = qdrant_client.search(
    collection_name="all_vectors",
    query_vector=[0.23, -0.45, ...],
    query_filter={"tenant_id": "acme"}  # ❌ Slow! Scans all vectors first
)
```
- Qdrant searches ALL vectors, then filters → O(N) performance hit
- No index on metadata filters in most vector DBs

**Solution: Collection-per-tenant**:
```python
# GOOD: Dedicated collection
results = qdrant_client.search(
    collection_name=f"tenant_{org_id}",  # ✅ Fast! Only searches this tenant's vectors
    query_vector=[0.23, -0.45, ...]
)
```
- Searches only tenant's vectors → O(log N) with HNSW index
- Easier to scale: Move hot tenants to dedicated Qdrant instances

**Trade-off**: Qdrant can handle 10,000+ collections, but beyond that we'd need multi-instance sharding.

---

## HOW: Tenant Context Propagation

### The Golden Rule

**Every request must carry `tenant_id` from authentication to database query.**

```
┌─────────────────────────────────────────────────────────────┐
│                    REQUEST LIFECYCLE                         │
└─────────────────────────────────────────────────────────────┘

1. HTTP Request
   └─> Headers: { Authorization: "Bearer <JWT>" }

2. TenantMiddleware
   └─> Decode JWT → Extract org_id → Store in request.state.tenant_id

3. Route Handler
   └─> Access request.state.tenant_id → Pass to service layer

4. Service Layer
   └─> All queries include tenant_id filter

5. Database/Vector Store
   └─> Returns only tenant's data
```

---

### Implementation: Tenant Middleware

```python
# middleware/tenant_middleware.py

from fastapi import Request, HTTPException
from starlette.middleware.base import BaseHTTPMiddleware
import jwt

class TenantMiddleware(BaseHTTPMiddleware):
    """
    Extracts tenant_id from JWT and injects into request state.
    ALL downstream code accesses tenant_id via request.state.tenant_id.
    """

    async def dispatch(self, request: Request, call_next):
        # Skip auth for public endpoints
        if request.url.path in ["/health", "/docs", "/openapi.json"]:
            return await call_next(request)

        # Extract JWT from Authorization header
        auth_header = request.headers.get("Authorization")
        if not auth_header or not auth_header.startswith("Bearer "):
            raise HTTPException(status_code=401, detail="Missing or invalid token")

        token = auth_header.split(" ")[1]

        try:
            # Decode JWT (validate signature, expiration)
            payload = jwt.decode(
                token,
                key=settings.JWT_SECRET,
                algorithms=["HS256"]
            )

            # Extract tenant_id (org_id) from token claims
            tenant_id = payload.get("org_id")
            if not tenant_id:
                raise HTTPException(status_code=403, detail="Token missing org_id claim")

            # Inject into request state (accessible in all route handlers)
            request.state.tenant_id = tenant_id
            request.state.user_id = payload.get("user_id")  # Bonus: user context

        except jwt.ExpiredSignatureError:
            raise HTTPException(status_code=401, detail="Token expired")
        except jwt.InvalidTokenError:
            raise HTTPException(status_code=401, detail="Invalid token")

        # Continue request processing
        response = await call_next(request)
        return response
```

**Register in FastAPI**:
```python
# app.py

from fastapi import FastAPI
from middleware.tenant_middleware import TenantMiddleware

app = FastAPI()

# Add tenant middleware (runs on EVERY request)
app.add_middleware(TenantMiddleware)
```

---

### Implementation: Tenant-Scoped Query Helpers

**Problem**: Developers might forget to add `tenant_id` to queries.

**Solution**: Helper functions that enforce tenant filtering.

```python
# utils/tenant_utils.py

from fastapi import Request, HTTPException
from typing import Dict, Any

def get_tenant_id(request: Request) -> str:
    """
    Extract tenant_id from request state.
    Raises 403 if missing (middleware should prevent this).
    """
    tenant_id = getattr(request.state, "tenant_id", None)
    if not tenant_id:
        raise HTTPException(status_code=403, detail="Tenant context missing")
    return tenant_id


def tenant_filter(request: Request, additional_filters: Dict[str, Any] = None) -> Dict[str, Any]:
    """
    Build MongoDB filter with tenant_id.

    Usage:
        filter = tenant_filter(request, {"status": "active"})
        # Returns: {"tenant_id": "acme", "status": "active"}
    """
    tenant_id = get_tenant_id(request)
    base_filter = {"tenant_id": tenant_id}

    if additional_filters:
        base_filter.update(additional_filters)

    return base_filter


def tenant_collection_name(request: Request, base_name: str) -> str:
    """
    Generate tenant-specific collection name for Qdrant.

    Usage:
        collection = tenant_collection_name(request, "knowledge")
        # Returns: "tenant_acme_knowledge"
    """
    tenant_id = get_tenant_id(request)
    return f"tenant_{tenant_id}_{base_name}"
```

---

### Implementation: FastAPI Dependency Injection

**Best practice**: Use FastAPI dependencies to automatically inject tenant context.

```python
# dependencies/tenant_deps.py

from fastapi import Depends, Request
from typing import Annotated

async def get_current_tenant(request: Request) -> str:
    """
    Dependency: Extract tenant_id from request.
    Use in route handlers for automatic injection.
    """
    tenant_id = getattr(request.state, "tenant_id", None)
    if not tenant_id:
        raise HTTPException(status_code=403, detail="Tenant context missing")
    return tenant_id

# Type alias for cleaner signatures
TenantId = Annotated[str, Depends(get_current_tenant)]
```

**Usage in route handlers**:
```python
# routes/conversations.py

from fastapi import APIRouter, Depends
from dependencies.tenant_deps import TenantId
from models.conversation import Conversation

router = APIRouter(prefix="/conversations", tags=["conversations"])

@router.get("/")
async def list_conversations(tenant_id: TenantId):
    """
    List conversations for authenticated tenant.
    tenant_id is automatically injected from JWT via dependency.
    """
    conversations = await Conversation.find(
        Conversation.tenant_id == tenant_id
    ).to_list()

    return {"conversations": conversations}


@router.post("/")
async def create_conversation(
    tenant_id: TenantId,
    data: ConversationCreate
):
    """
    Create new conversation scoped to tenant.
    """
    conversation = Conversation(
        tenant_id=tenant_id,  # ✅ Always set tenant_id
        messages=data.messages,
        status="active"
    )
    await conversation.insert()

    return {"conversation_id": str(conversation.id)}
```

---

### Implementation: Service Layer Enforcement

**Every service method must accept and use `tenant_id`.**

```python
# services/knowledge_service.py

from beanie import Document
from typing import List
from models.knowledge_doc import KnowledgeDoc

class KnowledgeService:
    """
    Business logic for knowledge management.
    ALL methods require tenant_id for data isolation.
    """

    async def add_document(
        self,
        tenant_id: str,
        content: str,
        source_url: str,
        metadata: dict
    ) -> str:
        """
        Add document to tenant's knowledge base.
        """
        doc = KnowledgeDoc(
            tenant_id=tenant_id,  # ✅ Critical: Scope to tenant
            content=content,
            source_url=source_url,
            metadata=metadata,
            status="pending_embedding"
        )
        await doc.insert()

        # Trigger background job to embed document
        await self._enqueue_embedding_task(tenant_id, str(doc.id))

        return str(doc.id)


    async def list_documents(
        self,
        tenant_id: str,
        skip: int = 0,
        limit: int = 50
    ) -> List[KnowledgeDoc]:
        """
        List tenant's knowledge documents.
        """
        docs = await KnowledgeDoc.find(
            KnowledgeDoc.tenant_id == tenant_id  # ✅ Always filter by tenant
        ).skip(skip).limit(limit).to_list()

        return docs


    async def delete_document(self, tenant_id: str, doc_id: str):
        """
        Delete document (tenant-scoped).
        Prevents cross-tenant deletion attacks.
        """
        doc = await KnowledgeDoc.find_one(
            KnowledgeDoc.id == doc_id,
            KnowledgeDoc.tenant_id == tenant_id  # ✅ Double-check tenant ownership
        )

        if not doc:
            raise ValueError("Document not found or access denied")

        await doc.delete()

        # Also delete from Qdrant
        await self._delete_from_vector_store(tenant_id, doc_id)
```

---

### Implementation: Qdrant Tenant Collections

```python
# services/vector_service.py

from qdrant_client import QdrantClient
from qdrant_client.models import Distance, VectorParams, PointStruct
from sentence_transformers import SentenceTransformer
from typing import List

class VectorService:
    def __init__(self):
        self.qdrant = QdrantClient(host="localhost", port=6333)
        self.embedding_model = SentenceTransformer("sentence-transformers/all-MiniLM-L6-v2")


    def _get_collection_name(self, tenant_id: str) -> str:
        """Generate tenant-specific collection name."""
        return f"tenant_{tenant_id}"


    async def ensure_tenant_collection(self, tenant_id: str):
        """
        Create Qdrant collection for tenant if it doesn't exist.
        Called when org is created.
        """
        collection_name = self._get_collection_name(tenant_id)

        # Check if collection exists
        collections = self.qdrant.get_collections().collections
        exists = any(c.name == collection_name for c in collections)

        if not exists:
            self.qdrant.create_collection(
                collection_name=collection_name,
                vectors_config=VectorParams(
                    size=384,  # MiniLM embedding dimension
                    distance=Distance.COSINE
                )
            )


    async def add_vectors(
        self,
        tenant_id: str,
        doc_id: str,
        chunks: List[str]
    ):
        """
        Embed text chunks and store in tenant's Qdrant collection.
        """
        collection_name = self._get_collection_name(tenant_id)

        # Generate embeddings
        embeddings = self.embedding_model.encode(chunks)

        # Prepare points
        points = [
            PointStruct(
                id=f"{doc_id}_chunk_{i}",
                vector=embedding.tolist(),
                payload={
                    "doc_id": doc_id,
                    "chunk_index": i,
                    "text": chunk
                }
            )
            for i, (chunk, embedding) in enumerate(zip(chunks, embeddings))
        ]

        # Upsert to tenant's collection
        self.qdrant.upsert(
            collection_name=collection_name,
            points=points
        )


    async def search(
        self,
        tenant_id: str,
        query: str,
        top_k: int = 5
    ) -> List[dict]:
        """
        Semantic search within tenant's knowledge base.
        """
        collection_name = self._get_collection_name(tenant_id)

        # Embed query
        query_vector = self.embedding_model.encode(query).tolist()

        # Search in tenant's collection only
        results = self.qdrant.search(
            collection_name=collection_name,
            query_vector=query_vector,
            limit=top_k
        )

        return [
            {
                "text": hit.payload["text"],
                "doc_id": hit.payload["doc_id"],
                "score": hit.score
            }
            for hit in results
        ]


    async def delete_tenant_collection(self, tenant_id: str):
        """
        Delete entire tenant collection (when org is deleted).
        """
        collection_name = self._get_collection_name(tenant_id)
        self.qdrant.delete_collection(collection_name)
```

---

## HOW: Multi-Tenant Data Flow Diagram

```
┌──────────────────────────────────────────────────────────────────┐
│                    MULTI-TENANT REQUEST FLOW                      │
└──────────────────────────────────────────────────────────────────┘

STEP 1: Authentication
━━━━━━━━━━━━━━━━━━━━━━
Admin (Acme Inc) → POST /api/conversations
Headers: { Authorization: "Bearer eyJhbGc..." }

        ↓

STEP 2: TenantMiddleware
━━━━━━━━━━━━━━━━━━━━━━━━
┌─────────────────────────────────────┐
│  Decode JWT                         │
│  {                                  │
│    "user_id": "user_123",           │
│    "org_id": "acme",  ← TENANT ID   │
│    "exp": 1735689600                │
│  }                                  │
└─────────────────────────────────────┘
        ↓
request.state.tenant_id = "acme"

        ↓

STEP 3: Route Handler (with Dependency Injection)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
async def create_conversation(tenant_id: TenantId):
    # tenant_id = "acme" (injected automatically)
    ...

        ↓

STEP 4: Service Layer
━━━━━━━━━━━━━━━━━━━━━━
await Conversation.find(
    Conversation.tenant_id == "acme",  ← CRITICAL FILTER
    Conversation.status == "active"
).to_list()

        ↓

STEP 5: MongoDB Query
━━━━━━━━━━━━━━━━━━━━━━
db.conversations.find({
    "tenant_id": "acme",  ← ISOLATION
    "status": "active"
})

Index used: { tenant_id: 1, status: 1 }

        ↓

STEP 6: Qdrant Vector Search (if needed)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
collection_name = "tenant_acme"  ← ISOLATED COLLECTION

qdrant.search(
    collection_name="tenant_acme",
    query_vector=[...],
    limit=5
)

        ↓

STEP 7: Response
━━━━━━━━━━━━━━━━
{ "conversations": [ ... ] }  ← Only Acme Inc's data


═══════════════════════════════════════════════════════════════════
SECURITY GUARANTEE: BigCorp's data is NEVER queried or returned
═══════════════════════════════════════════════════════════════════

MongoDB collections:
┌─────────────────────────────────────────┐
│ conversations                            │
│ ├─ { tenant_id: "acme", ... }      ✅   │
│ ├─ { tenant_id: "bigcorp", ... }   🚫   │ ← Never fetched
│ └─ { tenant_id: "techstart", ... } 🚫   │
└─────────────────────────────────────────┘

Qdrant collections:
┌─────────────────────────────────────────┐
│ tenant_acme         ✅ Searched          │
│ tenant_bigcorp      🚫 Not touched       │
│ tenant_techstart    🚫 Not touched       │
└─────────────────────────────────────────┘
```

---

## HOW: Production Best Practices

### 1. Database Indexes (Critical for Performance)

```python
# models/conversation.py

from beanie import Document, Indexed
from datetime import datetime
from typing import List

class Conversation(Document):
    # Always index tenant_id for fast filtering
    tenant_id: Indexed(str)  # ✅ Index for tenant isolation

    user_id: str
    messages: List[dict]
    status: str
    created_at: datetime = datetime.utcnow()

    class Settings:
        name = "conversations"
        indexes = [
            # Compound index for common queries
            [("tenant_id", 1), ("status", 1), ("created_at", -1)],

            # Index for user-specific queries
            [("tenant_id", 1), ("user_id", 1)],
        ]
```

**Why compound indexes matter**:
```python
# Without compound index: MongoDB scans ALL tenant's docs, then filters by status
# With compound index: MongoDB uses index to jump directly to matching docs

# Query: Active conversations for tenant (sorted by date)
await Conversation.find(
    Conversation.tenant_id == "acme",
    Conversation.status == "active"
).sort("-created_at").to_list()

# Index used: (tenant_id, status, created_at) → O(log N) lookup
```

---

### 2. Prevent Accidental Cross-Tenant Queries

**Linter rule** (using custom pre-commit hook):
```python
# scripts/lint_tenant_filter.py

import ast
import sys

def check_beanie_queries(file_path: str):
    """
    Ensure all Beanie .find() calls include tenant_id filter.
    """
    with open(file_path) as f:
        tree = ast.parse(f.read())

    errors = []

    for node in ast.walk(tree):
        if isinstance(node, ast.Call):
            # Check for Model.find() calls
            if (hasattr(node.func, 'attr') and
                node.func.attr == 'find'):

                # Verify tenant_id is in filter
                has_tenant_filter = any(
                    "tenant_id" in ast.unparse(arg)
                    for arg in node.args
                )

                if not has_tenant_filter:
                    errors.append(f"Line {node.lineno}: Missing tenant_id filter in .find()")

    if errors:
        print(f"❌ {file_path}:")
        for error in errors:
            print(f"  {error}")
        return False

    return True

# Run on all service files
if __name__ == "__main__":
    success = check_beanie_queries(sys.argv[1])
    sys.exit(0 if success else 1)
```

---

### 3. Testing Multi-Tenancy

```python
# tests/test_multi_tenancy.py

import pytest
from httpx import AsyncClient
from app import app

@pytest.mark.asyncio
async def test_tenant_isolation():
    """
    Ensure Tenant A cannot access Tenant B's data.
    """
    async with AsyncClient(app=app, base_url="http://test") as client:
        # Create conversation as Tenant A
        token_a = generate_jwt(org_id="acme", user_id="alice")
        resp_a = await client.post(
            "/api/conversations",
            headers={"Authorization": f"Bearer {token_a}"},
            json={"messages": [{"role": "user", "content": "Hello"}]}
        )
        assert resp_a.status_code == 201
        conv_id = resp_a.json()["conversation_id"]

        # Try to access as Tenant B
        token_b = generate_jwt(org_id="bigcorp", user_id="bob")
        resp_b = await client.get(
            f"/api/conversations/{conv_id}",
            headers={"Authorization": f"Bearer {token_b}"}
        )

        # Should return 404 (not 403, to avoid leaking existence)
        assert resp_b.status_code == 404


@pytest.mark.asyncio
async def test_vector_search_isolation():
    """
    Ensure vector search only returns tenant's knowledge.
    """
    # Add doc to Tenant A
    await vector_service.add_vectors(
        tenant_id="acme",
        doc_id="doc1",
        chunks=["Acme refund policy: 30 days"]
    )

    # Add doc to Tenant B
    await vector_service.add_vectors(
        tenant_id="bigcorp",
        doc_id="doc2",
        chunks=["BigCorp refund policy: 60 days"]
    )

    # Search as Tenant A
    results_a = await vector_service.search(
        tenant_id="acme",
        query="refund policy"
    )

    # Should only return Acme's doc
    assert len(results_a) == 1
    assert "Acme" in results_a[0]["text"]
    assert "BigCorp" not in results_a[0]["text"]
```

---

### 4. Monitoring & Alerting

**Track tenant activity**:
```python
# utils/telemetry.py

from prometheus_client import Counter, Histogram

# Metrics per tenant
tenant_requests = Counter(
    "tenant_requests_total",
    "Total requests per tenant",
    ["tenant_id", "endpoint"]
)

tenant_query_duration = Histogram(
    "tenant_query_duration_seconds",
    "Query duration per tenant",
    ["tenant_id", "collection"]
)

# Usage in service
async def list_conversations(tenant_id: str):
    with tenant_query_duration.labels(tenant_id=tenant_id, collection="conversations").time():
        conversations = await Conversation.find(
            Conversation.tenant_id == tenant_id
        ).to_list()

    tenant_requests.labels(tenant_id=tenant_id, endpoint="/conversations").inc()

    return conversations
```

**Alerts** (Prometheus + Grafana):
- **Noisy neighbor**: Alert if 1 tenant uses >50% of resources
- **Query slowness**: Alert if tenant queries >1s (missing index?)
- **Data growth**: Alert if tenant storage exceeds quota

---

## Interview Q&A

### Q1: How do you ensure data isolation between tenants?

**Answer**:
We use a **layered isolation strategy**:

1. **MongoDB** (Document-level): Every document has `tenant_id` field, all queries filtered
2. **Qdrant** (Collection-per-tenant): Each tenant gets dedicated vector collection (`tenant_{org_id}`)
3. **Middleware enforcement**: JWT decoded → `tenant_id` extracted → Injected into `request.state` → All queries automatically scoped

**Code enforcement**:
- Middleware runs on EVERY request (no bypass)
- FastAPI dependencies inject `tenant_id` (developers can't forget)
- Custom linter checks for missing `tenant_id` filters
- Integration tests verify cross-tenant access fails

**Audit trail**: Every DB write logs `tenant_id` + `user_id` for forensics.

### Q2: Why collection-per-tenant in Qdrant but document-level in MongoDB?

**Answer**:
**MongoDB**: Document-level isolation works because:
- Queries naturally filter by fields → Add `tenant_id` to `WHERE` clause
- Compound indexes make this efficient: `(tenant_id, status, created_at)`
- Cross-tenant analytics easy: `db.conversations.aggregate([{ $group: { _id: "$tenant_id", count: { $sum: 1 }}}])`

**Qdrant**: Collection-per-tenant necessary because:
- Vector search optimizes for similarity, not filtering
- Metadata filters (like `tenant_id`) are applied AFTER vector search → Performance hit
- Dedicated collections ensure tenant A's search never touches tenant B's vectors
- Easier to scale: Move hot tenants to dedicated Qdrant instances

**Trade-off**: Qdrant has collection limits (~10K), but we can shard across multiple Qdrant clusters if needed.

### Q3: What happens if a developer forgets to add `tenant_id` to a query?

**Answer**:
**Multiple safety layers**:

1. **Pre-commit hook**: Custom linter scans code for `.find()` calls without `tenant_id`
   ```bash
   # .git/hooks/pre-commit
   python scripts/lint_tenant_filter.py changed_files
   ```

2. **FastAPI dependencies**: Force developers to use `tenant_id: TenantId` in handlers
   ```python
   # This fails if tenant_id not in JWT
   async def handler(tenant_id: TenantId):
       ...
   ```

3. **Service layer convention**: All service methods MUST accept `tenant_id` as first param
   ```python
   async def list_docs(self, tenant_id: str, ...):
       # If tenant_id not passed, type checker complains
   ```

4. **Integration tests**: CI/CD runs cross-tenant access tests
   - Create data as Tenant A
   - Try to access as Tenant B
   - Assert 404 response

5. **Code review checklist**: PR template includes "Verified tenant_id in all queries?"

**Worst case**: If query leaks data, monitoring alerts on slow query (missing index) → Incident response.

### Q4: How do you handle "noisy neighbor" problems (one tenant slowing others)?

**Answer**:
**Noisy neighbor** = Tenant A's heavy usage impacts Tenant B's performance.

**Monitoring**:
- Track per-tenant metrics: Requests/sec, DB query time, vector search latency
- Alert if 1 tenant >50% of total load

**Mitigation strategies**:

1. **Rate limiting** (per-tenant):
   ```python
   from slowapi import Limiter
   limiter = Limiter(key_func=lambda req: req.state.tenant_id)

   @app.get("/api/search")
   @limiter.limit("100/minute")  # 100 searches per tenant per minute
   async def search(tenant_id: TenantId, query: str):
       ...
   ```

2. **MongoDB sharding** (shard key = `tenant_id`):
   - Large tenants automatically distributed across shards
   - Isolates heavy write load

3. **Qdrant dedicated instances**:
   - Detect hot tenant via metrics → Migrate collection to dedicated Qdrant node
   - Update service config: `tenant_routing = {"acme": "qdrant-2.internal"}`

4. **Queue prioritization**:
   - Celery tasks tagged with `tenant_id`
   - Large tenants routed to separate worker pool

5. **Tier-based limits** (business logic):
   - Free tier: 100 messages/month, 10MB storage
   - Pro tier: 10K messages/month, 1GB storage
   - Enforce in middleware or service layer

### Q5: How do you test multi-tenancy in development?

**Answer**:
**Challenges**: Hard to simulate multiple tenants locally.

**Solution: Seed script**:
```python
# scripts/seed_tenants.py

async def seed_dev_data():
    """
    Create 3 test tenants with realistic data.
    """
    tenants = [
        {"org_id": "acme", "name": "Acme Inc"},
        {"org_id": "bigcorp", "name": "BigCorp"},
        {"org_id": "techstart", "name": "TechStart"}
    ]

    for tenant in tenants:
        # Create org
        org = Organization(**tenant)
        await org.insert()

        # Create Qdrant collection
        await vector_service.ensure_tenant_collection(tenant["org_id"])

        # Add sample knowledge
        await knowledge_service.add_document(
            tenant_id=tenant["org_id"],
            content=f"Sample knowledge for {tenant['name']}",
            source_url=f"https://{tenant['org_id']}.com/docs"
        )

        # Create sample conversations
        for i in range(5):
            conv = Conversation(
                tenant_id=tenant["org_id"],
                user_id=f"user_{i}",
                messages=[{"role": "user", "content": "Hello"}],
                status="active"
            )
            await conv.insert()

# Run: python scripts/seed_tenants.py
```

**Testing workflow**:
1. Run seed script → 3 tenants with data
2. Generate JWTs for each tenant:
   ```python
   token_acme = generate_jwt(org_id="acme", user_id="alice")
   token_bigcorp = generate_jwt(org_id="bigcorp", user_id="bob")
   ```
3. Test API with different tokens:
   ```bash
   # Should see Acme's data
   curl -H "Authorization: Bearer $TOKEN_ACME" http://localhost:8000/api/conversations

   # Should see BigCorp's data
   curl -H "Authorization: Bearer $TOKEN_BIGCORP" http://localhost:8000/api/conversations
   ```

**Automated tests**: Pytest fixtures create/teardown tenant data per test.

---

## Key Takeaways

1. **Multi-tenancy = SaaS economics**: Shared infra reduces costs by 90%+
2. **Isolation strategy depends on layer**: Document-level for MongoDB (queryable), collection-per-tenant for Qdrant (search-optimized)
3. **Middleware is non-negotiable**: Extract `tenant_id` from JWT, inject into every request
4. **Compound indexes critical**: `(tenant_id, other_fields)` for fast queries
5. **Defense in depth**: Middleware + dependencies + linters + tests + monitoring

**Next section**: RAG pipeline implementation (ingestion, chunking, embedding, retrieval).
