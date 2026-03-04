# Section 17: Advanced Improvements & Roadmap

## WHY Plan for Evolution

Your MVP RAG system handles basic Q&A, but competitive differentiation requires advanced features: agentic workflows that take actions (process refunds, update orders), multi-modal support (search product images, understand voice queries), sophisticated retrieval (hybrid search, knowledge graphs, parent-child chunking), and enterprise features (white-labeling, fine-tuning per tenant, advanced analytics). This roadmap guides your evolution from MVP to market leader across three phases: Foundation (MVP), Growth (advanced retrieval/generation), and Enterprise (business features). Each phase builds on the previous, balancing technical innovation with business value.

## WHAT to Build: Three-Phase Roadmap

### Phase 1: Foundation (Months 1-3) - MVP Launch
- Basic RAG: Query → Retrieve → Generate
- Single embedding model (MiniLM), basic chunking (500 tokens)
- Multi-tenancy with data isolation
- REST API, simple UI, feedback collection
- Core metrics: Latency, error rate, user feedback

**Business value**: Validate product-market fit, onboard first 10-50 customers

### Phase 2: Growth (Months 4-9) - Advanced Retrieval & Generation
- **Advanced retrieval**: Hybrid search (vector + keyword), cross-encoder reranking, parent-child chunking
- **Advanced generation**: Self-RAG (model evaluates own outputs), chain-of-thought reasoning, multi-step answers
- **Agentic features**: Tool use (API calls, order lookups), multi-step workflows (LangGraph)
- **Multi-modal**: Image search, document OCR, video transcripts
- **Observability**: LangSmith/Langfuse integration, automated eval pipelines, A/B testing framework

**Business value**: Improve quality metrics (20-30% deflection increase), support 100-500 customers, competitive differentiation

### Phase 3: Enterprise (Months 10-18) - Business Features
- **White-labeling**: Custom domains, branded UI, tenant-specific prompts
- **Integrations**: Shopify, WordPress, Zendesk, Slack webhooks
- **Per-tenant fine-tuning**: Custom models for large customers
- **Advanced analytics**: Drift detection, topic modeling, customer journey tracking
- **Enterprise security**: SSO (SAML/OIDC), SOC 2 compliance, data residency
- **Voice support**: Speech-to-text, text-to-speech, phone integration

**Business value**: Land enterprise customers (1000+ employee companies), $50K+ ARR contracts, 80%+ retention

## HOW to Implement: Phase-by-Phase Details

### Phase 2: Agentic RAG with LangGraph

**WHY**: Simple RAG can't handle actions ("Refund my order") or multi-step reasoning ("Compare plans and upgrade me to Pro"). Agentic RAG gives your system decision-making capability.

**WHAT**: LangGraph is a framework for building stateful, multi-agent workflows. Agent decides actions (retrieve, call API, generate), executes them, evaluates results, and repeats until task complete.

**Example Workflow: Order Return Initiation**:
```
User: "I want to return order #12345"
  ↓
Agent: [Retrieve] Search knowledge base for return policy
  ↓
Context: "Returns within 30 days, need order status"
  ↓
Agent: [API Call] Check order date via get_order(12345)
  ↓
API Response: {order_date: "2026-01-15", status: "delivered"}
  ↓
Agent: [Reasoning] Order is 26 days old, eligible for return
  ↓
Agent: [API Call] Initiate return via create_return_label(12345)
  ↓
API Response: {return_label_url: "https://...", ticket_id: "RET-789"}
  ↓
Agent: [Generate] "I've initiated your return for order #12345. Download
                   your return label here: [link]. Ticket: RET-789."
```

**Implementation**:
```python
# tools/agent_tools.py - Define tools for agent
from langchain.tools import Tool
from typing import Annotated

@Tool
def search_knowledge_base(query: Annotated[str, "User query"]) -> str:
    """Search knowledge base for relevant information."""
    chunks = qdrant_client.search(
        collection_name=f"tenant_{tenant_id}",
        query_vector=embed_query(query),
        limit=5
    )
    return "\n".join([c.payload["text"] for c in chunks])

@Tool
def get_order_status(order_id: Annotated[str, "Order ID"]) -> dict:
    """Get order details from backend API."""
    response = httpx.get(f"{BACKEND_API}/orders/{order_id}")
    return response.json()

@Tool
def create_return_label(order_id: Annotated[str, "Order ID"]) -> dict:
    """Initiate return and generate label."""
    response = httpx.post(f"{BACKEND_API}/returns", json={"order_id": order_id})
    return response.json()

# Agent definition using LangGraph
from langgraph.prebuilt import create_react_agent
from langchain_openai import ChatOpenAI

def create_support_agent(tenant_id: str):
    llm = ChatOpenAI(model="gpt-4o", temperature=0)

    tools = [
        search_knowledge_base,
        get_order_status,
        create_return_label,
        # Add more tools: update_address, cancel_order, etc.
    ]

    system_prompt = """You are a helpful customer support agent.
    Use the provided tools to help users with their requests.
    - First search knowledge base for policies/procedures
    - Then call APIs to check status or take actions
    - Always confirm with user before taking irreversible actions
    - Cite sources and provide ticket numbers"""

    agent = create_react_agent(
        llm,
        tools,
        state_modifier=system_prompt
    )

    return agent

# Usage in API endpoint
@app.post("/api/v1/agent/query")
async def agent_query(request: QueryRequest, tenant_id: str = Depends(get_tenant_id)):
    agent = create_support_agent(tenant_id)

    # Run agent with streaming
    events = agent.stream(
        {"messages": [("user", request.query)]},
        stream_mode="values"
    )

    response_chunks = []
    async for event in events:
        # Stream tool calls and responses to user
        if "agent" in event:
            response_chunks.append(event["agent"]["messages"][-1].content)

    return {"response": " ".join(response_chunks)}
```

**Challenges**:
1. **Reliability**: Agent may call wrong tool or loop infinitely. Solution: Max iterations (10), validate tool inputs, human-in-the-loop for critical actions.
2. **Cost**: 5-10 LLM calls per query (vs 1 in simple RAG). Solution: Use GPT-4o-mini for tool selection, GPT-4o only for final generation.
3. **Latency**: 10-30 seconds per query. Solution: Stream intermediate steps ("Checking your order..."), set timeouts.

### Phase 2: Multi-Modal Support (Images)

**WHY**: 30-40% of support queries involve images ("Does this product come in blue?", "Why doesn't this part fit?"). Text-only RAG can't handle visual questions.

**WHAT**: Index product images with CLIP embeddings (768-dim vectors capturing visual+text semantics), search by image similarity, generate answers using GPT-4o vision.

**Implementation**:
```python
# services/multimodal_service.py - Image indexing and search
from transformers import CLIPProcessor, CLIPModel
import torch

# Load CLIP model (one-time at startup)
clip_model = CLIPModel.from_pretrained("openai/clip-vit-base-patch32")
clip_processor = CLIPProcessor.from_pretrained("openai/clip-vit-base-patch32")

def embed_image(image_url: str) -> list[float]:
    """Generate CLIP embedding for image."""
    image = Image.open(requests.get(image_url, stream=True).raw)
    inputs = clip_processor(images=image, return_tensors="pt")

    with torch.no_grad():
        image_features = clip_model.get_image_features(**inputs)

    # Normalize and convert to list
    embedding = image_features / image_features.norm(dim=-1, keepdim=True)
    return embedding.squeeze().tolist()

def embed_text_for_image_search(query: str) -> list[float]:
    """Generate CLIP embedding for text query (searches images)."""
    inputs = clip_processor(text=[query], return_tensors="pt", padding=True)

    with torch.no_grad():
        text_features = clip_model.get_text_features(**inputs)

    embedding = text_features / text_features.norm(dim=-1, keepdim=True)
    return embedding.squeeze().tolist()

# Ingest product images
async def ingest_product_images(product_id: str, image_urls: list[str], tenant_id: str):
    """Index product images in Qdrant."""
    points = []

    for i, url in enumerate(image_urls):
        embedding = embed_image(url)
        points.append(PointStruct(
            id=f"{product_id}_img_{i}",
            vector=embedding,
            payload={
                "product_id": product_id,
                "image_url": url,
                "tenant_id": tenant_id,
                "type": "image"
            }
        ))

    qdrant_client.upsert(
        collection_name=f"tenant_{tenant_id}_images",
        points=points
    )

# Search images by text query
@app.post("/api/v1/search/images")
async def search_images(query: str, tenant_id: str = Depends(get_tenant_id)):
    """Search product images by text description."""
    query_embedding = embed_text_for_image_search(query)

    results = qdrant_client.search(
        collection_name=f"tenant_{tenant_id}_images",
        query_vector=query_embedding,
        limit=10,
        query_filter=Filter(
            must=[FieldCondition(key="tenant_id", match=MatchValue(value=tenant_id))]
        )
    )

    return {
        "images": [
            {
                "product_id": r.payload["product_id"],
                "image_url": r.payload["image_url"],
                "score": r.score
            }
            for r in results
        ]
    }

# Generate answer with GPT-4o vision
async def answer_visual_query(query: str, image_results: list[dict], tenant_id: str):
    """Use GPT-4o vision to answer query about images."""
    # Prepare image URLs for vision model
    image_contents = [
        {"type": "image_url", "image_url": {"url": img["image_url"]}}
        for img in image_results[:3]  # Top 3 images
    ]

    messages = [
        {
            "role": "system",
            "content": "You are a helpful product assistant. Answer questions about the provided product images."
        },
        {
            "role": "user",
            "content": [
                {"type": "text", "text": f"User question: {query}"},
                *image_contents
            ]
        }
    ]

    response = await openai_client.chat.completions.create(
        model="gpt-4o",
        messages=messages,
        max_tokens=500
    )

    return response.choices[0].message.content
```

**Use Cases**:
- "Show me blue sneakers" → Visual search returns blue sneaker images
- "Does this jacket come in red?" → Search images, GPT-4o vision checks colors
- "Compare these two products" → Pass both images to GPT-4o vision for comparison

**Cost**: CLIP inference is cheap (self-hosted GPU, ~$100/month for 1M images). GPT-4o vision is expensive ($0.01/image), cache results.

### Phase 2: Advanced Retrieval - Parent-Child Chunking

**WHY**: Fixed-size chunking (500 tokens) loses context. A chunk about "30-day return policy" may not include crucial detail from previous paragraph ("except for electronics").

**WHAT**: Store hierarchical chunks: Parent (full section/page, 2000 tokens) and children (sentences, 200 tokens). Retrieve children for precision, return parent for context.

**Implementation**:
```python
# services/parent_child_chunking.py
from typing import List, Tuple

def create_parent_child_chunks(document: str, doc_id: str) -> List[Tuple[str, str, dict]]:
    """Split document into parent sections and child sentences."""
    chunks = []

    # Split into sections (parents) by headers
    sections = split_by_headers(document)  # Returns [(title, content), ...]

    for section_idx, (section_title, section_content) in enumerate(sections):
        parent_id = f"{doc_id}_section_{section_idx}"

        # Parent chunk (full section, 2000 tokens max)
        parent_text = f"{section_title}\n{section_content}"
        parent_text = truncate(parent_text, 2000)

        # Child chunks (sentences, 200 tokens each)
        sentences = split_into_sentences(section_content)
        child_chunks = group_sentences(sentences, target_tokens=200)

        for child_idx, child_text in enumerate(child_chunks):
            child_id = f"{parent_id}_child_{child_idx}"

            chunks.append((
                child_id,
                child_text,
                {
                    "parent_id": parent_id,
                    "parent_text": parent_text,  # Store for context
                    "section_title": section_title,
                    "type": "child"
                }
            ))

    return chunks

# Retrieval strategy: Search children, return parents
async def retrieve_with_parent_context(query: str, tenant_id: str, top_k: int = 5):
    """Retrieve child chunks, but return parent context."""
    query_embedding = embed_query(query)

    # Search children (precise matches)
    child_results = qdrant_client.search(
        collection_name=f"tenant_{tenant_id}",
        query_vector=query_embedding,
        limit=top_k,
        query_filter=Filter(
            must=[
                FieldCondition(key="tenant_id", match=MatchValue(value=tenant_id)),
                FieldCondition(key="type", match=MatchValue(value="child"))
            ]
        )
    )

    # Return parent context for each child
    contexts = []
    for result in child_results:
        contexts.append({
            "child_text": result.payload["text"],  # Matched sentence
            "parent_text": result.payload["parent_text"],  # Full section context
            "section_title": result.payload["section_title"],
            "score": result.score
        })

    return contexts

# Use in LLM prompt
def build_prompt_with_parent_context(query: str, contexts: list[dict]) -> str:
    context_str = "\n\n".join([
        f"### {ctx['section_title']}\n{ctx['parent_text']}\n(Relevant excerpt: {ctx['child_text']})"
        for ctx in contexts
    ])

    return f"""Answer the question using the context below.
Each context includes a full section with the most relevant excerpt highlighted.

Context:
{context_str}

Question: {query}

Answer:"""
```

**Benefits**: 20-30% better answer quality (more context), maintains citation precision (know which sentence matched).

### Phase 2: Self-RAG (Retrieval-Aware Generation)

**WHY**: Simple RAG always retrieves and generates, even when retrieval is poor or unnecessary. Self-RAG makes the model self-reflect on retrieval quality.

**WHAT**: After retrieval, model evaluates: (1) "Is retrieval helpful?" (2) "Is retrieved content relevant?" (3) After generation: "Is answer supported by context?" Model can trigger re-retrieval or refuse to answer.

**Implementation**:
```python
# services/self_rag_service.py
async def self_rag_query(query: str, tenant_id: str) -> dict:
    """RAG with self-reflection on retrieval and generation."""

    # Step 1: Retrieve context
    chunks = await retrieve_context(query, tenant_id, top_k=10)

    # Step 2: Evaluate retrieval quality
    relevance_prompt = f"""Rate the relevance of these retrieved chunks to the query.
Query: {query}

Retrieved chunks:
{format_chunks(chunks[:5])}

Are these chunks relevant? Answer "YES" if at least 3 chunks are highly relevant,
"PARTIAL" if 1-2 are relevant, "NO" if none are relevant."""

    relevance_check = await openai_client.chat.completions.create(
        model="gpt-4o-mini",
        messages=[{"role": "user", "content": relevance_prompt}],
        max_tokens=10
    )

    relevance = relevance_check.choices[0].message.content.strip()

    # Step 3: Decide action based on relevance
    if relevance == "NO":
        # Try alternative retrieval (hybrid search, broader query)
        chunks = await hybrid_search(query, tenant_id, top_k=10)

        if not chunks:
            return {
                "answer": "I don't have enough information to answer that question.",
                "confidence": "low",
                "retrieval_quality": "poor"
            }

    elif relevance == "PARTIAL":
        # Use smaller chunk set (top 3) to reduce noise
        chunks = chunks[:3]

    # Step 4: Generate answer
    answer_prompt = build_rag_prompt(query, chunks)
    answer = await generate_answer(answer_prompt)

    # Step 5: Validate answer is grounded in context
    validation_prompt = f"""Is the following answer fully supported by the context?
Context:
{format_chunks(chunks)}

Answer:
{answer}

Respond "SUPPORTED" if the answer only contains information from context,
"UNSUPPORTED" if it contains information not in context (hallucination)."""

    validation = await openai_client.chat.completions.create(
        model="gpt-4o-mini",
        messages=[{"role": "user", "content": validation_prompt}],
        max_tokens=20
    )

    is_supported = validation.choices[0].message.content.strip()

    # Step 6: Return with confidence
    return {
        "answer": answer if is_supported == "SUPPORTED" else "I cannot provide a confident answer.",
        "confidence": "high" if is_supported == "SUPPORTED" else "low",
        "retrieval_quality": relevance.lower(),
        "chunks_used": len(chunks)
    }
```

**Tradeoff**: 2-3x more LLM calls (cost/latency), but 30-50% reduction in hallucinations. Use for high-stakes domains.

### Phase 3: White-Labeling & Tenant Customization

**WHY**: Enterprise customers want branded experiences (their logo, domain, colors) and custom prompts (their support tone, policies).

**Implementation**:
```python
# models/tenant_config.py - Store per-tenant customization
class TenantConfig(Document):
    tenant_id: str
    branding: dict = {
        "logo_url": str,
        "primary_color": str,
        "domain": str  # e.g., support.acmeco.com
    }
    custom_prompt: Optional[str] = None  # Override system prompt
    model_preference: str = "gpt-4o-mini"  # Per-tenant model
    features: dict = {
        "enable_agentic": bool,
        "enable_multimodal": bool,
        "enable_voice": bool
    }

    class Settings:
        name = "tenant_configs"

# Apply customization at runtime
async def get_tenant_config(tenant_id: str) -> TenantConfig:
    config = await TenantConfig.find_one({"tenant_id": tenant_id})
    return config or TenantConfig(tenant_id=tenant_id)  # Default

@app.post("/api/v1/query")
async def query_endpoint(
    request: QueryRequest,
    tenant_id: str = Depends(get_tenant_id)
):
    config = await get_tenant_config(tenant_id)

    # Use tenant's custom prompt if provided
    system_prompt = config.custom_prompt or DEFAULT_SYSTEM_PROMPT

    # Use tenant's preferred model
    model = config.model_preference

    # Build prompt and generate
    prompt = build_prompt(request.query, system_prompt, ...)
    answer = await generate_answer(prompt, model=model)

    return {"answer": answer}

# Custom domain routing (Nginx/ALB)
# support.acmeco.com → API with tenant_id=acmeco extracted from domain
```

### Phase 3: Platform Integrations

**WHY**: Customers use Shopify, WordPress, Zendesk. Native integrations reduce setup time from days to minutes.

**Shopify Integration** (Sync Products):
```python
# integrations/shopify_integration.py
import shopify

async def sync_shopify_products(tenant_id: str, shop_url: str, access_token: str):
    """Sync Shopify products to knowledge base."""
    shopify.ShopifyResource.set_site(shop_url)
    shopify.ShopifyResource.headers = {"X-Shopify-Access-Token": access_token}

    products = shopify.Product.find()

    for product in products:
        # Convert product to document
        doc_text = f"""
        Product: {product.title}
        Description: {product.body_html}
        Price: ${product.variants[0].price}
        In stock: {product.variants[0].inventory_quantity > 0}
        """

        # Ingest into RAG
        await ingest_document(
            tenant_id=tenant_id,
            document_id=f"shopify_product_{product.id}",
            text=doc_text,
            metadata={
                "source": "shopify",
                "product_id": str(product.id),
                "url": f"{shop_url}/products/{product.handle}"
            }
        )

    # Set up webhook for real-time updates
    webhook = shopify.Webhook()
    webhook.topic = "products/update"
    webhook.address = f"{API_BASE_URL}/webhooks/shopify/product_update"
    webhook.save()

# Webhook handler
@app.post("/webhooks/shopify/product_update")
async def shopify_product_update(request: Request):
    payload = await request.json()

    # Verify webhook signature (HMAC)
    verify_shopify_webhook(request.headers, payload)

    # Re-ingest updated product
    await sync_shopify_products(
        tenant_id=extract_tenant_from_shop(payload["shop"]),
        shop_url=payload["shop"],
        access_token=get_tenant_shopify_token(...)
    )

    return {"status": "success"}
```

**WordPress Integration** (Sync Posts/Pages):
```python
# integrations/wordpress_integration.py
async def sync_wordpress_content(tenant_id: str, wp_site_url: str, api_key: str):
    """Sync WordPress posts and pages."""
    # WordPress REST API
    headers = {"Authorization": f"Bearer {api_key}"}

    # Fetch posts
    posts_response = httpx.get(f"{wp_site_url}/wp-json/wp/v2/posts", headers=headers)
    posts = posts_response.json()

    for post in posts:
        await ingest_document(
            tenant_id=tenant_id,
            document_id=f"wp_post_{post['id']}",
            text=f"{post['title']['rendered']}\n{strip_html(post['content']['rendered'])}",
            metadata={"source": "wordpress", "url": post["link"]}
        )
```

### Phase 3: Voice Support

**WHY**: 40% of users prefer voice for support. Phone integration expands TAM to non-technical users.

**Implementation**:
```python
# services/voice_service.py - Speech-to-text and text-to-speech
from openai import OpenAI

async def handle_voice_query(audio_file: bytes, tenant_id: str) -> bytes:
    """Process voice query and return voice response."""

    # 1. Speech-to-text (Whisper API)
    transcription = await openai_client.audio.transcriptions.create(
        model="whisper-1",
        file=audio_file,
        language="en"
    )

    query_text = transcription.text

    # 2. RAG query
    answer = await rag_query(query_text, tenant_id)

    # 3. Text-to-speech
    speech_response = await openai_client.audio.speech.create(
        model="tts-1",
        voice="alloy",
        input=answer["response"]
    )

    return speech_response.content  # MP3 audio bytes

# Twilio integration for phone support
from twilio.twiml.voice_response import VoiceResponse

@app.post("/webhooks/twilio/voice")
async def twilio_voice_webhook(request: Request):
    """Handle incoming phone calls."""
    form = await request.form()
    caller = form.get("From")

    # Play greeting
    response = VoiceResponse()
    response.say("Hi! I'm your AI support assistant. What can I help with?")

    # Record user's question
    response.record(
        max_length=30,
        action="/webhooks/twilio/recording",
        transcribe=True
    )

    return Response(content=str(response), media_type="application/xml")

@app.post("/webhooks/twilio/recording")
async def handle_recording(request: Request):
    """Process recorded question."""
    form = await request.form()
    recording_url = form.get("RecordingUrl")

    # Download audio
    audio = httpx.get(recording_url).content

    # Process with voice service
    answer_audio = await handle_voice_query(audio, tenant_id="...")

    # Upload answer audio and play back
    # (Implementation: upload to S3, return TwiML with Play verb)
```

**Cost**: Whisper ~$0.006/minute, TTS ~$0.015/1K chars, Twilio ~$0.02/minute. Total: ~$0.05/call.

### Phase 3: Observability - Automated Evaluation Pipelines

**WHY**: Manual testing doesn't scale. Need automated daily evals to catch regressions (new model version, prompt changes, data drift).

**Implementation**:
```python
# services/evaluation_service.py - Automated RAG evaluation
from typing import List
import pandas as pd

class EvaluationTestCase:
    query: str
    expected_answer: str
    expected_chunks: List[str]  # Document IDs that should be retrieved

async def run_daily_evaluation(tenant_id: str):
    """Run automated evaluation on golden test set."""
    # Load test cases (100-500 queries with ground truth)
    test_cases = await load_test_cases(tenant_id)

    results = []

    for case in test_cases:
        # Run RAG query
        rag_result = await rag_query(case.query, tenant_id)

        # Evaluate retrieval (precision/recall)
        retrieved_ids = [c["id"] for c in rag_result["chunks"]]
        precision = len(set(retrieved_ids) & set(case.expected_chunks)) / len(retrieved_ids)
        recall = len(set(retrieved_ids) & set(case.expected_chunks)) / len(case.expected_chunks)

        # Evaluate generation (LLM as judge)
        judge_prompt = f"""Compare the generated answer to the expected answer.
Expected: {case.expected_answer}
Generated: {rag_result["response"]}

Rate on a scale of 1-5 (1=completely wrong, 5=perfect). Respond with just the number."""

        judge_result = await openai_client.chat.completions.create(
            model="gpt-4o",
            messages=[{"role": "user", "content": judge_prompt}],
            max_tokens=5
        )

        score = int(judge_result.choices[0].message.content.strip())

        results.append({
            "query": case.query,
            "precision": precision,
            "recall": recall,
            "generation_score": score,
            "latency_ms": rag_result["latency_ms"]
        })

    # Aggregate metrics
    df = pd.DataFrame(results)
    metrics = {
        "avg_precision": df["precision"].mean(),
        "avg_recall": df["recall"].mean(),
        "avg_generation_score": df["generation_score"].mean(),
        "p95_latency_ms": df["latency_ms"].quantile(0.95),
        "pass_rate": (df["generation_score"] >= 4).mean()  # % scoring 4 or 5
    }

    # Store results
    await save_eval_results(tenant_id, date.today(), metrics)

    # Alert if regression
    if metrics["pass_rate"] < 0.85:  # SLO: 85% pass rate
        await send_alert(f"Evaluation regression for {tenant_id}: {metrics}")

    return metrics

# Schedule daily
@app.on_event("startup")
async def schedule_evals():
    scheduler.add_job(
        run_daily_evaluation,
        trigger="cron",
        hour=2,  # 2 AM daily
        args=["tenant_123"]
    )
```

**Metrics Tracked**:
- Retrieval: Precision@5, Recall@5, MRR
- Generation: LLM-as-judge score (1-5), ROUGE-L, human eval sample (10%)
- Latency: p50, p95, p99
- Cost: Tokens/query, cost/query

**Drift Detection**: Compare week-over-week metrics, alert if >10% drop.

---

## Evolution Roadmap (ASCII Diagram)

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         RAG Platform Evolution                          │
└─────────────────────────────────────────────────────────────────────────┘

Phase 1: MVP Foundation (Months 1-3)
┌────────────────────────────────────────────────────────────────┐
│  Core Features:                        Business Metrics:       │
│  • Basic RAG (retrieve + generate)     • 10-50 customers       │
│  • MiniLM embeddings                   • $5-10K MRR            │
│  • Fixed chunking (500 tokens)         • 60% deflection rate   │
│  • Multi-tenant collections            • 3s avg latency        │
│  • REST API + simple UI                • PMF validation        │
│  • Feedback (thumbs up/down)           • 70% user satisfaction │
└────────────────────────────────────────────────────────────────┘
                           │
                           │ + Advanced Retrieval & Generation
                           ▼
Phase 2: Growth Features (Months 4-9)
┌────────────────────────────────────────────────────────────────┐
│  Advanced Features:                    Business Metrics:       │
│  • Hybrid search (vector + keyword)    • 100-500 customers     │
│  • Cross-encoder reranking             • $50-100K MRR          │
│  • Parent-child chunking               • 75% deflection rate   │
│  • Self-RAG (reflection)               • 1.5s avg latency      │
│  • Agentic workflows (LangGraph)       • 80% user satisfaction │
│  • Multi-modal (images)                • 20% QoQ growth        │
│  • Observability (Langfuse)            • Competitive edge      │
│  • A/B testing framework               • Reduced churn         │
└────────────────────────────────────────────────────────────────┘
                           │
                           │ + Enterprise & Business Features
                           ▼
Phase 3: Enterprise Platform (Months 10-18)
┌────────────────────────────────────────────────────────────────┐
│  Enterprise Features:                  Business Metrics:       │
│  • White-labeling (custom domains)     • 500-2000 customers    │
│  • Shopify/WordPress integrations      • $200-500K MRR         │
│  • Per-tenant fine-tuning              • 85% deflection rate   │
│  • Voice support (Whisper + TTS)       • <1s avg latency       │
│  • SSO (SAML/OIDC)                     • $50K+ ARR contracts   │
│  • SOC 2 compliance                    • 85% retention         │
│  • Advanced analytics (drift, topics)  • Enterprise ready      │
│  • Webhooks for automation             • Market leader        │
└────────────────────────────────────────────────────────────────┘

                    Future: AI-Native Platform
┌────────────────────────────────────────────────────────────────┐
│  • Multi-agent collaboration (sales + support + ops)           │
│  • Proactive support (predict issues before user asks)         │
│  • Auto-resolution (close tickets end-to-end)                  │
│  • Cross-tenant learning (federated learning)                  │
│  • Real-time training (update models hourly)                   │
│  • Vertical-specific models (e-commerce, SaaS, healthcare)     │
└────────────────────────────────────────────────────────────────┘
```

---

## Interview Questions

**Q1: How would you implement agentic RAG with tool use?**

A: Use LangGraph to create a ReAct (Reasoning + Acting) agent. Define tools as Python functions with type hints and docstrings (e.g., `search_knowledge_base`, `get_order_status`, `create_return`). Agent receives user query, decides which tool to call based on descriptions, executes tool, evaluates result, and repeats until task complete. The LLM generates a plan in each step: "I should first check the order status using get_order_status(12345)". After tool execution, it reasons: "Order was delivered 26 days ago, within 30-day return window, so I can initiate return." Finally generates natural language response citing ticket number. Challenges: (1) Reliability—limit to 10 iterations max, validate tool inputs, (2) Cost—use GPT-4o-mini for tool selection, GPT-4o for final response, (3) Latency—stream intermediate steps ("Checking order...") to user. Monitor tool call success rate (target >95%) and avg iterations per query (target <3).

**Q2: What are the main approaches to multi-modal RAG, and when would you use each?**

A: (1) **CLIP for images**: Embed images and text in shared space (768-dim), search images by text query ("blue sneakers"), use GPT-4o vision to answer questions about images. Use for e-commerce (visual product search), real estate (search by room photos), fashion (style matching). (2) **OCR for documents**: Extract text from PDFs/images using Tesseract or cloud OCR, embed text normally. Use for invoices, contracts, scanned documents. (3) **Whisper for audio**: Transcribe audio to text, embed transcriptions. Use for call center recordings, podcasts, video tutorials. (4) **Video understanding**: Extract frames (1 fps), embed frames with CLIP, transcribe audio with Whisper, combine in multimodal retrieval. Use for training videos, tutorials. Each adds latency (OCR ~2s, CLIP inference ~100ms, Whisper ~3s per minute of audio) and storage (CLIP 768-dim vs MiniLM 384-dim). Start with text-only, add modalities based on user demand (if 30% of queries involve images, prioritize CLIP).

**Q3: Explain parent-child chunking and its benefits over fixed-size chunking.**

A: Parent-child chunking stores two levels: (1) Parent chunks (full sections/pages, 1500-2000 tokens) preserve broad context, (2) Child chunks (sentences/paragraphs, 200-400 tokens) enable precise retrieval. During retrieval, search children (high precision—only relevant sentences match), but return parent context to LLM (full context for better answers). Benefits: (1) Retrieval precision improves (find exact relevant sentence vs approximate chunk), (2) Generation quality improves (LLM sees full section context, not truncated), (3) Citation accuracy improves (cite specific sentence + broader context). Example: Query "Can I return electronics?" retrieves child chunk "Electronics are non-returnable" (precise match) with parent context "Return policy: 30 days for most items. Electronics, software, and gift cards are non-returnable." (full policy). Tradeoff: 2x storage (store both parent and child), more complex indexing logic. Typical improvement: 15-25% better answer quality on benchmarks.

**Q4: How would you implement A/B testing for different RAG configurations?**

A: (1) **Feature flags**: Use LaunchDarkly or custom flags to assign users to variants (50/50 split). Store assignment in Redis (`user_id → variant_A` or `variant_B`). (2) **Dual writes**: Ingest documents with both configurations (e.g., 400-token chunks in collection_v1, 600-token chunks in collection_v2, or MiniLM embeddings vs OpenAI embeddings in separate collections). (3) **Route queries**: Based on user's variant, query appropriate collection and use corresponding prompt/model. (4) **Track metrics**: Log per-variant: feedback score (thumbs up/down), p95 latency, deflection rate (% queries not escalating to human), tokens/query, cost/query. (5) **Statistical analysis**: Run for 2-4 weeks to gather 1000+ queries per variant. Use t-test to compare metrics (p<0.05 for significance). If variant B shows 10%+ improvement in key metric (e.g., feedback score), roll out to 100%. (6) **Gradual rollout**: After winning variant identified, roll out 25%→50%→100% over 1 week, monitoring for regressions. Track: variant assignment, query→variant mapping, per-variant metrics, user feedback, costs.

**Q5: What is Self-RAG and how does it improve answer quality?**

A: Self-RAG adds self-reflection steps where the model evaluates its own retrieval and generation. Process: (1) Retrieve context normally, (2) Model judges: "Is this retrieval relevant to the query?" (YES/PARTIAL/NO), (3) If NO, trigger alternative retrieval (hybrid search, broader query) or refuse to answer, (4) Generate answer, (5) Model validates: "Is my answer fully supported by the context?" (SUPPORTED/UNSUPPORTED), (6) If UNSUPPORTED, refuse or regenerate with stricter instructions. Benefits: (1) Reduces hallucinations by 30-50% (model refuses when uncertain), (2) Improves retrieval by triggering fallbacks when initial retrieval fails, (3) Increases user trust (model admits uncertainty vs confidently wrong answers). Tradeoff: 2-3x more LLM calls (cost increases from $0.002/query to $0.006/query), latency increases by 1-2s (extra generation steps). Use for high-stakes domains (legal, medical, financial) where accuracy >> speed. Implement with cheap model (GPT-4o-mini) for reflection steps, expensive model (GPT-4o) only for final answer.

