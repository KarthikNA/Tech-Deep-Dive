# LLM-Powered Chatbot - Overview

A 5-minute read covering the architecture of a production LLM chatbot. For the full design with deep dives, trade-offs, observability, and security, see [llm-chatbot.md](llm-chatbot.md).

---

## What We're Building

A conversational system where users send messages and get back coherent, grounded, safe responses from a Large Language Model (LLM). It needs to:

- Remember conversation history across turns
- Pull in relevant knowledge from a private corpus (RAG)
- Stream responses token-by-token for low perceived latency
- Block unsafe inputs and outputs
- Route between cheap and powerful models based on the query

---

## Architecture

![LLM Chatbot Architecture](../images/chat-service/chat_service.png)

The system has four logical layers:

- **Edge** - Client + API Gateway. Handles connections, auth, and traffic shaping.
- **Orchestration** - Chat Service. The brain - coordinates everything.
- **Augmentation** - Cache, RAG, Vector DB, Guardrails. Improves quality, cuts cost, enforces safety.
- **Inference** - Model Router + the LLMs themselves. Generates the response.

---

## The Components

### Client
Web, mobile, or embedded widget. Captures user input, opens a streaming connection, and renders tokens as they arrive.

### API Gateway
Single entry point for all client requests. Handles authentication, rate limiting, SSL termination, and proxies the streaming connection through to the Chat Service. Examples: Kong, NGINX, AWS API Gateway, or a purpose-built AI Gateway like Portkey or Cloudflare AI Gateway.

### Chat Service
The orchestrator. It is the only component that sees the whole request. It loads conversation history, decides whether the cache can serve the request, calls guardrails, runs RAG, builds the prompt, picks a model via the Router, streams the response back, and persists everything. Often built on FastAPI or Node.js, sometimes with LangGraph or LlamaIndex on top.

### Cache (Semantic Cache)
A Redis-backed cache that doesn't just match strings - it matches *meaning*. The incoming query is converted to a vector embedding and compared against past queries using cosine similarity. If a past query is close enough (say > 0.9), its stored response is returned without ever calling the LLM. Lookup is ~1-5ms using an HNSW index. This is the single biggest cost lever in the system.

### RAG + Vector DB
**Retrieval-Augmented Generation** grounds the LLM in your private knowledge. Documents are pre-chunked and embedded into a Vector DB (Qdrant, Pinecone, pgvector). At query time, the user's question is embedded, the top-K most similar chunks are retrieved (often using hybrid dense + keyword search), reranked by a cross-encoder for precision, and injected into the prompt as context. Result: fewer hallucinations, answers grounded in real documents, and the ability to cite sources.

### Guardrails
A content moderation layer powered by a classifier model. It runs **twice**:
- **Input guardrails** check the user's message for prompt injection, jailbreaks, PII, and off-topic content before it reaches the LLM.
- **Output guardrails** check the LLM's response for toxicity, leaked PII, hallucinations, and policy violations before it reaches the user.

Without this, you're trusting the model's own safety training - which is inconsistent. Examples: Llama Guard, NeMo Guardrails, AWS Bedrock Guardrails, Azure AI Content Safety.

### Model Router
Decides which LLM should handle each request. A simple FAQ doesn't need GPT-5 - a small fast model is fine. A complex reasoning task does. The router scores requests on signals like task complexity, cost budget, latency needs, and data sensitivity, then picks accordingly. Sensitive data (PII, regulated content) is routed to self-hosted models so it never leaves your infrastructure.

### LLMs
Two flavours:
- **Self-hosted** (open-weight models like Llama 4, Mistral, Qwen, DeepSeek) running on your own GPUs via vLLM or TGI. Full data control, fixed cost, operational burden.
- **External** (OpenAI GPT-5, Anthropic Claude Sonnet 4.5, Google Gemini 2.5) accessed via API. State-of-the-art, fully managed, pay-per-token.

Most production systems use both - external for general traffic, self-hosted for sensitive or high-volume workloads.

---

## Request Flow

For every user message:

1. **Client** sends a `POST /v1/chat` with the message and session ID
2. **API Gateway** authenticates and forwards to the Chat Service
3. **Chat Service** loads session history from Redis/Postgres
4. **Semantic Cache** check - if hit, stream cached response and stop
5. On miss, **Input Guardrails** validate the message (in parallel with step 6)
6. **RAG** retrieves relevant chunks from the Vector DB
7. **Chat Service** builds the final prompt: system prompt + RAG context + history + user message
8. **Model Router** picks the right LLM and dispatches the request
9. **LLM** generates a response, streaming tokens back
10. **Output Guardrails** inspect tokens as they stream; bad output is cut off
11. Tokens are forwarded to the client via SSE in real time
12. Conversation is persisted, cache is updated, telemetry is emitted

Steps 1-2 are typically <50ms. Steps 4-6 run partly in parallel and take ~100-300ms. Step 9 dominates - LLMs take hundreds of ms to first token and several seconds for a full response.

---

## Key Trade-offs to Know

- **RAG vs. Fine-tuning** - RAG keeps knowledge external and updatable; fine-tuning bakes it in. RAG wins for most production cases because knowledge changes faster than you can retrain.
- **Cache threshold** - Set it too high and you barely hit the cache. Set it too low and you serve wrong answers. Tune empirically.
- **Streaming vs. buffered** - Streaming dramatically improves perceived latency but complicates output guardrails (they must work on partial responses).
- **Self-hosted vs. external LLMs** - Self-hosted means data control and predictable cost but operational burden. External means state-of-the-art capabilities but per-token cost and third-party data exposure.
- **Quality vs. cost in routing** - The cheapest model for everything degrades quality; the most capable model for everything is unaffordable. The router's job is to find the right one per request.

---

## Where to Go Deeper

See [llm-chatbot](llm-chatbot.md) for:
- Concrete SLOs and latency budgets
- Component deep dives with technology comparisons
- Full API design (endpoints, status codes, error codes)
- Observability, alerting, evaluation, security, and capacity planning
- Twelve detailed trade-off discussions
