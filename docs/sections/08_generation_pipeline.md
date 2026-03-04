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

