---
layout: default
title: GenAI Latency & Cost Optimization
---

# Case Study: GenAI Economics & Latency Optimization

**Role:** Senior ML Engineer / ML Systems Engineer
**Tech Stack:** Python 3.11, FastAPI, Docker, Prometheus, OpenTelemetry, Enterprise LLM Gateway

---

## Executive Summary
I architected and deployed a semantic caching solution for a high-traffic Enterprise Support Agent. The primary goal was to decouple "static" instruction tokens from "dynamic" user context to leverage the **Prompt Caching** capabilities of modern LLMs (e.g., Anthropic Claude / OpenAI).

**Key Outcomes:**
* **Financial Impact:** Reduced inference costs by **61% per request** (driven by a 90% discount on cached input tokens).
* **Performance:** Achieved a **68.7% cache hit rate**, improving P95 latency by **34.8%**.
* **Scalability:** Increased system throughput by **29%** under peak load (75 concurrent users).
* **Efficiency:** Reduced static context retrieval overhead from ~1.5s to ~500ms via asynchronous parallelization.

---

## 1. The Challenge: The "Concat" Bottleneck
The legacy system used a string-based prompt template that concatenated all context into a single payload for every request.

* **Total Prompt Size:** ~7,200 tokens per request.
* **Static Overhead:** ~3,500 tokens (System Role, Persona, Guardrails) were identical across all calls.
* **The Inefficiency:** The LLM provider had to process these 3,500 static tokens from scratch for every single interaction. This resulted in high "Time to First Token" (TTFT) and unnecessary usage costs, as we were paying full price to re-process static instructions.

---

## 2. Solution Architecture

### Message-Based API & Cache Breakpoints
I refactored the prompt construction pipeline from raw string concatenation to a structured `ChatMessage` object model. This allowed us to inject specific metadata headers (`cache_breakpoint`) to instruct the LLM Gateway where to "freeze" the context.

**The "Split" Strategy:**
1.  **Block A (Static - Cached):** System instructions, JSON schemas, and Style Guidelines.
2.  **Block B (Dynamic - Uncached):** Current timestamp, RAG retrieval results (Policy docs), and User Conversation History.

#### Code Implementation: The Prompt Builder
I built a `PromptBuilder` service that logically separates these concerns. The `cache_breakpoint` flag allows the Gateway to hash and store the prefix server-side with a 5-minute TTL.

```python
# Refactored Prompt Construction Strategy
class PromptBuilder:
    def build_messages(self, query: str, context: dict):
        return [
            # 1. STATIC SYSTEM BLOCK (Cached)
            # ~3,500 tokens of immutable instructions.
            # We inject a custom field to signal the caching layer.
            ChatMessage(
                role=MessageRole.SYSTEM,
                content=BASE_PROMPT_CONTENT, 
                additional_kwargs={"custom_fields": {"cache_breakpoint": {}}}
            ),
            
            # 2. DYNAMIC CONTEXT BLOCK (Uncached)
            # Injected at runtime (Datetime, RAG results)
            ChatMessage(
                role=MessageRole.SYSTEM,
                content=f"<current_datetime>{context['now']}</current_datetime>..."
            ),
            
            # 3. USER QUERY
            ChatMessage(
                role=MessageRole.USER,
                content=query
            )
        ]
```

## Asynchronous Parallelization

During the refactor, I identified that context retrieval (fetching order history, RAG policy documents, and chat history) was executing sequentially. I optimized this using `asyncio.gather`.

**Before:** Sequential execution (~1.5s total overhead).  
**After:** Parallel execution (~500ms total overhead).

```python
# Optimization: Fetching context in parallel
chat_history, metadata, policies = await asyncio.gather(
    chat_message_db.get_messages(session_id),
    metadata_db.get(user_id),
    retriever.aretrieve_text(query)
)
```

## 3. Performance Data & Load Testing

Comprehensive load testing was performed using Locust against a DEV environment simulating production traffic patterns.

### Latency Reduction (Streaming Endpoint)

| Metric | 1 User (No Cache) | 1 User (With Cache) | Improvement |
| :--- | :--- | :--- | :--- |
| **Median** | 12,000 ms | 9,700 ms | ✅ 19.2% Faster |
| **P95** | 23,000 ms | 15,000 ms | ✅ 34.8% Faster |
| **Average** | 13,509 ms | 10,507 ms | ✅ 22.2% Faster |

### Stability Under Load

At peak traffic (75 concurrent streaming users), the system demonstrated massive stability gains:

*   **Throughput:** 29% increase in Requests Per Second (RPS).
*   **Reliability:** Zero failures across all caching-enabled test scenarios.

## 4. Financial Modeling & Cost Analysis

One of the critical drivers for this project was "GenAI Economics." By maximizing the cache hit rate, we effectively moved the majority of our traffic to a deeply discounted pricing tier.

| Metric | Value | Notes |
| :--- | :--- | :--- |
| **Static Prompt Size** | ~3,500 tokens | The "Fixed Cost" per call |
| **Cache Hit Rate** | 68.7% | Percentage of tokens served from hot cache |
| **Unit Cost** | $0.0083 | Down from $0.0216 per request |

### The Math

*   **Input Token Pricing:** Standard rate ($3.00/1M) vs. Cached rate ($0.30/1M).
*   **Result:** A 61% reduction in total cost per request. For a service handling 50k+ requests/month, this reduced projected infrastructure spend by thousands of dollars annually.

## 5. Engineering Insights: Shifting the Bottleneck

During load testing, I observed a counter-intuitive phenomenon: while LLM processing time dropped, the Vector DB and Guardrail services experienced increased pressure.

### Root Cause Analysis

*   **The "Good" Problem:** Because the LLM was returning tokens ~68% faster, the application server had to process streams at a much higher velocity.
*   **Bottleneck Migration:** The bottleneck shifted from external LLM inference (network I/O) to internal CPU-bound tasks (Guardrail validation on chunks) and database I/O (Semantic Search).

### Mitigation

This revealed that our downstream infrastructure needed to scale up to match the new speed of the LLM. It highlighted the importance of viewing GenAI latency as a holistic pipeline rather than just model inference time.

## 6. Observability & Instrumentation

I implemented custom Prometheus metrics to track cache performance in real-time, specifically monitoring `cache_hit_rate` and `cached_tokens`.

### Code Snippet: Streaming Instrumentation

```python
async def astream_chat(self, messages: list, **kwargs):
    # Stream chunks from LLM
    agen = await self.llm.astream_chat(messages, **kwargs)
    
    async for response in agen:
        yield response
        last_response = response

    # Post-stream: Record cache metrics to Prometheus
    if last_response:
        usage = getattr(last_response.raw, 'usage', None)
        cached_tokens = getattr(usage, 'cache_read_tokens', 0)
        record_metric(llm_prompt_cached_tokens, cached_tokens, call_type="streaming")
```
