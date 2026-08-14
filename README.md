# Meridian Platform (Sovereign AI Showcase)

An event-driven, real-time AI platform featuring asynchronous dynamic tool orchestration, multi-modal generative workflows, vector RAG retrieval, and zero-trust multi-tenancy.

> **Note:** This repository serves as a public architectural showcase and technical specification. Core backend orchestration logic, database migrations, and production API keys are maintained in a private repository.

---

## 🏗️ System Architecture & Data Flow

```mermaid
flowchart TD
    A["Client App (WebSocket / Custom UI)"] --> B["FastAPI WebSocket Endpoint (/ws)"]
    B --> C["LLM Intent & ReAct Router Engine"]
    
    subgraph Parallel ["Concurrent Execution (asyncio.as_completed)"]
        D["Tavily Web Search Engine"]
        E["Stock Analytics Engine"]
        F["ComfyUI API (Workflow API)"]
    end
    
    C --> D
    C --> E
    C --> F
    
    D --> G["Real-Time Async Stream Feed"]
    E --> G
    F --> G
    
    G --> H["PostgreSQL / ChromaDB / Storage"]
    H --> I["Client Response Output"]

Request Pipeline Tracing

    WebSocket Ingestion: The client initiates a connection to the FastAPI /ws endpoint. Queries are ingested as JSON payloads.

    Intent Routing: A central LLM router (powered by models such as Gemma 4) analyzes user intent using a ReAct (Reason + Act) pattern to determine which tools to dispatch.

    Asynchronous Parallel Dispatch: Selected tools execute concurrently using Python's asyncio event loop.

    External Engine Execution: The agent dispatches HTTP/WebSocket requests to external engines (e.g., executing generative workflows via ComfyUI's /prompt API).

    Real-Time Streaming: Output buffers stream back to the client over the active WebSocket connection as individual tasks complete, eliminating head-of-line blocking.

🛠️ Complete Tech Stack

    Core Runtime & Language: Python 3.9+

    Backend Gateway: FastAPI + Uvicorn for low-latency asynchronous processing and WebSocket route handling.

    Concurrency Engine: Python asyncio (asyncio.as_completed stream queues).

    Vector & RAG Storage: ChromaDB for high-dimensional document vector embeddings and contextual retrieval.

    Relational Storage & Caching:

        PostgreSQL for tenant metadata, access logs, and key isolation.

        Redis for distributed caching and sliding-window rate limiting.

    External Integrations:

        ComfyUI API: Custom REST/WebSocket wrapper interfacing with /prompt and /history endpoints.

        Search & Analytics: Tavily Web Search and financial market data APIs.

    Zero-Trust Network: Multi-tenant architecture designed for zero-trust isolation (e.g., OpenZiti / zrok overlay support).

💻 Asynchronous Orchestration Implementation

The platform utilizes non-blocking async generator patterns to execute tools in parallel:
Python

import asyncio
from fastapi import FastAPI, WebSocket, WebSocketDisconnect

app = FastAPI()

async def run_tool_safe(tool_func, params: dict):
    """Encapsulates tool execution with fault isolation to prevent pipeline crashes."""
    try:
        return await tool_func(params)
    except Exception as exc:
        return f"[Tool Error] {tool_func.__name__} failed: {str(exc)}"

@app.websocket("/ws")
async def websocket_endpoint(websocket: WebSocket):
    await websocket.accept()
    try:
        while True:
            raw_data = await websocket.receive_text()
            
            # 1. Resolve tools from router
            selected_tools = await llm_router.resolve_intent(raw_data)
            
            # 2. Build non-blocking concurrent tasks
            tasks = [
                run_tool_safe(tool.executable, tool.args) 
                for tool in selected_tools
            ]
            
            # 3. Yield results to client as they complete
            for completed_future in asyncio.as_completed(tasks):
                result = await completed_future
                await websocket.send_text(result)

    except WebSocketDisconnect:
        await manager.disconnect(websocket)

🔒 Security & Data Isolation

    Multi-Tenancy: PostgreSQL schema separation per tenant with enforced row-level security policy checks.

    Authentication: JWT-based bearer authentication on administrative HTTP management endpoints.

    Rate Limiting: Managed at the gateway layer via Redis sliding-window algorithms to enforce fair usage quotas per API key.
