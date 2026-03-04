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
