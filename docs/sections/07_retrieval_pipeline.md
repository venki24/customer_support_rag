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

