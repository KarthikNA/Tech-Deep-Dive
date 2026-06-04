# LLM-Powered Chatbot

## Problem Statement
Design a basic chatbot system that accepts user messages, sends them to a Large Language Model (LLM), and streams back coherent responses — supporting multi-turn conversations with context retention.

---

## Requirements

### Functional
- Users can send messages and receive responses from an LLM
- System maintains conversation history for multi-turn context
- Responses are streamed back in real time

### Non-Functional
- Low latency first token response
- Scalable to handle multiple concurrent users
- Conversation history persisted across sessions

---

## High-Level Architecture

![LLM Chatbot Architecture](images/chat-service/chat_service.png)

---

## Core Components

| Component | Responsibility |
|---|---|
| **Client** | Sends user messages, renders streamed responses |
| **API Gateway** | Auth, rate limiting, routes requests |
| **Chat Service** | Orchestrates conversation — builds prompt, calls LLM, persists messages |
| **Session Store (Redis)** | Caches recent conversation context for fast retrieval |
| **LLM Provider** | Generates responses (OpenAI, Anthropic, etc.) |
| **Message DB (PostgreSQL)** | Persists full conversation history |

---

## Data Flow

1. User sends a message from the client
2. API Gateway authenticates and forwards to the Chat Service
3. Chat Service fetches recent conversation history from Redis
4. Chat Service builds a prompt (system prompt + history + new message)
5. Prompt is sent to the LLM Provider; response is streamed back
6. Chat Service streams tokens to the client via WebSocket
7. Once complete, full message pair is persisted to PostgreSQL
8. Redis session is updated with the latest context window

---

## Trade-offs & Considerations

- **Context window limit** — older messages must be truncated or summarized as conversations grow long
- **Streaming vs. REST** — WebSockets give a better UX for streaming but add connection management complexity
- **LLM latency** — first-token latency dominates; choose providers with fast inference or self-host smaller models
- **Cost** — token usage scales with context length; trimming history aggressively reduces cost at the expense of coherence
