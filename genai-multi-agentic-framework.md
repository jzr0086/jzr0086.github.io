---
layout: default
title: Multi-Agent Framework Architecture
---

# Case Study: Decomposing a Monolith into a Multi-Agent Ecosystem

**Role:** AI Architect / Lead Engineer
**Tech Stack:** Python, LangChain (StructuredTools), LangGraph, FastAPI, Microservices Architecture

---

## Executive Summary
I led the architectural research and proof-of-concept for migrating a standalone, monolithic Customer Service AI application into a composable tool within a larger Multi-Agentic Framework.

The goal was to strip the domain-specific agent of non-core responsibilities (routing, state management, session handling) and transform it into a stateless, pure-logic microservice invokable by a central "Enterprise Orchestrator."

**Key Transformation:**
*   **From:** A heavy, standalone FastAPI application managing its own chat history, guardrails, and intent routing.
*   **To:** A lightweight, stateless `StructuredTool` that strictly performs RAG and response generation when invoked by the Orchestrator.

---

## 1. Architectural Evolution

### The "Before" State: Standalone Application
Originally, the Support Agent was a "thick" application. It had to own the entire lifecycle of a user interaction.
*   **Routing:** It had to decide if a user wanted to check an order or ask a policy question.
*   **State:** It managed session storage (S3/Redis) and conversation history.
*   **Guardrails:** It ran its own input/output safety checks.

**Pain Point:** This created duplication. Every new bot we built re-implemented state management and guardrails.

### The "After" State: Agent-as-a-Tool
In the new architecture, the Central Orchestrator (built on LangGraph) handles the "administrative" overhead. The Support Agent becomes a specialized worker.

| Responsibility | Old Owner (Standalone Agent) | New Owner (Orchestrator) | Benefit |
| :--- | :--- | :--- | :--- |
| **Routing** | Semantic Router | Central Orchestrator | Centralized intent classification across all enterprise tools. |
| **State Management** | Internal S3/DB | Orchestrator (Checkpointer) | The tool is now stateless; context is passed in arguments. |
| **Guardrails** | Local Validation | Global Guardrails | Consistent safety policy enforced before tools are even called. |
| **Streaming** | Local SSE Implementation | Orchestrator | Unified streaming experience for the client. |

---

## 2. Implementation: The Stateless Pattern

The core technical shift was defining the Agent's interface using Pydantic and LangChain's `StructuredTool`. Instead of looking up data itself, the Agent now expects all necessary context (User Info, Order History) to be passed as arguments.

### Defining the Schema
I defined a strict input schema that acts as the contract between the Orchestrator and the Tool.

```python
from langchain.tools import StructuredTool
from pydantic import BaseModel, Field

class SupportAgentInput(BaseModel):
    """Input schema for the Support Specialist Tool"""
    
    message: str = Field(
        description="The current user question to be answered."
    )
    conversation_history: list[dict] = Field(
        default_factory=list,
        description="Previous conversation messages for context."
    )
    order_context: dict = Field(
        default_factory=dict,
        description="The user's active orders and returns data."
    )
    user_metadata: dict = Field(
        default_factory=dict,
        description="User identifiers (session_id, user_id)."
    )
```

### The Logic Layer
The internal service was refactored to be purely functional. It takes inputs, performs RAG (Vector Search), formats the prompt, and returns a string. No side effects.

```python
async def support_agent_impl(
    message: str, 
    conversation_history: list, 
    order_context: dict, 
    user_metadata: dict
) -> str:
    """
    Core Logic:
    1. Search Policy Database (RAG)
    2. Format Order Context (Parse JSON to text)
    3. Generate LLM Response
    """
    # 1. RAG Retrieval
    policies = await retriever.aretrieve_text(message)

    # 2. Context Formatting
    formatted_orders = format_orders(order_context.get("orders", []))
    
    # 3. LLM Generation
    response = await llm.ainvoke(
        citation_rag_prompt, 
        query=message,
        context=policies,
        history=conversation_history
    )
    
    return response.content
```

---

## 3. Integration Patterns

I designed two methods for the Orchestrator to consume this new tool, allowing for flexibility in deployment.

### Option A: Direct Python Import (Monorepo)
For tight integration, the Orchestrator imports the tool directly.

```python
# In the Orchestrator
from support_package.tools import support_agent_tool

tools = [support_agent_tool, order_lookup_tool]
agent = create_react_agent(llm, tools=tools)
```

### Option B: HTTP Microservice (Distributed)
For decoupled deployment, I wrapped the tool in a lightweight FastAPI endpoint. The Orchestrator calls it via HTTP, allowing the Support Team to deploy updates without redeploying the Orchestrator.

```python
@StructuredTool.from_function
async def support_agent_http(message: str, ...):
    """Wrapper that calls the Support Service via HTTP"""
    async with httpx.AsyncClient() as client:
        response = await client.post(
            "http://support-service.internal/tool/invoke",
            json={
                "message": message,
                "order_context": order_context,
                # ...
            }
        )
    return response.json()["response"]
```

---

## 4. Key Results & Impact

*   **De-duplication of Code:** We removed ~40% of the codebase from the Support Agent (routing, state, and API overhead), allowing the team to focus purely on retrieval accuracy and prompt engineering.
*   **Universal Context:** Because the Orchestrator owns the state, the Support Agent now has access to context gathered by other agents (e.g., if a "Shipping Bot" finds a lost package, it passes that context to the Support Agent automatically).
*   **Scalability:** The stateless design allows the Support Agent to scale horizontally on serverless infrastructure (like AWS Lambda or K8s) with zero concern for sticky sessions or database connections.

### Why This Matters
This case study demonstrates a maturity in AI Engineering—moving beyond "building a chatbot" to "architecting an AI ecosystem." It shows an understanding of Service-Oriented Architecture (SOA) applied to Large Language Models, where agents are treated as deterministic, composable software components rather than black boxes.
