# Section 14: Analytics & Observability

## WHY: Measure What Matters

You've built a sophisticated RAG system, but **how do you know if it's actually helping customers?** Without observability, you're flying blind:

1. **Is RAG improving resolution rates?** Or are users still escalating to human agents?
2. **Are retrievals relevant?** Low relevance scores mean your embeddings or chunking strategy is broken.
3. **Is the system fast enough?** P95 latency > 5s will frustrate users.
4. **What questions can't you answer?** Knowledge gaps reveal where to focus content creation.
5. **Is it cost-effective?** Token costs can spiral out of control without tracking.

**Observability** transforms your product from a black box to a feedback loop:

```
User Question → RAG System → Response
                    ↓
              [Analytics]
                    ↓
     Insights → Improve Content/Model
                    ↓
              Better Answers
```

---

## WHAT: Key Metrics & Architecture

### Metrics to Track

#### 1. Resolution Rate
**Definition**: % of conversations where user didn't escalate or give negative feedback.

**Why it matters**: The ultimate success metric. If users still need human help, RAG isn't working.

**How to measure**:
```python
resolution_rate = (conversations_resolved / total_conversations) * 100

# Resolved = user gave thumbs up OR didn't ask "talk to human" OR session ended naturally
```

#### 2. Retrieval Quality
**Metrics**:
- **Average Relevance Score**: Mean of top-k retrieval scores (0.0-1.0)
- **Top-1 Precision**: % of times top result was actually used in answer
- **No-Result Rate**: % of queries with all scores < 0.5 (knowledge gap indicator)

**Target**: Avg relevance > 0.7, No-result rate < 10%

#### 3. Response Latency
**Metrics**:
- **P50 (Median)**: 50% of responses faster than X
- **P95**: 95% of responses faster than X (catch outliers)
- **P99**: 99% (catch worst cases)

**Target**: P95 < 3s for embedding + retrieval + generation

**Breakdown**:
```
Total Latency = Embedding (100ms) + Qdrant Query (200ms) + GPT Streaming (2s)
```

#### 4. User Satisfaction
**Metrics**:
- **Thumbs Up Rate**: % of responses rated positive
- **CSAT Score**: 1-5 star ratings (if collected)
- **Follow-up Rate**: % of users asking clarifying questions (indicates unclear answer)

**Target**: Thumbs up rate > 70%

#### 5. Popular Questions & Knowledge Gaps
**Questions to answer**:
- What are the top 20 questions asked?
- Which questions have lowest retrieval scores?
- Which topics have highest escalation rate?

**Action**: Prioritize content creation for gaps.

#### 6. Cost Tracking
**Per-Tenant Metrics**:
- **Tokens per conversation**: Track input + output tokens
- **Monthly cost**: `tokens * cost_per_token` (e.g., GPT-4: $0.03/1K input, $0.06/1K output)
- **Cost per resolved conversation**: `total_cost / resolved_conversations`

**Target**: Keep cost per conversation < $0.10

### Analytics Data Flow

```
┌──────────────────────────────────────────────────────────────┐
│                   Live Request Flow                          │
│                                                              │
│  User Message → RAG Pipeline → GPT Response                 │
│         │              │              │                      │
│         └──────────────┴──────────────┘                      │
│                        │                                     │
│                        ▼                                     │
│            ┌───────────────────────┐                        │
│            │  ChatLog (Raw Event)  │                        │
│            │  • tenant_id          │                        │
│            │  • session_id         │                        │
│            │  • message            │                        │
│            │  • retrieved_docs     │                        │
│            │  • response           │                        │
│            │  • latency_ms         │                        │
│            │  • tokens_used        │                        │
│            │  • timestamp          │                        │
│            │  • trace_id           │                        │
│            └───────────┬───────────┘                        │
│                        │                                     │
└────────────────────────┼─────────────────────────────────────┘
                         │
                         │ (Celery Async Aggregation)
                         │
                         ▼
            ┌────────────────────────────┐
            │  HourlyAnalytics           │
            │  • tenant_id               │
            │  • hour (2026-02-10 14:00) │
            │  • total_messages          │
            │  • avg_retrieval_score     │
            │  • p50_latency             │
            │  • p95_latency             │
            │  • thumbs_up_count         │
            │  • thumbs_down_count       │
            │  • total_tokens            │
            │  • total_cost              │
            └──────────┬─────────────────┘
                       │
                       │ (MongoDB Aggregation Pipeline)
                       │
                       ▼
            ┌────────────────────────────┐
            │  DailyAnalytics            │
            │  • tenant_id               │
            │  • date (2026-02-10)       │
            │  • resolution_rate         │
            │  • avg_retrieval_score     │
            │  • p95_latency             │
            │  • satisfaction_rate       │
            │  • cost                    │
            │  • top_questions [array]   │
            │  • knowledge_gaps [array]  │
            └──────────┬─────────────────┘
                       │
                       │ (Dashboard Queries)
                       │
                       ▼
            ┌────────────────────────────┐
            │  Analytics Dashboard       │
            │  • Real-time metrics       │
            │  • Historical trends       │
            │  • Alerts (P95 > 5s, etc.) │
            └────────────────────────────┘
```

### Structured Logging with Trace IDs

Every request gets a unique `trace_id` that flows through all components:

```
[2026-02-10 14:23:15] INFO  trace_id=req_abc123 event=chat_request tenant_id=tenant_x user_message="How do I reset password?"
[2026-02-10 14:23:15] INFO  trace_id=req_abc123 event=embedding_start model=minilm
[2026-02-10 14:23:15] INFO  trace_id=req_abc123 event=embedding_complete duration_ms=95
[2026-02-10 14:23:15] INFO  trace_id=req_abc123 event=retrieval_start top_k=5
[2026-02-10 14:23:15] INFO  trace_id=req_abc123 event=retrieval_complete num_results=5 avg_score=0.82 duration_ms=180
[2026-02-10 14:23:17] INFO  trace_id=req_abc123 event=gpt_streaming_start model=gpt-4o-mini
[2026-02-10 14:23:19] INFO  trace_id=req_abc123 event=gpt_streaming_complete tokens_in=450 tokens_out=120 duration_ms=2100
[2026-02-10 14:23:19] INFO  trace_id=req_abc123 event=chat_complete total_duration_ms=2375
```

**Benefits**:
1. **Debugging**: Find all logs for a specific failed request
2. **Performance Analysis**: Identify which stage is slow
3. **Alerting**: Trigger alerts when `total_duration_ms > 5000`

---

## HOW: Implementation

### 1. Chat Log Model (Raw Events)

```python
# models/chat_log.py
from beanie import Document
from pydantic import Field
from datetime import datetime
from typing import Optional

class RetrievedDoc(BaseModel):
    document_id: str
    score: float
    title: str

class ChatLog(Document):
    trace_id: str = Field(..., index=True)
    tenant_id: str = Field(..., index=True)
    session_id: str = Field(..., index=True)

    # Request
    user_message: str
    timestamp: datetime = Field(default_factory=datetime.utcnow, index=True)

    # Retrieval
    retrieved_docs: list[RetrievedDoc] = []
    avg_retrieval_score: float = 0.0
    retrieval_latency_ms: int = 0

    # Generation
    assistant_response: str
    tokens_input: int = 0
    tokens_output: int = 0
    generation_latency_ms: int = 0

    # Total
    total_latency_ms: int = 0

    # Feedback
    feedback: Optional[str] = None  # "positive", "negative", None
    escalated_to_human: bool = False

    class Settings:
        name = "chat_logs"
        indexes = [
            "trace_id",
            "tenant_id",
            "session_id",
            ("tenant_id", "timestamp"),  # Compound index for time-series queries
        ]
```

### 2. Analytics Aggregation Models

```python
# models/analytics.py
from beanie import Document
from pydantic import Field
from datetime import datetime

class HourlyAnalytics(Document):
    tenant_id: str = Field(..., index=True)
    hour: datetime = Field(..., index=True)  # Truncated to hour (e.g., 2026-02-10 14:00:00)

    # Volume
    total_messages: int = 0
    total_sessions: int = 0

    # Quality
    avg_retrieval_score: float = 0.0
    no_result_count: int = 0  # Queries with max score < 0.5

    # Latency (milliseconds)
    p50_latency: int = 0
    p95_latency: int = 0
    p99_latency: int = 0

    # Satisfaction
    thumbs_up_count: int = 0
    thumbs_down_count: int = 0
    escalation_count: int = 0

    # Cost
    total_tokens_input: int = 0
    total_tokens_output: int = 0
    total_cost_usd: float = 0.0

    class Settings:
        name = "hourly_analytics"
        indexes = [
            ("tenant_id", "hour"),
        ]

class DailyAnalytics(Document):
    tenant_id: str = Field(..., index=True)
    date: datetime = Field(..., index=True)  # Truncated to day

    # Aggregates from HourlyAnalytics
    total_messages: int = 0
    resolution_rate: float = 0.0  # % of sessions not escalated
    avg_retrieval_score: float = 0.0
    p95_latency: int = 0
    satisfaction_rate: float = 0.0  # thumbs_up / (thumbs_up + thumbs_down)
    total_cost_usd: float = 0.0

    # Top insights
    top_questions: list[dict] = []  # [{"question": "...", "count": 50}, ...]
    knowledge_gaps: list[dict] = []  # [{"question": "...", "avg_score": 0.3}, ...]

    class Settings:
        name = "daily_analytics"
        indexes = [
            ("tenant_id", "date"),
        ]
```

### 3. Analytics Service with Aggregation Pipeline

```python
# services/analytics_service.py
from models.chat_log import ChatLog
from models.analytics import HourlyAnalytics, DailyAnalytics
from datetime import datetime, timedelta
from typing import Optional
import numpy as np

class AnalyticsService:
    @staticmethod
    async def aggregate_hourly(tenant_id: str, hour: datetime):
        """Aggregate raw logs into hourly stats."""
        start = hour.replace(minute=0, second=0, microsecond=0)
        end = start + timedelta(hours=1)

        # MongoDB aggregation pipeline
        pipeline = [
            {
                "$match": {
                    "tenant_id": tenant_id,
                    "timestamp": {"$gte": start, "$lt": end}
                }
            },
            {
                "$group": {
                    "_id": None,
                    "total_messages": {"$sum": 1},
                    "total_sessions": {"$addToSet": "$session_id"},
                    "avg_retrieval_score": {"$avg": "$avg_retrieval_score"},
                    "no_result_count": {
                        "$sum": {"$cond": [{"$lt": ["$avg_retrieval_score", 0.5]}, 1, 0]}
                    },
                    "latencies": {"$push": "$total_latency_ms"},
                    "thumbs_up_count": {
                        "$sum": {"$cond": [{"$eq": ["$feedback", "positive"]}, 1, 0]}
                    },
                    "thumbs_down_count": {
                        "$sum": {"$cond": [{"$eq": ["$feedback", "negative"]}, 1, 0]}
                    },
                    "escalation_count": {
                        "$sum": {"$cond": ["$escalated_to_human", 1, 0]}
                    },
                    "total_tokens_input": {"$sum": "$tokens_input"},
                    "total_tokens_output": {"$sum": "$tokens_output"},
                }
            }
        ]

        result = await ChatLog.aggregate(pipeline).to_list()

        if not result:
            return

        data = result[0]

        # Calculate percentiles
        latencies = sorted(data["latencies"])
        p50 = np.percentile(latencies, 50) if latencies else 0
        p95 = np.percentile(latencies, 95) if latencies else 0
        p99 = np.percentile(latencies, 99) if latencies else 0

        # Calculate cost (GPT-4o-mini pricing)
        cost = (
            data["total_tokens_input"] * 0.00015 / 1000 +  # $0.15 per 1M input tokens
            data["total_tokens_output"] * 0.0006 / 1000    # $0.60 per 1M output tokens
        )

        # Upsert HourlyAnalytics
        hourly = await HourlyAnalytics.find_one(
            HourlyAnalytics.tenant_id == tenant_id,
            HourlyAnalytics.hour == start
        )

        if hourly:
            hourly.total_messages = data["total_messages"]
            hourly.total_sessions = len(data["total_sessions"])
            hourly.avg_retrieval_score = data["avg_retrieval_score"]
            hourly.no_result_count = data["no_result_count"]
            hourly.p50_latency = int(p50)
            hourly.p95_latency = int(p95)
            hourly.p99_latency = int(p99)
            hourly.thumbs_up_count = data["thumbs_up_count"]
            hourly.thumbs_down_count = data["thumbs_down_count"]
            hourly.escalation_count = data["escalation_count"]
            hourly.total_tokens_input = data["total_tokens_input"]
            hourly.total_tokens_output = data["total_tokens_output"]
            hourly.total_cost_usd = cost
            await hourly.save()
        else:
            hourly = HourlyAnalytics(
                tenant_id=tenant_id,
                hour=start,
                total_messages=data["total_messages"],
                total_sessions=len(data["total_sessions"]),
                avg_retrieval_score=data["avg_retrieval_score"],
                no_result_count=data["no_result_count"],
                p50_latency=int(p50),
                p95_latency=int(p95),
                p99_latency=int(p99),
                thumbs_up_count=data["thumbs_up_count"],
                thumbs_down_count=data["thumbs_down_count"],
                escalation_count=data["escalation_count"],
                total_tokens_input=data["total_tokens_input"],
                total_tokens_output=data["total_tokens_output"],
                total_cost_usd=cost
            )
            await hourly.insert()

    @staticmethod
    async def aggregate_daily(tenant_id: str, date: datetime):
        """Aggregate hourly stats into daily stats."""
        start = date.replace(hour=0, minute=0, second=0, microsecond=0)
        end = start + timedelta(days=1)

        # Aggregate HourlyAnalytics
        hourly_stats = await HourlyAnalytics.find(
            HourlyAnalytics.tenant_id == tenant_id,
            HourlyAnalytics.hour >= start,
            HourlyAnalytics.hour < end
        ).to_list()

        if not hourly_stats:
            return

        # Calculate daily metrics
        total_messages = sum(h.total_messages for h in hourly_stats)
        total_sessions = sum(h.total_sessions for h in hourly_stats)
        avg_retrieval_score = np.mean([h.avg_retrieval_score for h in hourly_stats])
        p95_latency = int(np.percentile([h.p95_latency for h in hourly_stats], 95))

        total_thumbs_up = sum(h.thumbs_up_count for h in hourly_stats)
        total_thumbs_down = sum(h.thumbs_down_count for h in hourly_stats)
        satisfaction_rate = (
            total_thumbs_up / (total_thumbs_up + total_thumbs_down)
            if (total_thumbs_up + total_thumbs_down) > 0
            else 0.0
        )

        total_escalations = sum(h.escalation_count for h in hourly_stats)
        resolution_rate = (
            (total_sessions - total_escalations) / total_sessions
            if total_sessions > 0
            else 0.0
        )

        total_cost = sum(h.total_cost_usd for h in hourly_stats)

        # Top questions (from raw logs)
        top_questions = await ChatLog.aggregate([
            {
                "$match": {
                    "tenant_id": tenant_id,
                    "timestamp": {"$gte": start, "$lt": end}
                }
            },
            {
                "$group": {
                    "_id": "$user_message",
                    "count": {"$sum": 1}
                }
            },
            {"$sort": {"count": -1}},
            {"$limit": 20}
        ]).to_list()

        # Knowledge gaps (questions with low retrieval scores)
        knowledge_gaps = await ChatLog.aggregate([
            {
                "$match": {
                    "tenant_id": tenant_id,
                    "timestamp": {"$gte": start, "$lt": end},
                    "avg_retrieval_score": {"$lt": 0.6}
                }
            },
            {
                "$group": {
                    "_id": "$user_message",
                    "avg_score": {"$avg": "$avg_retrieval_score"},
                    "count": {"$sum": 1}
                }
            },
            {"$sort": {"count": -1}},
            {"$limit": 20}
        ]).to_list()

        # Upsert DailyAnalytics
        daily = await DailyAnalytics.find_one(
            DailyAnalytics.tenant_id == tenant_id,
            DailyAnalytics.date == start
        )

        stats = {
            "total_messages": total_messages,
            "resolution_rate": resolution_rate,
            "avg_retrieval_score": float(avg_retrieval_score),
            "p95_latency": p95_latency,
            "satisfaction_rate": satisfaction_rate,
            "total_cost_usd": total_cost,
            "top_questions": [
                {"question": q["_id"], "count": q["count"]}
                for q in top_questions
            ],
            "knowledge_gaps": [
                {"question": g["_id"], "avg_score": g["avg_score"], "count": g["count"]}
                for g in knowledge_gaps
            ]
        }

        if daily:
            await daily.set(stats)
        else:
            daily = DailyAnalytics(
                tenant_id=tenant_id,
                date=start,
                **stats
            )
            await daily.insert()
```

### 4. Structured Logging Middleware

```python
# middleware/logging_middleware.py
from fastapi import Request
from loguru import logger
import time
import uuid
import json

async def logging_middleware(request: Request, call_next):
    """Inject trace_id and log request/response."""
    # Generate trace_id
    trace_id = str(uuid.uuid4())
    request.state.trace_id = trace_id

    # Log request
    logger.bind(
        trace_id=trace_id,
        event="request_start",
        method=request.method,
        path=request.url.path,
        client_ip=request.client.host
    ).info("Incoming request")

    start_time = time.time()

    # Process request
    response = await call_next(request)

    # Log response
    duration_ms = int((time.time() - start_time) * 1000)
    logger.bind(
        trace_id=trace_id,
        event="request_complete",
        status_code=response.status_code,
        duration_ms=duration_ms
    ).info("Request complete")

    # Add trace_id to response headers (helps with debugging)
    response.headers["X-Trace-ID"] = trace_id

    return response

# Configure loguru for structured JSON logging
logger.configure(
    handlers=[
        {
            "sink": "logs/app.log",
            "format": "{time:YYYY-MM-DD HH:mm:ss} | {level} | {extra[trace_id]} | {extra[event]} | {message}",
            "rotation": "500 MB",
            "retention": "30 days",
            "serialize": True  # JSON format
        }
    ]
)
```

### 5. Feedback Collection Endpoint

```python
# routes/feedback_api.py
from fastapi import APIRouter, HTTPException
from pydantic import BaseModel
from models.chat_log import ChatLog

router = APIRouter(prefix="/feedback", tags=["feedback"])

class FeedbackRequest(BaseModel):
    trace_id: str
    feedback: str  # "positive" or "negative"
    comment: Optional[str] = None

@router.post("/")
async def submit_feedback(request: FeedbackRequest):
    """User submits thumbs up/down after receiving response."""
    chat_log = await ChatLog.find_one(ChatLog.trace_id == request.trace_id)

    if not chat_log:
        raise HTTPException(status_code=404, detail="Chat log not found")

    chat_log.feedback = request.feedback
    await chat_log.save()

    logger.bind(
        trace_id=request.trace_id,
        event="feedback_received",
        feedback=request.feedback
    ).info("User feedback recorded")

    return {"status": "success"}
```

### 6. Health Check Endpoint

```python
# routes/health_api.py
from fastapi import APIRouter
from database.mongodb import db
from database.redis_client import redis_client
from database.qdrant_client import qdrant_client
from datetime import datetime

router = APIRouter(prefix="/health", tags=["health"])

@router.get("/")
async def health_check():
    """Check health of all dependencies."""
    status = {
        "status": "healthy",
        "timestamp": datetime.utcnow().isoformat(),
        "services": {}
    }

    # MongoDB
    try:
        await db.command("ping")
        status["services"]["mongodb"] = "healthy"
    except Exception as e:
        status["services"]["mongodb"] = f"unhealthy: {str(e)}"
        status["status"] = "degraded"

    # Redis
    try:
        await redis_client.ping()
        status["services"]["redis"] = "healthy"
    except Exception as e:
        status["services"]["redis"] = f"unhealthy: {str(e)}"
        status["status"] = "degraded"

    # Qdrant
    try:
        qdrant_client.get_collections()
        status["services"]["qdrant"] = "healthy"
    except Exception as e:
        status["services"]["qdrant"] = f"unhealthy: {str(e)}"
        status["status"] = "degraded"

    return status

@router.get("/metrics")
async def metrics():
    """Prometheus-style metrics endpoint."""
    # Example: expose key metrics in Prometheus format
    # (In production, use prometheus_client library)

    metrics_text = f"""
# HELP chat_requests_total Total number of chat requests
# TYPE chat_requests_total counter
chat_requests_total{{tenant="tenant_x"}} 1500

# HELP chat_latency_seconds Chat response latency
# TYPE chat_latency_seconds histogram
chat_latency_seconds_bucket{{le="1.0",tenant="tenant_x"}} 800
chat_latency_seconds_bucket{{le="3.0",tenant="tenant_x"}} 1200
chat_latency_seconds_bucket{{le="5.0",tenant="tenant_x"}} 1450
chat_latency_seconds_bucket{{le="+Inf",tenant="tenant_x"}} 1500
chat_latency_seconds_sum{{tenant="tenant_x"}} 3500
chat_latency_seconds_count{{tenant="tenant_x"}} 1500

# HELP retrieval_score Average retrieval relevance score
# TYPE retrieval_score gauge
retrieval_score{{tenant="tenant_x"}} 0.78
"""
    return Response(content=metrics_text, media_type="text/plain")
```

### 7. Celery Task for Hourly Aggregation

```python
# tasks/analytics_tasks.py
from celery import Celery
from datetime import datetime, timedelta
from services.analytics_service import AnalyticsService
from models.tenant import Tenant

celery_app = Celery('tasks', broker='redis://localhost:6379/0')

@celery_app.task
def aggregate_hourly_analytics():
    """Run every hour to aggregate previous hour's data."""
    previous_hour = datetime.utcnow().replace(minute=0, second=0, microsecond=0) - timedelta(hours=1)

    # Get all active tenants
    tenants = await Tenant.find(Tenant.is_active == True).to_list()

    for tenant in tenants:
        try:
            await AnalyticsService.aggregate_hourly(tenant.tenant_id, previous_hour)
            logger.info(f"Aggregated hourly analytics for {tenant.tenant_id}")
        except Exception as e:
            logger.error(f"Failed to aggregate for {tenant.tenant_id}: {e}")

@celery_app.task
def aggregate_daily_analytics():
    """Run daily at midnight to aggregate previous day's data."""
    previous_day = datetime.utcnow().replace(hour=0, minute=0, second=0, microsecond=0) - timedelta(days=1)

    tenants = await Tenant.find(Tenant.is_active == True).to_list()

    for tenant in tenants:
        try:
            await AnalyticsService.aggregate_daily(tenant.tenant_id, previous_day)
            logger.info(f"Aggregated daily analytics for {tenant.tenant_id}")
        except Exception as e:
            logger.error(f"Failed to aggregate for {tenant.tenant_id}: {e}")

# Schedule tasks
celery_app.conf.beat_schedule = {
    'aggregate-hourly': {
        'task': 'tasks.analytics_tasks.aggregate_hourly_analytics',
        'schedule': 3600.0,  # Every hour
    },
    'aggregate-daily': {
        'task': 'tasks.analytics_tasks.aggregate_daily_analytics',
        'schedule': crontab(hour=0, minute=5),  # Daily at 00:05
    },
}
```

---

## Production Tips

### Dashboard Design

1. **Real-Time Overview**:
   - Current P95 latency (line chart, last 24 hours)
   - Live resolution rate (gauge)
   - Avg retrieval score (gauge with threshold indicator)
   - Hourly message volume (bar chart)

2. **Historical Trends**:
   - Resolution rate over 30 days (spot degradation)
   - Cost per day (catch unexpected spikes)
   - Top 10 questions (word cloud or table)

3. **Alerts Panel**:
   - Red: P95 latency > 5s for 10 minutes
   - Yellow: Avg retrieval score < 0.6
   - Red: Escalation rate > 30%

### Alerting Thresholds

```python
# Example: PagerDuty integration
from services.alerting_service import AlertingService

async def check_and_alert(tenant_id: str):
    analytics = await HourlyAnalytics.find(
        HourlyAnalytics.tenant_id == tenant_id,
        HourlyAnalytics.hour >= datetime.utcnow() - timedelta(hours=1)
    ).to_list()

    latest = analytics[-1]

    # Alert on high latency
    if latest.p95_latency > 5000:
        await AlertingService.send_alert(
            severity="critical",
            title="High P95 Latency",
            description=f"P95 latency is {latest.p95_latency}ms (threshold: 5000ms)",
            tenant_id=tenant_id
        )

    # Alert on low retrieval quality
    if latest.avg_retrieval_score < 0.5:
        await AlertingService.send_alert(
            severity="warning",
            title="Low Retrieval Quality",
            description=f"Avg retrieval score is {latest.avg_retrieval_score} (threshold: 0.5)",
            tenant_id=tenant_id
        )

    # Alert on high cost
    if latest.total_cost_usd > 50:
        await AlertingService.send_alert(
            severity="warning",
            title="High Hourly Cost",
            description=f"Cost is ${latest.total_cost_usd} in the last hour",
            tenant_id=tenant_id
        )
```

### Data Retention

```python
# Cleanup old logs (run monthly)
@celery_app.task
async def cleanup_old_logs():
    """Delete raw logs older than 90 days."""
    cutoff = datetime.utcnow() - timedelta(days=90)

    result = await ChatLog.find(
        ChatLog.timestamp < cutoff
    ).delete()

    logger.info(f"Deleted {result.deleted_count} old chat logs")
```

---

## Interview Q&A

### Q1: How do you measure RAG quality in production?

**Answer**: Use a combination of quantitative and qualitative metrics:

**Quantitative**:
1. **Retrieval Score**: Avg cosine similarity of top-k results (target > 0.7). Track P50/P95 to catch outliers.
2. **No-Result Rate**: % of queries with max score < 0.5 (indicates knowledge gaps).
3. **Latency**: P95 total latency (embedding + retrieval + generation). Target < 3s.
4. **Resolution Rate**: % of sessions not escalated to human agents.

**Qualitative**:
1. **User Feedback**: Thumbs up/down on responses. Target > 70% positive.
2. **Manual Spot Checks**: Review random sample of 50 conversations weekly.
3. **Knowledge Gap Analysis**: Questions with low retrieval scores reveal missing content.

**Implementation**: Log every request with trace_id, retrieval scores, latency breakdown, and user feedback. Aggregate hourly → daily for dashboards.

### Q2: What metrics actually matter for a RAG-based customer support SaaS?

**Answer**: Focus on these 5 metrics (in priority order):

1. **Resolution Rate (70%+ target)**: Ultimate success metric. If users still need humans, RAG isn't working.
2. **P95 Latency (< 3s target)**: Slow responses kill UX. Track breakdown (embedding: ~100ms, retrieval: ~200ms, GPT: ~2s).
3. **Avg Retrieval Score (> 0.7 target)**: Low scores mean irrelevant results → bad answers.
4. **Cost per Resolved Conversation (< $0.10 target)**: Track tokens per tenant to prevent bill shock.
5. **Knowledge Gap Count**: Questions with score < 0.5 need new docs.

**Red Flags**:
- Resolution rate dropping → content stale or model drift
- Latency spike → Qdrant overloaded or GPT throttling
- Cost spike → Prompt too long or token inefficiency

### Q3: What are the benefits of structured logging with trace IDs?

**Answer**: Structured logging transforms logs from text soup to queryable data:

**1. Request Tracing**: Follow a single request through all components:
```
trace_id=abc123 → embedding → retrieval (5 docs, avg_score=0.78) → GPT (2.1s) → total: 2.4s
```

**2. Root Cause Analysis**: Filter all logs by trace_id to debug specific failures:
```bash
grep "trace_id=abc123" logs/app.log | jq
```

**3. Performance Profiling**: Find slow stages across all requests:
```sql
SELECT AVG(generation_latency_ms) FROM logs WHERE event='gpt_streaming_complete' GROUP BY model
```

**4. Alerting**: Trigger alerts on structured fields (e.g., `total_latency_ms > 5000`).

**5. Analytics**: Feed structured logs to Elasticsearch/Grafana for real-time dashboards.

**Implementation**: Use loguru with `.bind(trace_id=..., event=..., duration_ms=...)` and JSON serialization.

### Q4: How do you identify knowledge gaps in your RAG system?

**Answer**: Multi-pronged approach:

**1. Low Retrieval Score Queries**:
```python
# Questions with avg retrieval score < 0.5
knowledge_gaps = await ChatLog.find(
    ChatLog.avg_retrieval_score < 0.5
).sort(-ChatLog.timestamp).limit(100).to_list()
```

**2. Escalation Analysis**: If user asks "talk to human", previous question likely had no good answer.

**3. Negative Feedback**: Thumbs down → retrieve original question + response for manual review.

**4. Question Clustering**: Use embeddings to cluster similar questions, find topics with high frequency but low avg score.

**5. Missing Document Detection**: Track which document IDs are never retrieved (dead content or bad metadata).

**Action Plan**: Weekly review of top 20 knowledge gaps → create/update docs → re-ingest → measure improvement.

### Q5: How do you handle cost tracking and prevent runaway token usage?

**Answer**: Multi-layer cost controls:

**1. Per-Request Tracking**:
```python
cost = (tokens_input * COST_PER_INPUT_TOKEN) + (tokens_output * COST_PER_OUTPUT_TOKEN)
await ChatLog.update_one({"trace_id": trace_id}, {"$set": {"cost_usd": cost}})
```

**2. Per-Tenant Budgets**:
```python
# Monthly budget per tenant
tenant_config = await TenantConfig.find_one({"tenant_id": tenant_id})
if tenant_config.monthly_spend >= tenant_config.monthly_budget:
    raise HTTPException(status_code=429, detail="Monthly budget exceeded")
```

**3. Token Limit Enforcement**:
```python
# Max tokens per request
if len(prompt_tokens) > 4000:
    # Truncate context or use cheaper model
    prompt_tokens = prompt_tokens[:4000]
```

**4. Cost Alerts**:
```python
# Alert if hourly cost > threshold
if hourly_cost > tenant_config.hourly_alert_threshold:
    await send_alert(f"High cost: ${hourly_cost} in last hour")
```

**5. Model Selection**: Use GPT-4o-mini (15x cheaper) by default, GPT-4o only for complex queries flagged by heuristics (e.g., multi-step reasoning).

**Dashboard**: Real-time cost per tenant, projected monthly cost, cost per resolved conversation.

---

**Next Steps**: Congratulations! You now have a production-grade Customer Support RAG SaaS with observability. In the next sections, we'll cover advanced topics like query optimization, A/B testing, and handling multi-language support.
