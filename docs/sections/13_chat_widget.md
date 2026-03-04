# Section 13: Chat Widget & Frontend Integration

## WHY: Embeddable, User-Friendly Chat Interface

The chat widget is your product's public face. Customers need a seamless way to interact with your AI assistant without leaving their current page. Key requirements:

1. **Easy Integration**: Drop a single script tag into any website
2. **Isolation**: Widget shouldn't conflict with host page styles or JavaScript
3. **Real-time Streaming**: Show AI responses as they generate (better UX than waiting)
4. **Customization**: Match the widget to each tenant's brand
5. **Cross-Origin Security**: Handle authentication and CORS properly

A well-designed widget can be embedded in Shopify, WordPress, React apps, or plain HTML sites with zero friction.

---

## WHAT: Embeddable Widget Architecture

### Core Components

1. **Embed Snippet**: Lightweight JavaScript loader (< 5KB)
2. **Widget Application**: Full chat UI in an isolated iframe
3. **SSE Streaming**: Server-Sent Events for real-time AI responses
4. **Configuration API**: Per-tenant theming and behavior settings
5. **Session Management**: Handle auth tokens and conversation state

### Widget Communication Flow

```
┌─────────────────────────────────────────────────────────────┐
│                    Customer Website                          │
│  ┌────────────────────────────────────────────────────┐    │
│  │              Embed Script (widget-loader.js)       │    │
│  │  • Creates iframe with widget URL + tenant_id      │    │
│  │  • Passes config via URL params                    │    │
│  │  • Handles window.postMessage for cross-frame comm │    │
│  └─────────────┬──────────────────────────────────────┘    │
│                │                                             │
│  ┌─────────────▼──────────────────────────────────────┐    │
│  │           IFrame (widget.html - isolated)          │    │
│  │  ┌──────────────────────────────────────────────┐  │    │
│  │  │  Widget UI: Chat messages, input, buttons    │  │    │
│  │  │  • Loads config from /widget/config API      │  │    │
│  │  │  • Sends messages to /chat/stream endpoint   │  │    │
│  │  │  • Receives SSE stream events                │  │    │
│  │  └──────────────────┬───────────────────────────┘  │    │
│  └─────────────────────┼──────────────────────────────┘    │
└────────────────────────┼─────────────────────────────────────┘
                         │
                         │ HTTPS + API Key in headers
                         │
        ┌────────────────▼────────────────────┐
        │      FastAPI Backend                │
        │                                      │
        │  GET /widget/config/{tenant_id}     │
        │    → Widget settings (colors, logo) │
        │                                      │
        │  POST /chat/stream                  │
        │    → SSE: data: {"delta": "Hello"}  │
        │    → data: {"delta": " world"}      │
        │    → data: [DONE]                   │
        │                                      │
        │  POST /chat/feedback                │
        │    → Store thumbs up/down           │
        └─────────────────────────────────────┘
```

### SSE vs WebSockets: When to Use What?

**Server-Sent Events (SSE)**:
- **Best for**: Unidirectional server→client streaming (perfect for AI responses)
- **Advantages**: Built into browsers, simpler than WebSockets, auto-reconnect, HTTP/2 compatible
- **Disadvantages**: One-way only (use separate POST for client→server messages)

**WebSockets**:
- **Best for**: Bidirectional, low-latency communication (e.g., collaborative editing)
- **Advantages**: Full-duplex, lower overhead for frequent back-and-forth
- **Disadvantages**: More complex, requires connection state management

**For chat widgets**: **SSE wins** because:
- User sends message via POST → backend streams response via SSE
- Simpler infrastructure (no WebSocket server state)
- Works through most corporate firewalls
- Native browser EventSource API

---

## HOW: Implementation

### 1. Widget Configuration Model & API

Store per-tenant widget settings in MongoDB:

```python
# models/widget_config.py
from beanie import Document
from pydantic import Field, HttpUrl
from typing import Optional

class WidgetConfig(Document):
    tenant_id: str = Field(..., index=True)

    # Theming
    primary_color: str = Field(default="#007bff")
    secondary_color: str = Field(default="#6c757d")
    font_family: str = Field(default="'Inter', sans-serif")
    position: str = Field(default="bottom-right")  # bottom-right, bottom-left, etc.

    # Branding
    company_name: str = Field(default="Support Assistant")
    logo_url: Optional[HttpUrl] = None
    greeting_message: str = Field(default="Hi! How can I help you today?")

    # Behavior
    suggested_questions: list[str] = Field(default_factory=list)
    placeholder_text: str = Field(default="Type your message...")
    show_branding: bool = Field(default=True)  # "Powered by YourSaaS"

    # Security
    allowed_domains: list[str] = Field(default_factory=list)  # CORS whitelist

    class Settings:
        name = "widget_configs"
        indexes = ["tenant_id"]
```

```python
# routes/widget_api.py
from fastapi import APIRouter, HTTPException, Header
from models.widget_config import WidgetConfig
from auth.api_key import verify_api_key

router = APIRouter(prefix="/widget", tags=["widget"])

@router.get("/config/{tenant_id}")
async def get_widget_config(
    tenant_id: str,
    x_api_key: str = Header(...)
):
    """Public endpoint for widget to fetch configuration."""
    # Verify API key matches tenant
    verified_tenant = await verify_api_key(x_api_key)
    if verified_tenant != tenant_id:
        raise HTTPException(status_code=403, detail="Invalid API key")

    config = await WidgetConfig.find_one(WidgetConfig.tenant_id == tenant_id)
    if not config:
        # Return default config
        config = WidgetConfig(tenant_id=tenant_id)

    return config.dict(exclude={"id", "allowed_domains"})  # Don't expose internal fields

@router.put("/config")
async def update_widget_config(
    config: WidgetConfig,
    x_api_key: str = Header(...)
):
    """Tenant admin updates widget settings."""
    verified_tenant = await verify_api_key(x_api_key)
    if verified_tenant != config.tenant_id:
        raise HTTPException(status_code=403, detail="Cannot modify other tenant's config")

    existing = await WidgetConfig.find_one(WidgetConfig.tenant_id == config.tenant_id)
    if existing:
        await existing.set(config.dict(exclude_unset=True))
    else:
        await config.insert()

    return {"status": "success"}
```

### 2. SSE Streaming Endpoint

FastAPI's `StreamingResponse` with async generators:

```python
# routes/chat_api.py
from fastapi import APIRouter, Header
from fastapi.responses import StreamingResponse
from pydantic import BaseModel
from services.chat_service import ChatService
import json
import asyncio

router = APIRouter(prefix="/chat", tags=["chat"])

class ChatRequest(BaseModel):
    tenant_id: str
    session_id: str
    message: str
    conversation_history: list[dict] = []

@router.post("/stream")
async def stream_chat_response(
    request: ChatRequest,
    x_api_key: str = Header(...)
):
    """Stream AI response using Server-Sent Events."""
    # Verify API key
    verified_tenant = await verify_api_key(x_api_key)
    if verified_tenant != request.tenant_id:
        raise HTTPException(status_code=403, detail="Invalid API key")

    async def event_generator():
        """Async generator that yields SSE-formatted data."""
        try:
            chat_service = ChatService(tenant_id=request.tenant_id)

            # Send initial event
            yield f"data: {json.dumps({'type': 'start'})}\n\n"

            # Stream tokens from GPT
            full_response = ""
            async for chunk in chat_service.stream_response(
                message=request.message,
                session_id=request.session_id,
                history=request.conversation_history
            ):
                if chunk.get("type") == "token":
                    token = chunk["content"]
                    full_response += token
                    yield f"data: {json.dumps({'type': 'token', 'content': token})}\n\n"

                elif chunk.get("type") == "sources":
                    # Send retrieved sources
                    yield f"data: {json.dumps({'type': 'sources', 'sources': chunk['sources']})}\n\n"

            # Send completion event
            yield f"data: {json.dumps({'type': 'done', 'message_id': chunk.get('message_id')})}\n\n"

        except Exception as e:
            # Send error to client
            yield f"data: {json.dumps({'type': 'error', 'message': str(e)})}\n\n"

    return StreamingResponse(
        event_generator(),
        media_type="text/event-stream",
        headers={
            "Cache-Control": "no-cache",
            "Connection": "keep-alive",
            "X-Accel-Buffering": "no"  # Disable nginx buffering
        }
    )
```

### 3. ChatService with Streaming

```python
# services/chat_service.py
from services.rag_service import RAGService
from clients.openai_client import OpenAIClient
from models.conversation import Conversation, Message
from typing import AsyncIterator

class ChatService:
    def __init__(self, tenant_id: str):
        self.tenant_id = tenant_id
        self.rag = RAGService(tenant_id)
        self.openai = OpenAIClient()

    async def stream_response(
        self,
        message: str,
        session_id: str,
        history: list[dict]
    ) -> AsyncIterator[dict]:
        """Stream chat response with RAG context."""
        # 1. Retrieve relevant context
        retrieved_docs = await self.rag.retrieve(message, top_k=5)

        # Yield sources first
        yield {
            "type": "sources",
            "sources": [
                {"title": doc.metadata.get("title"), "score": doc.score}
                for doc in retrieved_docs
            ]
        }

        # 2. Build prompt with context
        context = "\n\n".join([doc.content for doc in retrieved_docs])
        system_prompt = f"""You are a helpful customer support assistant.
Use the following knowledge base context to answer questions:

{context}

If the answer isn't in the context, say so politely."""

        messages = [
            {"role": "system", "content": system_prompt},
            *history,
            {"role": "user", "content": message}
        ]

        # 3. Stream from GPT
        full_response = ""
        async for token in self.openai.stream_chat_completion(messages):
            full_response += token
            yield {"type": "token", "content": token}

        # 4. Save to conversation history
        conversation = await Conversation.find_one(
            Conversation.session_id == session_id
        )
        if not conversation:
            conversation = Conversation(
                tenant_id=self.tenant_id,
                session_id=session_id,
                messages=[]
            )

        conversation.messages.append(
            Message(role="user", content=message)
        )
        conversation.messages.append(
            Message(role="assistant", content=full_response)
        )
        await conversation.save()

        yield {"type": "done", "message_id": str(conversation.messages[-1].id)}
```

### 4. JavaScript Widget Loader (Embed Snippet)

This is what customers paste into their website:

```html
<!-- Embed snippet: widget-loader.js -->
<script>
(function() {
  // Configuration
  const WIDGET_HOST = 'https://your-saas.com';
  const TENANT_ID = 'tenant_abc123';  // Unique per customer
  const API_KEY = 'sk_live_xyz789';    // Public API key (read-only)

  // Create iframe container
  const container = document.createElement('div');
  container.id = 'support-widget-container';
  container.style.cssText = `
    position: fixed;
    bottom: 20px;
    right: 20px;
    width: 400px;
    height: 600px;
    z-index: 999999;
    box-shadow: 0 4px 12px rgba(0,0,0,0.15);
    border-radius: 12px;
    overflow: hidden;
    display: none;
  `;

  // Create iframe
  const iframe = document.createElement('iframe');
  iframe.src = `${WIDGET_HOST}/widget.html?tenant_id=${TENANT_ID}&api_key=${API_KEY}`;
  iframe.style.cssText = 'width: 100%; height: 100%; border: none;';
  iframe.allow = 'clipboard-write';

  container.appendChild(iframe);

  // Create launcher button
  const launcher = document.createElement('button');
  launcher.id = 'support-widget-launcher';
  launcher.innerHTML = `
    <svg width="24" height="24" viewBox="0 0 24 24" fill="none" stroke="currentColor">
      <path d="M21 11.5a8.38 8.38 0 0 1-.9 3.8 8.5 8.5 0 0 1-7.6 4.7 8.38 8.38 0 0 1-3.8-.9L3 21l1.9-5.7a8.38 8.38 0 0 1-.9-3.8 8.5 8.5 0 0 1 4.7-7.6 8.38 8.38 0 0 1 3.8-.9h.5a8.48 8.48 0 0 1 8 8v.5z"/>
    </svg>
  `;
  launcher.style.cssText = `
    position: fixed;
    bottom: 20px;
    right: 20px;
    width: 60px;
    height: 60px;
    border-radius: 50%;
    background: #007bff;
    color: white;
    border: none;
    cursor: pointer;
    box-shadow: 0 4px 12px rgba(0,0,0,0.15);
    z-index: 999998;
    display: flex;
    align-items: center;
    justify-content: center;
  `;

  // Toggle widget
  launcher.addEventListener('click', () => {
    const isVisible = container.style.display === 'block';
    container.style.display = isVisible ? 'none' : 'block';
    launcher.style.display = isVisible ? 'flex' : 'none';
  });

  // Listen for close message from iframe
  window.addEventListener('message', (event) => {
    if (event.origin !== WIDGET_HOST) return;
    if (event.data.type === 'close-widget') {
      container.style.display = 'none';
      launcher.style.display = 'flex';
    }
  });

  // Append to page
  document.body.appendChild(container);
  document.body.appendChild(launcher);
})();
</script>
```

### 5. Widget Frontend (Simplified)

Inside the iframe (`widget.html`):

```html
<!DOCTYPE html>
<html>
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Support Widget</title>
  <style>
    body {
      margin: 0;
      font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', sans-serif;
      height: 100vh;
      display: flex;
      flex-direction: column;
    }
    #chat-header {
      background: var(--primary-color, #007bff);
      color: white;
      padding: 16px;
      display: flex;
      justify-content: space-between;
      align-items: center;
    }
    #chat-messages {
      flex: 1;
      overflow-y: auto;
      padding: 16px;
      background: #f5f5f5;
    }
    .message {
      margin-bottom: 12px;
      padding: 10px;
      border-radius: 8px;
      max-width: 80%;
    }
    .message.user {
      background: var(--primary-color, #007bff);
      color: white;
      margin-left: auto;
    }
    .message.assistant {
      background: white;
    }
    #chat-input-container {
      padding: 16px;
      background: white;
      border-top: 1px solid #ddd;
    }
    #chat-input {
      width: 100%;
      padding: 10px;
      border: 1px solid #ddd;
      border-radius: 4px;
      font-size: 14px;
    }
  </style>
</head>
<body>
  <div id="chat-header">
    <span id="company-name">Support</span>
    <button id="close-btn">✕</button>
  </div>

  <div id="chat-messages">
    <div class="message assistant" id="greeting"></div>
  </div>

  <div id="chat-input-container">
    <input type="text" id="chat-input" placeholder="Type your message..." />
  </div>

  <script>
    // Parse URL params
    const params = new URLSearchParams(window.location.search);
    const TENANT_ID = params.get('tenant_id');
    const API_KEY = params.get('api_key');
    const API_BASE = window.location.origin;

    let config = {};
    let sessionId = `session_${Date.now()}_${Math.random()}`;
    let conversationHistory = [];

    // Fetch widget config
    fetch(`${API_BASE}/widget/config/${TENANT_ID}`, {
      headers: { 'X-API-Key': API_KEY }
    })
    .then(r => r.json())
    .then(data => {
      config = data;
      // Apply theming
      document.documentElement.style.setProperty('--primary-color', config.primary_color);
      document.getElementById('company-name').textContent = config.company_name;
      document.getElementById('greeting').textContent = config.greeting_message;
    });

    // Close button
    document.getElementById('close-btn').addEventListener('click', () => {
      window.parent.postMessage({ type: 'close-widget' }, '*');
    });

    // Send message
    document.getElementById('chat-input').addEventListener('keypress', async (e) => {
      if (e.key !== 'Enter') return;

      const input = e.target;
      const message = input.value.trim();
      if (!message) return;

      // Display user message
      addMessage('user', message);
      conversationHistory.push({ role: 'user', content: message });
      input.value = '';

      // Stream assistant response
      const assistantMsgEl = addMessage('assistant', '');

      const eventSource = new EventSource(`${API_BASE}/chat/stream`, {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json',
          'X-API-Key': API_KEY
        },
        body: JSON.stringify({
          tenant_id: TENANT_ID,
          session_id: sessionId,
          message: message,
          conversation_history: conversationHistory
        })
      });

      // Note: Native EventSource doesn't support POST, so use fetch with streaming
      const response = await fetch(`${API_BASE}/chat/stream`, {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json',
          'X-API-Key': API_KEY
        },
        body: JSON.stringify({
          tenant_id: TENANT_ID,
          session_id: sessionId,
          message: message,
          conversation_history: conversationHistory
        })
      });

      const reader = response.body.getReader();
      const decoder = new TextDecoder();

      while (true) {
        const { done, value } = await reader.read();
        if (done) break;

        const chunk = decoder.decode(value);
        const lines = chunk.split('\n');

        for (const line of lines) {
          if (line.startsWith('data: ')) {
            const data = JSON.parse(line.slice(6));
            if (data.type === 'token') {
              assistantMsgEl.textContent += data.content;
            }
          }
        }
      }

      conversationHistory.push({ role: 'assistant', content: assistantMsgEl.textContent });
    });

    function addMessage(role, content) {
      const messagesEl = document.getElementById('chat-messages');
      const msgEl = document.createElement('div');
      msgEl.className = `message ${role}`;
      msgEl.textContent = content;
      messagesEl.appendChild(msgEl);
      messagesEl.scrollTop = messagesEl.scrollHeight;
      return msgEl;
    }
  </script>
</body>
</html>
```

---

## Production Considerations

### Security

1. **API Key Scope**: Use separate read-only keys for widget (can't modify data)
2. **CORS**: Validate `allowed_domains` in widget config
3. **Rate Limiting**: Per-tenant and per-session limits to prevent abuse
4. **XSS Protection**: Sanitize all user input before rendering

### Performance

1. **CDN**: Host widget-loader.js on CDN (cache forever, version in filename)
2. **Lazy Loading**: Only load full widget UI when launcher is clicked
3. **Connection Pooling**: Reuse SSE connections for multiple messages in session

### Browser Compatibility

- SSE works in all modern browsers (IE11 needs polyfill)
- Fallback to long-polling for older browsers
- Mobile considerations: adjust widget size for responsive design

---

## Interview Q&A

### Q1: Why use Server-Sent Events instead of WebSockets for chat streaming?

**Answer**: SSE is simpler and sufficient for chat use cases:

1. **Unidirectional**: Chat is naturally request-response (user sends message → AI streams response). SSE handles server→client perfectly.
2. **Simpler Infrastructure**: No WebSocket server state, works over HTTP/2, auto-reconnects on disconnect.
3. **Better for AI Streaming**: Each user message is a new HTTP POST, then backend streams response via SSE. Clean separation.
4. **Firewall Friendly**: SSE is just HTTP GET, passes through corporate proxies that block WebSocket upgrades.

**When to use WebSockets**: Collaborative features (multiple users editing same doc), real-time presence indicators, or high-frequency bidirectional updates.

### Q2: How do you embed a widget in any website without conflicts?

**Answer**: Use iframe isolation + minimal loader script:

1. **IFrame Sandbox**: Widget UI lives in iframe with separate DOM/CSS scope. Host page styles can't leak in.
2. **Lightweight Loader**: Embed script is < 5KB, just creates iframe and launcher button. Zero dependencies.
3. **postMessage API**: Cross-frame communication without security risks.
4. **High z-index**: Position fixed with z-index: 999999 to stay on top.
5. **Namespacing**: All IDs/classes prefixed (`support-widget-*`) to avoid collisions.

**Testing**: Install on WordPress, Shopify, React SPA, and plain HTML to verify compatibility.

### Q3: How do you handle widget customization at scale (1000s of tenants)?

**Answer**: Per-tenant config with caching:

1. **Config Model**: Store theming (colors, fonts, position) in MongoDB per tenant.
2. **API Endpoint**: Widget fetches config on load via `/widget/config/{tenant_id}`.
3. **Redis Cache**: Cache config for 5 minutes (rarely changes). Invalidate on update.
4. **CSS Variables**: Use `--primary-color`, `--font-family` in widget CSS, set dynamically via JS.
5. **Admin UI**: Dashboard for tenants to preview and update widget settings in real-time.

**Performance**: Config fetch adds ~50ms on widget load (acceptable), subsequent messages don't refetch.

### Q4: How do you handle SSE connection failures and retries?

**Answer**: Built-in retry + client-side error handling:

1. **Native Reconnect**: EventSource API auto-retries with exponential backoff (3s, 6s, 12s...).
2. **Server Heartbeats**: Send `: keepalive\n\n` comment every 30s to detect dead connections.
3. **Client Timeout**: If no data for 60s, close and show "Connection lost" UI.
4. **Error Events**: Listen for `onerror` in EventSource, display user-friendly message.
5. **Fallback**: Degrade to polling (POST every 2s) if SSE repeatedly fails.

**Monitoring**: Alert if SSE error rate > 5% or avg connection duration < 10s (indicates network issues).

### Q5: How do you prevent widget abuse (spam, DDoS)?

**Answer**: Multi-layer rate limiting:

1. **Per-Session Limit**: Max 20 messages per session (reset after 1 hour).
2. **Per-Tenant Limit**: Max 1000 messages/hour across all sessions (protect shared resources).
3. **IP-Based Throttling**: Max 100 messages/hour per IP (catch bots without sessions).
4. **CAPTCHA**: Trigger after 10 messages in 5 minutes from same session.
5. **Token Cost Tracking**: Auto-pause widget if tenant exceeds token budget (billing safeguard).

**Implementation**: Use Redis sorted sets to track message counts with sliding window.

---

**Next Steps**: In Section 14, we'll cover analytics and observability—how to measure if your RAG system is actually helping customers and where to improve.
