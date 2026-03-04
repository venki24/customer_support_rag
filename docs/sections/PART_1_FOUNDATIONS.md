# Section 1: Product Overview & Vision

## WHY: The Customer Support Problem

Customer support teams face a critical bottleneck: **answering the same questions repeatedly**. Traditional solutions fall short:

- **Static FAQs**: Customers struggle to find answers, poor search UX
- **Human-only support**: Doesn't scale, expensive, slow response times
- **Rule-based chatbots**: Brittle, require manual scripting, limited to predefined flows
- **Knowledge silos**: Information scattered across docs, wikis, tickets, databases

**The cost**: Poor customer experience, high support costs, agent burnout.

**The opportunity**: AI that understands your business context and provides accurate, natural answers 24/7.

---

## WHAT: Introducing RAG (Retrieval-Augmented Generation)

### RAG Explained Simply

**Traditional AI chatbots** (fine-tuned LLMs):
- Train a model on your data → expensive, time-consuming
- Model "memorizes" information → can hallucinate, hard to update
- Black box → can't verify sources

**RAG approach** (our solution):
1. **Index**: Store knowledge in a searchable vector database
2. **Retrieve**: When user asks a question, find relevant docs using semantic search
3. **Generate**: Pass retrieved context to LLM (GPT-4) to generate natural answer
4. **Cite**: Show sources so users can verify information

```
User Question: "What's your refund policy?"
       ↓
   [Vector Search] → Find top 5 relevant docs from knowledge base
       ↓
   [GPT-4 Prompt] → "Given these docs: [context], answer: What's your refund policy?"
       ↓
   AI Answer + Source Citations
```

**Why RAG wins for customer support**:
- **Fresh knowledge**: Update instantly without retraining
- **Verifiable**: Every answer links to source documents
- **Cost-effective**: No expensive fine-tuning, use base GPT models
- **Transparent**: See exactly what knowledge informed each answer

---

## WHAT: Product Feature Breakdown

### 1. Knowledge Vault (Multi-Source Ingestion)

Businesses add knowledge through multiple channels:

**URL Crawling**:
- Admin inputs website URLs (docs, blog, support pages)
- System crawls, extracts text, chunks content
- Example: `https://docs.acme.com/api` → 50 indexed chunks

**File Uploads**:
- Support PDFs, DOCX, TXT, MD, HTML
- Parse and extract text, preserve structure
- Example: Product manuals, internal wikis

**Manual Text Entry**:
- Quick Q&A pairs, snippets
- Example: "What payment methods do you accept? → Visa, Mastercard, PayPal"

**Structured Data** (future):
- Connect to databases, CRMs, ticketing systems
- Example: Pull product specs from Postgres

### 2. AI Chat Widget

**Customer-facing chatbot**:
- Embeddable JavaScript widget
- Customizable UI (colors, logo, position)
- Real-time streaming responses
- Mobile-responsive

**Conversation flow**:
```
Customer → "How do I reset my password?"
AI → [Searches knowledge] → "Here's how to reset your password: 1) Go to..."
     [Shows source: docs.acme.com/password-reset]
Customer → "What if I don't receive the email?"
AI → [Uses conversation context] → "If you don't receive the email, check spam..."
```

### 3. Multi-Tenant Organizations

**SaaS model**:
- Each business = 1 organization (tenant)
- Isolated knowledge bases
- Team collaboration (invite users, role-based access)
- Usage quotas (messages/month, storage limits)

### 4. Ticketing Integration

**Escalation workflow**:
- If AI can't answer → Create support ticket
- Human agent reviews, provides answer
- Answer fed back into knowledge base → AI learns

### 5. Analytics Dashboard

**Track performance**:
- Conversation volume, resolution rate
- Most asked questions (identify knowledge gaps)
- User satisfaction scores (thumbs up/down)
- Response time, token usage costs

### 6. Customization

**Branding**:
- Widget colors, fonts, logo
- Custom welcome message
- Personality/tone settings ("professional" vs "casual")

**Behavior**:
- Fallback responses
- Confidence threshold (when to escalate)
- Language support

---

## HOW: User Journey

### Admin Journey: Setting Up AI Support

```
1. SIGNUP
   Admin visits app.answerhq.com → Sign up
   └─> Creates organization "Acme Inc"

2. ADD KNOWLEDGE
   Admin navigates to Knowledge Vault
   ├─> Crawl URLs: https://docs.acme.com, https://acme.com/faq
   ├─> Upload files: product_manual.pdf, troubleshooting_guide.docx
   └─> Add text: "Q: What's our SLA? A: 99.9% uptime guarantee"

   [Backend processing]
   - Text extraction → Chunking → Embedding (MiniLM) → Store in Qdrant

3. CONFIGURE WIDGET
   Admin → Settings → Chat Widget
   ├─> Set brand colors (#FF5733), upload logo
   ├─> Customize greeting: "Hi! I'm AcmeBot. How can I help?"
   └─> Copy embed code: <script src="https://cdn.answerhq.com/widget.js" data-org-id="acme123"></script>

4. DEPLOY
   Admin pastes script into website footer
   └─> Widget goes live instantly

5. MONITOR
   Admin → Analytics Dashboard
   └─> Sees 127 conversations, 89% resolved by AI, 11% escalated to tickets
```

### Customer Journey: Getting Help

```
1. ENCOUNTER ISSUE
   Customer on acme.com → Clicks chat widget (bottom-right)

2. ASK QUESTION
   Customer: "Do you integrate with Slack?"

3. AI RESPONSE
   [Backend: Vector search → GPT-4 generation]
   AI: "Yes! We integrate with Slack. You can connect your workspace via Settings > Integrations..."
       Source: docs.acme.com/integrations/slack

4. FOLLOW-UP
   Customer: "What about Microsoft Teams?"
   AI: [Context-aware] "We don't currently support Microsoft Teams, but it's on our roadmap..."

5. RESOLUTION or ESCALATION
   ├─> Resolved: Customer clicks thumbs-up → Conversation ends
   └─> Not resolved: AI → "I'll connect you with our team" → Creates ticket
```

---

## HOW: High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                         CUSTOMER SUPPORT RAG SAAS                │
└─────────────────────────────────────────────────────────────────┘

┌───────────────┐         ┌───────────────────────────────────────┐
│  ADMIN PANEL  │         │         CHAT WIDGET (JS)              │
│  (React SPA)  │         │    (Embedded on customer websites)    │
└───────┬───────┘         └────────────┬──────────────────────────┘
        │                              │
        │ REST API                     │ WebSocket (real-time chat)
        ▼                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                       FastAPI Backend                            │
│  ┌────────────┐  ┌─────────────┐  ┌──────────────┐             │
│  │   Auth     │  │   Orgs &    │  │     Chat     │             │
│  │  (JWT)     │  │   Knowledge │  │  (Streaming) │             │
│  └────────────┘  └─────────────┘  └──────────────┘             │
│  ┌────────────────────────────────────────────────┐             │
│  │         Tenant Isolation Middleware            │             │
│  │      (X-Tenant-ID → Filter all queries)        │             │
│  └────────────────────────────────────────────────┘             │
└───┬─────────────┬──────────────┬───────────────┬────────────────┘
    │             │              │               │
    │             │              │               │
┌───▼────┐  ┌─────▼──────┐  ┌───▼─────────┐  ┌─▼──────────────┐
│MongoDB │  │   Qdrant   │  │   Redis     │  │  Celery        │
│        │  │  (Vectors) │  │  (Cache +   │  │  (Background   │
│ Users  │  │            │  │   Sessions) │  │   Workers)     │
│ Orgs   │  │ Collection │  │             │  │                │
│ Docs   │  │ per Tenant │  │             │  │ - URL Crawling │
│ Convos │  │            │  │             │  │ - Embedding    │
│ Tickets│  │            │  │             │  │ - Analytics    │
└────────┘  └─────┬──────┘  └─────────────┘  └────────────────┘
                  │
            ┌─────▼──────┐
            │  MiniLM    │
            │ (sentence- │
            │transformers│
            │  384-dim)  │
            └────────────┘

┌────────────────────────────────────────────────────────────────┐
│                      EXTERNAL SERVICES                          │
│  ┌──────────────┐          ┌───────────────────┐              │
│  │   GPT-4 API  │          │  Email/SMS (SES,  │              │
│  │  (OpenAI)    │          │   Twilio)         │              │
│  └──────────────┘          └───────────────────┘              │
└────────────────────────────────────────────────────────────────┘

DATA FLOW (Chat Query):
1. Customer asks: "What's your refund policy?"
2. Widget → FastAPI /chat/message (WebSocket)
3. Middleware extracts tenant_id from JWT
4. Embed question using MiniLM → [0.23, -0.45, 0.12, ...]
5. Query Qdrant collection "tenant_acme123" → Top 5 chunks
6. Build GPT-4 prompt: "Context: [chunk1, chunk2...] Question: ..."
7. Stream GPT-4 response → WebSocket → Widget
8. Store conversation in MongoDB (tenant-scoped)
```

---

## Interview Q&A

### Q1: What is RAG and why use it for customer support?

**Answer**:
RAG (Retrieval-Augmented Generation) combines information retrieval with large language models. Instead of training a custom AI model on your data (expensive, slow), RAG:
1. Stores knowledge in a searchable vector database
2. Retrieves relevant documents when user asks a question
3. Passes context to GPT to generate natural answers

**Why for customer support**:
- **Fresh knowledge**: Update docs instantly without retraining
- **Verifiable**: Every answer cites sources
- **Cost-effective**: No fine-tuning, use base GPT models
- **Handles ambiguity**: Semantic search finds answers even with different phrasing

**Example**: Customer asks "How do I cancel?" → System finds "Cancellation Policy" doc → GPT generates conversational answer with citation.

### Q2: RAG vs fine-tuning for customer support?

**Answer**:

| Aspect | RAG (Our Choice) | Fine-Tuning |
|--------|------------------|-------------|
| **Setup time** | Hours (index docs) | Weeks (train model) |
| **Cost** | Low (embedding + GPT calls) | High (GPU training) |
| **Updates** | Instant (add new docs) | Retrain entire model |
| **Accuracy** | High (uses latest docs) | Degrades over time |
| **Transparency** | Shows sources | Black box |
| **Scalability** | Easy (add more docs) | Hard (retrain for each tenant) |

**When fine-tuning makes sense**: Highly specialized domains (medical, legal) where you need model to "memorize" rare patterns. For customer support, RAG wins because knowledge changes frequently (new features, policy updates).

### Q3: How does multi-tenancy work in this architecture?

**Answer**:
Each business (tenant) gets:
- **Isolated knowledge base**: Separate Qdrant collection (`tenant_{org_id}`)
- **Isolated data**: MongoDB queries filtered by `tenant_id`
- **Usage tracking**: Quota enforcement (messages/month, storage limits)

**Middleware** extracts `tenant_id` from JWT on every request and injects into all downstream queries. This ensures Acme Inc never sees BigCorp's knowledge or conversations.

**Why collection-per-tenant in Qdrant**: Better isolation, easier to scale (move hot tenants to dedicated instances), simpler deletion (drop collection vs scan-and-delete documents).

### Q4: What happens when AI can't answer a question?

**Answer**:
**Confidence scoring**: Each RAG response includes a confidence score (0-1) based on:
- Vector similarity of retrieved documents
- GPT's certainty (we prompt it to say "I don't know" if unsure)

**Fallback flow**:
1. Confidence < 0.6 → AI says "I don't have enough information. Let me connect you with our team."
2. Create support ticket in MongoDB
3. Notify human agent (email, Slack webhook)
4. Agent resolves ticket → Answer added back to knowledge base
5. **Feedback loop**: AI learns from human responses

**Analytics**: Track escalation rate to identify knowledge gaps.

### Q5: How do you handle conversational context (follow-up questions)?

**Answer**:
**Session management**:
- Store conversation history in Redis (fast access) and MongoDB (persistence)
- Each message includes `session_id` and `previous_messages[]`

**Context window**:
```python
# Simplified example
def build_prompt(current_question: str, history: list):
    context_chunks = retrieve_relevant_docs(current_question)  # Vector search

    conversation_context = "\n".join([
        f"User: {msg.question}\nAI: {msg.answer}"
        for msg in history[-3:]  # Last 3 exchanges
    ])

    return f"""
    Context from knowledge base: {context_chunks}

    Conversation history:
    {conversation_context}

    Current question: {current_question}

    Provide a helpful answer considering the conversation history.
    """
```

**Why Redis + MongoDB**: Redis for active sessions (TTL 30 min), MongoDB for long-term storage and analytics.

---

## Key Takeaways

1. **RAG > Fine-tuning** for customer support: Faster, cheaper, more transparent
2. **Multi-source knowledge**: URLs, files, text → Unified vector index
3. **Tenant isolation**: Collection-per-tenant in Qdrant, middleware-enforced filtering
4. **Human-in-the-loop**: AI escalates to tickets, learns from human answers
5. **Production-ready stack**: FastAPI (async), Qdrant (scale), Celery (background jobs), Redis (sessions)

**Next section**: Deep dive into multi-tenant architecture implementation.
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
# Section 4: Project Structure & Setup

## Overview

This section provides a complete walkthrough of how to structure a production-ready FastAPI application for a Customer Support RAG SaaS. We cover the directory layout, key files, configuration management, dependency injection, and development workflow.

**WHY this structure matters:**
- Clear separation of concerns makes the codebase maintainable as it grows
- Modular design enables independent testing and development of features
- Configuration management prevents hardcoded values and supports multiple environments
- Dependency injection makes code testable and loosely coupled

---

## Complete Directory Structure

```
customer-support-rag/
│
├── app/                              # Main application package
│   ├── __init__.py
│   ├── main.py                       # FastAPI application entry point
│   ├── config.py                     # Pydantic settings for configuration
│   ├── dependencies.py               # Dependency injection setup
│   │
│   ├── middleware/                   # Custom middleware components
│   │   ├── __init__.py
│   │   ├── auth.py                   # JWT authentication middleware
│   │   ├── tenant.py                 # Tenant isolation middleware
│   │   ├── rate_limit.py             # Rate limiting middleware
│   │   └── logging.py                # Request/response logging
│   │
│   ├── models/                       # Database models (MongoDB/Beanie)
│   │   ├── __init__.py
│   │   ├── organization.py           # Organization/tenant entity
│   │   ├── user.py                   # User and authentication
│   │   ├── assistant.py              # AI assistant configuration
│   │   ├── knowledge.py              # Knowledge base documents
│   │   ├── conversation.py           # Chat conversations and messages
│   │   ├── ticket.py                 # Support ticket integration
│   │   └── analytics.py              # Usage analytics and metrics
│   │
│   ├── schemas/                      # Pydantic request/response schemas
│   │   ├── __init__.py
│   │   ├── auth.py                   # Login, token, registration schemas
│   │   ├── chat.py                   # Chat request/response schemas
│   │   ├── knowledge.py              # Knowledge CRUD schemas
│   │   ├── assistant.py              # Assistant configuration schemas
│   │   └── common.py                 # Shared schemas (pagination, errors)
│   │
│   ├── routes/                       # API route handlers
│   │   ├── __init__.py
│   │   ├── auth.py                   # POST /auth/login, /auth/register
│   │   ├── chat.py                   # POST /chat, GET /chat/{id}
│   │   ├── knowledge.py              # CRUD for knowledge bases
│   │   ├── assistant.py              # CRUD for assistants
│   │   ├── tickets.py                # Ticket management endpoints
│   │   ├── analytics.py              # Analytics and reporting
│   │   └── health.py                 # Health check endpoints
│   │
│   ├── services/                     # Business logic layer
│   │   ├── __init__.py
│   │   ├── auth_service.py           # User authentication and JWT
│   │   ├── chat_service.py           # Main chat orchestration
│   │   ├── knowledge_service.py      # Knowledge management
│   │   ├── assistant_service.py      # Assistant configuration
│   │   │
│   │   ├── ingestion/                # Knowledge ingestion pipeline
│   │   │   ├── __init__.py
│   │   │   ├── ingestion_service.py  # Main ingestion orchestrator
│   │   │   ├── strategies.py         # Strategy pattern for sources
│   │   │   ├── parsers.py            # Content parsers (PDF, HTML, etc.)
│   │   │   └── validators.py         # Content validation
│   │   │
│   │   ├── retrieval/                # Retrieval pipeline
│   │   │   ├── __init__.py
│   │   │   ├── retrieval_pipeline.py # Main retrieval orchestrator
│   │   │   ├── embeddings.py         # Embedding generation
│   │   │   ├── vector_search.py      # Qdrant vector search
│   │   │   └── reranker.py           # Result re-ranking
│   │   │
│   │   └── generation/               # Response generation pipeline
│   │       ├── __init__.py
│   │       ├── generation_pipeline.py # Main generation orchestrator
│   │       ├── prompt_builder.py     # Prompt construction
│   │       ├── gpt_client.py         # OpenAI GPT client
│   │       └── circuit_breaker.py    # Circuit breaker implementation
│   │
│   ├── repositories/                 # Data access layer (Repository pattern)
│   │   ├── __init__.py
│   │   ├── organization_repo.py
│   │   ├── user_repo.py
│   │   ├── assistant_repo.py
│   │   ├── knowledge_repo.py
│   │   ├── conversation_repo.py
│   │   └── vector_repo.py            # Qdrant operations
│   │
│   ├── workers/                      # Celery background workers
│   │   ├── __init__.py
│   │   ├── celery_app.py             # Celery configuration
│   │   ├── ingestion_tasks.py        # Document ingestion tasks
│   │   ├── scheduled_tasks.py        # Periodic tasks (analytics, cleanup)
│   │   └── webhook_tasks.py          # External webhook processing
│   │
│   ├── utils/                        # Utility functions and helpers
│   │   ├── __init__.py
│   │   ├── chunking.py               # Text chunking strategies
│   │   ├── text_cleaner.py           # Text preprocessing
│   │   ├── token_counter.py          # Token counting for LLMs
│   │   ├── embeddings.py             # Embedding utilities
│   │   ├── cache.py                  # Redis caching helpers
│   │   └── logger.py                 # Structured logging setup
│   │
│   └── exceptions/                   # Custom exception classes
│       ├── __init__.py
│       ├── auth_exceptions.py
│       ├── knowledge_exceptions.py
│       └── rag_exceptions.py
│
├── tests/                            # Test suite
│   ├── __init__.py
│   ├── conftest.py                   # Pytest fixtures and configuration
│   ├── unit/                         # Unit tests
│   │   ├── services/
│   │   ├── repositories/
│   │   └── utils/
│   ├── integration/                  # Integration tests
│   │   ├── test_chat_flow.py
│   │   ├── test_ingestion_flow.py
│   │   └── test_auth_flow.py
│   └── e2e/                          # End-to-end tests
│       └── test_full_workflow.py
│
├── scripts/                          # Utility scripts
│   ├── init_db.py                    # Initialize database and collections
│   ├── seed_data.py                  # Seed test data
│   ├── migrate.py                    # Database migrations
│   └── backup.py                     # Backup utilities
│
├── docker/                           # Docker configuration
│   ├── Dockerfile                    # Main application Dockerfile
│   ├── Dockerfile.worker             # Celery worker Dockerfile
│   └── nginx.conf                    # Nginx reverse proxy config
│
├── alembic/                          # Database migrations (if using SQL)
│   ├── versions/
│   └── env.py
│
├── .env.example                      # Environment variable template
├── .env                              # Actual environment variables (gitignored)
├── .gitignore
├── docker-compose.yml                # Local development environment
├── docker-compose.prod.yml           # Production environment
├── requirements.txt                  # Python dependencies
├── requirements-dev.txt              # Development dependencies
├── pytest.ini                        # Pytest configuration
├── README.md                         # Project documentation
└── Makefile                          # Common commands
```

---

## Key Files Deep Dive

### 1. main.py - FastAPI Application Entry Point

**WHY:**
- Central configuration point for the entire application
- Lifespan management for startup/shutdown tasks
- Middleware registration in correct order
- Route registration with proper prefixes

**WHAT:**
```python
# app/main.py
from contextlib import asynccontextmanager
from fastapi import FastAPI, Request
from fastapi.middleware.cors import CORSMiddleware
from fastapi.responses import JSONResponse
import time

from app.config import settings
from app.database import init_mongodb, close_mongodb, init_redis, close_redis
from app.dependencies import get_qdrant_client
from app.routes import (
    auth_router,
    chat_router,
    knowledge_router,
    assistant_router,
    tickets_router,
    analytics_router,
    health_router
)
from app.middleware.auth import auth_middleware
from app.middleware.tenant import tenant_isolation_middleware
from app.middleware.rate_limit import rate_limit_middleware
from app.utils.logger import logger

@asynccontextmanager
async def lifespan(app: FastAPI):
    """
    Manage application lifespan: startup and shutdown events.
    Replaces deprecated @app.on_event("startup") and @app.on_event("shutdown").
    """
    # STARTUP
    logger.info("Starting Customer Support RAG API...")

    # Initialize databases
    await init_mongodb()
    logger.info("MongoDB connected")

    await init_redis()
    logger.info("Redis connected")

    # Initialize Qdrant
    qdrant_client = get_qdrant_client()
    logger.info("Qdrant client initialized")

    # Initialize embedding model
    from app.utils.embeddings import EmbeddingService
    embedding_service = EmbeddingService()
    await embedding_service.load_model()
    logger.info("Embedding model loaded")

    logger.info("Application startup complete")

    yield  # Application runs here

    # SHUTDOWN
    logger.info("Shutting down Customer Support RAG API...")

    await close_mongodb()
    logger.info("MongoDB connection closed")

    await close_redis()
    logger.info("Redis connection closed")

    logger.info("Application shutdown complete")

# Initialize FastAPI app
app = FastAPI(
    title="Customer Support RAG API",
    description="AI-powered customer support with RAG capabilities",
    version="1.0.0",
    lifespan=lifespan,
    docs_url="/docs" if settings.ENVIRONMENT != "production" else None,
    redoc_url="/redoc" if settings.ENVIRONMENT != "production" else None,
)

# CORS Configuration
app.add_middleware(
    CORSMiddleware,
    allow_origins=settings.ALLOWED_ORIGINS,
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# Custom Middleware (executes in reverse order of addition)
# Order: Request -> Rate Limit -> Tenant -> Auth -> Route Handler
@app.middleware("http")
async def add_request_id(request: Request, call_next):
    """Add unique request ID for tracing."""
    request_id = request.headers.get("X-Request-ID", f"req_{int(time.time() * 1000)}")
    request.state.request_id = request_id

    response = await call_next(request)
    response.headers["X-Request-ID"] = request_id
    return response

app.middleware("http")(rate_limit_middleware)
app.middleware("http")(tenant_isolation_middleware)
app.middleware("http")(auth_middleware)

# Global Exception Handler
@app.exception_handler(Exception)
async def global_exception_handler(request: Request, exc: Exception):
    logger.error(f"Unhandled exception: {exc}", exc_info=True)
    return JSONResponse(
        status_code=500,
        content={
            "error": "Internal server error",
            "request_id": getattr(request.state, "request_id", None)
        }
    )

# Register Routes
app.include_router(health_router, prefix="/health", tags=["Health"])
app.include_router(auth_router, prefix="/api/v1/auth", tags=["Authentication"])
app.include_router(chat_router, prefix="/api/v1/chat", tags=["Chat"])
app.include_router(knowledge_router, prefix="/api/v1/knowledge", tags=["Knowledge"])
app.include_router(assistant_router, prefix="/api/v1/assistants", tags=["Assistants"])
app.include_router(tickets_router, prefix="/api/v1/tickets", tags=["Tickets"])
app.include_router(analytics_router, prefix="/api/v1/analytics", tags=["Analytics"])

@app.get("/")
async def root():
    return {
        "name": "Customer Support RAG API",
        "version": "1.0.0",
        "status": "running"
    }

if __name__ == "__main__":
    import uvicorn
    uvicorn.run(
        "app.main:app",
        host=settings.HOST,
        port=settings.PORT,
        reload=settings.ENVIRONMENT == "development",
        log_level="info"
    )
```

**HOW to run:**
```bash
# Development with auto-reload
python app/main.py

# Production with Gunicorn + Uvicorn workers
gunicorn app.main:app -w 4 -k uvicorn.workers.UvicornWorker --bind 0.0.0.0:8000
```

---

### 2. config.py - Pydantic Settings for Configuration

**WHY:**
- Type-safe configuration with validation
- Environment-specific settings (dev, staging, prod)
- Secrets management (never commit .env to git)
- Single source of truth for all settings

**WHAT:**
```python
# app/config.py
from pydantic_settings import BaseSettings, SettingsConfigDict
from typing import List, Optional
from functools import lru_cache

class Settings(BaseSettings):
    """
    Application settings loaded from environment variables.
    Uses Pydantic Settings for validation and type safety.
    """
    model_config = SettingsConfigDict(
        env_file=".env",
        env_file_encoding="utf-8",
        case_sensitive=False,
        extra="ignore"
    )

    # Application
    APP_NAME: str = "Customer Support RAG API"
    ENVIRONMENT: str = "development"  # development, staging, production
    HOST: str = "0.0.0.0"
    PORT: int = 8000
    DEBUG: bool = True
    ALLOWED_ORIGINS: List[str] = ["http://localhost:3000", "http://localhost:8000"]

    # Security
    SECRET_KEY: str  # For JWT signing
    ALGORITHM: str = "HS256"
    ACCESS_TOKEN_EXPIRE_MINUTES: int = 30
    REFRESH_TOKEN_EXPIRE_DAYS: int = 7

    # MongoDB
    MONGO_HOST: str = "localhost"
    MONGO_PORT: int = 27017
    MONGO_USER: Optional[str] = None
    MONGO_PASSWORD: Optional[str] = None
    MONGO_DB_NAME: str = "customer_support_rag"
    MONGO_MAX_POOL_SIZE: int = 50
    MONGO_MIN_POOL_SIZE: int = 10

    @property
    def MONGO_URI(self) -> str:
        """Construct MongoDB connection URI."""
        if self.MONGO_USER and self.MONGO_PASSWORD:
            return (
                f"mongodb://{self.MONGO_USER}:{self.MONGO_PASSWORD}@"
                f"{self.MONGO_HOST}:{self.MONGO_PORT}/{self.MONGO_DB_NAME}"
            )
        return f"mongodb://{self.MONGO_HOST}:{self.MONGO_PORT}/{self.MONGO_DB_NAME}"

    # Redis
    REDIS_HOST: str = "localhost"
    REDIS_PORT: int = 6379
    REDIS_PASSWORD: Optional[str] = None
    REDIS_DB: int = 0
    REDIS_CACHE_TTL: int = 3600  # 1 hour

    @property
    def REDIS_URI(self) -> str:
        """Construct Redis connection URI."""
        if self.REDIS_PASSWORD:
            return f"redis://:{self.REDIS_PASSWORD}@{self.REDIS_HOST}:{self.REDIS_PORT}/{self.REDIS_DB}"
        return f"redis://{self.REDIS_HOST}:{self.REDIS_PORT}/{self.REDIS_DB}"

    # Qdrant Vector Database
    QDRANT_HOST: str = "localhost"
    QDRANT_PORT: int = 6333
    QDRANT_API_KEY: Optional[str] = None
    QDRANT_COLLECTION_PREFIX: str = "org"
    QDRANT_VECTOR_SIZE: int = 384  # MiniLM embedding dimension

    # OpenAI / LLM
    OPENAI_API_KEY: str
    OPENAI_MODEL: str = "gpt-4"
    OPENAI_MAX_TOKENS: int = 1000
    OPENAI_TEMPERATURE: float = 0.7
    OPENAI_TIMEOUT: int = 30  # seconds

    # Embeddings
    EMBEDDING_MODEL: str = "sentence-transformers/all-MiniLM-L6-v2"
    EMBEDDING_BATCH_SIZE: int = 32
    EMBEDDING_MAX_LENGTH: int = 512

    # RAG Configuration
    RAG_CHUNK_SIZE: int = 512
    RAG_CHUNK_OVERLAP: int = 50
    RAG_TOP_K: int = 20
    RAG_RERANK_TOP_K: int = 5
    RAG_MAX_CONTEXT_TOKENS: int = 3000

    # Celery
    CELERY_BROKER_URL: str = "redis://localhost:6379/1"
    CELERY_RESULT_BACKEND: str = "redis://localhost:6379/2"
    CELERY_TASK_ALWAYS_EAGER: bool = False  # Set True for synchronous testing

    # Rate Limiting
    RATE_LIMIT_ENABLED: bool = True
    RATE_LIMIT_REQUESTS_PER_MINUTE: int = 60
    RATE_LIMIT_BURST: int = 10

    # Logging
    LOG_LEVEL: str = "INFO"
    LOG_FORMAT: str = "json"  # json or text

    # Circuit Breaker
    CIRCUIT_BREAKER_FAILURE_THRESHOLD: int = 5
    CIRCUIT_BREAKER_RECOVERY_TIMEOUT: int = 60  # seconds

    # File Upload
    MAX_UPLOAD_SIZE_MB: int = 10
    ALLOWED_FILE_TYPES: List[str] = [".pdf", ".docx", ".txt", ".md", ".html"]

    class Config:
        case_sensitive = True

@lru_cache()
def get_settings() -> Settings:
    """
    Get cached settings instance.
    Uses lru_cache to ensure settings are loaded only once.
    """
    return Settings()

settings = get_settings()
```

**HOW to use:**
```python
# In any module
from app.config import settings

# Access settings
print(settings.MONGO_URI)
print(settings.RAG_CHUNK_SIZE)
```

---

### 3. dependencies.py - Dependency Injection Setup

**WHY:**
- Centralized dependency management
- Easy to mock for testing
- Lifecycle management (connection pooling)
- Loose coupling between components

**WHAT:**
```python
# app/dependencies.py
from typing import Annotated, Optional
from fastapi import Depends, HTTPException, status, Header
from motor.motor_asyncio import AsyncIOMotorClient, AsyncIOMotorDatabase
from redis.asyncio import Redis
from qdrant_client import QdrantClient
import jwt

from app.config import settings
from app.models.user import User
from app.repositories.user_repo import UserRepository

# ============================================================================
# Database Dependencies
# ============================================================================

# MongoDB Client (singleton)
_mongo_client: Optional[AsyncIOMotorClient] = None

async def get_mongo_client() -> AsyncIOMotorClient:
    """Get MongoDB client instance."""
    global _mongo_client
    if _mongo_client is None:
        _mongo_client = AsyncIOMotorClient(
            settings.MONGO_URI,
            maxPoolSize=settings.MONGO_MAX_POOL_SIZE,
            minPoolSize=settings.MONGO_MIN_POOL_SIZE
        )
    return _mongo_client

async def get_database(
    client: Annotated[AsyncIOMotorClient, Depends(get_mongo_client)]
) -> AsyncIOMotorDatabase:
    """Get MongoDB database instance."""
    return client[settings.MONGO_DB_NAME]

# Redis Client (singleton)
_redis_client: Optional[Redis] = None

async def get_redis_client() -> Redis:
    """Get Redis client instance."""
    global _redis_client
    if _redis_client is None:
        _redis_client = await Redis.from_url(
            settings.REDIS_URI,
            encoding="utf-8",
            decode_responses=True
        )
    return _redis_client

# Qdrant Client (singleton)
_qdrant_client: Optional[QdrantClient] = None

def get_qdrant_client() -> QdrantClient:
    """Get Qdrant client instance."""
    global _qdrant_client
    if _qdrant_client is None:
        _qdrant_client = QdrantClient(
            host=settings.QDRANT_HOST,
            port=settings.QDRANT_PORT,
            api_key=settings.QDRANT_API_KEY
        )
    return _qdrant_client

# ============================================================================
# Authentication Dependencies
# ============================================================================

async def get_current_user(
    authorization: Annotated[Optional[str], Header()] = None,
    db: Annotated[AsyncIOMotorDatabase, Depends(get_database)] = None
) -> User:
    """
    Extract and validate JWT token, return current user.
    Used as a dependency for protected routes.
    """
    if not authorization:
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="Authorization header missing"
        )

    try:
        # Extract token from "Bearer <token>"
        scheme, token = authorization.split()
        if scheme.lower() != "bearer":
            raise HTTPException(
                status_code=status.HTTP_401_UNAUTHORIZED,
                detail="Invalid authentication scheme"
            )

        # Decode JWT
        payload = jwt.decode(
            token,
            settings.SECRET_KEY,
            algorithms=[settings.ALGORITHM]
        )
        user_id: str = payload.get("sub")
        if user_id is None:
            raise HTTPException(
                status_code=status.HTTP_401_UNAUTHORIZED,
                detail="Invalid token payload"
            )

        # Fetch user from database
        user_repo = UserRepository(db)
        user = await user_repo.find_by_id(user_id)
        if user is None:
            raise HTTPException(
                status_code=status.HTTP_401_UNAUTHORIZED,
                detail="User not found"
            )

        return user

    except jwt.ExpiredSignatureError:
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="Token expired"
        )
    except jwt.InvalidTokenError:
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="Invalid token"
        )
    except ValueError:
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="Invalid authorization header format"
        )

async def get_current_organization_id(
    current_user: Annotated[User, Depends(get_current_user)]
) -> str:
    """
    Extract organization_id from current user.
    Used for tenant isolation in queries.
    """
    if not current_user.organization_id:
        raise HTTPException(
            status_code=status.HTTP_403_FORBIDDEN,
            detail="User not associated with any organization"
        )
    return current_user.organization_id

# ============================================================================
# Repository Dependencies
# ============================================================================

async def get_user_repository(
    db: Annotated[AsyncIOMotorDatabase, Depends(get_database)]
) -> UserRepository:
    """Get user repository instance."""
    return UserRepository(db)

# Add similar factory functions for other repositories...

# ============================================================================
# Service Dependencies
# ============================================================================

from app.services.chat_service import ChatService
from app.services.knowledge_service import KnowledgeService
from app.utils.embeddings import EmbeddingService

async def get_embedding_service() -> EmbeddingService:
    """Get embedding service instance (singleton)."""
    # This would be initialized in lifespan and stored globally
    from app.main import embedding_service
    return embedding_service

async def get_chat_service(
    db: Annotated[AsyncIOMotorDatabase, Depends(get_database)],
    redis: Annotated[Redis, Depends(get_redis_client)],
    qdrant: Annotated[QdrantClient, Depends(get_qdrant_client)],
    embedding_service: Annotated[EmbeddingService, Depends(get_embedding_service)]
) -> ChatService:
    """Get chat service instance with all dependencies."""
    return ChatService(
        db=db,
        redis=redis,
        qdrant=qdrant,
        embedding_service=embedding_service
    )

async def get_knowledge_service(
    db: Annotated[AsyncIOMotorDatabase, Depends(get_database)],
    qdrant: Annotated[QdrantClient, Depends(get_qdrant_client)]
) -> KnowledgeService:
    """Get knowledge service instance with all dependencies."""
    return KnowledgeService(db=db, qdrant=qdrant)
```

**HOW to use in routes:**
```python
# app/routes/chat.py
from fastapi import APIRouter, Depends
from app.dependencies import get_current_user, get_chat_service
from app.models.user import User
from app.services.chat_service import ChatService

router = APIRouter()

@router.post("/")
async def create_chat(
    message: str,
    current_user: Annotated[User, Depends(get_current_user)],
    chat_service: Annotated[ChatService, Depends(get_chat_service)]
):
    """Create a new chat message and get AI response."""
    response = await chat_service.handle_chat(
        user_id=current_user.id,
        organization_id=current_user.organization_id,
        message=message
    )
    return response
```

---

## Environment Configuration

### .env.example Template

```bash
# .env.example - Template for environment variables
# Copy to .env and fill in actual values

# Application
APP_NAME="Customer Support RAG API"
ENVIRONMENT="development"  # development, staging, production
HOST="0.0.0.0"
PORT=8000
DEBUG=true
ALLOWED_ORIGINS=["http://localhost:3000"]

# Security
SECRET_KEY="your-secret-key-here-min-32-chars"
ALGORITHM="HS256"
ACCESS_TOKEN_EXPIRE_MINUTES=30

# MongoDB
MONGO_HOST="localhost"
MONGO_PORT=27017
MONGO_USER="admin"
MONGO_PASSWORD="password"
MONGO_DB_NAME="customer_support_rag"

# Redis
REDIS_HOST="localhost"
REDIS_PORT=6379
REDIS_PASSWORD=""
REDIS_DB=0

# Qdrant
QDRANT_HOST="localhost"
QDRANT_PORT=6333
QDRANT_API_KEY=""

# OpenAI
OPENAI_API_KEY="sk-..."
OPENAI_MODEL="gpt-4"

# Embeddings
EMBEDDING_MODEL="sentence-transformers/all-MiniLM-L6-v2"

# RAG
RAG_CHUNK_SIZE=512
RAG_CHUNK_OVERLAP=50

# Celery
CELERY_BROKER_URL="redis://localhost:6379/1"
CELERY_RESULT_BACKEND="redis://localhost:6379/2"
```

---

## Docker Compose Setup

### docker-compose.yml for Local Development

**WHY:**
- One command to start all services
- Consistent environment across team members
- Isolated networking and volumes

**WHAT:**
```yaml
# docker-compose.yml
version: '3.8'

services:
  # MongoDB
  mongodb:
    image: mongo:7
    container_name: rag_mongodb
    ports:
      - "27017:27017"
    environment:
      MONGO_INITDB_ROOT_USERNAME: admin
      MONGO_INITDB_ROOT_PASSWORD: password
      MONGO_INITDB_DATABASE: customer_support_rag
    volumes:
      - mongodb_data:/data/db
    networks:
      - rag_network

  # Redis
  redis:
    image: redis:7-alpine
    container_name: rag_redis
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data
    networks:
      - rag_network

  # Qdrant Vector Database
  qdrant:
    image: qdrant/qdrant:latest
    container_name: rag_qdrant
    ports:
      - "6333:6333"
      - "6334:6334"
    volumes:
      - qdrant_data:/qdrant/storage
    networks:
      - rag_network

  # FastAPI Application
  api:
    build:
      context: .
      dockerfile: docker/Dockerfile
    container_name: rag_api
    ports:
      - "8000:8000"
    environment:
      - MONGO_HOST=mongodb
      - REDIS_HOST=redis
      - QDRANT_HOST=qdrant
      - CELERY_BROKER_URL=redis://redis:6379/1
    env_file:
      - .env
    volumes:
      - ./app:/app/app
      - ./logs:/app/logs
    depends_on:
      - mongodb
      - redis
      - qdrant
    networks:
      - rag_network
    command: uvicorn app.main:app --host 0.0.0.0 --port 8000 --reload

  # Celery Worker
  worker:
    build:
      context: .
      dockerfile: docker/Dockerfile.worker
    container_name: rag_worker
    environment:
      - MONGO_HOST=mongodb
      - REDIS_HOST=redis
      - QDRANT_HOST=qdrant
      - CELERY_BROKER_URL=redis://redis:6379/1
    env_file:
      - .env
    volumes:
      - ./app:/app/app
    depends_on:
      - redis
      - mongodb
      - qdrant
    networks:
      - rag_network
    command: celery -A app.workers.celery_app worker --loglevel=info --concurrency=4

  # Celery Beat (Scheduler)
  beat:
    build:
      context: .
      dockerfile: docker/Dockerfile.worker
    container_name: rag_beat
    environment:
      - REDIS_HOST=redis
      - CELERY_BROKER_URL=redis://redis:6379/1
    env_file:
      - .env
    volumes:
      - ./app:/app/app
    depends_on:
      - redis
    networks:
      - rag_network
    command: celery -A app.workers.celery_app beat --loglevel=info

volumes:
  mongodb_data:
  redis_data:
  qdrant_data:

networks:
  rag_network:
    driver: bridge
```

**HOW to use:**
```bash
# Start all services
docker-compose up -d

# View logs
docker-compose logs -f api

# Stop all services
docker-compose down

# Rebuild after code changes
docker-compose up -d --build
```

---

## Makefile for Common Commands

```makefile
# Makefile
.PHONY: install test run docker-up docker-down clean

install:
	pip install -r requirements.txt

install-dev:
	pip install -r requirements-dev.txt

test:
	pytest tests/ -v --cov=app --cov-report=html

test-unit:
	pytest tests/unit/ -v

test-integration:
	pytest tests/integration/ -v

run:
	python app/main.py

run-worker:
	celery -A app.workers.celery_app worker --loglevel=info

run-beat:
	celery -A app.workers.celery_app beat --loglevel=info

docker-up:
	docker-compose up -d

docker-down:
	docker-compose down

docker-logs:
	docker-compose logs -f

docker-rebuild:
	docker-compose up -d --build

clean:
	find . -type d -name __pycache__ -exec rm -rf {} +
	find . -type f -name "*.pyc" -delete
	rm -rf .pytest_cache htmlcov .coverage

lint:
	black app/ tests/
	isort app/ tests/
	flake8 app/ tests/

migrate:
	python scripts/migrate.py

seed:
	python scripts/seed_data.py

init-db:
	python scripts/init_db.py
```

---

## Interview Questions & Answers

### Q1: How do you structure a FastAPI project for scalability?

**A:** I structure FastAPI projects using a layered architecture:

1. **Routes Layer**: Thin handlers that validate input via Pydantic schemas, extract auth context, and delegate to services. No business logic here.

2. **Services Layer**: Core business logic organized by domain (chat, knowledge, auth). Services orchestrate multiple repositories and external APIs.

3. **Repository Layer**: Data access abstraction using the Repository pattern. Each model has a repository with methods like `find_by_id`, `create`, `update`. This makes it easy to switch databases or write tests with mocks.

4. **Models vs Schemas**: Models are database entities (MongoDB documents), Schemas are request/response DTOs. This separation prevents leaking database internals to the API.

5. **Dependency Injection**: All dependencies (database connections, services) are injected via FastAPI's `Depends()`. This makes code testable and enables lifecycle management (connection pooling).

6. **Configuration**: All settings in `config.py` using Pydantic Settings. Type-safe, validated, environment-specific. Never hardcode values.

This structure scales because each layer can be developed, tested, and scaled independently.

### Q2: Explain your dependency injection strategy in FastAPI.

**A:** I use FastAPI's built-in dependency injection system with a centralized `dependencies.py` file:

1. **Singleton Dependencies**: Database clients (MongoDB, Redis, Qdrant) are created once and reused. I use module-level variables with `lru_cache` or global state in the lifespan function.

2. **Per-Request Dependencies**: Services are created per request but reuse singleton database clients. For example, `ChatService` depends on `get_database`, `get_redis_client`, and `get_qdrant_client`.

3. **Dependency Chain**: I compose dependencies. `get_current_user` depends on `get_database`, and `get_current_organization_id` depends on `get_current_user`. FastAPI resolves the entire chain automatically.

4. **Testing**: In tests, I override dependencies with mocks:
```python
app.dependency_overrides[get_database] = lambda: mock_db
```

This makes testing isolated and fast without real databases.

5. **Type Annotations**: Using `Annotated[Type, Depends(func)]` provides better IDE support and clarity.

The benefits: testability, loose coupling, lifecycle management, and clear dependency graphs.

### Q3: How do you manage configuration across different environments?

**A:** I use Pydantic Settings with a `.env` file approach:

1. **Settings Class**: All configuration in `app/config.py` as a Pydantic Settings class. This provides type validation, defaults, and computed properties (like constructing `MONGO_URI` from components).

2. **Environment Files**:
   - `.env.example`: Template committed to git
   - `.env`: Actual secrets, gitignored
   - `.env.production`: Production overrides (loaded via environment variable `ENV_FILE`)

3. **Environment-Specific Values**: The `ENVIRONMENT` variable (dev/staging/prod) controls behavior like enabling debug mode or API docs.

4. **Secrets Management**: In production, I use cloud secret managers (AWS Secrets Manager, Azure Key Vault) and inject them as environment variables via Kubernetes or Docker.

5. **Validation**: Pydantic validates all settings on startup. If a required value is missing (like `OPENAI_API_KEY`), the app fails fast with a clear error.

6. **Caching**: Settings are cached using `@lru_cache` so they're loaded once, not on every import.

This approach prevents hardcoded values, supports multiple environments, and ensures type safety.

### Q4: Walk through your project's startup and shutdown lifecycle.

**A:** I use FastAPI's `lifespan` context manager (replaces deprecated startup/shutdown events):

**Startup sequence:**
1. **Database Connections**: Initialize MongoDB client with connection pool settings. This creates a persistent connection pool that all requests share.

2. **Redis Connection**: Initialize Redis client for caching and Celery queue.

3. **Qdrant Client**: Initialize vector database client.

4. **Embedding Model**: Load the sentence-transformer model into memory. This is expensive, so we do it once at startup.

5. **Health Check**: Ping all services to ensure they're reachable. Fail fast if any are down.

**Shutdown sequence:**
1. **Close Connections**: Gracefully close MongoDB and Redis connections, flushing any pending writes.

2. **Cancel Background Tasks**: If using `asyncio.create_task`, cancel them gracefully.

3. **Cleanup Resources**: Delete temp files, flush logs.

This ensures clean startup/shutdown and prevents connection leaks. In Kubernetes, this integrates with readiness/liveness probes.

### Q5: How do you organize tests for a FastAPI project?

**A:** I use a three-layer test structure:

1. **Unit Tests** (`tests/unit/`): Test individual functions/classes in isolation with mocked dependencies. Fast (milliseconds), high coverage.
   - Example: Test chunking logic in `utils/chunking.py` with string inputs
   - Mock all external dependencies (database, API calls)

2. **Integration Tests** (`tests/integration/`): Test interactions between components with real dependencies (test database, Redis).
   - Example: Test full ingestion flow from file upload to Qdrant storage
   - Use `pytest-asyncio` for async tests
   - Use fixtures to set up test data

3. **E2E Tests** (`tests/e2e/`): Test complete workflows from API request to response.
   - Example: POST to `/knowledge`, verify status, then query via `/chat`
   - Use FastAPI's `TestClient` for HTTP requests
   - Run against docker-compose test environment

**Key testing patterns:**
- `conftest.py`: Shared fixtures (mock database, test client, test users)
- `pytest.ini`: Configure test discovery, async support, coverage
- Dependency overrides: Replace real services with mocks
- Database cleanup: Ensure each test starts with a clean state

This structure gives confidence at all levels while keeping tests fast and maintainable.

---

## Summary

This project structure provides:

- **Clarity**: Each directory has a single responsibility
- **Scalability**: Layered architecture allows independent component scaling
- **Testability**: Dependency injection and repository pattern enable easy mocking
- **Maintainability**: Consistent patterns and configuration management
- **Production-Ready**: Docker support, health checks, graceful shutdown

The next section will cover the data models and database schema design.
# Section 5: Environment Setup & Docker Compose

## Overview

This section provides a complete, production-ready local development environment using Docker Compose. You'll understand why containerization matters for RAG systems, what components are required, and how to configure everything for seamless development.

---

## Why: The Case for Containerized Development

### Multi-Service Complexity
A Customer Support RAG SaaS requires orchestrating 6+ interconnected services: FastAPI application, MongoDB (document storage), Redis (caching/queue), Qdrant (vector database), Celery worker (async processing), and Celery beat (scheduling). Managing these manually across different environments leads to "works on my machine" problems.

### Dependency Isolation
Vector databases like Qdrant have specific version requirements. Sentence transformers need CUDA libraries for GPU acceleration. Docker ensures every developer and CI/CD pipeline runs identical environments with pinned dependencies.

### Infrastructure Parity
Your production Kubernetes cluster uses containerized services. Docker Compose mirrors this architecture locally, reducing deployment surprises. Health checks, restart policies, and volume mounts in development translate directly to production manifests.

### Rapid Onboarding
New engineers run `docker-compose up` and have a fully functional RAG system in 5 minutes, including pre-configured databases, vector stores, and queue workers.

---

## What: Architecture Components

```
┌─────────────────────────────────────────────────────────────┐
│                     Docker Compose Network                   │
├─────────────────────────────────────────────────────────────┤
│                                                               │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐     │
│  │   FastAPI   │───▶│   MongoDB   │    │    Redis    │     │
│  │  (app:8000) │    │   (27017)   │    │   (6379)    │     │
│  └──────┬──────┘    └─────────────┘    └──────┬──────┘     │
│         │                                       │            │
│         │           ┌─────────────┐            │            │
│         ├──────────▶│   Qdrant    │            │            │
│         │           │   (6333)    │            │            │
│         │           └─────────────┘            │            │
│         │                                       │            │
│  ┌──────▼──────┐                        ┌──────▼──────┐    │
│  │   Celery    │◀───────────────────────│   Celery    │    │
│  │   Worker    │                        │    Beat     │    │
│  └─────────────┘                        └─────────────┘    │
│                                                               │
│  Volumes: mongodb_data, qdrant_data, redis_data             │
└─────────────────────────────────────────────────────────────┘
```

**Services Breakdown**:
1. **FastAPI App**: Main API server handling HTTP requests, authentication, document ingestion orchestration
2. **MongoDB**: Stores user accounts, workspaces, document metadata, conversation history
3. **Redis**: Session cache, Celery task queue/results, rate limiting counters
4. **Qdrant**: Vector database storing document embeddings (MiniLM-L6-v2, 384 dimensions)
5. **Celery Worker**: Background tasks (document parsing, embedding generation, batch processing)
6. **Celery Beat**: Scheduled jobs (workspace analytics, cleanup tasks, health monitoring)

---

## How: Complete Environment Setup

### 1. Docker Compose Configuration

**File: `docker-compose.yml`**

```yaml
version: '3.8'

services:
  # FastAPI Application
  app:
    build:
      context: .
      dockerfile: Dockerfile
      target: development
    container_name: rag_saas_api
    ports:
      - "8000:8000"
    environment:
      - ENV=development
      - RELOAD=true
    env_file:
      - .env
    volumes:
      # Mount source code for hot reload
      - ./app:/app/app
      - ./tests:/app/tests
      # Prevent overwriting node_modules if needed
      - /app/__pycache__
    depends_on:
      mongodb:
        condition: service_healthy
      redis:
        condition: service_healthy
      qdrant:
        condition: service_started
    networks:
      - rag_network
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8000/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s
    command: uvicorn app.main:app --host 0.0.0.0 --port 8000 --reload

  # MongoDB Database
  mongodb:
    image: mongo:7.0
    container_name: rag_saas_mongodb
    ports:
      - "27017:27017"
    environment:
      MONGO_INITDB_ROOT_USERNAME: ${MONGO_USER}
      MONGO_INITDB_ROOT_PASSWORD: ${MONGO_PASSWORD}
      MONGO_INITDB_DATABASE: ${MONGO_DBNAME}
    volumes:
      - mongodb_data:/data/db
      - mongodb_config:/data/configdb
      # Optional: initialization scripts
      - ./scripts/mongo-init.js:/docker-entrypoint-initdb.d/mongo-init.js:ro
    networks:
      - rag_network
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "mongosh", "--eval", "db.adminCommand('ping')"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 20s

  # Redis Cache & Queue
  redis:
    image: redis:7.2-alpine
    container_name: rag_saas_redis
    ports:
      - "6379:6379"
    command: redis-server --requirepass ${REDIS_PASSWORD} --maxmemory 512mb --maxmemory-policy allkeys-lru
    volumes:
      - redis_data:/data
    networks:
      - rag_network
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "redis-cli", "--raw", "incr", "ping"]
      interval: 10s
      timeout: 3s
      retries: 5
      start_period: 10s

  # Qdrant Vector Database
  qdrant:
    image: qdrant/qdrant:v1.7.4
    container_name: rag_saas_qdrant
    ports:
      - "6333:6333"  # HTTP API
      - "6334:6334"  # gRPC API (optional)
    environment:
      - QDRANT__SERVICE__GRPC_PORT=6334
      - QDRANT__SERVICE__HTTP_PORT=6333
      # Disable telemetry for local dev
      - QDRANT__TELEMETRY_DISABLED=true
    volumes:
      - qdrant_data:/qdrant/storage
    networks:
      - rag_network
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:6333/health"]
      interval: 15s
      timeout: 5s
      retries: 3
      start_period: 30s

  # Celery Worker
  celery_worker:
    build:
      context: .
      dockerfile: Dockerfile
      target: development
    container_name: rag_saas_celery_worker
    environment:
      - ENV=development
      - C_FORCE_ROOT=true  # Allow running as root (dev only)
    env_file:
      - .env
    volumes:
      - ./app:/app/app
      # Shared volume for temporary file processing
      - celery_temp:/tmp/celery
    depends_on:
      redis:
        condition: service_healthy
      mongodb:
        condition: service_healthy
      qdrant:
        condition: service_started
    networks:
      - rag_network
    restart: unless-stopped
    command: celery -A app.tasks.celery_app worker --loglevel=info --concurrency=4 --max-tasks-per-child=50

  # Celery Beat Scheduler
  celery_beat:
    build:
      context: .
      dockerfile: Dockerfile
      target: development
    container_name: rag_saas_celery_beat
    environment:
      - ENV=development
      - C_FORCE_ROOT=true
    env_file:
      - .env
    volumes:
      - ./app:/app/app
      # Persist schedule database
      - celery_beat_schedule:/app/celerybeat-schedule
    depends_on:
      redis:
        condition: service_healthy
    networks:
      - rag_network
    restart: unless-stopped
    command: celery -A app.tasks.celery_app beat --loglevel=info --scheduler=redis_scheduler.schedulers:RedisScheduler

volumes:
  mongodb_data:
    driver: local
  mongodb_config:
    driver: local
  redis_data:
    driver: local
  qdrant_data:
    driver: local
  celery_temp:
    driver: local
  celery_beat_schedule:
    driver: local

networks:
  rag_network:
    driver: bridge
```

### 2. Environment Variables Template

**File: `.env.example`**

```bash
# ============================================
# ENVIRONMENT
# ============================================
ENV=development
DEBUG=true
LOG_LEVEL=INFO
APP_NAME=CustomerSupportRAGSaaS
APP_VERSION=1.0.0

# ============================================
# API SERVER
# ============================================
HOST=0.0.0.0
PORT=8000
RELOAD=true
WORKERS=4

# ============================================
# MONGODB DATABASE
# ============================================
MONGO_HOST=mongodb
MONGO_PORT=27017
MONGO_USER=rag_admin
MONGO_PASSWORD=secure_mongo_password_change_in_production
MONGO_DBNAME=rag_saas_db
# Full connection string (overrides individual params if set)
MONGO_CONN_STRING=mongodb://${MONGO_USER}:${MONGO_PASSWORD}@${MONGO_HOST}:${MONGO_PORT}/${MONGO_DBNAME}?authSource=admin

# ============================================
# REDIS CACHE & QUEUE
# ============================================
REDIS_HOST=redis
REDIS_PORT=6379
REDIS_PASSWORD=secure_redis_password_change_in_production
REDIS_DB=0
REDIS_MAX_CONNECTIONS=50
# Session expiry (seconds)
REDIS_SESSION_TTL=86400
# Cache expiry for embeddings (seconds)
REDIS_CACHE_TTL=3600

# ============================================
# QDRANT VECTOR DATABASE
# ============================================
QDRANT_HOST=qdrant
QDRANT_PORT=6333
QDRANT_GRPC_PORT=6334
# Set to empty for no API key in local dev
QDRANT_API_KEY=
QDRANT_COLLECTION_NAME=support_docs
# Vector dimensions for MiniLM-L6-v2
QDRANT_VECTOR_SIZE=384
# Distance metric: Cosine, Euclid, Dot
QDRANT_DISTANCE_METRIC=Cosine

# ============================================
# OPENAI API (GPT)
# ============================================
OPENAI_API_KEY=sk-proj-xxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
# Model for answer generation
OPENAI_MODEL=gpt-4o-mini
# Model for query rewriting
OPENAI_REWRITE_MODEL=gpt-4o-mini
# Max tokens for generation
OPENAI_MAX_TOKENS=1500
OPENAI_TEMPERATURE=0.3
# Request timeout (seconds)
OPENAI_TIMEOUT=30
# Retry configuration
OPENAI_MAX_RETRIES=3

# ============================================
# EMBEDDING MODEL (Sentence Transformers)
# ============================================
EMBEDDING_MODEL_NAME=sentence-transformers/all-MiniLM-L6-v2
EMBEDDING_DEVICE=cpu
# Set to 'cuda' for GPU acceleration
# EMBEDDING_DEVICE=cuda
EMBEDDING_BATCH_SIZE=32
# Max sequence length for embeddings
EMBEDDING_MAX_LENGTH=512

# ============================================
# CELERY TASK QUEUE
# ============================================
CELERY_BROKER_URL=redis://:${REDIS_PASSWORD}@${REDIS_HOST}:${REDIS_PORT}/1
CELERY_RESULT_BACKEND=redis://:${REDIS_PASSWORD}@${REDIS_HOST}:${REDIS_PORT}/2
CELERY_TASK_SERIALIZER=json
CELERY_RESULT_SERIALIZER=json
CELERY_ACCEPT_CONTENT=json
# Task time limits (seconds)
CELERY_TASK_TIME_LIMIT=600
CELERY_TASK_SOFT_TIME_LIMIT=540
# Worker concurrency
CELERY_WORKER_CONCURRENCY=4
CELERY_WORKER_MAX_TASKS_PER_CHILD=50

# ============================================
# JWT AUTHENTICATION
# ============================================
JWT_SECRET_KEY=super_secret_jwt_key_min_32_chars_change_in_production_use_secrets_generator
JWT_ALGORITHM=HS256
# Access token expiry (minutes)
JWT_ACCESS_TOKEN_EXPIRE_MINUTES=60
# Refresh token expiry (days)
JWT_REFRESH_TOKEN_EXPIRE_DAYS=7

# ============================================
# RAG CONFIGURATION
# ============================================
# Number of chunks to retrieve
RAG_TOP_K=5
# Minimum similarity score (0-1)
RAG_SCORE_THRESHOLD=0.7
# Chunk size for document splitting (characters)
CHUNK_SIZE=800
CHUNK_OVERLAP=200
# Enable hybrid search (keyword + vector)
ENABLE_HYBRID_SEARCH=true
# Reranking model (optional)
RERANKER_MODEL=
# Enable query rewriting
ENABLE_QUERY_REWRITE=true

# ============================================
# FILE PROCESSING
# ============================================
# Max file size for upload (bytes) - 10MB
MAX_FILE_SIZE=10485760
# Allowed file extensions
ALLOWED_EXTENSIONS=pdf,txt,docx,md,html
# Temporary file storage path
TEMP_FILE_PATH=/tmp/celery

# ============================================
# RATE LIMITING
# ============================================
# Requests per minute per user
RATE_LIMIT_PER_MINUTE=60
# Requests per hour per workspace
WORKSPACE_RATE_LIMIT_PER_HOUR=1000

# ============================================
# CORS CONFIGURATION
# ============================================
CORS_ALLOWED_ORIGINS=http://localhost:3000,http://localhost:8000
CORS_ALLOW_CREDENTIALS=true
CORS_ALLOWED_METHODS=GET,POST,PUT,DELETE,PATCH
CORS_ALLOWED_HEADERS=*

# ============================================
# MONITORING & OBSERVABILITY
# ============================================
# Enable Sentry error tracking
ENABLE_SENTRY=false
SENTRY_DSN=
# Enable metrics export
ENABLE_METRICS=false
PROMETHEUS_PORT=9090
```

### 3. Python Dependencies

**File: `requirements.txt`**

```txt
# ============================================
# FASTAPI & WEB SERVER
# ============================================
fastapi==0.109.0
uvicorn[standard]==0.27.0
python-multipart==0.0.6
httpx==0.26.0

# ============================================
# DATABASE & ORM
# ============================================
# MongoDB async driver
motor==3.3.2
# Beanie ODM for MongoDB
beanie==1.24.0
# Pydantic for data validation
pydantic==2.5.3
pydantic-settings==2.1.0

# ============================================
# REDIS & CACHING
# ============================================
redis==5.0.1
hiredis==2.3.2

# ============================================
# CELERY TASK QUEUE
# ============================================
celery==5.3.6
celery-redbeat==2.1.1
# Flower for Celery monitoring (optional)
flower==2.0.1

# ============================================
# VECTOR DATABASE
# ============================================
qdrant-client==1.7.3

# ============================================
# AI & EMBEDDINGS
# ============================================
# OpenAI GPT API
openai==1.10.0
# Sentence transformers for embeddings
sentence-transformers==2.3.1
torch==2.1.2
# For CPU-only (smaller): torch==2.1.2+cpu
transformers==4.37.1

# ============================================
# DOCUMENT PROCESSING
# ============================================
# PDF parsing
pypdf2==3.0.1
pdfplumber==0.10.4
# Word documents
python-docx==1.1.0
# HTML parsing
beautifulsoup4==4.12.3
lxml==5.1.0
# Markdown
markdown==3.5.2
# Text splitting
langchain-text-splitters==0.0.1

# ============================================
# AUTHENTICATION & SECURITY
# ============================================
pyjwt[crypto]==2.8.0
passlib[bcrypt]==1.7.4
python-jose[cryptography]==3.3.0
cryptography==42.0.0

# ============================================
# UTILITIES
# ============================================
# Environment variable loading
python-dotenv==1.0.0
# Logging
loguru==0.7.2
# HTTP requests
requests==2.31.0
# Date/time utilities
python-dateutil==2.8.2
# UUID generation
uuid==1.30
# JSON handling
orjson==3.9.12

# ============================================
# DEVELOPMENT & TESTING
# ============================================
pytest==7.4.4
pytest-asyncio==0.23.3
pytest-mock==3.12.0
pytest-cov==4.1.0
httpx==0.26.0
mongomock==4.1.2
fakeredis==2.21.1

# ============================================
# CODE QUALITY
# ============================================
black==24.1.1
ruff==0.1.14
mypy==1.8.0
pre-commit==3.6.0

# ============================================
# MONITORING (OPTIONAL)
# ============================================
# sentry-sdk==1.40.0
# prometheus-client==0.19.0
```

### 4. Multi-Stage Dockerfile

**File: `Dockerfile`**

```dockerfile
# ============================================
# Stage 1: Base Image with Dependencies
# ============================================
FROM python:3.11-slim as base

# Set environment variables
ENV PYTHONUNBUFFERED=1 \
    PYTHONDONTWRITEBYTECODE=1 \
    PIP_NO_CACHE_DIR=1 \
    PIP_DISABLE_PIP_VERSION_CHECK=1

# Install system dependencies
RUN apt-get update && apt-get install -y --no-install-recommends \
    curl \
    gcc \
    g++ \
    git \
    && rm -rf /var/lib/apt/lists/*

# Set working directory
WORKDIR /app

# ============================================
# Stage 2: Dependencies Installation
# ============================================
FROM base as dependencies

# Copy requirements first (better caching)
COPY requirements.txt .

# Install Python dependencies
RUN pip install --upgrade pip setuptools wheel && \
    pip install -r requirements.txt

# ============================================
# Stage 3: Development Image
# ============================================
FROM dependencies as development

# Copy application code
COPY ./app /app/app

# Create non-root user for development
RUN useradd -m -u 1000 appuser && \
    chown -R appuser:appuser /app

USER appuser

# Expose port
EXPOSE 8000

# Health check
HEALTHCHECK --interval=30s --timeout=10s --start-period=40s --retries=3 \
    CMD curl -f http://localhost:8000/health || exit 1

# Default command (can be overridden in docker-compose)
CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000", "--reload"]

# ============================================
# Stage 4: Production Image (Optimized)
# ============================================
FROM dependencies as production

# Copy only necessary application code
COPY ./app /app/app

# Create non-root user
RUN useradd -m -u 1000 appuser && \
    chown -R appuser:appuser /app

USER appuser

# Expose port
EXPOSE 8000

# Health check
HEALTHCHECK --interval=30s --timeout=10s --start-period=40s --retries=3 \
    CMD curl -f http://localhost:8000/health || exit 1

# Production command with multiple workers
CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000", "--workers", "4"]
```

### 5. Configuration Management with Pydantic

**File: `app/config.py`**

```python
"""
Configuration management using Pydantic Settings.

This module loads configuration from environment variables with:
- Type validation and coercion
- Default values
- Computed fields
- Environment-specific settings
"""

from functools import lru_cache
from typing import Literal, Optional
from pydantic import Field, field_validator, computed_field
from pydantic_settings import BaseSettings, SettingsConfigDict


class Settings(BaseSettings):
    """
    Application settings with environment variable loading.

    Loads from .env file in development, environment variables in production.
    All settings have type validation and sensible defaults.
    """

    model_config = SettingsConfigDict(
        env_file=".env",
        env_file_encoding="utf-8",
        case_sensitive=False,
        extra="ignore"
    )

    # ========================================
    # ENVIRONMENT
    # ========================================
    env: Literal["development", "staging", "production"] = Field(
        default="development",
        description="Application environment"
    )
    debug: bool = Field(default=False, description="Debug mode")
    log_level: Literal["DEBUG", "INFO", "WARNING", "ERROR"] = Field(
        default="INFO",
        description="Logging level"
    )
    app_name: str = Field(default="CustomerSupportRAGSaaS", description="Application name")
    app_version: str = Field(default="1.0.0", description="Application version")

    # ========================================
    # API SERVER
    # ========================================
    host: str = Field(default="0.0.0.0", description="API host")
    port: int = Field(default=8000, description="API port")
    reload: bool = Field(default=False, description="Auto-reload on code changes")
    workers: int = Field(default=4, description="Number of uvicorn workers")

    # ========================================
    # MONGODB
    # ========================================
    mongo_host: str = Field(default="localhost", description="MongoDB host")
    mongo_port: int = Field(default=27017, description="MongoDB port")
    mongo_user: str = Field(default="admin", description="MongoDB username")
    mongo_password: str = Field(default="password", description="MongoDB password")
    mongo_dbname: str = Field(default="rag_saas_db", description="MongoDB database name")
    mongo_conn_string: Optional[str] = Field(default=None, description="Full MongoDB URI")

    @computed_field
    @property
    def mongodb_url(self) -> str:
        """Construct MongoDB connection URL."""
        if self.mongo_conn_string:
            return self.mongo_conn_string
        return (
            f"mongodb://{self.mongo_user}:{self.mongo_password}"
            f"@{self.mongo_host}:{self.mongo_port}/{self.mongo_dbname}"
            f"?authSource=admin"
        )

    # ========================================
    # REDIS
    # ========================================
    redis_host: str = Field(default="localhost", description="Redis host")
    redis_port: int = Field(default=6379, description="Redis port")
    redis_password: str = Field(default="", description="Redis password")
    redis_db: int = Field(default=0, description="Redis database number")
    redis_max_connections: int = Field(default=50, description="Redis connection pool size")
    redis_session_ttl: int = Field(default=86400, description="Session TTL (seconds)")
    redis_cache_ttl: int = Field(default=3600, description="Cache TTL (seconds)")

    @computed_field
    @property
    def redis_url(self) -> str:
        """Construct Redis connection URL."""
        if self.redis_password:
            return f"redis://:{self.redis_password}@{self.redis_host}:{self.redis_port}/{self.redis_db}"
        return f"redis://{self.redis_host}:{self.redis_port}/{self.redis_db}"

    # ========================================
    # QDRANT
    # ========================================
    qdrant_host: str = Field(default="localhost", description="Qdrant host")
    qdrant_port: int = Field(default=6333, description="Qdrant HTTP port")
    qdrant_grpc_port: int = Field(default=6334, description="Qdrant gRPC port")
    qdrant_api_key: Optional[str] = Field(default=None, description="Qdrant API key")
    qdrant_collection_name: str = Field(default="support_docs", description="Qdrant collection")
    qdrant_vector_size: int = Field(default=384, description="Vector dimensions")
    qdrant_distance_metric: Literal["Cosine", "Euclid", "Dot"] = Field(
        default="Cosine",
        description="Distance metric"
    )

    @computed_field
    @property
    def qdrant_url(self) -> str:
        """Construct Qdrant connection URL."""
        return f"http://{self.qdrant_host}:{self.qdrant_port}"

    # ========================================
    # OPENAI
    # ========================================
    openai_api_key: str = Field(..., description="OpenAI API key (required)")
    openai_model: str = Field(default="gpt-4o-mini", description="GPT model for answers")
    openai_rewrite_model: str = Field(default="gpt-4o-mini", description="GPT model for queries")
    openai_max_tokens: int = Field(default=1500, description="Max tokens per response")
    openai_temperature: float = Field(default=0.3, ge=0, le=2, description="Generation temperature")
    openai_timeout: int = Field(default=30, description="Request timeout (seconds)")
    openai_max_retries: int = Field(default=3, description="Max retry attempts")

    @field_validator("openai_temperature")
    @classmethod
    def validate_temperature(cls, v: float) -> float:
        """Ensure temperature is in valid range."""
        if not 0 <= v <= 2:
            raise ValueError("Temperature must be between 0 and 2")
        return v

    # ========================================
    # EMBEDDING MODEL
    # ========================================
    embedding_model_name: str = Field(
        default="sentence-transformers/all-MiniLM-L6-v2",
        description="Sentence transformer model"
    )
    embedding_device: Literal["cpu", "cuda"] = Field(default="cpu", description="Device for embeddings")
    embedding_batch_size: int = Field(default=32, description="Batch size for embeddings")
    embedding_max_length: int = Field(default=512, description="Max sequence length")

    # ========================================
    # CELERY
    # ========================================
    celery_broker_url: Optional[str] = Field(default=None, description="Celery broker URL")
    celery_result_backend: Optional[str] = Field(default=None, description="Celery result backend")
    celery_task_serializer: str = Field(default="json", description="Task serializer")
    celery_result_serializer: str = Field(default="json", description="Result serializer")
    celery_accept_content: list[str] = Field(default=["json"], description="Accepted content types")
    celery_task_time_limit: int = Field(default=600, description="Task hard timeout (seconds)")
    celery_task_soft_time_limit: int = Field(default=540, description="Task soft timeout (seconds)")
    celery_worker_concurrency: int = Field(default=4, description="Worker concurrency")
    celery_worker_max_tasks_per_child: int = Field(default=50, description="Tasks before worker restart")

    @computed_field
    @property
    def celery_broker_computed(self) -> str:
        """Compute Celery broker URL if not provided."""
        if self.celery_broker_url:
            return self.celery_broker_url
        # Use Redis DB 1 for broker
        return f"redis://:{self.redis_password}@{self.redis_host}:{self.redis_port}/1"

    @computed_field
    @property
    def celery_backend_computed(self) -> str:
        """Compute Celery result backend if not provided."""
        if self.celery_result_backend:
            return self.celery_result_backend
        # Use Redis DB 2 for results
        return f"redis://:{self.redis_password}@{self.redis_host}:{self.redis_port}/2"

    # ========================================
    # JWT AUTHENTICATION
    # ========================================
    jwt_secret_key: str = Field(..., description="JWT secret key (required)")
    jwt_algorithm: str = Field(default="HS256", description="JWT algorithm")
    jwt_access_token_expire_minutes: int = Field(default=60, description="Access token expiry")
    jwt_refresh_token_expire_days: int = Field(default=7, description="Refresh token expiry")

    @field_validator("jwt_secret_key")
    @classmethod
    def validate_jwt_secret(cls, v: str) -> str:
        """Ensure JWT secret is sufficiently long."""
        if len(v) < 32:
            raise ValueError("JWT secret must be at least 32 characters")
        return v

    # ========================================
    # RAG CONFIGURATION
    # ========================================
    rag_top_k: int = Field(default=5, ge=1, le=20, description="Number of chunks to retrieve")
    rag_score_threshold: float = Field(default=0.7, ge=0, le=1, description="Minimum similarity")
    chunk_size: int = Field(default=800, description="Chunk size (characters)")
    chunk_overlap: int = Field(default=200, description="Chunk overlap (characters)")
    enable_hybrid_search: bool = Field(default=True, description="Enable hybrid search")
    reranker_model: Optional[str] = Field(default=None, description="Reranking model")
    enable_query_rewrite: bool = Field(default=True, description="Enable query rewriting")

    # ========================================
    # FILE PROCESSING
    # ========================================
    max_file_size: int = Field(default=10485760, description="Max file size (bytes)")
    allowed_extensions: list[str] = Field(
        default=["pdf", "txt", "docx", "md", "html"],
        description="Allowed file types"
    )
    temp_file_path: str = Field(default="/tmp/celery", description="Temp file storage")

    # ========================================
    # RATE LIMITING
    # ========================================
    rate_limit_per_minute: int = Field(default=60, description="Requests per minute per user")
    workspace_rate_limit_per_hour: int = Field(default=1000, description="Requests per hour per workspace")

    # ========================================
    # CORS
    # ========================================
    cors_allowed_origins: list[str] = Field(
        default=["http://localhost:3000"],
        description="Allowed CORS origins"
    )
    cors_allow_credentials: bool = Field(default=True, description="Allow credentials")
    cors_allowed_methods: list[str] = Field(
        default=["GET", "POST", "PUT", "DELETE", "PATCH"],
        description="Allowed HTTP methods"
    )
    cors_allowed_headers: list[str] = Field(default=["*"], description="Allowed headers")

    # ========================================
    # MONITORING
    # ========================================
    enable_sentry: bool = Field(default=False, description="Enable Sentry")
    sentry_dsn: Optional[str] = Field(default=None, description="Sentry DSN")
    enable_metrics: bool = Field(default=False, description="Enable Prometheus metrics")
    prometheus_port: int = Field(default=9090, description="Prometheus port")

    # ========================================
    # COMPUTED PROPERTIES
    # ========================================
    @computed_field
    @property
    def is_production(self) -> bool:
        """Check if running in production."""
        return self.env == "production"

    @computed_field
    @property
    def is_development(self) -> bool:
        """Check if running in development."""
        return self.env == "development"


@lru_cache()
def get_settings() -> Settings:
    """
    Get cached settings instance.

    Uses lru_cache to ensure settings are loaded once and reused.
    Perfect for FastAPI dependency injection.

    Returns:
        Settings: Validated settings instance
    """
    return Settings()


# Convenience export
settings = get_settings()
```

### 6. Local Development Setup Guide

#### Step 1: Clone & Environment Setup

```bash
# Clone repository
git clone https://github.com/your-org/customer-support-rag-saas.git
cd customer-support-rag-saas

# Create .env from template
cp .env.example .env

# Edit .env with your values (CRITICAL: Set OPENAI_API_KEY and JWT_SECRET_KEY)
nano .env
```

#### Step 2: Start All Services

```bash
# Build and start all containers
docker-compose up --build

# Or run in detached mode
docker-compose up -d --build

# View logs
docker-compose logs -f

# View specific service logs
docker-compose logs -f app
docker-compose logs -f celery_worker
```

#### Step 3: Verify Services

```bash
# Check all services are healthy
docker-compose ps

# Test API health
curl http://localhost:8000/health

# Test Qdrant
curl http://localhost:6333/health

# Test MongoDB
docker-compose exec mongodb mongosh --eval "db.adminCommand('ping')"

# Test Redis
docker-compose exec redis redis-cli -a your_password ping
```

#### Step 4: Initialize Database & Collections

```bash
# Run database initialization script
docker-compose exec app python -m app.scripts.init_db

# Create Qdrant collection
docker-compose exec app python -m app.scripts.init_qdrant

# Create admin user (optional)
docker-compose exec app python -m app.scripts.create_admin_user
```

#### Step 5: Run Tests

```bash
# Run full test suite
docker-compose exec app pytest

# Run with coverage
docker-compose exec app pytest --cov=app --cov-report=html

# Run specific test file
docker-compose exec app pytest tests/test_embedding_service.py -v
```

#### Step 6: Development Workflow

```bash
# Hot reload is enabled - just edit code and save
# Logs will show reload events

# Access Celery Flower monitoring (optional)
docker-compose exec celery_worker celery -A app.tasks.celery_app flower --port=5555
# Visit http://localhost:5555

# Access MongoDB with GUI (optional - install MongoDB Compass)
# Connection string: mongodb://rag_admin:your_password@localhost:27017/rag_saas_db

# Rebuild specific service
docker-compose up -d --build app

# Stop all services
docker-compose down

# Stop and remove volumes (CAUTION: deletes data)
docker-compose down -v
```

#### Common Development Commands

```bash
# Execute command in app container
docker-compose exec app python -m app.scripts.seed_data

# Run Python shell with app context
docker-compose exec app python

# Restart Celery worker (after task changes)
docker-compose restart celery_worker

# View resource usage
docker stats

# Clean up unused Docker resources
docker system prune -a --volumes
```

---

## Interview Q&A

### Q1: Why use Docker Compose for local development instead of installing services natively?

**Answer**: Docker Compose solves the "dependency hell" problem inherent in multi-service RAG architectures. Here's why it's superior for local development:

**Service Orchestration**: A RAG SaaS requires 6+ interconnected services (FastAPI, MongoDB, Redis, Qdrant, Celery worker, Celery beat). Docker Compose defines all dependencies, startup order (via `depends_on` with health checks), and networking in one declarative file. Without it, developers manually start services, handle port conflicts, and debug connectivity issues.

**Environment Consistency**: Qdrant 1.7.4 works differently than 1.6.x. MongoDB 7.0 has different authentication than 5.x. Docker pins exact versions (`image: mongo:7.0`, `image: qdrant/qdrant:v1.7.4`), ensuring your local environment matches staging and production. This eliminates "works on my machine" bugs.

**Onboarding Speed**: New engineers run `docker-compose up` and have a working RAG system in 5 minutes. Without Docker, they'd spend hours installing MongoDB, configuring Redis, compiling sentence transformers with correct CUDA libraries, and debugging path issues.

**Isolation**: Docker networks (`rag_network`) isolate services. You can run multiple projects simultaneously without port conflicts (map different host ports). Native installations pollute your system and cause conflicts.

**Production Parity**: Your production Kubernetes cluster uses containers. Docker Compose mirrors this architecture locally. Health checks (`healthcheck`), restart policies (`restart: unless-stopped`), and resource limits translate directly to production manifests.

**Trade-offs**: Docker adds ~500MB overhead per project and slower filesystem I/O on macOS/Windows (solved with named volumes for data). But for multi-service systems, the benefits vastly outweigh costs.

---

### Q2: How do you manage configuration across development, staging, and production environments?

**Answer**: Configuration management is critical for RAG systems due to sensitive credentials (OpenAI API keys, JWT secrets) and environment-specific settings (GPU vs CPU embeddings, connection strings). Here's a robust strategy:

**Pydantic Settings with Environment Variables**: Use `pydantic-settings` BaseSettings class (see `config.py`). It loads from environment variables with type validation, defaults, and computed fields. Benefits:
- **Type Safety**: `port: int` ensures you don't accidentally pass a string. `temperature: float = Field(ge=0, le=2)` validates ranges.
- **Computed Fields**: `@computed_field` auto-generates `mongodb_url` from individual `MONGO_HOST`, `MONGO_PORT`, etc., reducing duplication.
- **Validation**: `@field_validator` ensures JWT secrets are ≥32 chars, preventing weak security in production.
- **Caching**: `@lru_cache()` loads settings once, avoiding repeated file I/O.

**Environment-Specific .env Files**:
- Development: `.env` with local service hostnames (`MONGO_HOST=mongodb`, `REDIS_HOST=redis` - Docker Compose service names)
- Staging/Production: Environment variables injected by Kubernetes secrets/ConfigMaps. **Never commit .env to Git**.
- Template: `.env.example` documents all required variables without sensitive values.

**Secrets Management**:
- Development: `.env` (gitignored)
- Production: Kubernetes secrets or AWS Secrets Manager. Fetch at runtime with `boto3`:
  ```python
  if settings.is_production:
      settings.openai_api_key = get_secret("prod/openai-key")
  ```

**Environment Detection**: `ENV=development|staging|production` toggles behaviors:
- Development: `DEBUG=true`, hot reload, verbose logging
- Production: `DEBUG=false`, multiple workers, Sentry error tracking

**Configuration Validation**: At startup, `Settings()` raises `ValidationError` if required fields are missing (e.g., `OPENAI_API_KEY`). This fails fast instead of cryptic runtime errors.

**Example**: `EMBEDDING_DEVICE=cpu` in development (no GPU), `EMBEDDING_DEVICE=cuda` in production (GPU acceleration). Pydantic validates it's a valid Literal type.

**Anti-patterns to Avoid**:
- Hard-coding credentials in code
- Using `config.json` (merge conflict hell with teams)
- Different config loading logic across environments

---

### Q3: Explain the health check configuration in docker-compose.yml. Why is it important for a RAG system?

**Answer**: Health checks (`healthcheck`) in Docker Compose determine when a service is truly ready, not just running. For RAG systems with complex startup dependencies, they're critical:

**Dependency Ordering**: `depends_on` with `condition: service_healthy` ensures the FastAPI app doesn't start until MongoDB, Redis, and Qdrant are operational. Without health checks, the app crashes trying to connect to MongoDB that's still initializing.

**Example - MongoDB Health Check**:
```yaml
healthcheck:
  test: ["CMD", "mongosh", "--eval", "db.adminCommand('ping')"]
  interval: 10s       # Check every 10 seconds
  timeout: 5s         # Fail if check takes >5s
  retries: 5          # Try 5 times before marking unhealthy
  start_period: 20s   # Grace period (don't fail during first 20s)
```
This executes `mongosh --eval "db.adminCommand('ping')"` inside the container. MongoDB returns a successful response only after authentication is ready and databases are mounted.

**Why It Matters for RAG**:
1. **Vector Database Initialization**: Qdrant takes 10-30s to load collections from disk. If the app queries Qdrant during initialization, you get cryptic 500 errors. Health check (`curl http://localhost:6333/health`) ensures Qdrant's HTTP API is responsive.

2. **Embedding Service Warm-up**: Sentence transformers load a ~400MB model on first use. If a user query hits the app before the model is loaded, the request times out. A custom health check can verify the model is ready:
   ```python
   @app.get("/health")
   async def health_check():
       await embedding_service.get_embeddings(["test"])  # Triggers model load
       return {"status": "healthy"}
   ```

3. **Celery Worker Readiness**: The worker must connect to Redis (queue) and Qdrant (embedding storage) before processing tasks. Without health checks, tasks fail immediately on startup.

**Orchestration Flow**:
1. `docker-compose up` starts all services in parallel
2. MongoDB health check passes after 20s
3. Redis health check passes after 10s
4. Qdrant starts (no health check, uses `service_started`)
5. FastAPI app starts only after MongoDB and Redis are healthy
6. Celery worker starts after Redis and MongoDB are healthy

**Monitoring**: `docker-compose ps` shows health status. Production Kubernetes uses liveness/readiness probes with identical logic.

**Best Practices**:
- Use lightweight checks (avoid heavy queries)
- Set realistic `start_period` (don't fail during initialization)
- Log check failures (`curl -f` returns non-zero on failure, visible in logs)

---

### Q4: How do you handle Celery task development and debugging in Docker Compose?

**Answer**: Celery debugging is notoriously tricky because tasks run in background workers with separate logs. Docker Compose requires specific strategies:

**Log Aggregation**:
```bash
# View worker logs in real-time
docker-compose logs -f celery_worker

# View both app and worker logs
docker-compose logs -f app celery_worker

# Follow all logs with timestamps
docker-compose logs -f --timestamps
```

**Task Visibility with Flower**: Add Celery Flower as a service for web-based task monitoring:
```yaml
celery_flower:
  build: .
  command: celery -A app.tasks.celery_app flower --port=5555
  ports:
    - "5555:5555"
  depends_on:
    - redis
```
Access `http://localhost:5555` to see task queue, active tasks, success/failure rates, and full tracebacks.

**Shared Volumes for File Processing**: Document parsing tasks write temporary files. Share volumes between app and worker:
```yaml
volumes:
  - celery_temp:/tmp/celery
```
Both services access the same filesystem path, preventing "file not found" errors.

**Interactive Debugging**: Celery doesn't support `breakpoint()` because it's a daemon. Instead:
1. **Synchronous Testing**: Call tasks directly without `.delay()`:
   ```python
   # In tests or shell
   result = embed_document_task(doc_id="test123")  # Synchronous
   ```
2. **Celery Shell**: Execute tasks with `docker-compose exec`:
   ```bash
   docker-compose exec celery_worker python
   >>> from app.tasks import embed_document_task
   >>> embed_document_task.delay("doc123")
   <AsyncResult: task-id>
   ```

**Task Retry Configuration**: RAG tasks often fail (OpenAI rate limits, transient Qdrant errors). Configure retries in task definition:
```python
@celery_app.task(bind=True, max_retries=3, default_retry_delay=60)
def embed_document_task(self, doc_id: str):
    try:
        # Generate embeddings
        pass
    except Exception as exc:
        # Exponential backoff
        self.retry(exc=exc, countdown=60 * (2 ** self.request.retries))
```

**Hot Reload for Task Changes**: Celery workers don't auto-reload. After editing task code:
```bash
docker-compose restart celery_worker
```
For frequent changes, reduce `--max-tasks-per-child=5` (worker restarts after 5 tasks, picking up new code).

**Environment Consistency**: Both app and worker must use identical code. Mount source code in both services:
```yaml
app:
  volumes:
    - ./app:/app/app
celery_worker:
  volumes:
    - ./app:/app/app  # Same mount
```

**Debugging Serialization Errors**: Celery serializes arguments to JSON. Complex objects (Pydantic models) fail. Use:
```python
# Convert Pydantic model to dict
task.delay(**document.model_dump())
```

**Task Timeout Handling**: Embedding large PDFs can exceed task time limits. Configure in `.env`:
```bash
CELERY_TASK_TIME_LIMIT=600        # Hard kill after 10 minutes
CELERY_TASK_SOFT_TIME_LIMIT=540   # Warning at 9 minutes
```
Tasks receive `SoftTimeLimitExceeded` exception to cleanup gracefully before hard kill.

**Redis Inspection**: Check task queue status:
```bash
docker-compose exec redis redis-cli -a your_password
> KEYS celery*                  # View Celery keys
> LLEN celery                   # Queue length
> LRANGE celery 0 10            # View queued tasks
```

---

### Q5: What are the security considerations for this docker-compose.yml configuration?

**Answer**: The provided `docker-compose.yml` is optimized for local development. Production deployments require hardening:

**Secrets Management**:
- **Issue**: `.env` file contains plaintext passwords (`MONGO_PASSWORD`, `REDIS_PASSWORD`, `OPENAI_API_KEY`)
- **Fix**: Use Docker secrets or Kubernetes secrets. Never commit `.env`:
  ```bash
  echo ".env" >> .gitignore
  ```
- **Production**: Mount secrets from external vault:
  ```yaml
  secrets:
    mongo_password:
      external: true
  services:
    mongodb:
      secrets:
        - mongo_password
  ```

**Non-Root Users**: The Dockerfile creates `appuser` and runs as UID 1000. This prevents privilege escalation if the container is compromised. **Never run containers as root in production**.

**Network Isolation**:
- **Current**: All services on `rag_network` bridge network, can communicate freely
- **Improved**: Separate networks for different security zones:
  ```yaml
  networks:
    frontend:  # Only app service
    backend:   # App, MongoDB, Redis, Qdrant
  app:
    networks:
      - frontend
      - backend
  mongodb:
    networks:
      - backend  # Not exposed to frontend
  ```

**Port Exposure**:
- **Issue**: `ports: - "27017:27017"` exposes MongoDB to host. Attackers on your network can access the database.
- **Fix**: Only expose services that need external access (app:8000). Remove port mappings for MongoDB, Redis, Qdrant (they communicate via internal Docker network).

**Resource Limits**: Prevent DOS attacks by limiting resources:
```yaml
app:
  deploy:
    resources:
      limits:
        cpus: '2.0'
        memory: 2G
      reservations:
        cpus: '0.5'
        memory: 512M
```

**Image Vulnerabilities**: Use specific tags (`mongo:7.0`, not `mongo:latest`) and scan for CVEs:
```bash
docker scan rag_saas_api:latest
```

**CORS Configuration**: `.env` allows `http://localhost:3000`. Production should restrict to your actual frontend domain:
```bash
CORS_ALLOWED_ORIGINS=https://app.yourdomain.com
```

**Redis Authentication**: Current setup uses `--requirepass`. Production should use ACLs (Access Control Lists) with role-based permissions:
```bash
# Redis 6+ ACLs
redis-cli ACL SETUSER celery_user on >celery_password ~celery* +@all
```

**TLS/SSL**: Internal service communication is unencrypted. Production should use TLS:
- MongoDB: Enable TLS with certificates
- Redis: `redis-server --tls-port 6379 --tls-cert-file cert.pem`
- Qdrant: Configure HTTPS

**Environment Variable Injection**: Using `env_file: .env` exposes all variables to containers. Follow least privilege - only inject necessary variables per service.

**Health Check Security**: The MongoDB health check uses `mongosh --eval "db.adminCommand('ping')"`. Ensure the container's user has minimal permissions (not `admin` role).

**Logging Sensitive Data**: Ensure Celery and FastAPI logs don't leak API keys or user data. Configure loguru with redaction:
```python
logger.add("app.log", filter=lambda r: redact_secrets(r))
```

**Development vs Production**: This docker-compose.yml is for **local development only**. Production should use:
- Kubernetes with secrets management
- Managed services (MongoDB Atlas, Redis Cloud)
- API gateways with authentication
- Network policies isolating pods

# Section 6: Knowledge Ingestion Pipeline

## WHY: RAG Quality Depends on Ingestion Quality

The retrieval quality of your RAG system is fundamentally limited by the quality of your ingestion pipeline. Even the most sophisticated retrieval algorithms cannot compensate for poorly chunked, inadequately embedded, or incorrectly stored documents.

**Critical ingestion decisions that impact retrieval quality:**
- **Chunking strategy** determines context preservation vs. granularity
- **Embedding quality** determines semantic search accuracy
- **Metadata design** enables filtering and attribution
- **Deduplication** prevents redundant results
- **Error handling** ensures completeness of knowledge base

Poor ingestion leads to: irrelevant search results, lost context, duplicate answers, missing information, and degraded user experience. This section builds a production-grade ingestion pipeline that handles multiple content sources, optimizes for retrieval quality, and scales with your knowledge base.

---

## WHAT: Three Ingestion Strategies

### 1. URL Crawling (BeautifulSoup)
**Use case:** External documentation, help centers, blog posts
- Extract content from web pages
- Clean HTML tags and navigation elements
- Handle pagination and rate limiting
- Respect robots.txt

### 2. File Upload (PDF/DOCX)
**Use case:** Internal documentation, manuals, guides
- Parse PDF with PyPDF2 (text extraction)
- Parse DOCX with python-docx (structured content)
- Handle multi-page documents
- Preserve formatting hints (headers, sections)

### 3. Manual Text/Articles
**Use case:** FAQs, quick knowledge entries, KB articles
- Direct text input via API
- Markdown support
- Immediate availability
- Version control

---

## Chunking Strategies Comparison

### Fixed-Size Chunking
```
[0────512][512────1024][1024────1536]
```
- **Pros:** Simple, predictable token counts
- **Cons:** Splits mid-sentence, loses context, poor for structured content

### Sentence-Based Chunking
```
[Sentence 1. Sentence 2.][Sentence 3. Sentence 4.]
```
- **Pros:** Natural boundaries, preserves sentence context
- **Cons:** Variable sizes, may split related paragraphs

### Recursive Chunking
```
Split by paragraph → if too large, split by sentence → if too large, split by token
```
- **Pros:** Adaptive, handles edge cases
- **Cons:** Complex logic, unpredictable chunk sizes

### Section-Aware Chunking (Our Choice)
```
[# Header 1
  ## Subheader 1.1
    Content...][# Header 2
                 Content...]
```
- **Pros:** Preserves document hierarchy, maintains topic coherence, better retrieval
- **Cons:** Requires structured input, may produce variable sizes

**WHY Section-Aware:**
- Documents have inherent structure (headers, sections, subsections)
- Users search by topics/concepts that align with document structure
- Retrieval precision improves when chunks represent complete concepts
- Context windows benefit from hierarchical information (header path)

---

## Embedding with MiniLM

### WHY MiniLM?

**Model:** `sentence-transformers/all-MiniLM-L6-v2`

| Aspect | MiniLM | OpenAI text-embedding-3-small |
|--------|---------|-------------------------------|
| **Speed** | ~1000 docs/sec (local GPU) | ~100 docs/sec (API) |
| **Cost** | Free (self-hosted) | $0.02 per 1M tokens |
| **Dimensions** | 384 | 1536 |
| **Latency** | <10ms (batch) | 50-200ms (API) |
| **Offline** | Yes | No |
| **Quality** | Good for support docs | Excellent for nuanced queries |

**Decision:** Use MiniLM for ingestion (cost + speed), optionally use OpenAI for query-time reranking.

### Batch Embedding Strategy
```python
# Bad: One-by-one (slow)
for chunk in chunks:
    embedding = model.encode(chunk)  # 1000 API calls

# Good: Batched (fast)
embeddings = model.encode(chunks, batch_size=32)  # 32 API calls
```

---

## Qdrant Storage Strategy

### Collection-Per-Tenant Architecture
```
qdrant/
├── tenant_acme_corp/
│   ├── point_1 (vector + payload)
│   ├── point_2 (vector + payload)
│   └── ...
└── tenant_globex_inc/
    ├── point_1 (vector + payload)
    └── ...
```

**WHY:**
- **Isolation:** Tenant data never mixes
- **Performance:** Smaller collections = faster search
- **Scalability:** Independent collection optimization
- **Security:** Delete tenant = drop collection

### Point Structure
```json
{
  "id": "doc_123_chunk_5",
  "vector": [0.123, -0.456, ...],  // 384 dimensions
  "payload": {
    "text": "Original chunk text...",
    "document_id": "doc_123",
    "source": "https://example.com/docs/api",
    "source_type": "url",
    "chunk_index": 5,
    "total_chunks": 12,
    "section_path": "API > Authentication > OAuth2",
    "created_at": "2026-02-10T10:00:00Z",
    "metadata": {
      "title": "API Documentation",
      "author": "Engineering Team",
      "language": "en"
    }
  }
}
```

### HNSW Configuration
```python
# Hierarchical Navigable Small World graph configuration
{
    "hnsw_config": {
        "m": 16,               # Connections per layer (trade-off: recall vs. speed)
        "ef_construct": 100,   # Neighbors during construction (higher = better quality)
    },
    "optimizer_config": {
        "indexing_threshold": 20000,  # Rebuild index after N points
    }
}
```

---

## MongoDB: Knowledge Document Metadata Tracking

```python
{
    "_id": ObjectId("..."),
    "tenant_id": "acme_corp",
    "document_id": "doc_123",
    "title": "API Authentication Guide",
    "source_type": "url",  # url | pdf | docx | manual
    "source_url": "https://example.com/docs/api",
    "ingestion_status": "completed",  # pending | processing | completed | failed
    "total_chunks": 12,
    "chunks_embedded": 12,
    "embedding_model": "all-MiniLM-L6-v2",
    "ingestion_started_at": ISODate("2026-02-10T10:00:00Z"),
    "ingestion_completed_at": ISODate("2026-02-10T10:05:23Z"),
    "error_message": null,
    "metadata": {
        "file_size_bytes": 45000,
        "word_count": 5000,
        "language": "en",
        "author": "Engineering Team"
    },
    "created_at": ISODate("2026-02-10T10:00:00Z"),
    "updated_at": ISODate("2026-02-10T10:05:23Z")
}
```

---

## ASCII Diagram: Ingestion Pipeline

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         KNOWLEDGE INGESTION PIPELINE                         │
└─────────────────────────────────────────────────────────────────────────────┘

SOURCE LAYER
┌──────────┐   ┌──────────┐   ┌──────────┐   ┌──────────┐
│   URL    │   │   PDF    │   │   DOCX   │   │  Manual  │
│ Crawler  │   │  Upload  │   │  Upload  │   │   Text   │
└────┬─────┘   └────┬─────┘   └────┬─────┘   └────┬─────┘
     │              │              │              │
     └──────────────┴──────────────┴──────────────┘
                          │
                          ▼
PARSE & CLEAN LAYER
┌─────────────────────────────────────────────────────────┐
│  HTML Cleaner    │  PDF Parser  │  DOCX Parser          │
│  - Strip tags    │  - PyPDF2    │  - python-docx        │
│  - Extract text  │  - OCR (opt) │  - Preserve headers   │
│  - Remove nav    │              │                       │
└──────────────────────────┬──────────────────────────────┘
                           │
                           ▼
CHUNKING LAYER
┌─────────────────────────────────────────────────────────┐
│            Section-Aware Chunker                        │
│  1. Split by headers (H1, H2, H3)                       │
│  2. Sub-chunk large sections (max 500 tokens)           │
│  3. Add overlap (50 tokens between chunks)              │
│  4. Preserve section path in metadata                   │
└──────────────────────────┬──────────────────────────────┘
                           │
                           ▼
                    ┌─────────────┐
                    │   Chunks    │
                    │  [C1, C2,   │
                    │   C3, ...]  │
                    └──────┬──────┘
                           │
                           ▼
EMBEDDING LAYER
┌─────────────────────────────────────────────────────────┐
│           MiniLM Embedding Service                      │
│  - Batch encode (32 chunks/batch)                       │
│  - GPU acceleration (CUDA if available)                 │
│  - Output: 384-dim vectors                              │
└──────────────────────────┬──────────────────────────────┘
                           │
                           ▼
                 ┌──────────────────┐
                 │   Vector + Text  │
                 │  [(v1, c1),      │
                 │   (v2, c2), ...] │
                 └────────┬─────────┘
                          │
         ┌────────────────┴────────────────┐
         ▼                                 ▼
STORAGE LAYER
┌──────────────────────┐     ┌──────────────────────┐
│      Qdrant          │     │      MongoDB         │
│  - Vector storage    │     │  - Document metadata │
│  - HNSW indexing     │     │  - Ingestion status  │
│  - Point payloads    │     │  - Error tracking    │
└──────────────────────┘     └──────────────────────┘

ASYNC PROCESSING
┌─────────────────────────────────────────────────────────┐
│  Celery Worker Pool                                     │
│  - Task: ingest_document(tenant_id, doc_id, source)    │
│  - Retry on failure (3 attempts)                        │
│  - Progress updates to MongoDB                          │
└─────────────────────────────────────────────────────────┘
```

---

## Code: Section-Aware Chunker

```python
# services/chunking_service.py
import re
from typing import List, Dict, Any
from dataclasses import dataclass


@dataclass
class Chunk:
    """Represents a text chunk with metadata."""
    text: str
    section_path: str
    chunk_index: int
    token_count: int
    metadata: Dict[str, Any]


class SectionAwareChunker:
    """
    Chunks documents by section structure, preserving hierarchy.

    WHY: Section-aware chunking maintains document structure, improving
    retrieval relevance by keeping related content together.
    """

    def __init__(
        self,
        max_tokens: int = 500,
        overlap_tokens: int = 50,
        min_chunk_tokens: int = 50
    ):
        self.max_tokens = max_tokens
        self.overlap_tokens = overlap_tokens
        self.min_chunk_tokens = min_chunk_tokens

        # Header patterns (Markdown and HTML)
        self.header_patterns = [
            (r'^#{1,6}\s+(.+)$', 'markdown'),  # # Header
            (r'^(.+)\n={3,}$', 'setext_h1'),   # Header\n===
            (r'^(.+)\n-{3,}$', 'setext_h2'),   # Header\n---
            (r'<h([1-6])>(.*?)</h\1>', 'html') # <h1>Header</h1>
        ]

    def chunk(self, text: str, document_metadata: Dict[str, Any] = None) -> List[Chunk]:
        """
        Chunk text using section-aware strategy.

        Args:
            text: Raw document text
            document_metadata: Additional metadata to include in chunks

        Returns:
            List of Chunk objects with section hierarchy preserved
        """
        # Split into sections by headers
        sections = self._split_by_headers(text)

        # Sub-chunk large sections
        chunks = []
        for section_index, section in enumerate(sections):
            section_chunks = self._sub_chunk_section(
                section['text'],
                section['path'],
                section_index
            )
            chunks.extend(section_chunks)

        # Add metadata and numbering
        for i, chunk in enumerate(chunks):
            chunk.chunk_index = i
            chunk.metadata.update(document_metadata or {})
            chunk.metadata['total_chunks'] = len(chunks)

        return chunks

    def _split_by_headers(self, text: str) -> List[Dict[str, Any]]:
        """
        Split text into sections based on header hierarchy.

        Returns:
            List of sections with text and hierarchical path
        """
        lines = text.split('\n')
        sections = []
        current_section = {'text': '', 'headers': [], 'path': ''}

        header_stack = []  # Track header hierarchy

        for line in lines:
            header_match = self._match_header(line)

            if header_match:
                # Save previous section if it has content
                if current_section['text'].strip():
                    current_section['path'] = ' > '.join(current_section['headers'])
                    sections.append(current_section)

                # Update header stack
                level, title = header_match
                header_stack = header_stack[:level-1] + [title.strip()]

                # Start new section
                current_section = {
                    'text': line + '\n',
                    'headers': header_stack.copy(),
                    'path': ''
                }
            else:
                current_section['text'] += line + '\n'

        # Add final section
        if current_section['text'].strip():
            current_section['path'] = ' > '.join(current_section['headers'])
            sections.append(current_section)

        return sections

    def _match_header(self, line: str) -> tuple | None:
        """
        Check if line is a header and return (level, title).

        Returns:
            (level, title) tuple or None
        """
        # Markdown headers: # H1, ## H2, etc.
        md_match = re.match(r'^(#{1,6})\s+(.+)$', line)
        if md_match:
            level = len(md_match.group(1))
            title = md_match.group(2)
            return (level, title)

        # HTML headers: <h1>Title</h1>
        html_match = re.search(r'<h([1-6])>(.*?)</h\1>', line)
        if html_match:
            level = int(html_match.group(1))
            title = html_match.group(2)
            return (level, title)

        return None

    def _sub_chunk_section(
        self,
        section_text: str,
        section_path: str,
        section_index: int
    ) -> List[Chunk]:
        """
        Sub-chunk a section if it exceeds max_tokens.
        Uses overlapping windows to preserve context.
        """
        # Simple token estimation (words * 1.3)
        words = section_text.split()
        estimated_tokens = len(words) * 1.3

        if estimated_tokens <= self.max_tokens:
            # Section fits in one chunk
            return [Chunk(
                text=section_text,
                section_path=section_path,
                chunk_index=0,
                token_count=int(estimated_tokens),
                metadata={'section_index': section_index}
            )]

        # Need to sub-chunk
        chunks = []
        words_per_chunk = int(self.max_tokens / 1.3)
        overlap_words = int(self.overlap_tokens / 1.3)

        start_idx = 0
        sub_chunk_idx = 0

        while start_idx < len(words):
            end_idx = min(start_idx + words_per_chunk, len(words))
            chunk_words = words[start_idx:end_idx]
            chunk_text = ' '.join(chunk_words)

            chunks.append(Chunk(
                text=chunk_text,
                section_path=section_path,
                chunk_index=sub_chunk_idx,
                token_count=int(len(chunk_words) * 1.3),
                metadata={
                    'section_index': section_index,
                    'sub_chunk': sub_chunk_idx
                }
            ))

            # Move start with overlap
            start_idx += (words_per_chunk - overlap_words)
            sub_chunk_idx += 1

        return chunks

    def estimate_tokens(self, text: str) -> int:
        """Rough token estimation for chunking decisions."""
        # Simple heuristic: 1 word ≈ 1.3 tokens on average
        return int(len(text.split()) * 1.3)


# Example usage
if __name__ == "__main__":
    sample_doc = """
# API Authentication

OAuth2 is the recommended authentication method.

## Getting Started

First, register your application to get API credentials.

### Client Credentials

Navigate to the developer portal and create a new app.

## Access Tokens

Access tokens expire after 1 hour. Refresh tokens are valid for 30 days.
"""

    chunker = SectionAwareChunker(max_tokens=100, overlap_tokens=20)
    chunks = chunker.chunk(sample_doc, {'source': 'docs'})

    for chunk in chunks:
        print(f"\nChunk {chunk.chunk_index}:")
        print(f"Section: {chunk.section_path}")
        print(f"Tokens: {chunk.token_count}")
        print(f"Text: {chunk.text[:100]}...")
```

---

## Code: Embedding Service

```python
# services/embedding_service.py
from typing import List
import numpy as np
from sentence_transformers import SentenceTransformer
import torch
from loguru import logger


class EmbeddingService:
    """
    Generates embeddings using MiniLM model with batch processing.

    WHY MiniLM:
    - Fast: ~1000 docs/sec on GPU
    - Free: Self-hosted, no API costs
    - Good quality: 384 dims, trained on 1B sentence pairs
    - Offline: No external dependencies
    """

    def __init__(
        self,
        model_name: str = "sentence-transformers/all-MiniLM-L6-v2",
        batch_size: int = 32,
        use_gpu: bool = True
    ):
        self.model_name = model_name
        self.batch_size = batch_size
        self.device = self._get_device(use_gpu)

        logger.info(f"Loading embedding model: {model_name} on {self.device}")
        self.model = SentenceTransformer(model_name, device=self.device)
        self.embedding_dim = self.model.get_sentence_embedding_dimension()
        logger.info(f"Model loaded. Embedding dimension: {self.embedding_dim}")

    def _get_device(self, use_gpu: bool) -> str:
        """Determine compute device (GPU if available and requested)."""
        if use_gpu and torch.cuda.is_available():
            return "cuda"
        return "cpu"

    def embed_single(self, text: str) -> np.ndarray:
        """
        Embed a single text string.

        Args:
            text: Input text

        Returns:
            Embedding vector (384 dimensions)
        """
        return self.model.encode(text, convert_to_numpy=True)

    def embed_batch(
        self,
        texts: List[str],
        show_progress: bool = False
    ) -> np.ndarray:
        """
        Embed multiple texts in batches (efficient).

        Args:
            texts: List of input texts
            show_progress: Show progress bar

        Returns:
            Array of embedding vectors (N x 384)
        """
        if not texts:
            return np.array([])

        logger.info(f"Embedding {len(texts)} texts in batches of {self.batch_size}")

        embeddings = self.model.encode(
            texts,
            batch_size=self.batch_size,
            show_progress_bar=show_progress,
            convert_to_numpy=True
        )

        logger.info(f"Generated embeddings: shape {embeddings.shape}")
        return embeddings

    def compute_similarity(
        self,
        embedding1: np.ndarray,
        embedding2: np.ndarray
    ) -> float:
        """
        Compute cosine similarity between two embeddings.

        Returns:
            Similarity score (0 to 1)
        """
        return float(
            np.dot(embedding1, embedding2) /
            (np.linalg.norm(embedding1) * np.linalg.norm(embedding2))
        )

    def get_model_info(self) -> dict:
        """Return model metadata."""
        return {
            "model_name": self.model_name,
            "embedding_dim": self.embedding_dim,
            "device": self.device,
            "max_seq_length": self.model.max_seq_length
        }


# Example usage
if __name__ == "__main__":
    service = EmbeddingService()

    # Single embedding
    text = "How do I reset my password?"
    embedding = service.embed_single(text)
    print(f"Single embedding shape: {embedding.shape}")

    # Batch embedding (efficient)
    texts = [
        "How do I reset my password?",
        "What is your refund policy?",
        "How to contact support?",
        "Is there a mobile app?"
    ]
    embeddings = service.embed_batch(texts)
    print(f"Batch embeddings shape: {embeddings.shape}")

    # Similarity
    sim = service.compute_similarity(embeddings[0], embeddings[1])
    print(f"Similarity between first two: {sim:.3f}")
```

---

## Code: Qdrant Service

```python
# services/qdrant_service.py
from typing import List, Dict, Any, Optional
from qdrant_client import QdrantClient
from qdrant_client.models import (
    Distance, VectorParams, PointStruct, Filter, FieldCondition, MatchValue
)
from loguru import logger
import uuid


class QdrantService:
    """
    Manages Qdrant vector storage with collection-per-tenant isolation.

    WHY Qdrant:
    - Fast: HNSW indexing for sub-10ms search
    - Scalable: Handles billions of vectors
    - Flexible: Rich filtering on payload fields
    - Open source: Self-hosted option
    """

    def __init__(self, host: str = "localhost", port: int = 6333):
        self.client = QdrantClient(host=host, port=port)
        logger.info(f"Connected to Qdrant at {host}:{port}")

    def get_collection_name(self, tenant_id: str) -> str:
        """Generate collection name for tenant."""
        return f"kb_{tenant_id}"

    def create_collection(
        self,
        tenant_id: str,
        vector_size: int = 384,
        distance: Distance = Distance.COSINE
    ):
        """
        Create a new collection for tenant if it doesn't exist.

        Args:
            tenant_id: Tenant identifier
            vector_size: Embedding dimension (384 for MiniLM)
            distance: Similarity metric (COSINE, EUCLID, DOT)
        """
        collection_name = self.get_collection_name(tenant_id)

        # Check if collection exists
        collections = self.client.get_collections().collections
        if any(c.name == collection_name for c in collections):
            logger.info(f"Collection {collection_name} already exists")
            return

        # Create collection with HNSW config
        self.client.create_collection(
            collection_name=collection_name,
            vectors_config=VectorParams(
                size=vector_size,
                distance=distance
            ),
            hnsw_config={
                "m": 16,              # Connections per layer
                "ef_construct": 100,  # Quality during construction
            },
            optimizer_config={
                "indexing_threshold": 20000,
            }
        )

        logger.info(f"Created collection: {collection_name}")

    def upsert_points(
        self,
        tenant_id: str,
        vectors: List[List[float]],
        payloads: List[Dict[str, Any]],
        point_ids: Optional[List[str]] = None
    ):
        """
        Insert or update points in tenant's collection.

        Args:
            tenant_id: Tenant identifier
            vectors: List of embedding vectors
            payloads: List of metadata dicts (must include 'text' field)
            point_ids: Optional custom IDs (generated if not provided)
        """
        collection_name = self.get_collection_name(tenant_id)

        if not point_ids:
            point_ids = [str(uuid.uuid4()) for _ in vectors]

        points = [
            PointStruct(
                id=point_id,
                vector=vector,
                payload=payload
            )
            for point_id, vector, payload in zip(point_ids, vectors, payloads)
        ]

        self.client.upsert(
            collection_name=collection_name,
            points=points
        )

        logger.info(f"Upserted {len(points)} points to {collection_name}")

    def search(
        self,
        tenant_id: str,
        query_vector: List[float],
        limit: int = 10,
        filters: Optional[Dict[str, Any]] = None,
        score_threshold: float = 0.0
    ) -> List[Dict[str, Any]]:
        """
        Search for similar vectors in tenant's collection.

        Args:
            tenant_id: Tenant identifier
            query_vector: Query embedding
            limit: Max results to return
            filters: Payload field filters (e.g., {"source_type": "url"})
            score_threshold: Minimum similarity score

        Returns:
            List of results with text, score, and metadata
        """
        collection_name = self.get_collection_name(tenant_id)

        # Build filter if provided
        query_filter = None
        if filters:
            conditions = [
                FieldCondition(key=key, match=MatchValue(value=value))
                for key, value in filters.items()
            ]
            query_filter = Filter(must=conditions)

        results = self.client.search(
            collection_name=collection_name,
            query_vector=query_vector,
            limit=limit,
            query_filter=query_filter,
            score_threshold=score_threshold
        )

        # Format results
        formatted_results = []
        for result in results:
            formatted_results.append({
                "id": result.id,
                "score": result.score,
                "text": result.payload.get("text", ""),
                "metadata": {
                    k: v for k, v in result.payload.items() if k != "text"
                }
            })

        logger.info(f"Found {len(formatted_results)} results for {tenant_id}")
        return formatted_results

    def delete_document(self, tenant_id: str, document_id: str):
        """Delete all chunks from a document."""
        collection_name = self.get_collection_name(tenant_id)

        self.client.delete(
            collection_name=collection_name,
            points_selector=Filter(
                must=[
                    FieldCondition(
                        key="document_id",
                        match=MatchValue(value=document_id)
                    )
                ]
            )
        )

        logger.info(f"Deleted document {document_id} from {collection_name}")

    def delete_collection(self, tenant_id: str):
        """Delete entire tenant collection (use with caution)."""
        collection_name = self.get_collection_name(tenant_id)
        self.client.delete_collection(collection_name)
        logger.warning(f"Deleted collection: {collection_name}")

    def get_collection_info(self, tenant_id: str) -> Dict[str, Any]:
        """Get collection statistics."""
        collection_name = self.get_collection_name(tenant_id)
        info = self.client.get_collection(collection_name)

        return {
            "name": collection_name,
            "vector_count": info.points_count,
            "vector_size": info.config.params.vectors.size,
            "distance": info.config.params.vectors.distance.name
        }


# Example usage
if __name__ == "__main__":
    service = QdrantService()
    tenant_id = "acme_corp"

    # Create collection
    service.create_collection(tenant_id)

    # Upsert points
    vectors = [
        [0.1] * 384,  # Mock embedding
        [0.2] * 384
    ]
    payloads = [
        {
            "text": "How to reset password?",
            "document_id": "doc_1",
            "source": "faq.html",
            "chunk_index": 0
        },
        {
            "text": "Contact support at support@example.com",
            "document_id": "doc_1",
            "source": "faq.html",
            "chunk_index": 1
        }
    ]
    service.upsert_points(tenant_id, vectors, payloads)

    # Search
    query_vector = [0.15] * 384
    results = service.search(tenant_id, query_vector, limit=5)
    print(f"Found {len(results)} results")
```

---

## Code: URL Crawler

```python
# services/url_crawler.py
import httpx
from bs4 import BeautifulSoup
from typing import Dict, Any, Optional
from loguru import logger
import re


class URLCrawler:
    """
    Crawls web pages and extracts clean text content.

    Handles:
    - HTML parsing and cleaning
    - Navigation/footer removal
    - Rate limiting
    - Error handling
    """

    def __init__(
        self,
        timeout: int = 30,
        user_agent: str = "CustomerSupportRAG/1.0 (Knowledge Ingestion Bot)"
    ):
        self.timeout = timeout
        self.headers = {"User-Agent": user_agent}

    async def crawl(self, url: str) -> Dict[str, Any]:
        """
        Crawl a URL and extract clean text.

        Args:
            url: URL to crawl

        Returns:
            Dict with text, title, and metadata
        """
        logger.info(f"Crawling: {url}")

        try:
            async with httpx.AsyncClient(timeout=self.timeout) as client:
                response = await client.get(url, headers=self.headers, follow_redirects=True)
                response.raise_for_status()

                html_content = response.text
                final_url = str(response.url)

                # Parse HTML
                soup = BeautifulSoup(html_content, 'html.parser')

                # Extract metadata
                title = self._extract_title(soup)
                description = self._extract_description(soup)

                # Clean and extract text
                clean_text = self._extract_clean_text(soup)

                return {
                    "success": True,
                    "url": final_url,
                    "title": title,
                    "description": description,
                    "text": clean_text,
                    "word_count": len(clean_text.split()),
                    "status_code": response.status_code
                }

        except httpx.HTTPStatusError as e:
            logger.error(f"HTTP error crawling {url}: {e}")
            return {
                "success": False,
                "url": url,
                "error": f"HTTP {e.response.status_code}",
                "text": ""
            }

        except Exception as e:
            logger.error(f"Error crawling {url}: {e}")
            return {
                "success": False,
                "url": url,
                "error": str(e),
                "text": ""
            }

    def _extract_title(self, soup: BeautifulSoup) -> str:
        """Extract page title."""
        # Try <title> tag first
        if soup.title and soup.title.string:
            return soup.title.string.strip()

        # Try <h1> as fallback
        h1 = soup.find('h1')
        if h1:
            return h1.get_text().strip()

        return "Untitled"

    def _extract_description(self, soup: BeautifulSoup) -> Optional[str]:
        """Extract meta description."""
        meta = soup.find('meta', attrs={'name': 'description'})
        if meta and meta.get('content'):
            return meta['content'].strip()
        return None

    def _extract_clean_text(self, soup: BeautifulSoup) -> str:
        """
        Extract clean text from HTML, removing navigation and clutter.

        Strategy:
        1. Remove script, style, nav, footer tags
        2. Extract from main content area if identifiable
        3. Clean whitespace
        """
        # Remove unwanted tags
        for tag in soup(['script', 'style', 'nav', 'footer', 'header', 'aside']):
            tag.decompose()

        # Try to find main content area
        main_content = (
            soup.find('main') or
            soup.find('article') or
            soup.find('div', class_=re.compile(r'content|article|main', re.I)) or
            soup.find('body')
        )

        if not main_content:
            main_content = soup

        # Extract text
        text = main_content.get_text(separator='\n', strip=True)

        # Clean whitespace
        text = re.sub(r'\n{3,}', '\n\n', text)  # Max 2 newlines
        text = re.sub(r'[ \t]+', ' ', text)     # Single spaces

        return text.strip()


# Example usage
if __name__ == "__main__":
    import asyncio

    async def main():
        crawler = URLCrawler()
        result = await crawler.crawl("https://example.com/docs/api")

        print(f"Title: {result['title']}")
        print(f"Word count: {result['word_count']}")
        print(f"Text preview: {result['text'][:200]}...")

    asyncio.run(main())
```

---

## Code: File Parser

```python
# services/file_parser.py
from typing import Dict, Any
import PyPDF2
from docx import Document
from loguru import logger
import os


class FileParser:
    """
    Parses PDF and DOCX files to extract text.

    Supports:
    - PDF: PyPDF2 for text extraction
    - DOCX: python-docx for structured content
    """

    def parse_pdf(self, file_path: str) -> Dict[str, Any]:
        """
        Parse PDF file and extract text.

        Args:
            file_path: Path to PDF file

        Returns:
            Dict with text and metadata
        """
        logger.info(f"Parsing PDF: {file_path}")

        try:
            with open(file_path, 'rb') as file:
                pdf_reader = PyPDF2.PdfReader(file)

                # Extract metadata
                metadata = pdf_reader.metadata or {}
                num_pages = len(pdf_reader.pages)

                # Extract text from all pages
                text_parts = []
                for page_num, page in enumerate(pdf_reader.pages):
                    text = page.extract_text()
                    if text:
                        text_parts.append(f"\n--- Page {page_num + 1} ---\n")
                        text_parts.append(text)

                full_text = ''.join(text_parts)

                return {
                    "success": True,
                    "text": full_text,
                    "num_pages": num_pages,
                    "word_count": len(full_text.split()),
                    "metadata": {
                        "title": metadata.get('/Title', ''),
                        "author": metadata.get('/Author', ''),
                        "subject": metadata.get('/Subject', ''),
                        "creator": metadata.get('/Creator', '')
                    }
                }

        except Exception as e:
            logger.error(f"Error parsing PDF {file_path}: {e}")
            return {
                "success": False,
                "error": str(e),
                "text": ""
            }

    def parse_docx(self, file_path: str) -> Dict[str, Any]:
        """
        Parse DOCX file and extract text with structure.

        Args:
            file_path: Path to DOCX file

        Returns:
            Dict with text and metadata
        """
        logger.info(f"Parsing DOCX: {file_path}")

        try:
            doc = Document(file_path)

            # Extract text preserving structure
            text_parts = []
            for para in doc.paragraphs:
                # Detect headers (based on style)
                if para.style.name.startswith('Heading'):
                    level = para.style.name.replace('Heading ', '')
                    try:
                        level = int(level)
                        text_parts.append(f"\n{'#' * level} {para.text}\n")
                    except ValueError:
                        text_parts.append(f"\n{para.text}\n")
                else:
                    text_parts.append(para.text)

            full_text = '\n'.join(text_parts)

            # Extract metadata
            core_properties = doc.core_properties

            return {
                "success": True,
                "text": full_text,
                "num_paragraphs": len(doc.paragraphs),
                "word_count": len(full_text.split()),
                "metadata": {
                    "title": core_properties.title or '',
                    "author": core_properties.author or '',
                    "subject": core_properties.subject or '',
                    "created": str(core_properties.created) if core_properties.created else ''
                }
            }

        except Exception as e:
            logger.error(f"Error parsing DOCX {file_path}: {e}")
            return {
                "success": False,
                "error": str(e),
                "text": ""
            }

    def parse_file(self, file_path: str) -> Dict[str, Any]:
        """
        Auto-detect file type and parse accordingly.

        Args:
            file_path: Path to file

        Returns:
            Dict with text and metadata
        """
        _, ext = os.path.splitext(file_path)
        ext = ext.lower()

        if ext == '.pdf':
            return self.parse_pdf(file_path)
        elif ext in ['.docx', '.doc']:
            return self.parse_docx(file_path)
        else:
            return {
                "success": False,
                "error": f"Unsupported file type: {ext}",
                "text": ""
            }


# Example usage
if __name__ == "__main__":
    parser = FileParser()

    # Parse PDF
    pdf_result = parser.parse_file("/path/to/document.pdf")
    print(f"PDF pages: {pdf_result.get('num_pages')}")
    print(f"PDF text preview: {pdf_result['text'][:200]}...")

    # Parse DOCX
    docx_result = parser.parse_file("/path/to/document.docx")
    print(f"DOCX paragraphs: {docx_result.get('num_paragraphs')}")
    print(f"DOCX text preview: {docx_result['text'][:200]}...")
```

---

## Code: Celery Ingestion Task

```python
# tasks/ingestion_tasks.py
from celery import Celery, Task
from typing import Dict, Any
from datetime import datetime
from loguru import logger
from motor.motor_asyncio import AsyncIOMotorClient
import asyncio

from services.url_crawler import URLCrawler
from services.file_parser import FileParser
from services.chunking_service import SectionAwareChunker
from services.embedding_service import EmbeddingService
from services.qdrant_service import QdrantService


# Celery app configuration
celery_app = Celery(
    'ingestion_tasks',
    broker='redis://localhost:6379/0',
    backend='redis://localhost:6379/0'
)

celery_app.conf.update(
    task_serializer='json',
    accept_content=['json'],
    result_serializer='json',
    timezone='UTC',
    enable_utc=True,
    task_track_started=True,
    task_time_limit=600,  # 10 minutes
    task_soft_time_limit=540,  # 9 minutes
)


class IngestionTask(Task):
    """Base task with shared service instances."""

    _crawler = None
    _parser = None
    _chunker = None
    _embedding_service = None
    _qdrant_service = None
    _mongo_client = None

    @property
    def crawler(self):
        if self._crawler is None:
            self._crawler = URLCrawler()
        return self._crawler

    @property
    def parser(self):
        if self._parser is None:
            self._parser = FileParser()
        return self._parser

    @property
    def chunker(self):
        if self._chunker is None:
            self._chunker = SectionAwareChunker(max_tokens=500, overlap_tokens=50)
        return self._chunker

    @property
    def embedding_service(self):
        if self._embedding_service is None:
            self._embedding_service = EmbeddingService()
        return self._embedding_service

    @property
    def qdrant_service(self):
        if self._qdrant_service is None:
            self._qdrant_service = QdrantService()
        return self._qdrant_service

    @property
    def mongo_client(self):
        if self._mongo_client is None:
            self._mongo_client = AsyncIOMotorClient("mongodb://localhost:27017")
        return self._mongo_client


@celery_app.task(
    bind=True,
    base=IngestionTask,
    max_retries=3,
    autoretry_for=(Exception,),
    retry_backoff=True
)
def ingest_document(
    self,
    tenant_id: str,
    document_id: str,
    source_type: str,
    source: str,
    metadata: Dict[str, Any] = None
):
    """
    Celery task for document ingestion pipeline.

    Args:
        tenant_id: Tenant identifier
        document_id: Unique document ID
        source_type: 'url', 'pdf', 'docx', or 'manual'
        source: URL, file path, or direct text
        metadata: Additional document metadata
    """
    logger.info(f"Starting ingestion: {document_id} ({source_type})")

    # Async wrapper
    async def _ingest():
        try:
            # Update status to processing
            await _update_document_status(
                self.mongo_client,
                tenant_id,
                document_id,
                "processing"
            )

            # STEP 1: Extract text based on source type
            if source_type == "url":
                result = await self.crawler.crawl(source)
                text = result["text"]
                extracted_metadata = {
                    "title": result.get("title", ""),
                    "description": result.get("description", ""),
                    "word_count": result.get("word_count", 0)
                }

            elif source_type in ["pdf", "docx"]:
                result = self.parser.parse_file(source)
                text = result["text"]
                extracted_metadata = result.get("metadata", {})
                extracted_metadata["word_count"] = result.get("word_count", 0)

            elif source_type == "manual":
                text = source
                extracted_metadata = {"word_count": len(text.split())}

            else:
                raise ValueError(f"Unknown source type: {source_type}")

            if not result.get("success", True):
                raise Exception(result.get("error", "Extraction failed"))

            # STEP 2: Chunk text
            chunks = self.chunker.chunk(text, metadata or {})
            logger.info(f"Created {len(chunks)} chunks")

            # STEP 3: Generate embeddings (batch)
            chunk_texts = [chunk.text for chunk in chunks]
            embeddings = self.embedding_service.embed_batch(chunk_texts)

            # STEP 4: Prepare Qdrant points
            points_payloads = []
            for chunk, embedding in zip(chunks, embeddings):
                payload = {
                    "text": chunk.text,
                    "document_id": document_id,
                    "source": source,
                    "source_type": source_type,
                    "chunk_index": chunk.chunk_index,
                    "total_chunks": len(chunks),
                    "section_path": chunk.section_path,
                    "token_count": chunk.token_count,
                    "created_at": datetime.utcnow().isoformat(),
                    "metadata": {**extracted_metadata, **(metadata or {})}
                }
                points_payloads.append(payload)

            # STEP 5: Store in Qdrant
            self.qdrant_service.create_collection(tenant_id)  # Idempotent
            point_ids = [f"{document_id}_chunk_{i}" for i in range(len(chunks))]
            self.qdrant_service.upsert_points(
                tenant_id,
                embeddings.tolist(),
                points_payloads,
                point_ids
            )

            # STEP 6: Update MongoDB document status
            await _update_document_status(
                self.mongo_client,
                tenant_id,
                document_id,
                "completed",
                {
                    "total_chunks": len(chunks),
                    "chunks_embedded": len(chunks),
                    "ingestion_completed_at": datetime.utcnow(),
                    "metadata": extracted_metadata
                }
            )

            logger.info(f"Ingestion completed: {document_id}")
            return {"status": "completed", "chunks": len(chunks)}

        except Exception as e:
            logger.error(f"Ingestion failed for {document_id}: {e}")

            # Update status to failed
            await _update_document_status(
                self.mongo_client,
                tenant_id,
                document_id,
                "failed",
                {"error_message": str(e)}
            )

            raise

    # Run async code
    loop = asyncio.get_event_loop()
    return loop.run_until_complete(_ingest())


async def _update_document_status(
    mongo_client: AsyncIOMotorClient,
    tenant_id: str,
    document_id: str,
    status: str,
    updates: Dict[str, Any] = None
):
    """Update document status in MongoDB."""
    db = mongo_client["customer_support_rag"]
    collection = db["knowledge_documents"]

    update_data = {
        "ingestion_status": status,
        "updated_at": datetime.utcnow()
    }

    if updates:
        update_data.update(updates)

    await collection.update_one(
        {"tenant_id": tenant_id, "document_id": document_id},
        {"$set": update_data}
    )


# Example usage
if __name__ == "__main__":
    # Enqueue ingestion task
    result = ingest_document.delay(
        tenant_id="acme_corp",
        document_id="doc_123",
        source_type="url",
        source="https://example.com/docs/api",
        metadata={"category": "API Documentation"}
    )

    print(f"Task ID: {result.id}")
    print(f"Task status: {result.status}")
```

---

## Production Tips

### 1. Duplicate Detection

**Problem:** Re-ingesting the same content creates duplicate vectors, degrading search quality.

**Solution:** Content hashing before ingestion.

```python
import hashlib
from typing import Optional

async def check_duplicate(
    mongo_client,
    tenant_id: str,
    content_hash: str
) -> Optional[str]:
    """Check if content already exists. Returns document_id if found."""
    collection = mongo_client["customer_support_rag"]["knowledge_documents"]
    existing = await collection.find_one({
        "tenant_id": tenant_id,
        "content_hash": content_hash
    })
    return existing["document_id"] if existing else None

def compute_content_hash(text: str) -> str:
    """Generate SHA256 hash of content."""
    return hashlib.sha256(text.encode('utf-8')).hexdigest()

# In ingestion pipeline:
content_hash = compute_content_hash(text)
existing_doc_id = await check_duplicate(mongo_client, tenant_id, content_hash)
if existing_doc_id:
    logger.info(f"Duplicate detected: {existing_doc_id}")
    return {"status": "duplicate", "document_id": existing_doc_id}
```

### 2. Retry Failed Ingestion

**Problem:** Transient failures (network, API limits) should not require manual re-ingestion.

**Solution:** Celery automatic retry with exponential backoff.

```python
@celery_app.task(
    bind=True,
    max_retries=3,
    autoretry_for=(httpx.HTTPError, TimeoutError),
    retry_backoff=True,          # Exponential backoff
    retry_backoff_max=600,        # Max 10 minutes
    retry_jitter=True             # Add randomness to prevent thundering herd
)
def ingest_document(self, ...):
    # Task code
```

### 3. Progress Tracking

**Problem:** Users need visibility into long-running ingestion jobs.

**Solution:** Update MongoDB with progress checkpoints.

```python
async def track_progress(mongo_client, tenant_id, document_id, stage, progress_pct):
    """Update ingestion progress."""
    await mongo_client["customer_support_rag"]["knowledge_documents"].update_one(
        {"tenant_id": tenant_id, "document_id": document_id},
        {
            "$set": {
                "ingestion_stage": stage,  # "parsing", "chunking", "embedding", "storing"
                "progress_percentage": progress_pct,
                "updated_at": datetime.utcnow()
            }
        }
    )

# In ingestion pipeline:
await track_progress(mongo_client, tenant_id, document_id, "parsing", 20)
# ... parse text ...
await track_progress(mongo_client, tenant_id, document_id, "chunking", 40)
# ... chunk text ...
await track_progress(mongo_client, tenant_id, document_id, "embedding", 70)
# ... generate embeddings ...
await track_progress(mongo_client, tenant_id, document_id, "storing", 90)
# ... store in Qdrant ...
```

### 4. Batch Ingestion Optimization

For ingesting many documents at once:

```python
@celery_app.task
def ingest_batch(tenant_id: str, document_specs: List[Dict]):
    """Ingest multiple documents efficiently."""
    # Deduplicate within batch
    unique_specs = deduplicate_batch(document_specs)

    # Chunk all documents first
    all_chunks = []
    chunk_to_doc = {}
    for spec in unique_specs:
        chunks = chunk_document(spec)
        all_chunks.extend([c.text for c in chunks])
        for chunk in chunks:
            chunk_to_doc[len(all_chunks) - 1] = spec['document_id']

    # Embed all chunks in one large batch (efficient)
    embeddings = embedding_service.embed_batch(all_chunks)

    # Bulk upsert to Qdrant
    qdrant_service.upsert_points(tenant_id, embeddings.tolist(), payloads)
```

---

## Interview Q&A

### Q1: How do you choose chunk size?

**Answer:**
Chunk size is a balance between context and granularity. Key considerations:

1. **Context window:** Too small (< 100 tokens) loses context. Too large (> 1000 tokens) dilutes relevance.
2. **Retrieval precision:** Smaller chunks = more precise but may miss context. Larger chunks = more context but less precise.
3. **Embedding model:** MiniLM max sequence is 256 tokens. Longer chunks get truncated.
4. **LLM context limits:** Retrieved chunks must fit in prompt (e.g., 5 chunks × 500 tokens = 2500 tokens, leaves room for conversation).

**Our choice: 500 tokens with section-aware splitting**
- Aligns with document structure (sections, subsections)
- Fits well in LLM context (5-10 chunks per query)
- Preserves semantic coherence
- Tested empirically: better retrieval precision than 200 or 1000 token chunks

**Trade-off:** If user queries are very specific (e.g., "What is the API rate limit?"), smaller chunks (200 tokens) retrieve more precise answers. If queries are conceptual (e.g., "How does authentication work?"), larger chunks (500-800 tokens) provide better context.

---

### Q2: Why overlap chunks?

**Answer:**
Overlap prevents information loss at chunk boundaries and improves retrieval recall.

**Problem without overlap:**
```
[... the authentication process involves |] [| obtaining an access token from ...]
```
Query: "How to get access token?"
- Both chunks mention tokens, but neither is a complete answer
- Retrieval may miss the full context

**Solution with 50-token overlap:**
```
[... the authentication process involves obtaining an access token from the /oauth/token endpoint ...]
[... obtaining an access token from the /oauth/token endpoint. The token expires after ...]
```
Query: "How to get access token?"
- First chunk contains complete answer
- Second chunk provides additional context (expiration)

**Overlap size:**
- Too small (10 tokens): Doesn't capture meaningful context
- Too large (200 tokens): Redundant storage and slower search
- **Sweet spot: 50-100 tokens (10-20% of chunk size)**

**Cost:** Increased storage (10-20%) and embedding compute. Worth it for improved retrieval quality.

---

### Q3: MiniLM vs OpenAI embeddings?

**Answer:**

| Criterion | MiniLM (all-MiniLM-L6-v2) | OpenAI (text-embedding-3-small) |
|-----------|---------------------------|----------------------------------|
| **Speed** | ~1000 docs/sec (local GPU) | ~100 docs/sec (API rate limits) |
| **Cost** | Free (self-hosted) | $0.02 per 1M tokens (~$0.20 per 100K docs) |
| **Dimensions** | 384 | 1536 |
| **Quality** | Good (trained on 1B pairs) | Excellent (proprietary training) |
| **Latency** | <10ms (local) | 50-200ms (API roundtrip) |
| **Offline** | Yes | No (requires internet) |
| **Maintenance** | Model updates manual | Model updates automatic |

**When to use MiniLM:**
- High-volume ingestion (millions of documents)
- Budget-constrained
- Low-latency requirements
- Offline/air-gapped deployments
- Sufficient quality for domain-specific support docs

**When to use OpenAI:**
- Maximum retrieval quality needed
- Low document volume (< 100K)
- Budget available
- Multi-language, multi-domain corpus

**Hybrid approach (recommended for production):**
1. Use MiniLM for ingestion (cost + speed)
2. Use OpenAI embeddings for query-time reranking (quality)

```python
# Hybrid retrieval
async def hybrid_search(query: str):
    # Stage 1: Fast MiniLM search (retrieve 50 candidates)
    query_embedding_minilm = minilm_service.embed_single(query)
    candidates = qdrant_service.search(tenant_id, query_embedding_minilm, limit=50)

    # Stage 2: Rerank with OpenAI embeddings (top 10)
    query_embedding_openai = await openai_service.embed(query)
    reranked = rerank_with_openai(candidates, query_embedding_openai)

    return reranked[:10]
```

This gives 90% of OpenAI quality at 10% of the cost.

---

### Q4: How do you handle large documents (1000+ pages)?

**Answer:**

Large documents present challenges:
1. **Memory:** Loading entire document into memory may fail
2. **Chunking:** Creates thousands of chunks, slowing retrieval
3. **Context loss:** Chunks from page 1 and page 500 may lack connection

**Solution: Hierarchical chunking + smart metadata**

```python
class HierarchicalChunker:
    def chunk_large_document(self, file_path: str, max_pages_per_section: int = 50):
        """Chunk large PDFs in sections."""
        chunks = []

        with open(file_path, 'rb') as file:
            pdf_reader = PyPDF2.PdfReader(file)
            num_pages = len(pdf_reader.pages)

            # Process in sections
            for section_start in range(0, num_pages, max_pages_per_section):
                section_end = min(section_start + max_pages_per_section, num_pages)

                # Extract section text
                section_text = ""
                for page_num in range(section_start, section_end):
                    section_text += pdf_reader.pages[page_num].extract_text()

                # Chunk section
                section_chunks = self.chunker.chunk(section_text)

                # Add section metadata
                for chunk in section_chunks:
                    chunk.metadata['page_range'] = f"{section_start + 1}-{section_end}"
                    chunk.metadata['section_index'] = section_start // max_pages_per_section
                    chunks.append(chunk)

        return chunks
```

**Additional optimizations:**
- **Streaming parsing:** Process page-by-page instead of loading entire document
- **Separate indexing:** Create per-chapter or per-section collections
- **Summary embeddings:** Generate document-level summary, embed separately for high-level queries
- **Metadata filtering:** Allow users to filter by page range, chapter, etc.

---

### Q5: How do you validate ingestion quality?

**Answer:**

Validation is critical to catch issues before they degrade retrieval.

**1. Syntax validation:**
```python
def validate_chunk(chunk: Chunk) -> bool:
    """Basic sanity checks."""
    if len(chunk.text) < 10:
        logger.warning("Chunk too short")
        return False

    if chunk.token_count > 1000:
        logger.warning("Chunk exceeds max tokens")
        return False

    if not chunk.section_path:
        logger.warning("Missing section path")
        return False

    return True
```

**2. Embedding quality:**
```python
def validate_embeddings(embeddings: np.ndarray) -> bool:
    """Check embedding sanity."""
    # Check for NaN or Inf
    if np.any(np.isnan(embeddings)) or np.any(np.isinf(embeddings)):
        logger.error("Invalid embedding values")
        return False

    # Check dimensionality
    if embeddings.shape[1] != 384:
        logger.error(f"Wrong dimension: {embeddings.shape[1]}")
        return False

    # Check for zero vectors (indicates encoding failure)
    norms = np.linalg.norm(embeddings, axis=1)
    if np.any(norms < 0.01):
        logger.error("Zero or near-zero embedding detected")
        return False

    return True
```

**3. Retrieval smoke test:**
```python
async def smoke_test_ingestion(tenant_id: str, document_id: str):
    """Test that ingested document is retrievable."""
    # Get first chunk text
    collection = mongo_client["customer_support_rag"]["knowledge_documents"]
    doc = await collection.find_one({"document_id": document_id})

    # Extract query from first chunk
    qdrant_results = qdrant_service.search(
        tenant_id,
        query_vector=None,  # Retrieve by filter only
        limit=1,
        filters={"document_id": document_id}
    )

    if not qdrant_results:
        raise ValueError(f"Document {document_id} not found in Qdrant")

    first_chunk_text = qdrant_results[0]['text'][:100]  # First 100 chars

    # Search for this text
    query_embedding = embedding_service.embed_single(first_chunk_text)
    search_results = qdrant_service.search(tenant_id, query_embedding, limit=10)

    # Check if document appears in results
    result_doc_ids = [r['metadata']['document_id'] for r in search_results]
    if document_id not in result_doc_ids:
        raise ValueError(f"Smoke test failed: document {document_id} not retrieved")

    logger.info(f"Smoke test passed for {document_id}")
```

**4. Human spot-check (production):**
- Randomly sample 1% of ingested chunks
- Manual review for text quality, metadata accuracy
- Track spot-check failures in metrics dashboard

---

## Summary

This section built a production-grade knowledge ingestion pipeline with:

1. **Three ingestion strategies:** URL crawling, file upload, manual text
2. **Section-aware chunking:** Preserves document structure for better retrieval
3. **MiniLM embeddings:** Fast, cost-effective, local processing
4. **Qdrant storage:** Collection-per-tenant isolation, HNSW indexing
5. **MongoDB tracking:** Document metadata, ingestion status, error handling
6. **Celery tasks:** Asynchronous processing, retry logic, progress tracking
7. **Production tips:** Duplicate detection, retry strategies, batch optimization

**Key takeaway:** Ingestion quality determines RAG quality. Invest in robust parsing, smart chunking, and validation to build a reliable knowledge base.

**Next section:** Section 7 will cover the retrieval pipeline, including semantic search, hybrid search, reranking, and relevance tuning.
# Section 7: Retrieval Pipeline

## WHY: Retrieval Quality = Answer Quality

The retrieval pipeline is the critical bridge between a user's question and the knowledge base. **Poor retrieval directly causes hallucinations** - if the right context doesn't reach the LLM, it will fabricate answers or return "I don't know." In production customer support RAG systems, retrieval quality has a **direct 1:1 correlation with answer accuracy**.

Key insight: **Not all queries are semantic**. Pure vector search excels at semantic similarity ("How do I reset my password?" matches "Password recovery steps") but fails on exact matches like product codes ("SKU-1234"), error messages ("ERROR_CODE_500"), or specific terminology. This is why hybrid search combining vector embeddings with keyword matching is essential for production systems.

## WHAT: Pipeline Architecture

```
User Query
    │
    ├──> Query Processor
    │      ├─ Expand conversational queries
    │      ├─ Extract filters (date, category, source)
    │      └─ Normalize text
    │
    ├──> Embedding Generator (MiniLM)
    │      └─ Convert to 384-dim vector
    │
    ├──> Vector Search (Qdrant)
    │      ├─ Cosine similarity
    │      ├─ top_K = 5-10
    │      └─ score_threshold = 0.7
    │
    ├──> BM25 Keyword Search
    │      ├─ Exact term matching
    │      ├─ TF-IDF weighting
    │      └─ top_K = 5-10
    │
    └──> Reciprocal Rank Fusion (RRF)
           ├─ Merge vector + BM25 results
           ├─ Score = Σ 1/(k + rank_i)
           └─ k = 60 (standard)

Merged Results
    │
    ├──> Optional Reranker (Cross-Encoder)
    │      ├─ Compute query-document relevance
    │      ├─ Higher quality than bi-encoder
    │      └─ More expensive (run on top-20)
    │
    └──> Context Assembler
           ├─ Deduplicate chunks
           ├─ Order by score
           ├─ Trim to token budget (4000 tokens)
           └─ Format for LLM consumption

Final Context → Generation Pipeline
```

## HOW: Implementation

### 1. Query Processor

Handles conversation context and query expansion:

```python
# services/query_processor.py
from typing import List, Dict, Optional
import re
from datetime import datetime
from loguru import logger

class QueryProcessor:
    """
    Processes user queries for retrieval optimization.
    Handles conversation context, filter extraction, and normalization.
    """

    def __init__(self):
        self.category_keywords = {
            "billing": ["invoice", "payment", "charge", "subscription", "refund"],
            "technical": ["error", "bug", "crash", "not working", "issue"],
            "account": ["login", "password", "profile", "settings", "access"],
        }

    async def process_query(
        self,
        query: str,
        conversation_history: Optional[List[Dict[str, str]]] = None,
        metadata_filters: Optional[Dict[str, any]] = None
    ) -> Dict[str, any]:
        """
        Process user query with conversation context.

        Args:
            query: User's raw query text
            conversation_history: Previous messages for context
            metadata_filters: Explicit filters (date, category, source)

        Returns:
            Processed query with expanded text and filters
        """
        # Expand query with conversation context
        expanded_query = self._expand_conversational_query(query, conversation_history)

        # Extract implicit filters
        extracted_filters = self._extract_filters(expanded_query)

        # Merge with explicit filters
        final_filters = {**extracted_filters, **(metadata_filters or {})}

        # Normalize text
        normalized_query = self._normalize_text(expanded_query)

        logger.info(
            f"Processed query: '{query[:50]}...' → '{normalized_query[:50]}...'",
            filters=final_filters
        )

        return {
            "original_query": query,
            "expanded_query": expanded_query,
            "normalized_query": normalized_query,
            "filters": final_filters,
        }

    def _expand_conversational_query(
        self,
        query: str,
        conversation_history: Optional[List[Dict[str, str]]]
    ) -> str:
        """
        Expand pronouns and references using conversation context.

        Example:
            User: "How do I reset my password?"
            Bot: "Go to Settings > Security."
            User: "What if that doesn't work?"  # "that" refers to password reset

        Returns expanded query with resolved references.
        """
        if not conversation_history or len(conversation_history) < 2:
            return query

        # Simple pronoun resolution (production: use coreference resolution model)
        pronouns = ["it", "that", "this", "those", "these"]
        query_lower = query.lower()

        if any(pronoun in query_lower.split() for pronoun in pronouns):
            # Get last user message for context
            last_user_msg = None
            for msg in reversed(conversation_history):
                if msg.get("role") == "user":
                    last_user_msg = msg.get("content", "")
                    break

            if last_user_msg:
                # Append previous context (simple approach)
                expanded = f"{last_user_msg} {query}"
                return expanded

        return query

    def _extract_filters(self, query: str) -> Dict[str, any]:
        """
        Extract metadata filters from query text.

        Examples:
            "billing issues last month" → category=billing, date_from=last_month
            "login problems" → category=account
        """
        filters = {}
        query_lower = query.lower()

        # Category detection
        for category, keywords in self.category_keywords.items():
            if any(keyword in query_lower for keyword in keywords):
                filters["category"] = category
                break

        # Date range detection (simple patterns)
        if "last week" in query_lower:
            filters["date_from"] = datetime.now().timestamp() - 7 * 86400
        elif "last month" in query_lower:
            filters["date_from"] = datetime.now().timestamp() - 30 * 86400

        return filters

    def _normalize_text(self, text: str) -> str:
        """
        Normalize text for retrieval:
        - Lowercase
        - Remove extra whitespace
        - Preserve important punctuation (for error codes, versions)
        """
        # Lowercase
        text = text.lower()

        # Collapse whitespace
        text = re.sub(r'\s+', ' ', text).strip()

        return text
```

### 2. Retrieval Service with Hybrid Search

Combines vector search and BM25 keyword search using Reciprocal Rank Fusion:

```python
# services/retrieval_service.py
from typing import List, Dict, Optional
from sentence_transformers import SentenceTransformer
from qdrant_client import QdrantClient
from qdrant_client.models import Filter, FieldCondition, MatchValue, SearchParams
from rank_bm25 import BM25Okapi
import tiktoken
from loguru import logger

class RetrievalService:
    """
    Production-grade retrieval service with hybrid search.
    Combines vector search (semantic) + BM25 (keyword) for robustness.
    """

    def __init__(
        self,
        qdrant_client: QdrantClient,
        embedding_model: SentenceTransformer,
        collection_name: str = "knowledge_base",
        cache_client = None  # Redis for caching frequent queries
    ):
        self.qdrant = qdrant_client
        self.encoder = embedding_model
        self.collection_name = collection_name
        self.cache = cache_client
        self.tokenizer = tiktoken.get_encoding("cl100k_base")  # GPT-4 tokenizer

        # BM25 index (in-memory, rebuild periodically from Qdrant)
        self.bm25_index = None
        self.bm25_corpus = []
        self._build_bm25_index()

    async def hybrid_search(
        self,
        query: str,
        top_k: int = 5,
        vector_weight: float = 0.7,
        bm25_weight: float = 0.3,
        score_threshold: float = 0.7,
        filters: Optional[Dict[str, any]] = None
    ) -> List[Dict[str, any]]:
        """
        Hybrid search combining vector and BM25 with Reciprocal Rank Fusion.

        Args:
            query: User query text
            top_k: Number of results to return
            vector_weight: Weight for vector search in fusion (0-1)
            bm25_weight: Weight for BM25 search in fusion (0-1)
            score_threshold: Minimum similarity score (0-1)
            filters: Metadata filters (category, date, source)

        Returns:
            List of chunks with scores, ordered by relevance
        """
        # Check cache
        cache_key = f"hybrid:{query}:{top_k}:{filters}"
        if self.cache:
            cached = await self.cache.get(cache_key)
            if cached:
                logger.info(f"Cache hit for query: {query[:50]}...")
                return cached

        # Run vector and BM25 searches in parallel
        vector_results = await self.vector_search(
            query, top_k=top_k * 2, score_threshold=score_threshold, filters=filters
        )
        bm25_results = await self.bm25_search(query, top_k=top_k * 2)

        # Reciprocal Rank Fusion
        fused_results = self._reciprocal_rank_fusion(
            vector_results, bm25_results, k=60,
            vector_weight=vector_weight, bm25_weight=bm25_weight
        )

        # Take top K
        final_results = fused_results[:top_k]

        # Cache results
        if self.cache:
            await self.cache.setex(cache_key, 3600, final_results)  # 1 hour TTL

        logger.info(
            f"Hybrid search: {len(vector_results)} vector + {len(bm25_results)} BM25 "
            f"→ {len(final_results)} fused results"
        )

        return final_results

    async def vector_search(
        self,
        query: str,
        top_k: int = 10,
        score_threshold: float = 0.7,
        filters: Optional[Dict[str, any]] = None
    ) -> List[Dict[str, any]]:
        """
        Pure vector search using Qdrant.

        Args:
            query: User query text
            top_k: Number of results
            score_threshold: Minimum cosine similarity (0-1)
            filters: Metadata filters

        Returns:
            List of chunks with similarity scores
        """
        # Embed query
        query_vector = self.encoder.encode(query, convert_to_tensor=False).tolist()

        # Build Qdrant filter
        qdrant_filter = self._build_qdrant_filter(filters) if filters else None

        # Search
        results = self.qdrant.search(
            collection_name=self.collection_name,
            query_vector=query_vector,
            limit=top_k,
            score_threshold=score_threshold,
            query_filter=qdrant_filter,
            search_params=SearchParams(
                hnsw_ef=128,  # Higher = better quality, slower
                exact=False  # Use HNSW approximate search
            )
        )

        # Format results
        chunks = []
        for hit in results:
            chunks.append({
                "id": hit.id,
                "text": hit.payload.get("text", ""),
                "metadata": hit.payload.get("metadata", {}),
                "score": hit.score,
                "search_type": "vector"
            })

        return chunks

    async def bm25_search(
        self,
        query: str,
        top_k: int = 10
    ) -> List[Dict[str, any]]:
        """
        BM25 keyword search for exact term matching.

        WHY: Handles product codes, error messages, specific terminology
        that pure semantic search misses.

        Args:
            query: User query text
            top_k: Number of results

        Returns:
            List of chunks with BM25 scores
        """
        if not self.bm25_index:
            logger.warning("BM25 index not built, returning empty results")
            return []

        # Tokenize query
        query_tokens = query.lower().split()

        # BM25 scoring
        scores = self.bm25_index.get_scores(query_tokens)

        # Get top K indices
        top_indices = sorted(
            range(len(scores)), key=lambda i: scores[i], reverse=True
        )[:top_k]

        # Format results
        chunks = []
        for idx in top_indices:
            if scores[idx] > 0:  # Only include non-zero scores
                chunks.append({
                    "id": self.bm25_corpus[idx].get("id"),
                    "text": self.bm25_corpus[idx].get("text", ""),
                    "metadata": self.bm25_corpus[idx].get("metadata", {}),
                    "score": float(scores[idx]),
                    "search_type": "bm25"
                })

        return chunks

    def _reciprocal_rank_fusion(
        self,
        vector_results: List[Dict[str, any]],
        bm25_results: List[Dict[str, any]],
        k: int = 60,
        vector_weight: float = 0.7,
        bm25_weight: float = 0.3
    ) -> List[Dict[str, any]]:
        """
        Reciprocal Rank Fusion (RRF) to merge vector and BM25 results.

        Formula: RRF_score = Σ weight_i / (k + rank_i)

        WHY RRF: Simple, effective, no score normalization needed.
        Ranks matter more than absolute scores.

        Args:
            vector_results: Results from vector search
            bm25_results: Results from BM25 search
            k: Constant for RRF (default 60, from literature)
            vector_weight: Weight for vector rankings
            bm25_weight: Weight for BM25 rankings

        Returns:
            Merged and reranked results
        """
        # Build rank maps
        rrf_scores = {}

        # Add vector results
        for rank, result in enumerate(vector_results):
            doc_id = result["id"]
            rrf_scores[doc_id] = {
                "score": vector_weight / (k + rank),
                "data": result
            }

        # Add BM25 results
        for rank, result in enumerate(bm25_results):
            doc_id = result["id"]
            if doc_id in rrf_scores:
                rrf_scores[doc_id]["score"] += bm25_weight / (k + rank)
            else:
                rrf_scores[doc_id] = {
                    "score": bm25_weight / (k + rank),
                    "data": result
                }

        # Sort by RRF score
        sorted_results = sorted(
            rrf_scores.items(),
            key=lambda x: x[1]["score"],
            reverse=True
        )

        # Format output
        merged = []
        for doc_id, data in sorted_results:
            result = data["data"].copy()
            result["rrf_score"] = data["score"]
            result["search_type"] = "hybrid"
            merged.append(result)

        return merged

    def _build_qdrant_filter(self, filters: Dict[str, any]) -> Filter:
        """Build Qdrant filter from metadata filters."""
        conditions = []

        if "category" in filters:
            conditions.append(
                FieldCondition(
                    key="metadata.category",
                    match=MatchValue(value=filters["category"])
                )
            )

        if "source" in filters:
            conditions.append(
                FieldCondition(
                    key="metadata.source",
                    match=MatchValue(value=filters["source"])
                )
            )

        if "date_from" in filters:
            conditions.append(
                FieldCondition(
                    key="metadata.timestamp",
                    range={
                        "gte": filters["date_from"]
                    }
                )
            )

        return Filter(must=conditions) if conditions else None

    def _build_bm25_index(self):
        """
        Build in-memory BM25 index from Qdrant collection.
        Run periodically (e.g., every hour) to refresh.
        """
        try:
            # Fetch all documents from Qdrant (paginated)
            offset = None
            all_docs = []

            while True:
                results = self.qdrant.scroll(
                    collection_name=self.collection_name,
                    limit=1000,
                    offset=offset,
                    with_payload=True,
                    with_vectors=False
                )

                points, next_offset = results

                for point in points:
                    all_docs.append({
                        "id": point.id,
                        "text": point.payload.get("text", ""),
                        "metadata": point.payload.get("metadata", {})
                    })

                if next_offset is None:
                    break
                offset = next_offset

            # Tokenize corpus
            tokenized_corpus = [doc["text"].lower().split() for doc in all_docs]

            # Build BM25 index
            self.bm25_index = BM25Okapi(tokenized_corpus)
            self.bm25_corpus = all_docs

            logger.info(f"Built BM25 index with {len(all_docs)} documents")

        except Exception as e:
            logger.error(f"Failed to build BM25 index: {e}")
            self.bm25_index = None
            self.bm25_corpus = []
```

### 3. Context Assembler

Deduplicates, orders, and trims chunks to fit token budget:

```python
# services/context_assembler.py
from typing import List, Dict
import tiktoken
from loguru import logger

class ContextAssembler:
    """
    Assembles retrieved chunks into final context for LLM.
    Handles deduplication, ordering, and token budget.
    """

    def __init__(self, max_tokens: int = 4000):
        self.max_tokens = max_tokens
        self.tokenizer = tiktoken.get_encoding("cl100k_base")

    def assemble_context(
        self,
        chunks: List[Dict[str, any]],
        include_metadata: bool = True
    ) -> str:
        """
        Assemble chunks into final context string.

        Args:
            chunks: Retrieved chunks with scores
            include_metadata: Include source metadata in context

        Returns:
            Formatted context string within token budget
        """
        # Deduplicate by text similarity (fuzzy matching)
        deduplicated = self._deduplicate_chunks(chunks)

        # Order by score (already sorted from retrieval)
        ordered = sorted(deduplicated, key=lambda x: x.get("rrf_score", x.get("score", 0)), reverse=True)

        # Trim to token budget
        context_parts = []
        total_tokens = 0

        for i, chunk in enumerate(ordered):
            # Format chunk
            chunk_text = self._format_chunk(chunk, i + 1, include_metadata)
            chunk_tokens = len(self.tokenizer.encode(chunk_text))

            # Check budget
            if total_tokens + chunk_tokens > self.max_tokens:
                logger.warning(
                    f"Token budget reached: {total_tokens}/{self.max_tokens}. "
                    f"Including {len(context_parts)}/{len(ordered)} chunks."
                )
                break

            context_parts.append(chunk_text)
            total_tokens += chunk_tokens

        # Join with separators
        context = "\n\n---\n\n".join(context_parts)

        logger.info(
            f"Assembled context: {len(context_parts)} chunks, "
            f"{total_tokens} tokens ({total_tokens/self.max_tokens*100:.1f}% of budget)"
        )

        return context

    def _deduplicate_chunks(self, chunks: List[Dict[str, any]]) -> List[Dict[str, any]]:
        """
        Deduplicate chunks with similar text.
        Uses simple character-level similarity (production: use MinHash or LSH).
        """
        seen_texts = set()
        deduplicated = []

        for chunk in chunks:
            text = chunk.get("text", "")
            # Simple normalization
            normalized = text.lower().replace(" ", "").replace("\n", "")

            if normalized not in seen_texts:
                seen_texts.add(normalized)
                deduplicated.append(chunk)

        return deduplicated

    def _format_chunk(self, chunk: Dict[str, any], index: int, include_metadata: bool) -> str:
        """
        Format chunk with metadata for LLM context.

        Example output:
        [Source 1] (Category: Billing, Score: 0.92)
        To cancel your subscription, go to Account > Billing > Cancel Plan.
        """
        text = chunk.get("text", "")
        metadata = chunk.get("metadata", {})
        score = chunk.get("rrf_score", chunk.get("score", 0))

        if include_metadata:
            meta_parts = []
            if "category" in metadata:
                meta_parts.append(f"Category: {metadata['category']}")
            if "source" in metadata:
                meta_parts.append(f"Source: {metadata['source']}")
            meta_parts.append(f"Score: {score:.2f}")

            meta_str = ", ".join(meta_parts)
            return f"[Source {index}] ({meta_str})\n{text}"
        else:
            return f"[Source {index}]\n{text}"
```

### 4. Optional: Cross-Encoder Reranker

For higher precision, rerank top results with a cross-encoder:

```python
# services/reranker.py
from sentence_transformers import CrossEncoder
from typing import List, Dict
from loguru import logger

class CrossEncoderReranker:
    """
    Rerank retrieved chunks using cross-encoder for higher precision.

    WHY: Bi-encoders (MiniLM) encode query and document independently.
    Cross-encoders compute joint query-document relevance → more accurate.

    TRADEOFF: ~10x slower, so only rerank top-K candidates (e.g., top 20).
    """

    def __init__(self, model_name: str = "cross-encoder/ms-marco-MiniLM-L-6-v2"):
        self.model = CrossEncoder(model_name)
        logger.info(f"Loaded cross-encoder: {model_name}")

    def rerank(
        self,
        query: str,
        chunks: List[Dict[str, any]],
        top_k: int = 5
    ) -> List[Dict[str, any]]:
        """
        Rerank chunks using cross-encoder.

        Args:
            query: User query
            chunks: Retrieved chunks (typically top 20 from hybrid search)
            top_k: Number of final results to return

        Returns:
            Reranked chunks with cross-encoder scores
        """
        if not chunks:
            return []

        # Prepare query-document pairs
        pairs = [[query, chunk.get("text", "")] for chunk in chunks]

        # Compute cross-encoder scores
        scores = self.model.predict(pairs)

        # Attach scores to chunks
        for chunk, score in zip(chunks, scores):
            chunk["rerank_score"] = float(score)

        # Sort by rerank score
        reranked = sorted(chunks, key=lambda x: x["rerank_score"], reverse=True)

        logger.info(
            f"Reranked {len(chunks)} chunks → returning top {top_k} "
            f"(score range: {scores.max():.2f} - {scores.min():.2f})"
        )

        return reranked[:top_k]
```

## Production Tips

### 1. Monitor Retrieval Quality Metrics

Track these metrics in production:

```python
# utils/retrieval_metrics.py
from typing import List, Dict
import numpy as np
from loguru import logger

class RetrievalMetrics:
    """Track retrieval quality metrics."""

    @staticmethod
    def calculate_mrr(results: List[Dict[str, any]], relevant_ids: List[str]) -> float:
        """
        Mean Reciprocal Rank: 1 / rank of first relevant result.
        Production: Track MRR per query type.
        """
        for rank, result in enumerate(results, start=1):
            if result["id"] in relevant_ids:
                return 1.0 / rank
        return 0.0

    @staticmethod
    def calculate_precision_at_k(
        results: List[Dict[str, any]],
        relevant_ids: List[str],
        k: int = 5
    ) -> float:
        """Precision@K: % of top-K results that are relevant."""
        top_k = results[:k]
        relevant_count = sum(1 for r in top_k if r["id"] in relevant_ids)
        return relevant_count / k if k > 0 else 0.0

    @staticmethod
    def log_retrieval_metrics(
        query: str,
        results: List[Dict[str, any]],
        user_feedback: Optional[str] = None
    ):
        """Log metrics for monitoring (send to metrics backend)."""
        logger.info(
            "Retrieval metrics",
            query=query[:50],
            num_results=len(results),
            avg_score=np.mean([r.get("score", 0) for r in results]),
            min_score=min([r.get("score", 0) for r in results]) if results else 0,
            user_feedback=user_feedback
        )
```

### 2. Cache Frequent Queries

Use Redis with TTL for caching:

```python
# Cache key format: "hybrid:{query_hash}:{top_k}:{filters_hash}"
# TTL: 1 hour for hot queries, evict LRU
```

### 3. Refresh BM25 Index Periodically

Schedule Celery task to rebuild BM25 index every hour:

```python
# tasks/refresh_bm25.py
from celery import shared_task
from services.retrieval_service import RetrievalService

@shared_task
def refresh_bm25_index():
    """Rebuild BM25 index from Qdrant collection."""
    retrieval_service = RetrievalService(...)
    retrieval_service._build_bm25_index()
```

### 4. A/B Test Retrieval Strategies

Track which strategy (pure vector vs hybrid vs reranked) performs best:

```python
# Assign users to groups, log strategy + user feedback
if user_id % 3 == 0:
    results = await retrieval_service.vector_search(query)
elif user_id % 3 == 1:
    results = await retrieval_service.hybrid_search(query)
else:
    results = await retrieval_service.hybrid_search(query)
    results = reranker.rerank(query, results)
```

## Interview Q&A

### Q1: Explain hybrid search and Reciprocal Rank Fusion (RRF). When would you use it over pure vector search?

**Answer:**

Hybrid search combines **semantic vector search** (embeddings) with **keyword-based BM25 search** to leverage both approaches' strengths:

- **Vector search**: Excels at semantic similarity ("How do I reset password?" matches "Password recovery steps")
- **BM25 search**: Excels at exact term matching (product codes "SKU-1234", error codes "ERROR_500")

**Reciprocal Rank Fusion (RRF)** merges results from both methods using ranks, not raw scores:

```
RRF_score(doc) = Σ weight_i / (k + rank_i)
```

Where k=60 (standard constant), and rank_i is the position in each result list.

**WHY RRF?** It avoids score normalization issues (vector scores are cosine similarities 0-1, BM25 scores are unbounded). Ranks are comparable across search methods.

**When to use hybrid:**
- Customer support systems with mixed query types (semantic questions + exact product/error codes)
- Domain-specific terminology that embeddings haven't seen
- When pure vector search has low precision on exact matches

**Production tip:** A/B test vector vs hybrid. In our system, hybrid improved accuracy by 15% for technical support queries.

---

### Q2: How do you evaluate retrieval quality in production? What metrics matter?

**Answer:**

Retrieval quality directly impacts answer accuracy, so monitoring is critical:

**1. Offline Metrics (with labeled test set):**
- **Precision@K**: % of top-K results that are relevant (target: >80% at K=5)
- **Recall@K**: % of relevant docs retrieved in top-K
- **Mean Reciprocal Rank (MRR)**: 1 / rank of first relevant result (target: >0.7)
- **NDCG@K**: Normalized Discounted Cumulative Gain (accounts for graded relevance)

**2. Online Metrics (in production):**
- **User feedback**: Thumbs up/down on answers (proxy for retrieval quality)
- **Context utilization**: Does LLM cite retrieved sources in answer? (track via parsing)
- **"I don't know" rate**: High rate may indicate retrieval failures
- **Query latency**: P50/P95/P99 retrieval times

**3. Monitoring Setup:**
```python
# Log every retrieval with query, results, scores, user feedback
logger.info("retrieval_event", query=query, results=results, feedback=feedback)

# Aggregate in monitoring dashboard:
# - MRR trend over time
# - Precision@5 by query category
# - Latency distribution
```

**4. Human Evaluation:**
- Sample 100 random queries weekly
- Annotators rate top-5 results (relevant/not relevant)
- Calculate Precision@5, identify failure patterns

**Red flags:**
- MRR dropping → degraded relevance
- P95 latency spiking → index issues, network problems
- Low feedback scores → retrieval or generation issues

---

### Q3: Cosine similarity vs dot product for vector search - what's the difference and when does it matter?

**Answer:**

**Cosine similarity** measures angle between vectors (normalized dot product):
```
cosine(A, B) = (A · B) / (||A|| × ||B||)
```
Range: -1 to 1 (or 0 to 1 for non-negative embeddings)

**Dot product** measures both angle and magnitude:
```
dot(A, B) = A · B = Σ a_i × b_i
```
Range: Unbounded

**Key difference:** Cosine is **magnitude-invariant**, dot product is **magnitude-sensitive**.

**When it matters:**

1. **Use cosine similarity when:**
   - Document length varies significantly (long docs vs short docs)
   - You want to compare semantic content regardless of document size
   - Embeddings aren't normalized (most sentence-transformers models)
   - **This is the default for most RAG systems**

2. **Use dot product when:**
   - Embeddings are pre-normalized to unit length (then dot = cosine)
   - You want to penalize shorter documents (less information)
   - Speed is critical (dot product is faster, no sqrt computation)

**Production implementation:**

Most vector DBs (Qdrant, Pinecone, Weaviate) support both. For MiniLM embeddings (our stack), use **cosine** because:
- Embeddings aren't normalized by default
- Document lengths vary (50 chars to 1000+ chars)
- Semantic similarity is what matters, not document length

**Gotcha:** If you normalize embeddings to unit length during indexing, cosine = dot product (mathematically equivalent). Some systems (Pinecone) auto-normalize, so dot product is faster with same results.

---

### Q4: How do you handle queries that span multiple chunks? What about context fragmentation?

**Answer:**

**Problem:** Answers may require information spread across multiple chunks (e.g., "What's the refund policy for enterprise plans?" needs chunks about refunds AND enterprise plans).

**Solutions:**

**1. Overlap in Chunking (Ingestion Pipeline):**
```python
# When chunking, use 10-15% overlap
chunk_size = 500
overlap = 75  # 15% overlap
# Ensures split concepts have context in adjacent chunks
```

**2. Parent-Child Chunking:**
- Index small chunks (for precision)
- But return parent document (for full context)
```python
# Store in Qdrant payload:
payload = {
    "text": small_chunk,
    "parent_text": full_document,  # Return this to LLM
    "chunk_index": 2
}
```

**3. Sliding Window in Context Assembly:**
```python
# If chunk #3 and #5 are retrieved, also include #4 (bridging chunk)
def include_bridging_chunks(chunks, window=1):
    chunk_ids = sorted([c["chunk_index"] for c in chunks])
    for i in range(len(chunk_ids) - 1):
        if chunk_ids[i+1] - chunk_ids[i] <= window + 1:
            # Fetch intermediate chunks
            ...
```

**4. Query Decomposition (Advanced):**
- Use LLM to break complex query into sub-queries
- Retrieve for each sub-query
- Merge results
```python
# "What's the refund policy for enterprise plans?"
sub_queries = [
    "What is the refund policy?",
    "What are enterprise plan features?"
]
```

**5. Increase top_K and Let LLM Synthesize:**
- Retrieve more chunks (e.g., top_K=10 instead of 5)
- LLM's context window (128K tokens for GPT-4) can handle it
- Trade more context for less fragmentation

**Production approach:** Use overlap + parent-child chunking. This covers 90% of cases without complex query decomposition.

---

### Q5: Your BM25 index is in-memory. What happens when it doesn't fit in RAM? How do you scale it?

**Answer:**

**Current limitation:** In-memory BM25 (rank_bm25 library) doesn't scale beyond ~1M documents (~2-3GB RAM).

**Scaling strategies:**

**1. Use Elasticsearch for BM25 (Recommended for >1M docs):**
```python
# Replace in-memory BM25 with Elasticsearch
from elasticsearch import AsyncElasticsearch

class RetrievalService:
    def __init__(self, ..., es_client: AsyncElasticsearch):
        self.es = es_client

    async def bm25_search(self, query: str, top_k: int):
        results = await self.es.search(
            index="knowledge_base",
            body={
                "query": {"match": {"text": query}},
                "size": top_k
            }
        )
        return results["hits"]["hits"]
```
Elasticsearch handles BM25 at scale, with sharding and persistence.

**2. Shard BM25 by Category:**
```python
# Build separate BM25 indices per category (billing, technical, account)
self.bm25_indices = {
    "billing": BM25Okapi(...),
    "technical": BM25Okapi(...),
}
# Search only relevant shard based on query classification
```

**3. Use Approximate BM25 (Sampling):**
```python
# For massive corpora (10M+ docs), sample top 100K most recent docs
# Rebuild daily with rolling window
# Trade completeness for memory efficiency
```

**4. Move to Qdrant Sparse Vectors (Experimental):**
- Qdrant supports sparse vector search (BM25-like)
- Single DB for dense + sparse vectors
- Currently experimental, not production-ready

**Production recommendation:**
- <1M docs: In-memory BM25 (rebuild hourly, ~2GB RAM)
- 1M-10M docs: Elasticsearch for BM25, Qdrant for vector search
- >10M docs: Shard by category + Elasticsearch

**Our stack (customer support SaaS):** Start with in-memory BM25 (typical knowledge bases <100K docs), migrate to Elasticsearch if growth exceeds 500K documents.

# Section 8: Generation Pipeline & Prompt Engineering

## WHY: Turning Context into Helpful Answers

The generation pipeline is where retrieval context transforms into human-like, accurate customer support responses. **This is the user-facing interface** - poor generation ruins excellent retrieval. The key challenge: **preventing hallucinations** while maintaining helpfulness.

Critical insight: **Prompt engineering is your primary defense against hallucination.** A well-designed system prompt with grounding instructions, few-shot examples, and explicit "I don't know" handling is worth more than any post-processing filter. In production systems, 80% of answer quality comes from prompt design, 20% from model selection.

## WHAT: Generation Architecture

```
Retrieved Context + User Query + Conversation History
    │
    ├──> Prompt Builder
    │      ├─ System Prompt (grounding instructions)
    │      ├─ Few-Shot Examples (in-context learning)
    │      ├─ Retrieved Context (formatted with sources)
    │      ├─ Conversation History (sliding window)
    │      └─ User Query
    │
    ├──> Token Budget Manager
    │      ├─ Count tokens (tiktoken)
    │      ├─ Prioritize: Context > History > Examples
    │      └─ Trim to fit model context window
    │
    ├──> GPT API Client (OpenAI)
    │      ├─ Model: GPT-4o (quality) vs GPT-4o-mini (cost)
    │      ├─ Streaming: Yield chunks for real-time UX
    │      ├─ Temperature: 0.3 (low randomness)
    │      ├─ Max tokens: 500 (concise answers)
    │      └─ Circuit Breaker (failure handling)
    │
    └──> Response Post-Processor
           ├─ Parse markdown
           ├─ Extract source citations
           ├─ Confidence scoring
           └─ Format for display

Final Answer → User
```

## HOW: Implementation

### 1. Prompt Builder

Constructs the full chat prompt with grounding and examples:

```python
# services/prompt_builder.py
from typing import List, Dict, Optional
import tiktoken
from loguru import logger

class PromptBuilder:
    """
    Builds prompts for GPT API with grounding, examples, and context.
    Handles token budgeting and sliding window conversation history.
    """

    def __init__(self, max_tokens: int = 8000):
        self.max_tokens = max_tokens
        self.tokenizer = tiktoken.get_encoding("cl100k_base")

        # System prompt with grounding instructions
        self.system_prompt = self._build_system_prompt()

        # Few-shot examples for in-context learning
        self.few_shot_examples = self._build_few_shot_examples()

    def build_chat_prompt(
        self,
        user_query: str,
        context: str,
        conversation_history: Optional[List[Dict[str, str]]] = None,
        include_examples: bool = True
    ) -> List[Dict[str, str]]:
        """
        Build full chat prompt for GPT API.

        Args:
            user_query: Current user question
            context: Retrieved context from RAG pipeline
            conversation_history: Previous messages (list of {role, content})
            include_examples: Whether to include few-shot examples

        Returns:
            List of chat messages in OpenAI format
        """
        messages = []

        # 1. System prompt (always first)
        messages.append({
            "role": "system",
            "content": self.system_prompt
        })

        # 2. Few-shot examples (optional, improves consistency)
        if include_examples:
            messages.extend(self.few_shot_examples)

        # 3. Conversation history (sliding window)
        if conversation_history:
            history_messages = self._apply_history_window(
                conversation_history, max_messages=5
            )
            messages.extend(history_messages)

        # 4. Retrieved context (formatted)
        context_message = self._format_context(context)
        messages.append({
            "role": "system",
            "content": context_message
        })

        # 5. Current user query
        messages.append({
            "role": "user",
            "content": user_query
        })

        # 6. Token budget check (trim if needed)
        messages = self._trim_to_budget(messages)

        logger.info(
            f"Built prompt: {len(messages)} messages, "
            f"{self._count_tokens(messages)} tokens"
        )

        return messages

    def _build_system_prompt(self) -> str:
        """
        System prompt with grounding instructions.

        Critical components:
        1. Role definition (customer support assistant)
        2. Grounding instructions (only use provided context)
        3. "I don't know" handling (explicit permission to refuse)
        4. Citation requirements (reference sources)
        5. Tone and style guidelines
        """
        return """You are a helpful customer support assistant. Your goal is to provide accurate, concise answers to user questions based ONLY on the provided context.

CRITICAL RULES:
1. **Answer ONLY from the provided context**. Do not use external knowledge or make assumptions.
2. **If the context doesn't contain enough information to answer, say "I don't have enough information to answer that question. Let me connect you with a human agent who can help."**
3. **Cite your sources** by referencing [Source N] numbers from the context.
4. **Be concise** - aim for 2-3 sentences unless more detail is needed.
5. **Be empathetic** - acknowledge user frustration if present.
6. **Use markdown formatting** for readability (bullets, bold, code blocks).
7. **If multiple solutions exist**, present the most common one first.

NEVER:
- Fabricate information not in the context
- Make promises about policies you're unsure of
- Provide troubleshooting steps not explicitly mentioned
- Guess at product features or pricing

When you use information from context, cite it like: "According to [Source 2], you can..."
"""

    def _build_few_shot_examples(self) -> List[Dict[str, str]]:
        """
        Few-shot examples for in-context learning.

        Demonstrates:
        1. Proper citation format
        2. "I don't know" handling
        3. Markdown formatting
        4. Tone and style
        """
        return [
            # Example 1: Good answer with citation
            {
                "role": "user",
                "content": "How do I reset my password?"
            },
            {
                "role": "assistant",
                "content": """To reset your password:

1. Go to the login page and click "Forgot Password"
2. Enter your email address
3. Check your email for a reset link (valid for 1 hour)
4. Click the link and create a new password

According to [Source 1], your new password must be at least 8 characters with one uppercase letter and one number.

If you don't receive the email within 5 minutes, check your spam folder or contact support."""
            },

            # Example 2: "I don't know" when context lacks info
            {
                "role": "user",
                "content": "Can I get a refund after 60 days?"
            },
            {
                "role": "assistant",
                "content": """I don't have enough information about refunds after 60 days in the provided documentation. This is a specific policy question that's best answered by our billing team.

Let me connect you with a human agent who can review your account and provide accurate guidance."""
            },

            # Example 3: Citing multiple sources
            {
                "role": "user",
                "content": "What's included in the Pro plan?"
            },
            {
                "role": "assistant",
                "content": """The Pro plan includes:

- **Unlimited projects** ([Source 1])
- **50 GB storage** ([Source 1])
- **Priority support** with 1-hour response time ([Source 2])
- **Advanced analytics** dashboard ([Source 3])
- **API access** with 10,000 requests/month ([Source 3])

You can upgrade anytime from your Account Settings. According to [Source 2], upgrades are prorated based on your billing cycle."""
            }
        ]

    def _format_context(self, context: str) -> str:
        """
        Format retrieved context for prompt.

        Adds header and instructions to make context clear to LLM.
        """
        if not context or context.strip() == "":
            return "No relevant context found in the knowledge base."

        return f"""Below is relevant information from our knowledge base. Use this to answer the user's question.

---
{context}
---

Remember: Only use information from the context above. If the context doesn't answer the user's question, say so clearly."""

    def _apply_history_window(
        self,
        conversation_history: List[Dict[str, str]],
        max_messages: int = 5
    ) -> List[Dict[str, str]]:
        """
        Apply sliding window to conversation history.

        Keep last N messages (default 5 = ~2-3 turns).
        WHY: Prevents context from growing unbounded, focuses on recent context.
        """
        if len(conversation_history) <= max_messages:
            return conversation_history

        # Take last N messages
        windowed = conversation_history[-max_messages:]

        logger.info(
            f"Applied history window: {len(conversation_history)} → {len(windowed)} messages"
        )

        return windowed

    def _trim_to_budget(self, messages: List[Dict[str, str]]) -> List[Dict[str, str]]:
        """
        Trim messages to fit token budget.

        Priority (keep in order):
        1. System prompt (always keep)
        2. Current user query (always keep)
        3. Retrieved context (most important)
        4. Conversation history (trim oldest first)
        5. Few-shot examples (remove if needed)
        """
        total_tokens = self._count_tokens(messages)

        if total_tokens <= self.max_tokens:
            return messages  # Fits budget

        logger.warning(
            f"Prompt exceeds budget: {total_tokens}/{self.max_tokens} tokens. Trimming..."
        )

        # Separate message types
        system_prompt = messages[0]  # Always system
        few_shot_end = 1 + len(self.few_shot_examples) * 2  # User + assistant pairs
        few_shot = messages[1:few_shot_end]
        history = messages[few_shot_end:-2]  # Between few-shot and context+query
        context = messages[-2]
        user_query = messages[-1]

        # Strategy 1: Remove few-shot examples
        trimmed = [system_prompt] + history + [context, user_query]
        if self._count_tokens(trimmed) <= self.max_tokens:
            logger.info("Removed few-shot examples to fit budget")
            return trimmed

        # Strategy 2: Trim conversation history (oldest first)
        while history and self._count_tokens(trimmed) > self.max_tokens:
            history = history[2:]  # Remove oldest user+assistant pair
            trimmed = [system_prompt] + history + [context, user_query]

        if self._count_tokens(trimmed) <= self.max_tokens:
            logger.info(f"Trimmed history to {len(history)} messages")
            return trimmed

        # Strategy 3: Trim context (last resort, bad for quality)
        context_text = context["content"]
        while self._count_tokens(trimmed) > self.max_tokens:
            # Remove last paragraph
            paragraphs = context_text.split("\n\n")
            if len(paragraphs) <= 1:
                break
            paragraphs = paragraphs[:-1]
            context_text = "\n\n".join(paragraphs)
            trimmed[-2]["content"] = context_text

        logger.warning("Had to trim context to fit budget - answer quality may degrade")
        return trimmed

    def _count_tokens(self, messages: List[Dict[str, str]]) -> int:
        """Count tokens in messages (OpenAI chat format)."""
        # Approximate: 4 tokens per message overhead + content tokens
        total = 0
        for message in messages:
            total += 4  # Message overhead
            total += len(self.tokenizer.encode(message["content"]))
        return total
```

### 2. GPT Client with Streaming

Handles API calls with streaming for real-time UX:

```python
# services/gpt_client.py
from typing import AsyncGenerator, List, Dict, Optional
import openai
from openai import AsyncOpenAI
import asyncio
from loguru import logger

class GPTClient:
    """
    Wrapper for OpenAI GPT API with streaming, retries, and circuit breaker.
    """

    def __init__(
        self,
        api_key: str,
        model: str = "gpt-4o",  # or "gpt-4o-mini" for cost savings
        temperature: float = 0.3,
        max_tokens: int = 500,
        circuit_breaker = None
    ):
        self.client = AsyncOpenAI(api_key=api_key)
        self.model = model
        self.temperature = temperature
        self.max_tokens = max_tokens
        self.circuit_breaker = circuit_breaker

    async def generate_streaming(
        self,
        messages: List[Dict[str, str]],
        temperature: Optional[float] = None,
        max_tokens: Optional[int] = None
    ) -> AsyncGenerator[str, None]:
        """
        Generate answer with streaming (yields chunks as they arrive).

        WHY streaming: Improves perceived latency - user sees response start in <1s
        instead of waiting 5-10s for full response.

        Args:
            messages: Chat messages in OpenAI format
            temperature: Override default (lower = more deterministic)
            max_tokens: Override default max response length

        Yields:
            String chunks as they arrive from API
        """
        # Check circuit breaker
        if self.circuit_breaker and not self.circuit_breaker.allow_request():
            logger.error("Circuit breaker open, rejecting request")
            raise Exception("Service temporarily unavailable (circuit breaker open)")

        temp = temperature if temperature is not None else self.temperature
        max_tok = max_tokens if max_tokens is not None else self.max_tokens

        try:
            logger.info(
                f"GPT API request: model={self.model}, temp={temp}, "
                f"max_tokens={max_tok}, messages={len(messages)}"
            )

            # Streaming request
            stream = await self.client.chat.completions.create(
                model=self.model,
                messages=messages,
                temperature=temp,
                max_tokens=max_tok,
                stream=True
            )

            # Yield chunks
            async for chunk in stream:
                if chunk.choices[0].delta.content:
                    content = chunk.choices[0].delta.content
                    yield content

            # Record success in circuit breaker
            if self.circuit_breaker:
                self.circuit_breaker.record_success()

        except openai.RateLimitError as e:
            logger.error(f"Rate limit exceeded: {e}")
            if self.circuit_breaker:
                self.circuit_breaker.record_failure()
            raise

        except openai.APIError as e:
            logger.error(f"OpenAI API error: {e}")
            if self.circuit_breaker:
                self.circuit_breaker.record_failure()
            raise

        except Exception as e:
            logger.error(f"Unexpected error in GPT generation: {e}")
            if self.circuit_breaker:
                self.circuit_breaker.record_failure()
            raise

    async def generate(
        self,
        messages: List[Dict[str, str]],
        temperature: Optional[float] = None,
        max_tokens: Optional[int] = None
    ) -> str:
        """
        Generate complete answer (non-streaming).

        Use for batch processing or when streaming isn't needed.
        """
        full_response = ""
        async for chunk in self.generate_streaming(messages, temperature, max_tokens):
            full_response += chunk

        return full_response

    async def generate_with_retry(
        self,
        messages: List[Dict[str, str]],
        max_retries: int = 3,
        backoff_factor: float = 2.0
    ) -> str:
        """
        Generate with exponential backoff retry.

        WHY: Handles transient API failures (rate limits, 503 errors).
        """
        for attempt in range(max_retries):
            try:
                return await self.generate(messages)
            except (openai.RateLimitError, openai.APIError) as e:
                if attempt == max_retries - 1:
                    raise  # Last attempt, propagate error

                wait_time = backoff_factor ** attempt
                logger.warning(
                    f"API error, retrying in {wait_time}s (attempt {attempt+1}/{max_retries}): {e}"
                )
                await asyncio.sleep(wait_time)
```

### 3. Circuit Breaker Pattern

Prevents cascading failures when OpenAI API is down:

```python
# services/circuit_breaker.py
from enum import Enum
import time
from loguru import logger

class CircuitState(Enum):
    CLOSED = "closed"      # Normal operation
    OPEN = "open"          # Failing, reject requests
    HALF_OPEN = "half_open"  # Testing if service recovered

class CircuitBreaker:
    """
    Circuit breaker pattern for external API calls.

    States:
    - CLOSED: Normal operation, requests pass through
    - OPEN: Too many failures, reject requests immediately (fail fast)
    - HALF_OPEN: Allow limited requests to test recovery

    WHY: Prevents cascading failures when OpenAI API is down.
    Instead of waiting for timeouts (30s+), fail fast (100ms).
    """

    def __init__(
        self,
        failure_threshold: int = 5,  # Open after N failures
        recovery_timeout: int = 60,  # Retry after N seconds
        half_open_max_requests: int = 3  # Test with N requests
    ):
        self.failure_threshold = failure_threshold
        self.recovery_timeout = recovery_timeout
        self.half_open_max_requests = half_open_max_requests

        self.state = CircuitState.CLOSED
        self.failure_count = 0
        self.last_failure_time = None
        self.half_open_requests = 0

    def allow_request(self) -> bool:
        """
        Check if request should be allowed.

        Returns:
            True if request can proceed, False if should be rejected
        """
        if self.state == CircuitState.CLOSED:
            return True

        if self.state == CircuitState.OPEN:
            # Check if recovery timeout elapsed
            if time.time() - self.last_failure_time >= self.recovery_timeout:
                logger.info("Circuit breaker: Transitioning OPEN → HALF_OPEN")
                self.state = CircuitState.HALF_OPEN
                self.half_open_requests = 0
                return True
            else:
                # Still in timeout, reject request
                return False

        if self.state == CircuitState.HALF_OPEN:
            # Allow limited requests to test recovery
            if self.half_open_requests < self.half_open_max_requests:
                self.half_open_requests += 1
                return True
            else:
                return False

    def record_success(self):
        """Record successful request."""
        if self.state == CircuitState.HALF_OPEN:
            # Recovery confirmed, close circuit
            logger.info("Circuit breaker: Recovery confirmed, HALF_OPEN → CLOSED")
            self.state = CircuitState.CLOSED
            self.failure_count = 0
            self.half_open_requests = 0

        elif self.state == CircuitState.CLOSED:
            # Reset failure count on success
            if self.failure_count > 0:
                self.failure_count = 0

    def record_failure(self):
        """Record failed request."""
        self.failure_count += 1
        self.last_failure_time = time.time()

        if self.state == CircuitState.HALF_OPEN:
            # Recovery failed, reopen circuit
            logger.warning("Circuit breaker: Recovery failed, HALF_OPEN → OPEN")
            self.state = CircuitState.OPEN
            self.failure_count = 0

        elif self.state == CircuitState.CLOSED:
            # Check if threshold exceeded
            if self.failure_count >= self.failure_threshold:
                logger.error(
                    f"Circuit breaker: Failure threshold reached "
                    f"({self.failure_count}/{self.failure_threshold}), CLOSED → OPEN"
                )
                self.state = CircuitState.OPEN

    def get_state(self) -> CircuitState:
        """Get current circuit state."""
        return self.state
```

### 4. Conversation Manager

Manages conversation history with sliding window:

```python
# services/conversation_manager.py
from typing import List, Dict
import tiktoken
from loguru import logger

class ConversationManager:
    """
    Manages conversation history with sliding window and token budgeting.
    """

    def __init__(
        self,
        max_history_messages: int = 10,  # Max messages to keep
        max_history_tokens: int = 2000   # Max tokens for history
    ):
        self.max_history_messages = max_history_messages
        self.max_history_tokens = max_history_tokens
        self.tokenizer = tiktoken.get_encoding("cl100k_base")

    def add_message(
        self,
        conversation_history: List[Dict[str, str]],
        role: str,
        content: str
    ) -> List[Dict[str, str]]:
        """
        Add message to conversation history.

        Args:
            conversation_history: Existing history
            role: "user" or "assistant"
            content: Message content

        Returns:
            Updated history (may be trimmed)
        """
        new_message = {"role": role, "content": content}
        updated_history = conversation_history + [new_message]

        # Trim if needed
        updated_history = self._trim_history(updated_history)

        return updated_history

    def _trim_history(self, history: List[Dict[str, str]]) -> List[Dict[str, str]]:
        """
        Trim history to fit message count and token budget.

        Strategy:
        1. Keep last N messages (sliding window)
        2. If still over token budget, remove oldest pairs
        """
        # Trim by message count
        if len(history) > self.max_history_messages:
            history = history[-self.max_history_messages:]

        # Trim by token budget
        while self._count_history_tokens(history) > self.max_history_tokens:
            if len(history) <= 2:  # Keep at least one turn
                break
            # Remove oldest user+assistant pair
            history = history[2:]

        return history

    def _count_history_tokens(self, history: List[Dict[str, str]]) -> int:
        """Count tokens in conversation history."""
        total = 0
        for message in history:
            total += len(self.tokenizer.encode(message["content"]))
        return total

    def get_last_user_message(self, history: List[Dict[str, str]]) -> str:
        """Extract last user message (for query expansion)."""
        for message in reversed(history):
            if message["role"] == "user":
                return message["content"]
        return ""
```

### 5. Response Post-Processor

Parses and formats LLM responses:

```python
# services/response_processor.py
import re
from typing import List, Tuple
from loguru import logger

class ResponseProcessor:
    """
    Post-processes LLM responses for display.
    Extracts citations, calculates confidence, formats markdown.
    """

    def process_response(self, raw_response: str) -> dict:
        """
        Process raw LLM response.

        Args:
            raw_response: Raw text from GPT API

        Returns:
            Processed response with citations, confidence, formatted text
        """
        # Extract citations
        citations = self._extract_citations(raw_response)

        # Calculate confidence score
        confidence = self._calculate_confidence(raw_response, citations)

        # Format markdown
        formatted_text = self._format_markdown(raw_response)

        # Detect "I don't know" responses
        is_uncertain = self._is_uncertain_response(raw_response)

        logger.info(
            f"Processed response: {len(citations)} citations, "
            f"confidence={confidence:.2f}, uncertain={is_uncertain}"
        )

        return {
            "text": formatted_text,
            "citations": citations,
            "confidence": confidence,
            "is_uncertain": is_uncertain,
            "raw_text": raw_response
        }

    def _extract_citations(self, text: str) -> List[int]:
        """
        Extract [Source N] citations from response.

        Returns:
            List of source numbers cited
        """
        # Regex: [Source N] or [Source 1, 2, 3]
        pattern = r'\[Source\s+(\d+(?:\s*,\s*\d+)*)\]'
        matches = re.findall(pattern, text, re.IGNORECASE)

        citations = set()
        for match in matches:
            # Handle comma-separated numbers
            numbers = [int(n.strip()) for n in match.split(',')]
            citations.update(numbers)

        return sorted(list(citations))

    def _calculate_confidence(self, text: str, citations: List[int]) -> float:
        """
        Calculate confidence score based on heuristics.

        Heuristics:
        - Has citations: +0.4
        - Multiple citations: +0.2
        - No uncertain phrases: +0.2
        - Specific details (numbers, steps): +0.2

        Returns:
            Confidence score 0-1
        """
        score = 0.0

        # Has citations
        if len(citations) > 0:
            score += 0.4
            if len(citations) >= 2:
                score += 0.2

        # No uncertain phrases
        uncertain_phrases = [
            "i don't know", "not sure", "might", "possibly",
            "i don't have", "unclear", "i cannot"
        ]
        text_lower = text.lower()
        if not any(phrase in text_lower for phrase in uncertain_phrases):
            score += 0.2

        # Contains specific details (numbers, steps, URLs)
        if re.search(r'\d+', text) or "http" in text or "step" in text_lower:
            score += 0.2

        return min(score, 1.0)

    def _is_uncertain_response(self, text: str) -> bool:
        """Check if response indicates uncertainty."""
        uncertain_phrases = [
            "i don't have enough information",
            "i cannot answer",
            "i don't know",
            "connect you with a human agent",
            "contact support"
        ]
        text_lower = text.lower()
        return any(phrase in text_lower for phrase in uncertain_phrases)

    def _format_markdown(self, text: str) -> str:
        """
        Format markdown for display.

        Currently a pass-through (assume LLM generates valid markdown).
        Future: Sanitize HTML, add CSS classes, etc.
        """
        return text.strip()
```

### 6. Complete Generation Service

Orchestrates the full generation pipeline:

```python
# services/generation_service.py
from typing import AsyncGenerator, List, Dict, Optional
from loguru import logger

class GenerationService:
    """
    Complete generation pipeline orchestration.
    """

    def __init__(
        self,
        gpt_client,
        prompt_builder,
        conversation_manager,
        response_processor
    ):
        self.gpt = gpt_client
        self.prompt_builder = prompt_builder
        self.conversation = conversation_manager
        self.processor = response_processor

    async def generate_answer(
        self,
        user_query: str,
        context: str,
        conversation_history: Optional[List[Dict[str, str]]] = None
    ) -> dict:
        """
        Generate complete answer (non-streaming).

        Args:
            user_query: User's question
            context: Retrieved context from RAG pipeline
            conversation_history: Previous messages

        Returns:
            Processed response with text, citations, confidence
        """
        # Build prompt
        messages = self.prompt_builder.build_chat_prompt(
            user_query=user_query,
            context=context,
            conversation_history=conversation_history or []
        )

        # Generate
        raw_response = await self.gpt.generate_with_retry(messages)

        # Process response
        processed = self.processor.process_response(raw_response)

        # Update conversation history
        updated_history = self.conversation.add_message(
            conversation_history or [],
            role="user",
            content=user_query
        )
        updated_history = self.conversation.add_message(
            updated_history,
            role="assistant",
            content=raw_response
        )

        processed["conversation_history"] = updated_history

        return processed

    async def generate_answer_streaming(
        self,
        user_query: str,
        context: str,
        conversation_history: Optional[List[Dict[str, str]]] = None
    ) -> AsyncGenerator[str, None]:
        """
        Generate answer with streaming (yields chunks).

        For real-time UI updates.
        """
        # Build prompt
        messages = self.prompt_builder.build_chat_prompt(
            user_query=user_query,
            context=context,
            conversation_history=conversation_history or []
        )

        # Stream response
        full_response = ""
        async for chunk in self.gpt.generate_streaming(messages):
            full_response += chunk
            yield chunk

        # Post-process complete response (for metrics/logging)
        processed = self.processor.process_response(full_response)
        logger.info(
            f"Streaming generation complete: {len(full_response)} chars, "
            f"confidence={processed['confidence']:.2f}"
        )
```

## Production Tips

### 1. Prompt Versioning

Track prompt changes for A/B testing and rollbacks:

```python
# services/prompt_versions.py
from enum import Enum

class PromptVersion(Enum):
    V1_BASELINE = "v1_baseline"
    V2_MORE_EXAMPLES = "v2_more_examples"
    V3_STRICTER_GROUNDING = "v3_stricter_grounding"

# Store prompts in database/config
PROMPT_TEMPLATES = {
    PromptVersion.V1_BASELINE: "You are a helpful...",
    PromptVersion.V2_MORE_EXAMPLES: "You are a helpful...",
    # ...
}

# Assign users to versions (A/B testing)
def get_prompt_version(user_id: str) -> PromptVersion:
    # Hash user_id to assign to group
    if hash(user_id) % 3 == 0:
        return PromptVersion.V2_MORE_EXAMPLES
    else:
        return PromptVersion.V1_BASELINE

# Log version with each request for analysis
logger.info("generation_request", user_id=user_id, prompt_version=version)
```

### 2. Cost Monitoring

Track OpenAI API costs per request:

```python
# utils/cost_tracking.py
import tiktoken

# Pricing (as of 2026, check OpenAI pricing page)
PRICING = {
    "gpt-4o": {"input": 5.00 / 1_000_000, "output": 15.00 / 1_000_000},
    "gpt-4o-mini": {"input": 0.15 / 1_000_000, "output": 0.60 / 1_000_000},
}

def calculate_cost(model: str, input_tokens: int, output_tokens: int) -> float:
    """Calculate cost for API call."""
    pricing = PRICING.get(model, PRICING["gpt-4o-mini"])
    cost = (input_tokens * pricing["input"]) + (output_tokens * pricing["output"])
    return cost

# Log cost per request
logger.info(
    "api_cost",
    model=model,
    input_tokens=input_tokens,
    output_tokens=output_tokens,
    cost=cost
)

# Aggregate daily costs in monitoring dashboard
```

**Model Selection Strategy:**
- **GPT-4o**: Higher quality, use for complex queries or when confidence <0.7
- **GPT-4o-mini**: 90% quality at 15% cost, use as default
- Dynamic routing: Start with mini, escalate to 4o if uncertain

### 3. Prompt A/B Testing Framework

```python
# tests/prompt_ab_test.py
import pytest
from services.generation_service import GenerationService

@pytest.fixture
def test_queries():
    return [
        "How do I reset my password?",
        "What's included in the Pro plan?",
        "Can I get a refund after 60 days?",
        # ... 50+ queries covering common scenarios
    ]

async def test_prompt_versions(test_queries):
    """
    Compare prompt versions on test set.

    Metrics:
    - Citation rate (% of responses with sources)
    - "I don't know" rate (when appropriate)
    - Avg confidence score
    - Manual quality rating (sample 20 responses)
    """
    results = {}

    for version in [PromptVersion.V1, PromptVersion.V2]:
        generation_service = GenerationService(prompt_version=version)

        citations_count = 0
        idk_count = 0
        confidence_scores = []

        for query in test_queries:
            response = await generation_service.generate_answer(
                query, context="...", history=[]
            )

            if response["citations"]:
                citations_count += 1
            if response["is_uncertain"]:
                idk_count += 1
            confidence_scores.append(response["confidence"])

        results[version] = {
            "citation_rate": citations_count / len(test_queries),
            "idk_rate": idk_count / len(test_queries),
            "avg_confidence": sum(confidence_scores) / len(confidence_scores)
        }

    # Compare and log
    print(results)
```

### 4. Streaming Response Timeout

Handle slow API responses:

```python
# In GPTClient.generate_streaming
import asyncio

async def generate_streaming_with_timeout(self, messages, timeout=30):
    """Streaming with timeout per chunk."""
    try:
        async with asyncio.timeout(timeout):
            async for chunk in self.generate_streaming(messages):
                yield chunk
    except asyncio.TimeoutError:
        logger.error("Streaming timeout exceeded")
        yield "\n\n[Response timeout - please try again]"
```

## Interview Q&A

### Q1: How do you prevent hallucination in RAG systems? What specific prompt engineering techniques work?

**Answer:**

Hallucination prevention requires **multi-layered defense**:

**1. System Prompt Grounding (Most Important):**
```
"Answer ONLY from the provided context. If the context doesn't contain
the answer, say 'I don't have enough information' instead of guessing."
```

**WHY it works:** Explicit instruction is the primary control mechanism. LLMs follow instructions well when they're clear and repeated.

**2. Few-Shot Examples:**
Show 2-3 examples of:
- Good answers with proper citations
- "I don't know" responses when context lacks info
- Avoiding speculation

**WHY it works:** In-context learning reinforces desired behavior. LLMs pattern-match to examples.

**3. Low Temperature (0.1-0.3):**
Reduces randomness → more deterministic outputs → less creative fabrication.

**4. Citation Requirements:**
Force model to cite sources: "According to [Source 2], ..."

**WHY it works:** Citing creates accountability. Model must point to specific context, harder to fabricate.

**5. Context Window Management:**
Don't let conversation history dominate retrieved context. Priority: Context > History.

**What DOESN'T work well:**
- Post-generation fact-checking (slow, unreliable)
- Keyword filtering (too brittle)
- Confidence scores alone (LLMs overconfident)

**Production metrics to monitor:**
- **Citation rate**: % of responses with sources (target: >80%)
- **"I don't know" rate**: Should be >10% (if 0%, model is guessing)
- **User feedback**: "Was this answer accurate?" thumbs up/down

**Red flags:**
- Citation rate suddenly drops → prompt drift
- Zero "I don't know" responses → model fabricating

---

### Q2: Explain streaming in the context of LLM APIs. How does it improve UX and what are the implementation challenges?

**Answer:**

**Streaming = Server-Sent Events (SSE)** where API yields response chunks as tokens are generated, instead of waiting for complete response.

**UX Impact:**

Non-streaming (traditional):
```
[User waits 5-10 seconds...]
[Complete answer appears]
```

Streaming:
```
[User waits <1 second]
[Answer starts appearing word-by-word, like typing]
[User reads while rest generates]
```

**Perceived latency improvement:** ~80% reduction. User sees progress in 800ms instead of 8 seconds.

**Implementation (AsyncGenerator in Python):**

```python
async def generate_streaming(messages):
    stream = await openai_client.chat.completions.create(
        model="gpt-4o",
        messages=messages,
        stream=True  # Enable streaming
    )

    async for chunk in stream:
        if chunk.choices[0].delta.content:
            yield chunk.choices[0].delta.content  # Yield each token
```

**Frontend (FastAPI SSE):**

```python
from fastapi.responses import StreamingResponse

@app.post("/chat/stream")
async def chat_stream(request: ChatRequest):
    async def event_generator():
        async for chunk in generation_service.generate_streaming(...):
            yield f"data: {json.dumps({'chunk': chunk})}\n\n"

    return StreamingResponse(event_generator(), media_type="text/event-stream")
```

**Challenges:**

1. **Error Handling Mid-Stream:**
   - Can't send HTTP 500 after headers sent
   - Solution: Yield error message as data chunk
   ```python
   try:
       async for chunk in stream:
           yield chunk
   except Exception as e:
       yield f"\n\n[Error: {e}]"
   ```

2. **Circuit Breaker Interaction:**
   - Circuit breaker must fail before streaming starts
   - Check state before creating stream

3. **Cost Tracking:**
   - Can't count output tokens until stream completes
   - Solution: Accumulate chunks, count at end

4. **Retries Don't Work:**
   - Can't retry mid-stream (user already saw partial response)
   - Solution: Retry at connection level, not mid-generation

5. **Frontend Complexity:**
   - Must handle SSE connection, reconnection, buffering
   - Libraries: `EventSource` (JS), `httpx-sse` (Python client)

**When NOT to use streaming:**
- Batch processing (no human watching)
- When you need full response before proceeding (e.g., classification)
- Very short responses (<50 tokens, overhead not worth it)

**Production tips:**
- Timeout per chunk (if no data for 30s, close stream)
- Heartbeat messages to keep connection alive
- Client-side reconnection with exponential backoff

---

### Q3: How do you handle token budgeting across prompt, context, history, and response? What gets priority when you exceed the model's context window?

**Answer:**

**Context Window Example (GPT-4o):**
- Total: 128K tokens
- Budget allocation:
  - System prompt: 500 tokens (fixed)
  - Few-shot examples: 800 tokens (optional)
  - Retrieved context: 4000 tokens (critical)
  - Conversation history: 2000 tokens (variable)
  - User query: 200 tokens (variable)
  - Response: 500 tokens (max_tokens setting)
  - **Buffer**: 1000 tokens (safety margin)

**Total:** ~9K tokens (leaves 119K for growth)

**Priority When Exceeding Budget:**

```
1. System Prompt (ALWAYS KEEP) - Core instructions
2. User Query (ALWAYS KEEP) - Current question
3. Retrieved Context (HIGHEST PRIORITY) - RAG core
4. Conversation History (TRIM OLDEST FIRST) - Nice to have
5. Few-Shot Examples (REMOVE IF NEEDED) - Optional
```

**Trimming Strategy:**

```python
def trim_to_budget(messages, max_tokens=8000):
    current = count_tokens(messages)

    if current <= max_tokens:
        return messages

    # Step 1: Remove few-shot examples
    messages_no_examples = [system, context, history, query]
    if count_tokens(messages_no_examples) <= max_tokens:
        return messages_no_examples

    # Step 2: Trim conversation history (oldest first)
    while len(history) > 2 and count_tokens(messages) > max_tokens:
        history = history[2:]  # Remove oldest user+assistant pair

    # Step 3: Trim context (LAST RESORT, degrades quality)
    while count_tokens(messages) > max_tokens:
        context = context[:-100]  # Remove last 100 chars

    return messages
```

**Why Context > History?**
- RAG's value is grounding in knowledge base
- Without context, LLM will hallucinate
- History provides continuity but isn't critical for single query

**Token Counting:**

```python
import tiktoken

tokenizer = tiktoken.get_encoding("cl100k_base")  # GPT-4 tokenizer

def count_tokens(text: str) -> int:
    return len(tokenizer.encode(text))
```

**Production Monitoring:**

Track these metrics:
- **% of requests requiring trimming** (target: <5%)
- **Average tokens used per component** (identify bloat)
- **Context trimming events** (red flag, hurts quality)

**Advanced Optimization:**

1. **Summarize Long History:**
   ```python
   # After 10 messages, summarize to 1-2 sentences
   if len(history) > 10:
       summary = await llm.summarize(history[:8])
       history = [{"role": "system", "content": f"Previous context: {summary}"}] + history[-2:]
   ```

2. **Compress Context:**
   - Use LLMLingua to compress context by 50% while preserving meaning
   - Trade compression latency for token savings

3. **Dynamic Context Budget:**
   - Simple query ("What's your email?") → less context
   - Complex query → more context
   - Use query complexity classifier

**Gotcha:** OpenAI chat format has overhead (4 tokens per message for role/structure). Account for this:
```python
def count_chat_tokens(messages):
    total = 0
    for msg in messages:
        total += 4  # Format overhead
        total += count_tokens(msg["content"])
    return total
```

---

### Q4: What's your strategy for model selection (GPT-4o vs GPT-4o-mini)? How do you balance cost and quality?

**Answer:**

**Cost vs Quality Trade-off:**

| Model | Cost (per 1M tokens) | Quality | Latency |
|-------|---------------------|---------|---------|
| GPT-4o | $5 input / $15 output | 100% (baseline) | ~3s |
| GPT-4o-mini | $0.15 input / $0.60 output | ~90% | ~2s |

**Cost ratio:** GPT-4o-mini is **~15-20x cheaper** with only **~10% quality drop**.

**Default Strategy: Start with GPT-4o-mini**

Use mini for 80% of queries, escalate to 4o for:
1. Low retrieval confidence (<0.7)
2. Complex multi-step reasoning queries
3. User explicitly requests escalation ("That doesn't help")

**Implementation:**

```python
async def select_model(query: str, retrieval_results: List, history: List) -> str:
    """Dynamic model selection based on query complexity."""

    # Check retrieval quality
    avg_score = sum(r["score"] for r in retrieval_results) / len(retrieval_results)
    if avg_score < 0.7:
        return "gpt-4o"  # Poor retrieval, need better reasoning

    # Check query complexity (heuristics)
    complexity_signals = [
        len(query.split()) > 30,  # Long query
        "why" in query.lower() or "how does" in query.lower(),  # Reasoning
        len(history) > 10,  # Long conversation
    ]
    if sum(complexity_signals) >= 2:
        return "gpt-4o"

    # Default: mini
    return "gpt-4o-mini"
```

**A/B Testing Results (Production Data):**

Our customer support SaaS:
- **90% of queries work well with mini** (user satisfaction 4.3/5)
- **10% benefit from 4o** (complex billing questions, multi-step troubleshooting)
- **Cost savings:** 70% reduction ($15K → $4.5K/month) with minimal quality impact

**Monitoring:**

Track model usage + user feedback:
```python
logger.info(
    "generation_complete",
    model=model,
    query_complexity=complexity_score,
    retrieval_score=avg_retrieval_score,
    user_rating=rating  # 1-5 stars
)
```

**Red flags:**
- Mini rating drops below 4.0 → queries too complex, increase 4o usage
- 4o usage >30% → over-escalating, tune threshold

**Advanced: Learned Model Router**

Train small classifier on query → model → user rating data:
```python
# Features: query length, question type, retrieval scores, conversation length
# Target: best model for query

router = XGBoostClassifier(...)
router.fit(features, best_model_labels)

# At inference
model = router.predict(query_features)
```

**Cost Monitoring Dashboard:**

Track daily:
- Requests by model
- Cost per model
- Total cost
- Cost per query
- User satisfaction by model

**Budget Alerts:**
- Alert if daily cost exceeds $500
- Alert if 4o usage >40% (unusual spike)

---

### Q5: How do you version and test prompts in production? What's your deployment process for prompt changes?

**Answer:**

**Problem:** Prompts are code. Changes can break production. Need version control, testing, and gradual rollout.

**1. Prompt Version Control:**

```python
# prompts/system_prompts.py
from enum import Enum

class PromptVersion(Enum):
    V1_0_BASELINE = "v1.0_baseline"
    V1_1_STRICTER_GROUNDING = "v1.1_stricter_grounding"
    V2_0_FEW_SHOT = "v2.0_few_shot"

SYSTEM_PROMPTS = {
    PromptVersion.V1_0_BASELINE: """
        You are a helpful customer support assistant...
    """,
    PromptVersion.V1_1_STRICTER_GROUNDING: """
        You are a helpful customer support assistant...
        CRITICAL: Never use information outside the provided context...
    """,
    # ...
}

# Git commit each change with description
# Tag releases: v1.0, v1.1, v2.0
```

**2. Offline Testing (Before Production):**

```python
# tests/test_prompts.py
import pytest

@pytest.fixture
def test_cases():
    return [
        {
            "query": "How do I reset my password?",
            "context": "...",
            "expected_keywords": ["settings", "security", "reset"],
            "should_cite": True
        },
        {
            "query": "What's your company's secret sauce?",  # Out of scope
            "context": "",
            "expected_keywords": ["don't have information"],
            "should_cite": False
        },
        # ... 50+ test cases
    ]

async def test_prompt_version(test_cases):
    """Test new prompt version on standardized test set."""

    prompt_builder = PromptBuilder(version=PromptVersion.V1_1)
    generation_service = GenerationService(...)

    results = []
    for case in test_cases:
        response = await generation_service.generate(
            case["query"], case["context"]
        )

        # Check expectations
        has_keywords = all(
            kw in response["text"].lower()
            for kw in case["expected_keywords"]
        )
        has_citations = len(response["citations"]) > 0

        results.append({
            "passed": has_keywords and (has_citations == case["should_cite"]),
            "response": response
        })

    pass_rate = sum(r["passed"] for r in results) / len(results)
    assert pass_rate >= 0.85, f"Pass rate {pass_rate} below threshold"
```

**3. A/B Testing (Gradual Rollout):**

```python
# services/prompt_router.py
def get_prompt_version(user_id: str) -> PromptVersion:
    """Route users to prompt versions for A/B test."""

    # Hash user_id to assign to group (stable assignment)
    hash_val = hash(user_id) % 100

    # Rollout plan:
    # - 80% on v1.0 (baseline)
    # - 20% on v1.1 (new version)
    if hash_val < 80:
        return PromptVersion.V1_0_BASELINE
    else:
        return PromptVersion.V1_1_STRICTER_GROUNDING
```

**4. Production Monitoring:**

Track metrics per prompt version:

```python
# Log every generation with version
logger.info(
    "generation_event",
    prompt_version=version,
    user_id=user_id,
    query=query,
    citations=len(citations),
    confidence=confidence,
    user_rating=rating  # Thumbs up/down
)

# Dashboard queries:
# - User rating by prompt version
# - Citation rate by version
# - "I don't know" rate by version
```

**5. Decision Criteria:**

After 7 days of A/B test (minimum 1000 queries per version):

| Metric | V1.0 Baseline | V1.1 New | Change |
|--------|--------------|----------|--------|
| User rating | 4.2/5 | 4.4/5 | **+4.8%** |
| Citation rate | 75% | 82% | **+7%** |
| "I don't know" rate | 8% | 12% | +4% (good!) |
| Avg confidence | 0.72 | 0.78 | **+8%** |

**Decision:** V1.1 wins → Roll out to 100%

**6. Rollback Plan:**

If new version causes issues:

```python
# Instant rollback via config change (no code deploy)
PROMPT_VERSION_OVERRIDE = PromptVersion.V1_0_BASELINE  # Emergency override

# All users immediately get baseline version
# Fix issue, re-test, re-rollout
```

**7. Deployment Process:**

```
1. Developer creates new prompt version in code
2. Commit to git with description
3. Write offline tests, ensure >85% pass rate
4. Deploy to staging, manual QA
5. A/B test on 20% of production traffic
6. Monitor for 7 days, review metrics
7. If successful: Roll out to 100%
8. If failed: Rollback, iterate
```

**Advanced: Prompt Optimization:**

Use DSPy or similar frameworks to auto-optimize prompts:

```python
# Define optimization goal
def evaluation_metric(prompt, test_cases):
    score = 0
    for case in test_cases:
        response = generate(prompt, case["query"], case["context"])
        if check_quality(response, case["expected"]):
            score += 1
    return score / len(test_cases)

# DSPy optimizes prompt wording to maximize metric
optimized_prompt = dspy.optimize(
    base_prompt,
    test_cases,
    metric=evaluation_metric
)
```

**Production Best Practices:**

1. **Never change prompts without testing** (treat like code)
2. **Version everything** (git tags, semantic versioning)
3. **A/B test major changes** (20% rollout first)
4. **Monitor continuously** (user ratings, citation rates)
5. **Document changes** (why changed, expected impact)
6. **Keep rollback ready** (config-based version override)

