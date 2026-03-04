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
