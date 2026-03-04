# Section 10: Caching Strategy

## WHY: The Business Case for Caching in RAG Systems

Caching is not optional for production RAG SaaS—it's essential for economic viability and performance:

### Cost Impact
- **GPT API costs**: GPT-4 costs $0.03/1K input tokens. A 1000-token query costs $0.03. With 10K queries/day, that's $300/day = $9K/month.
- **Cache hit rate**: 60% cache hit rate saves $5.4K/month on GPT costs alone
- **Embedding API costs**: MiniLM self-hosted is free, but OpenAI embeddings cost $0.0001/1K tokens

### Performance Impact
- **GPT latency**: 2-5 seconds for streaming response
- **Cache latency**: 5-20ms for Redis lookup
- **User experience**: 100x faster response for cached queries
- **Throughput**: Cached responses don't consume GPT rate limits

### System Scalability
- **Database load**: Caching reduces MongoDB/Qdrant queries
- **Rate limits**: GPT API has 10K requests/min limit (Enterprise)
- **Concurrent users**: Cache enables handling 10x more users without scaling GPT spend

### Real-World Example
A customer support bot answering "How do I reset my password?" 50 times/day:
- **Without cache**: 50 × $0.03 = $1.50/day = $45/month for ONE question
- **With cache**: First query $0.03, remaining 49 queries cached = $0.03/day = $0.90/month
- **ROI**: 98% cost reduction on repeated queries

## WHAT: Multi-Layer Caching Architecture

A production RAG system needs **four distinct cache layers**, each with different invalidation strategies:

### 1. Response Cache Layer
**Purpose**: Cache complete chat responses to identical questions

**Cache Key Design:**
```
chat:response:{org_id}:{assistant_id}:{query_hash}:{context_hash}
```

**What to cache:**
- Complete assistant response (text + sources)
- Retrieved knowledge chunks
- Response metadata (model, tokens, latency)

**TTL Strategy:**
- Default: 1 hour (frequently changing knowledge)
- Configurable per assistant (e.g., 24h for stable FAQ bots)
- Invalidate on knowledge base updates

**When to use:**
- Identical user questions (exact match)
- FAQ-style queries with stable answers
- High-traffic support questions

**When NOT to use:**
- Personalized responses (user-specific data)
- Time-sensitive queries ("What's today's status?")
- Multi-turn conversations with context

### 2. Embedding Cache Layer
**Purpose**: Cache text embeddings to avoid recomputing vectors

**Cache Key Design:**
```
embed:{model_name}:{text_hash}
```

**What to cache:**
- Query embeddings (768-dim vectors for MiniLM)
- Document chunk embeddings during ingestion
- TTL: 24 hours (embeddings are deterministic)

**Impact:**
- Embedding generation: 50-200ms (local MiniLM)
- Cache lookup: 2-5ms
- **40x faster** for repeated queries

### 3. Session Cache Layer
**Purpose**: Store conversation history for multi-turn chat

**Cache Key Design:**
```
session:{conversation_id}
```

**What to cache:**
- Message history (last N messages)
- Conversation metadata (user_id, org_id, assistant_id)
- Session state (language, user preferences)

**TTL Strategy:**
- Active sessions: 30 minutes (sliding window)
- Inactive sessions: Moved to MongoDB cold storage
- Max messages in cache: 50 (older messages in DB)

### 4. Rate Limit Cache Layer
**Purpose**: Track API usage for per-tenant rate limiting

**Cache Key Design:**
```
ratelimit:{org_id}:{endpoint}:{window_timestamp}
```

**What to cache:**
- Request count per time window
- TTL: Window duration (e.g., 60 seconds for 1-minute windows)
- Atomic increment operations (INCR command)

**Example:**
```
ratelimit:org_123:/chat/message:1709251200 → 45 requests
```

## HOW: Implementation with Redis

### Cache Service Core Implementation

```python
# services/cache_service.py
import redis.asyncio as redis
import hashlib
import json
from typing import Any, Optional, Callable
from datetime import timedelta
import pickle
from functools import wraps

from config import settings
from utils.logging_utils import logger


class CacheService:
    """
    Redis-based multi-layer caching service.

    Provides get/set/invalidate operations with TTL support,
    hit rate tracking, and structured logging.
    """

    def __init__(self, redis_client: redis.Redis):
        self.redis = redis_client
        self._hit_count = 0
        self._miss_count = 0

    @staticmethod
    def generate_key_hash(text: str) -> str:
        """
        Generate deterministic hash for cache keys.

        Uses SHA256 for collision resistance.
        """
        return hashlib.sha256(text.encode()).hexdigest()[:16]

    async def get(self, key: str) -> Optional[Any]:
        """
        Get value from cache.

        Returns None if key doesn't exist.
        Tracks cache hit/miss metrics.
        """
        try:
            value = await self.redis.get(key)

            if value:
                self._hit_count += 1
                logger.debug("Cache hit", key=key, hit_rate=self.hit_rate)
                return pickle.loads(value)
            else:
                self._miss_count += 1
                logger.debug("Cache miss", key=key, hit_rate=self.hit_rate)
                return None

        except Exception as e:
            logger.error("Cache get failed", key=key, error=str(e))
            # Fail open: return None on cache errors
            return None

    async def set(
        self,
        key: str,
        value: Any,
        ttl: Optional[int] = None,
    ) -> bool:
        """
        Set value in cache with optional TTL.

        Args:
            key: Cache key
            value: Any Python object (will be pickled)
            ttl: Time-to-live in seconds (None = no expiry)

        Returns:
            True if successful, False otherwise
        """
        try:
            serialized = pickle.dumps(value)

            if ttl:
                await self.redis.setex(key, ttl, serialized)
            else:
                await self.redis.set(key, serialized)

            logger.debug("Cache set", key=key, ttl=ttl, size_bytes=len(serialized))
            return True

        except Exception as e:
            logger.error("Cache set failed", key=key, error=str(e))
            return False

    async def delete(self, key: str) -> bool:
        """Delete key from cache."""
        try:
            result = await self.redis.delete(key)
            logger.debug("Cache delete", key=key, existed=bool(result))
            return bool(result)
        except Exception as e:
            logger.error("Cache delete failed", key=key, error=str(e))
            return False

    async def delete_pattern(self, pattern: str) -> int:
        """
        Delete all keys matching a pattern.

        Example: delete_pattern("chat:response:org_123:*")

        Returns number of keys deleted.
        """
        try:
            cursor = 0
            deleted_count = 0

            # Use SCAN to avoid blocking Redis with KEYS
            while True:
                cursor, keys = await self.redis.scan(
                    cursor=cursor,
                    match=pattern,
                    count=100,
                )

                if keys:
                    deleted_count += await self.redis.delete(*keys)

                if cursor == 0:
                    break

            logger.info("Cache pattern delete", pattern=pattern, deleted=deleted_count)
            return deleted_count

        except Exception as e:
            logger.error("Cache pattern delete failed", pattern=pattern, error=str(e))
            return 0

    async def invalidate_knowledge_cache(self, org_id: str, source_id: Optional[str] = None):
        """
        Invalidate caches after knowledge base update.

        Deletes:
            - All response caches for organization
            - Embedding caches for affected source (if provided)
        """
        # Invalidate all response caches for this org
        pattern = f"chat:response:{org_id}:*"
        deleted = await self.delete_pattern(pattern)

        logger.info(
            "Knowledge cache invalidated",
            org_id=org_id,
            source_id=source_id,
            response_caches_deleted=deleted,
        )

    @property
    def hit_rate(self) -> float:
        """Calculate cache hit rate as percentage."""
        total = self._hit_count + self._miss_count
        return (self._hit_count / total * 100) if total > 0 else 0.0

    async def get_stats(self) -> dict:
        """Get cache statistics for monitoring."""
        info = await self.redis.info("stats")

        return {
            "hit_count": self._hit_count,
            "miss_count": self._miss_count,
            "hit_rate_pct": round(self.hit_rate, 2),
            "keyspace_hits": info.get("keyspace_hits", 0),
            "keyspace_misses": info.get("keyspace_misses", 0),
            "total_keys": await self.redis.dbsize(),
        }


# Dependency injection
async def get_cache_service() -> CacheService:
    """Provide CacheService instance for FastAPI dependency injection."""
    redis_client = await get_redis_client()
    return CacheService(redis_client)
```

### Response Caching Decorator

```python
# decorators/cached.py
from functools import wraps
from typing import Callable, Optional
import inspect

from services.cache_service import CacheService, get_cache_service


def cached(
    key_prefix: str,
    ttl: int = 3600,  # 1 hour default
    key_builder: Optional[Callable] = None,
):
    """
    Decorator to cache function results.

    Args:
        key_prefix: Prefix for cache key (e.g., "chat:response")
        ttl: Time-to-live in seconds
        key_builder: Custom function to build cache key from args

    Example:
        @cached(key_prefix="chat:response", ttl=3600)
        async def get_chat_response(org_id: str, query: str):
            # Expensive GPT call
            return response
    """

    def decorator(func: Callable):
        @wraps(func)
        async def wrapper(*args, **kwargs):
            # Get cache service
            cache = await get_cache_service()

            # Build cache key
            if key_builder:
                cache_key = key_builder(*args, **kwargs)
            else:
                # Default: hash all arguments
                sig = inspect.signature(func)
                bound = sig.bind(*args, **kwargs)
                bound.apply_defaults()

                # Build key from function name and args
                args_str = json.dumps(bound.arguments, sort_keys=True)
                args_hash = cache.generate_key_hash(args_str)
                cache_key = f"{key_prefix}:{func.__name__}:{args_hash}"

            # Try cache first
            cached_result = await cache.get(cache_key)
            if cached_result is not None:
                logger.debug(
                    "Cache hit for function",
                    function=func.__name__,
                    cache_key=cache_key,
                )
                return cached_result

            # Cache miss: execute function
            logger.debug(
                "Cache miss for function",
                function=func.__name__,
                cache_key=cache_key,
            )
            result = await func(*args, **kwargs)

            # Store in cache
            await cache.set(cache_key, result, ttl=ttl)

            return result

        return wrapper

    return decorator


# Usage example
@cached(key_prefix="chat:response", ttl=3600)
async def get_chat_response(
    org_id: str,
    assistant_id: str,
    query: str,
) -> dict:
    """This function's results will be automatically cached."""
    # Expensive operations: embedding, vector search, GPT call
    response = await chat_service.generate_response(org_id, assistant_id, query)
    return response
```

### Embedding Cache Implementation

```python
# services/embedding_cache_service.py
import numpy as np
from typing import Optional

from services.cache_service import CacheService
from utils.logging_utils import logger


class EmbeddingCacheService:
    """
    Specialized cache for text embeddings.

    Stores 768-dim vectors efficiently in Redis.
    """

    def __init__(self, cache_service: CacheService):
        self.cache = cache_service
        self.ttl = 86400  # 24 hours

    def _build_key(self, text: str, model_name: str = "minilm") -> str:
        """Build cache key for embedding."""
        text_hash = self.cache.generate_key_hash(text)
        return f"embed:{model_name}:{text_hash}"

    async def get_embedding(
        self,
        text: str,
        model_name: str = "minilm",
    ) -> Optional[np.ndarray]:
        """
        Get cached embedding for text.

        Returns None if not cached.
        """
        key = self._build_key(text, model_name)
        cached = await self.cache.get(key)

        if cached:
            logger.debug("Embedding cache hit", text_preview=text[:50], model=model_name)
            return np.array(cached)

        logger.debug("Embedding cache miss", text_preview=text[:50], model=model_name)
        return None

    async def set_embedding(
        self,
        text: str,
        embedding: np.ndarray,
        model_name: str = "minilm",
    ) -> bool:
        """
        Cache embedding for text.

        Args:
            text: Input text
            embedding: NumPy array (768-dim for MiniLM)
            model_name: Embedding model name
        """
        key = self._build_key(text, model_name)

        # Convert NumPy array to list for serialization
        embedding_list = embedding.tolist()

        success = await self.cache.set(key, embedding_list, ttl=self.ttl)

        if success:
            logger.debug(
                "Embedding cached",
                text_preview=text[:50],
                model=model_name,
                dimensions=len(embedding_list),
            )

        return success

    async def get_or_compute(
        self,
        text: str,
        compute_fn: Callable,
        model_name: str = "minilm",
    ) -> np.ndarray:
        """
        Get embedding from cache or compute if not cached.

        Args:
            text: Input text
            compute_fn: Async function to compute embedding
            model_name: Model name for cache key

        Returns:
            Embedding vector
        """
        # Try cache first
        cached = await self.get_embedding(text, model_name)
        if cached is not None:
            return cached

        # Cache miss: compute embedding
        embedding = await compute_fn(text)

        # Store in cache
        await self.set_embedding(text, embedding, model_name)

        return embedding
```

### Session Cache for Conversation History

```python
# services/session_cache_service.py
from typing import List, Optional
from datetime import datetime, timedelta

from services.cache_service import CacheService
from models.conversation import Message
from utils.logging_utils import logger


class SessionCacheService:
    """
    Cache conversation sessions for multi-turn chat.

    Stores recent messages in Redis for fast access,
    with automatic eviction to MongoDB for older messages.
    """

    def __init__(self, cache_service: CacheService):
        self.cache = cache_service
        self.ttl = 1800  # 30 minutes
        self.max_messages = 50  # Max messages to keep in cache

    def _build_key(self, conversation_id: str) -> str:
        """Build cache key for conversation."""
        return f"session:{conversation_id}"

    async def get_conversation(self, conversation_id: str) -> Optional[dict]:
        """
        Get conversation from cache.

        Returns:
            {
                "conversation_id": "conv_123",
                "messages": [...],
                "metadata": {...},
                "last_activity": "2024-01-15T10:30:00Z"
            }
        """
        key = self._build_key(conversation_id)
        conversation = await self.cache.get(key)

        if conversation:
            # Refresh TTL on access (sliding window)
            await self.cache.redis.expire(key, self.ttl)
            logger.debug("Session cache hit", conversation_id=conversation_id)

        return conversation

    async def save_conversation(
        self,
        conversation_id: str,
        messages: List[Message],
        metadata: dict,
    ) -> bool:
        """
        Save conversation to cache.

        Automatically trims to max_messages if needed.
        """
        key = self._build_key(conversation_id)

        # Trim to max messages (keep most recent)
        if len(messages) > self.max_messages:
            messages = messages[-self.max_messages:]
            logger.info(
                "Conversation trimmed in cache",
                conversation_id=conversation_id,
                kept_messages=len(messages),
            )

        conversation_data = {
            "conversation_id": conversation_id,
            "messages": [msg.dict() for msg in messages],
            "metadata": metadata,
            "last_activity": datetime.utcnow().isoformat(),
        }

        success = await self.cache.set(key, conversation_data, ttl=self.ttl)

        if success:
            logger.debug(
                "Conversation cached",
                conversation_id=conversation_id,
                message_count=len(messages),
            )

        return success

    async def append_message(
        self,
        conversation_id: str,
        message: Message,
    ) -> bool:
        """
        Append message to cached conversation.

        If conversation not in cache, loads from DB first.
        """
        conversation = await self.get_conversation(conversation_id)

        if not conversation:
            # Load from database
            conversation = await self._load_from_db(conversation_id)
            if not conversation:
                logger.warning("Conversation not found", conversation_id=conversation_id)
                return False

        # Append new message
        conversation["messages"].append(message.dict())
        conversation["last_activity"] = datetime.utcnow().isoformat()

        # Save back to cache
        return await self.save_conversation(
            conversation_id=conversation_id,
            messages=[Message(**m) for m in conversation["messages"]],
            metadata=conversation["metadata"],
        )

    async def _load_from_db(self, conversation_id: str) -> Optional[dict]:
        """Load conversation from MongoDB (cache miss fallback)."""
        # Implementation depends on your database layer
        pass
```

### Rate Limiting with Cache

```python
# services/rate_limiter_service.py
import time
from fastapi import HTTPException, status

from services.cache_service import CacheService
from utils.logging_utils import logger


class RateLimiterService:
    """
    Token bucket rate limiter using Redis.

    Tracks requests per tenant per endpoint.
    """

    def __init__(self, cache_service: CacheService):
        self.cache = cache_service

    def _build_key(
        self,
        org_id: str,
        endpoint: str,
        window_seconds: int,
    ) -> str:
        """Build rate limit key with time window."""
        window_start = int(time.time() / window_seconds) * window_seconds
        return f"ratelimit:{org_id}:{endpoint}:{window_start}"

    async def check_rate_limit(
        self,
        org_id: str,
        endpoint: str,
        max_requests: int,
        window_seconds: int = 60,
    ) -> dict:
        """
        Check if request is within rate limit.

        Args:
            org_id: Organization ID
            endpoint: API endpoint path
            max_requests: Max requests allowed in window
            window_seconds: Time window in seconds

        Returns:
            {
                "allowed": bool,
                "current": int,
                "limit": int,
                "remaining": int,
                "reset_at": int (timestamp)
            }

        Raises:
            HTTPException 429 if rate limit exceeded
        """
        key = self._build_key(org_id, endpoint, window_seconds)

        # Increment counter atomically
        current_count = await self.cache.redis.incr(key)

        # Set expiry on first request in window
        if current_count == 1:
            await self.cache.redis.expire(key, window_seconds)

        # Calculate remaining requests
        remaining = max(0, max_requests - current_count)
        reset_at = int(time.time()) + window_seconds

        # Check if over limit
        if current_count > max_requests:
            logger.warning(
                "Rate limit exceeded",
                org_id=org_id,
                endpoint=endpoint,
                current=current_count,
                limit=max_requests,
            )

            raise HTTPException(
                status_code=status.HTTP_429_TOO_MANY_REQUESTS,
                detail={
                    "error_code": "RATE_LIMIT_EXCEEDED",
                    "message": f"Rate limit exceeded: {max_requests} requests per {window_seconds}s",
                    "retry_after": window_seconds,
                },
                headers={
                    "X-RateLimit-Limit": str(max_requests),
                    "X-RateLimit-Remaining": "0",
                    "X-RateLimit-Reset": str(reset_at),
                    "Retry-After": str(window_seconds),
                },
            )

        return {
            "allowed": True,
            "current": current_count,
            "limit": max_requests,
            "remaining": remaining,
            "reset_at": reset_at,
        }
```

## Cache Flow Diagram

```
┌─────────────────────────────────────────────────────────────────────┐
│                         INCOMING CHAT REQUEST                        │
│                    "How do I reset my password?"                     │
└────────────────────────────────┬────────────────────────────────────┘
                                 │
                                 ▼
                    ┌────────────────────────┐
                    │  1. RATE LIMIT CHECK   │
                    │   Redis: ratelimit:*   │
                    └────────┬───────────────┘
                             │ allowed?
                             ▼
                    ┌────────────────────────┐
                    │  2. RESPONSE CACHE     │
                    │   Key: chat:response:  │
                    │   {org}:{query_hash}   │
                    └────────┬───────────────┘
                             │
                    ┌────────┴────────┐
                    │                 │
                  HIT ✓             MISS ✗
                    │                 │
                    │                 ▼
                    │        ┌────────────────────┐
                    │        │  3. EMBEDDING      │
                    │        │     CACHE          │
                    │        │  Key: embed:*      │
                    │        └─────┬──────────────┘
                    │              │
                    │     ┌────────┴────────┐
                    │     │                 │
                    │   HIT ✓             MISS ✗
                    │     │                 │
                    │     │                 ▼
                    │     │        ┌────────────────┐
                    │     │        │ Compute        │
                    │     │        │ Embedding      │
                    │     │        │ (50-200ms)     │
                    │     │        └────────┬───────┘
                    │     │                 │
                    │     └─────────────────┤
                    │                       ▼
                    │              ┌────────────────┐
                    │              │  4. QDRANT     │
                    │              │     SEARCH     │
                    │              │  (Vector DB)   │
                    │              └────────┬───────┘
                    │                       │
                    │                       ▼
                    │              ┌────────────────┐
                    │              │  5. GPT API    │
                    │              │     CALL       │
                    │              │  (2-5 sec)     │
                    │              └────────┬───────┘
                    │                       │
                    │                       ▼
                    │              ┌────────────────┐
                    │              │ Cache Response │
                    │              │   TTL: 1hr     │
                    │              └────────┬───────┘
                    │                       │
                    └───────────────────────┘
                                 │
                                 ▼
                    ┌────────────────────────┐
                    │  6. SESSION CACHE      │
                    │   Update conversation  │
                    │   TTL: 30min sliding   │
                    └────────┬───────────────┘
                             │
                             ▼
                    ┌────────────────────────┐
                    │  RETURN TO CLIENT      │
                    │  Latency: 5ms (hit)    │
                    │  or 3-7s (miss)        │
                    └────────────────────────┘
```

## Cache Invalidation Strategies

### 1. Time-Based Expiry (TTL)

**Best for:** Semi-stable data that changes infrequently

```python
# Different TTLs for different data types
CACHE_TTL = {
    "chat_response": 3600,      # 1 hour
    "embedding": 86400,          # 24 hours
    "session": 1800,             # 30 minutes
    "analytics": 300,            # 5 minutes
    "rate_limit": 60,            # 1 minute
}
```

### 2. Event-Driven Invalidation

**Best for:** Data that must be immediately consistent

```python
# Event handlers for knowledge base changes
async def on_knowledge_source_added(event):
    """Invalidate caches when new content is added."""
    org_id = event.org_id
    source_id = event.source_id

    cache = await get_cache_service()

    # Invalidate all response caches for this org
    await cache.invalidate_knowledge_cache(org_id, source_id)

    logger.info(
        "Cache invalidated due to knowledge update",
        org_id=org_id,
        source_id=source_id,
        trigger="knowledge_source_added",
    )


async def on_assistant_settings_changed(event):
    """Invalidate caches when assistant config changes."""
    org_id = event.org_id
    assistant_id = event.assistant_id

    cache = await get_cache_service()

    # Invalidate all responses for this assistant
    pattern = f"chat:response:{org_id}:{assistant_id}:*"
    deleted = await cache.delete_pattern(pattern)

    logger.info(
        "Assistant cache invalidated",
        org_id=org_id,
        assistant_id=assistant_id,
        deleted_keys=deleted,
    )
```

### 3. Versioned Caching

**Best for:** Data that changes atomically

```python
def _build_versioned_key(org_id: str, resource_id: str, version: int) -> str:
    """Include version in cache key to avoid invalidation."""
    return f"resource:{org_id}:{resource_id}:v{version}"

# When version changes, old cache entries naturally expire
```

### 4. Cache Stampede Prevention

**Best for:** High-traffic scenarios with expensive misses

```python
async def get_with_lock(
    cache: CacheService,
    key: str,
    compute_fn: Callable,
    ttl: int,
) -> Any:
    """
    Prevent cache stampede using Redis locks.

    Only one request computes the value; others wait.
    """
    # Try cache first
    cached = await cache.get(key)
    if cached:
        return cached

    # Acquire lock for this key
    lock_key = f"lock:{key}"
    lock = await cache.redis.set(lock_key, "1", nx=True, ex=10)

    if lock:
        # This request won the race: compute value
        try:
            value = await compute_fn()
            await cache.set(key, value, ttl=ttl)
            return value
        finally:
            await cache.redis.delete(lock_key)
    else:
        # Another request is computing: wait and retry
        await asyncio.sleep(0.1)
        return await get_with_lock(cache, key, compute_fn, ttl)
```

## Production Tips

### 1. Cache Warming Strategy

Pre-populate cache with frequently accessed data:

```python
async def warm_cache_on_startup():
    """Warm critical caches during application startup."""
    cache = await get_cache_service()

    # Load top 100 FAQ questions and their embeddings
    faq_questions = await db.faq.find().sort("view_count", -1).limit(100).to_list()

    for faq in faq_questions:
        # Compute and cache embedding
        embedding = await embedding_service.get_embedding(faq.question)
        await embedding_cache.set_embedding(faq.question, embedding)

        logger.info("FAQ cached", question=faq.question, id=faq.id)

    logger.info("Cache warming complete", faq_count=len(faq_questions))


# In app.py lifespan
@asynccontextmanager
async def lifespan(app: FastAPI):
    # Startup
    await warm_cache_on_startup()
    yield
    # Shutdown
    pass
```

### 2. Hit Rate Monitoring

Track cache effectiveness in production:

```python
# Metrics endpoint for monitoring
@router.get("/metrics/cache")
async def get_cache_metrics(cache: CacheService = Depends()):
    stats = await cache.get_stats()

    return {
        "response_cache": {
            "hit_rate_pct": stats["hit_rate_pct"],
            "total_requests": stats["hit_count"] + stats["miss_count"],
        },
        "redis_stats": {
            "total_keys": stats["total_keys"],
            "keyspace_hits": stats["keyspace_hits"],
            "keyspace_misses": stats["keyspace_misses"],
        },
    }


# Alert if hit rate drops below threshold
if cache.hit_rate < 40:
    logger.warning(
        "Cache hit rate below threshold",
        hit_rate=cache.hit_rate,
        threshold=40,
    )
    # Trigger alert to ops team
```

### 3. Memory Management

Redis memory limits and eviction policies:

```python
# redis.conf settings
maxmemory 2gb
maxmemory-policy allkeys-lru  # Evict least recently used keys

# Monitor memory usage
async def check_redis_memory():
    info = await redis.info("memory")

    used_memory_mb = info["used_memory"] / 1024 / 1024
    max_memory_mb = info["maxmemory"] / 1024 / 1024
    usage_pct = (used_memory_mb / max_memory_mb) * 100

    if usage_pct > 80:
        logger.warning(
            "Redis memory usage high",
            used_mb=used_memory_mb,
            max_mb=max_memory_mb,
            usage_pct=usage_pct,
        )
```

### 4. Cache Key Namespace Design

Organize keys for easy invalidation:

```
# Good: Hierarchical namespace
chat:response:{org_id}:{assistant_id}:{query_hash}
embed:{model}:{text_hash}
session:{conversation_id}
ratelimit:{org_id}:{endpoint}:{window}

# Bad: Flat namespace (hard to invalidate patterns)
cache:abc123def456
user_session_789
rl_org_123_chat
```

### 5. Serialization Performance

Choose serialization format based on data type:

```python
# JSON: Human-readable, slow for large objects
await redis.set(key, json.dumps(data))

# Pickle: Fast, binary, not human-readable
await redis.set(key, pickle.dumps(data))

# MessagePack: Faster than JSON, smaller than Pickle
import msgpack
await redis.set(key, msgpack.packb(data))

# Recommendation:
# - JSON for small config data
# - Pickle for complex Python objects
# - MessagePack for high-throughput paths
```

## Interview Q&A

### Q1: How do you cache chat responses without serving stale data after knowledge base updates?

**Answer:**

The key is **event-driven cache invalidation** triggered by knowledge base changes:

**1. Cache Key Structure:**
```python
key = f"chat:response:{org_id}:{assistant_id}:{query_hash}"
```

Include `org_id` and `assistant_id` so we can invalidate by scope.

**2. Invalidation Trigger:**
```python
async def on_knowledge_source_updated(org_id: str, source_id: str):
    """Webhook/event handler when knowledge base changes."""
    cache = CacheService()

    # Invalidate ALL response caches for this organization
    pattern = f"chat:response:{org_id}:*"
    deleted = await cache.delete_pattern(pattern)

    logger.info("Invalidated response cache", org_id=org_id, keys=deleted)
```

**3. Trade-offs:**

| Strategy | Pros | Cons |
|----------|------|------|
| **Invalidate all org caches** | Simple, guarantees freshness | Overkill if only 1 doc changed |
| **Invalidate by source** | More precise | Need to track query → source mapping |
| **Short TTL (5min)** | Simple, eventual consistency | Higher GPT costs |

**Production approach:**
- Use **TTL + event invalidation** combo
- Set TTL to 1 hour (cost savings)
- Invalidate immediately on knowledge updates (freshness)
- Track cache hit rate per org to detect issues

**Implementation:**
```python
class KnowledgeService:
    async def add_document(self, org_id: str, document: str):
        # Add to vector DB
        await qdrant.add_document(org_id, document)

        # Invalidate caches
        await cache.invalidate_knowledge_cache(org_id)

        # Publish event for other services
        await event_bus.publish("knowledge.updated", {
            "org_id": org_id,
            "timestamp": datetime.utcnow(),
        })
```

**Key principle:** Accept brief staleness (up to TTL) for cost savings, but invalidate immediately when accuracy is critical.

### Q2: What cache invalidation strategies are available, and when do you use each?

**Answer:**

**1. Time-Based (TTL):**
```python
await cache.set(key, value, ttl=3600)  # 1 hour
```

**When to use:**
- Data changes infrequently (FAQ responses)
- Eventual consistency is acceptable
- Simple to implement

**Pros:** Zero maintenance, prevents memory leaks
**Cons:** May serve stale data until TTL expires

---

**2. Event-Driven Invalidation:**
```python
async def on_data_updated(event):
    await cache.delete_pattern(f"resource:{event.id}:*")
```

**When to use:**
- Data must be immediately consistent
- You control the data source
- Events are reliable

**Pros:** Immediate consistency, cache only invalidated when needed
**Cons:** Requires event infrastructure, risk of missed events

---

**3. Write-Through Caching:**
```python
async def update_data(id: str, data: dict):
    # Update database
    await db.update(id, data)

    # Update cache immediately
    await cache.set(f"resource:{id}", data)
```

**When to use:**
- Reads far exceed writes
- Strong consistency required
- Single writer (no race conditions)

**Pros:** Always consistent
**Cons:** Writes are slower (update 2 places)

---

**4. Cache-Aside (Lazy Loading):**
```python
async def get_data(id: str):
    cached = await cache.get(f"resource:{id}")
    if cached:
        return cached

    data = await db.get(id)
    await cache.set(f"resource:{id}", data)
    return data
```

**When to use:**
- Read-heavy workloads
- Not all data needs to be cached
- Default pattern for most use cases

**Pros:** Only cache what's accessed, simple logic
**Cons:** First request is slow (cache miss)

---

**5. Versioned Caching:**
```python
key = f"resource:{id}:v{version}"
```

**When to use:**
- Immutable data (old versions never change)
- Multiple versions exist simultaneously
- High read volume

**Pros:** No invalidation needed, old versions coexist
**Cons:** Requires version tracking

---

**Decision Matrix:**

| Data Type | Strategy | TTL |
|-----------|----------|-----|
| Chat responses (with knowledge updates) | Event-driven + TTL | 1h |
| Embeddings (deterministic) | TTL only | 24h |
| User sessions | TTL sliding window | 30min |
| Analytics (eventually consistent) | TTL only | 5min |
| Configuration (rare changes) | Event-driven | No expiry |

**Best practice:** Use **hybrid approach**—TTL as safety net + event-driven for critical updates.

### Q3: How do you prevent cache stampede when many requests miss cache simultaneously?

**Answer:**

**Cache stampede** occurs when:
1. Cached item expires
2. 100+ concurrent requests all miss cache
3. All 100 requests hit expensive backend (e.g., GPT API) simultaneously
4. Results in 100x cost and 100x latency spike

**Solution: Distributed Locking Pattern**

```python
async def get_with_lock(
    key: str,
    compute_fn: Callable,
    ttl: int = 3600,
) -> Any:
    """
    Only one request computes value; others wait for result.
    """
    # Check cache first
    cached = await cache.get(key)
    if cached:
        return cached

    # Try to acquire lock
    lock_key = f"lock:{key}"
    lock_acquired = await redis.set(lock_key, "1", nx=True, ex=30)

    if lock_acquired:
        # This request won the race: compute value
        try:
            logger.info("Computing cache value", key=key)
            value = await compute_fn()

            # Store in cache
            await cache.set(key, value, ttl=ttl)
            return value
        finally:
            # Always release lock
            await redis.delete(lock_key)
    else:
        # Another request is computing: wait and retry
        logger.debug("Waiting for lock", key=key)

        # Poll cache until value appears or timeout
        for _ in range(50):  # Max 5 seconds
            await asyncio.sleep(0.1)
            cached = await cache.get(key)
            if cached:
                return cached

        # Timeout: fallback to computing ourselves
        logger.warning("Lock timeout, computing anyway", key=key)
        return await compute_fn()
```

**Alternative: Probabilistic Early Expiration**

Refresh cache *before* it expires with decreasing probability:

```python
async def get_with_early_refresh(key: str, compute_fn: Callable):
    """Refresh cache probabilistically before expiry."""
    result = await cache.get(key)
    if not result:
        # Normal cache miss
        result = await compute_fn()
        await cache.set(key, result, ttl=3600)
        return result

    # Check TTL
    ttl_remaining = await redis.ttl(key)

    # Refresh if TTL < 10% AND random chance
    if ttl_remaining < 360:  # Less than 10% of 3600s
        refresh_probability = 1 - (ttl_remaining / 360)

        if random.random() < refresh_probability:
            # Refresh in background
            asyncio.create_task(refresh_cache(key, compute_fn))

    return result
```

**Best Practice:**
1. Use locking for **expensive operations** (GPT API calls)
2. Use early refresh for **moderate cost operations** (database queries)
3. Set lock timeout **longer than compute time** to avoid duplicate work
4. Monitor lock contention metrics (how often requests wait)

**Key Insight:** Stampede prevention is about **coordinating concurrent requests** so only one does the expensive work.

### Q4: How do you determine optimal TTL values for different cache layers?

**Answer:**

TTL selection requires balancing **cost**, **freshness**, and **hit rate**:

**Framework:**

```
Optimal TTL = f(update_frequency, cost_per_miss, staleness_tolerance)
```

**1. Response Cache (Chat Responses):**

**Factors:**
- Update frequency: Knowledge base changes 1-10x/day
- Cost per miss: $0.02-0.05 per GPT call
- Staleness tolerance: Medium (users expect recent info)

**Calculation:**
- If knowledge updates every 6 hours → TTL = 1 hour (safe buffer)
- If 60% hit rate with 1h TTL → saves $3K/month
- Event-driven invalidation reduces staleness risk

**Recommended: 1 hour TTL + event invalidation**

---

**2. Embedding Cache:**

**Factors:**
- Update frequency: Never (deterministic function)
- Cost per miss: 50-200ms compute time
- Staleness tolerance: N/A (always correct)

**Calculation:**
- Embeddings are pure functions (same input → same output)
- Only risk is memory usage
- Keep until evicted by LRU

**Recommended: 24 hours TTL** (balance memory vs recompute)

---

**3. Session Cache:**

**Factors:**
- Update frequency: Every message (high)
- Cost per miss: Database query (~20ms)
- Staleness tolerance: Zero (must be consistent)

**Calculation:**
- Users expect instant context in multi-turn chat
- Sliding window: Refresh TTL on every access
- Move to cold storage after inactivity

**Recommended: 30 minutes sliding window**

---

**4. Rate Limit Cache:**

**Factors:**
- Update frequency: Every request
- Cost per miss: N/A (atomic counter)
- Staleness tolerance: Zero (security critical)

**Calculation:**
- TTL = rate limit window (e.g., 60s for per-minute limits)
- No sliding window needed

**Recommended: Match window duration (60s, 3600s, etc.)**

---

**5. Analytics Cache:**

**Factors:**
- Update frequency: Continuous
- Cost per miss: Expensive aggregation query (~500ms)
- Staleness tolerance: High (dashboards can lag)

**Calculation:**
- Users don't expect real-time analytics
- Aggregate every 5 minutes in background job
- Cache results for frontend

**Recommended: 5 minutes TTL**

---

**Rule of Thumb:**

| Cache Type | TTL | Rationale |
|------------|-----|-----------|
| **Immutable data** | 24h+ | Never changes, safe to cache long |
| **Slow-changing data** | 1-6h | FAQ, docs, config |
| **User session** | 30min sliding | Active use pattern |
| **Real-time data** | 1-5min | Dashboards, counters |
| **Security tokens** | Match token expiry | JWT, API keys |
| **Rate limits** | Window duration | 60s, 3600s |

**Testing Your TTL:**

```python
# A/B test different TTLs in production
async def test_cache_ttl():
    """Monitor cost vs freshness trade-off."""
    for ttl in [300, 900, 1800, 3600]:  # 5min to 1hr
        cache.ttl = ttl

        # Track metrics
        metrics = {
            "ttl": ttl,
            "hit_rate": await cache.hit_rate,
            "gpt_cost_daily": await calculate_gpt_cost(),
            "avg_latency_ms": await get_avg_latency(),
            "stale_rate": await measure_staleness(),
        }

        logger.info("TTL experiment", **metrics)
```

**Key Insight:** Start conservative (short TTL), then increase gradually while monitoring staleness complaints and cost savings.

### Q5: How do you handle cache consistency in a distributed system with multiple API servers?

**Answer:**

With multiple API servers (horizontal scaling), cache consistency requires coordination:

**Problem:**
- Server A updates database
- Server B's cache still has old value
- User gets inconsistent results depending on which server handles request

**Solution Patterns:**

**1. Centralized Cache (Redis):**

```python
# All servers share the same Redis instance
cache = redis.Redis(host="cache.example.com")

# Server A writes:
await db.update(id=123, data=new_data)
await cache.delete(f"resource:123")  # Invalidate

# Server B reads:
# Cache miss triggers fresh DB read, consistent across all servers
```

**Pros:** Automatic consistency, simple
**Cons:** Redis is single point of failure (use Redis Cluster)

---

**2. Cache Invalidation via Pub/Sub:**

```python
# Server A publishes invalidation event
await redis.publish("cache:invalidate", json.dumps({
    "key": "resource:123",
    "timestamp": time.time(),
}))

# All servers (A, B, C) subscribe and invalidate local caches
async def listen_for_invalidations():
    pubsub = redis.pubsub()
    await pubsub.subscribe("cache:invalidate")

    async for message in pubsub.listen():
        data = json.loads(message["data"])
        await local_cache.delete(data["key"])
```

**Use case:** Hybrid local + Redis caching
**Pros:** Fast local cache + eventual consistency
**Cons:** Network partition can cause brief inconsistency

---

**3. Write-Through to All Layers:**

```python
async def update_resource(id: str, data: dict):
    # Update database
    await db.update(id, data)

    # Update Redis cache
    await redis_cache.set(f"resource:{id}", data)

    # Update local cache on this server
    await local_cache.set(f"resource:{id}", data)

    # Notify other servers via pub/sub
    await redis.publish("cache:update", json.dumps({
        "key": f"resource:{id}",
        "value": data,
    }))
```

**Pros:** Strong consistency
**Cons:** Complex, slower writes

---

**4. Versioned Caching with Consensus:**

```python
# Include version in cache key
key = f"resource:{id}:v{version}"

# On update, increment version in database (atomic)
new_version = await db.increment_version(id)

# New version means old caches naturally invalidate
await cache.set(f"resource:{id}:v{new_version}", data)
```

**Pros:** No invalidation needed, old versions coexist
**Cons:** Requires version field in DB

---

**Best Practice for Multi-Server Deployments:**

1. **Use centralized Redis** for primary cache (not local memory)
2. **Enable Redis Cluster** for high availability
3. **Pub/Sub for critical invalidations** that must propagate immediately
4. **Monitor cache key distribution** to avoid hot keys
5. **Use consistent hashing** if sharding Redis across nodes

**Production Setup:**

```yaml
# docker-compose.yml
services:
  redis-cluster:
    image: redis:7-alpine
    command: redis-server --cluster-enabled yes
    volumes:
      - redis-data:/data
    ports:
      - "6379:6379"

  api-server:
    image: api:latest
    environment:
      REDIS_URL: redis://redis-cluster:6379
    depends_on:
      - redis-cluster
    deploy:
      replicas: 3  # 3 API servers
```

**Key Insight:** Centralized cache (Redis) is simplest and most reliable for distributed systems. Only add complexity (local cache + pub/sub) if Redis latency becomes a bottleneck.

---

**Next Steps:**
- Section 11: Implement background jobs with Celery for async tasks
- Section 12: Design monitoring and observability with Prometheus + Grafana
- Section 13: Build deployment pipeline with Docker + Kubernetes
