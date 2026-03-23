# Integration Architecture Patterns

This reference covers the major patterns for integrating AI into enterprise systems. Use the decision tree at the end to select the right pattern for a given situation.

## Table of Contents
1. MCP Server Pattern
2. API Gateway / Middleware Pattern
3. RAG (Retrieval-Augmented Generation) Pattern
4. Event-Driven Agent Pattern
5. Orchestrator / Multi-Agent Pattern
6. Sidecar / Copilot Pattern
7. Batch Processing Pattern
8. Decision Tree

---

## 1. MCP Server Pattern

**When to use:** You want Claude (or another LLM with MCP support) to directly interact with an external system — read data, take actions, or both.

**How it works:** An MCP server exposes "tools" that the LLM can call. Each tool maps to an API endpoint or database query on the target system. The LLM decides when and how to call tools based on the user's request.

**Best for:**
- Connecting Claude Code/Cowork to internal APIs, databases, or SaaS tools
- Replacing manual copy-paste workflows where someone looks up data in one system to use in another
- Giving AI agents the ability to take actions (create tickets, send messages, update records)

**Architecture:**
```
User → Claude → MCP Server → [Auth Layer] → Enterprise System
                    ↕
              Tool Definitions
              (list, read, create, update)
```

**Key decisions:**
- **Transport**: stdio (local) vs SSE/HTTP (remote). Use stdio for local dev, SSE for shared/deployed servers.
- **Tool granularity**: One tool per API endpoint is too fine-grained; one tool for everything is too coarse. Group by user intent: "search_customers", "create_ticket", "get_order_status".
- **Auth forwarding**: Does the MCP server use its own service account, or forward the user's credentials? Service accounts are simpler but have audit implications.

**Pitfalls:**
- Tool descriptions matter enormously — the LLM uses them to decide when to call a tool. Vague descriptions lead to wrong tool selection.
- Rate limiting must happen at the MCP server level, not just the upstream API, to prevent the LLM from hammering endpoints.
- Large response payloads should be summarized or paginated — don't send 10MB of JSON back to the LLM context.

---

## 2. API Gateway / Middleware Pattern

**When to use:** AI is exposed as a service that other systems call, rather than AI calling out to other systems. Think: "our CRM triggers an AI analysis when a deal moves to Stage 3."

**How it works:** An API gateway sits in front of the LLM, handling authentication, rate limiting, request transformation, and response formatting. Enterprise systems call this gateway just like any other API.

**Best for:**
- Embedding AI analysis into existing workflows without changing those workflows
- Exposing AI capabilities to multiple internal consumers
- Maintaining consistent API contracts while the AI backend evolves

**Architecture:**
```
Enterprise System → API Gateway → [Prompt Builder] → LLM API
                        ↕                              ↕
                  Auth / Rate Limit            Context / RAG
                  Logging / Metrics
```

**Key decisions:**
- **Prompt templates vs. dynamic prompts**: Use templates for well-defined use cases (e.g., "summarize this contract"), dynamic construction for open-ended ones.
- **Sync vs. async**: If LLM latency (2-30s) is acceptable, use sync. If not, accept the request, return a job ID, and deliver results via webhook or polling.
- **Model routing**: Route simple requests to cheaper/faster models (Haiku), complex ones to capable models (Opus). This can cut costs 60-80%.

**Pitfalls:**
- Don't expose raw LLM output to downstream systems — always parse and validate it first.
- Set hard timeout limits; LLM calls can occasionally hang or take much longer than expected.
- Cache identical requests aggressively — many enterprise queries are repetitive.

---

## 3. RAG (Retrieval-Augmented Generation) Pattern

**When to use:** AI needs access to enterprise knowledge that isn't in its training data — internal docs, policies, product catalogs, customer records, Confluence pages, SharePoint sites.

**How it works:** Documents are ingested, chunked, and embedded into a vector database. When a user asks a question, relevant chunks are retrieved and injected into the LLM prompt as context.

**Best for:**
- Internal knowledge bases and documentation search
- Customer support with company-specific answers
- Policy and compliance Q&A
- Any "ask your data" use case

**Architecture:**
```
[Document Sources] → Ingestion Pipeline → Vector DB
                                              ↕
User Query → Embedding → Similarity Search → Top-K Chunks → LLM → Response
```

**Key decisions:**
- **Chunking strategy**: 500-1000 tokens per chunk is a good starting point. Use semantic chunking (by section/paragraph) over fixed-size when possible. Always include metadata (source, date, section).
- **Embedding model**: Use the same model for ingestion and query. Anthropic's Voyage embeddings, OpenAI's ada-002, or open-source models like BGE/E5 are common choices.
- **Vector store**: Pinecone, Weaviate, Qdrant, pgvector (if you're already on Postgres), or ChromaDB for prototyping.
- **Retrieval strategy**: Start with simple top-K similarity. Add reranking (Cohere Rerank, cross-encoder models) if precision matters. Consider hybrid search (vector + keyword) for technical content.
- **Update frequency**: Real-time ingestion for fast-changing data, batch for stable document sets.

**Pitfalls:**
- Garbage in, garbage out — if your documents are poorly structured, RAG will retrieve irrelevant chunks. Clean your data first.
- Don't stuff too many chunks into context. 5-10 highly relevant chunks beat 50 loosely related ones.
- Always cite sources in the response so users can verify. Hallucination on enterprise data is a trust-killer.
- Test with real queries from actual users, not synthetic ones. The gap between "works on paper" and "works in practice" is often about retrieval quality.

---

## 4. Event-Driven Agent Pattern

**When to use:** AI should react to things happening in your systems — a new support ticket, a deployment failure, a contract approaching expiration, an anomaly in metrics.

**How it works:** Events from enterprise systems are captured (via webhooks, message queues, or change data capture) and routed to an AI agent that analyzes the event, decides on an action, and either executes it or requests human approval.

**Best for:**
- Automated triage (support tickets, alerts, PRs)
- Proactive monitoring and anomaly response
- Workflow automation triggered by system state changes
- "If X happens, have AI do Y" scenarios

**Architecture:**
```
Enterprise System → Event Bus (Kafka/SQS/webhooks) → Event Router
                                                         ↕
                                                    AI Agent
                                                    ↕      ↕
                                              Action Queue  Human Approval
                                                    ↕
                                              Target System
```

**Key decisions:**
- **Approval gates**: Define which actions the AI can take autonomously vs. which need human approval. Default to requiring approval for write operations until trust is established.
- **Event filtering**: Not every event needs AI attention. Use rules or a lightweight classifier to filter before invoking the LLM (saves cost and latency).
- **Idempotency**: Events can be delivered multiple times. Make sure the AI's actions are safe to repeat.
- **Dead letter queues**: Failed events must go somewhere for investigation, not silently disappear.

**Pitfalls:**
- Event storms can trigger massive LLM costs if you don't have rate limiting and deduplication.
- Time-sensitivity varies — a P0 incident needs seconds, a contract review can wait days. Route accordingly.
- AI actions on production systems need rollback capabilities. Always ask: "What happens if the AI gets this wrong?"

---

## 5. Orchestrator / Multi-Agent Pattern

**When to use:** A task requires coordinating multiple AI agents, each specialized in a different domain or system — e.g., one agent reads the CRM, another queries the knowledge base, a third drafts the response, and an orchestrator coordinates them.

**How it works:** A supervisor agent receives the task, breaks it into subtasks, dispatches to specialist agents (each with their own tools/context), collects results, and synthesizes a final output.

**Best for:**
- Complex workflows spanning multiple systems
- Tasks requiring different expertise (data analysis + writing + system actions)
- When a single agent's context window isn't enough for all the information needed

**Architecture:**
```
User Request → Orchestrator Agent
                    ↕
    ┌───────────────┼───────────────┐
    ↓               ↓               ↓
CRM Agent      KB Agent       Action Agent
(Salesforce)   (RAG/docs)     (Jira/Slack)
    ↓               ↓               ↓
    └───────────────┼───────────────┘
                    ↕
            Final Response
```

**Key decisions:**
- **Orchestration model**: Supervisor (one agent controls others), pipeline (sequential hand-off), or peer-to-peer (agents communicate directly). Supervisor is simplest to debug.
- **Context sharing**: How much context do agents share? Minimal sharing = better isolation but potential redundancy. Full sharing = coherent but expensive.
- **Failure handling**: If one agent fails, does the whole task fail? Use graceful degradation — return partial results and flag what's missing.

**Pitfalls:**
- Multi-agent systems are significantly harder to debug than single-agent ones. Start with a single agent and only split when you hit clear limitations.
- Cost multiplies quickly — N agents means N times the token usage. Make sure the orchestration overhead is worth it.
- Agent-to-agent communication should use structured formats (JSON), not natural language, to reduce errors.

---

## 6. Sidecar / Copilot Pattern

**When to use:** AI assists a human user within an existing application — think "AI sidebar" in your CRM, "AI assistant" in your IDE, or "smart suggestions" in your email client.

**How it works:** The AI runs alongside the application, receives context about what the user is doing (current page, selected record, open document), and provides suggestions, drafts, or analysis without leaving the app.

**Best for:**
- Augmenting existing tools without replacing them
- Context-aware assistance (the AI sees what the user sees)
- Low-risk entry point for AI adoption (suggestions, not actions)

**Architecture:**
```
Existing App ←→ Sidecar/Extension ←→ AI Backend
     ↕                                    ↕
 App Context                        Knowledge Base
(current page,                      (RAG, tools)
 user actions)
```

---

## 7. Batch Processing Pattern

**When to use:** AI processes large volumes of data on a schedule — classifying thousands of support tickets, enriching a customer database, generating weekly reports from multiple data sources.

**How it works:** A scheduled job pulls data, sends it through an AI pipeline (possibly with parallelism), writes results back, and produces a summary report.

**Best for:**
- Data enrichment and classification at scale
- Periodic report generation
- Bulk content generation (product descriptions, summaries)
- Migration and transformation tasks

**Architecture:**
```
Scheduler (cron/Airflow) → Data Extraction → AI Processing (parallel)
                                                    ↕
                                             Results Store → Reporting
```

---

## Decision Tree

Use these questions to select the right pattern:

```
Q: Is AI the caller or the callee?
├── AI calls external systems → Q: How many systems?
│   ├── One system → MCP Server Pattern
│   ├── Multiple, sequential → Pipeline (variant of Multi-Agent)
│   └── Multiple, parallel → Orchestrator / Multi-Agent Pattern
│
├── External systems call AI → Q: Real-time needed?
│   ├── Yes → API Gateway Pattern
│   └── No → Batch Processing Pattern
│
└── AI reacts to events → Event-Driven Agent Pattern

Q: Does AI need enterprise knowledge?
├── Yes → Layer RAG Pattern on top of any of the above
└── No → Proceed with chosen pattern

Q: Is AI embedded in an existing UI?
├── Yes → Sidecar / Copilot Pattern
└── No → Standalone service (any pattern above)
```

When multiple patterns apply, it's common to combine them. For example: an event-driven agent (Pattern 4) that uses RAG (Pattern 3) and talks to systems via MCP (Pattern 1). Start with one pattern and layer others as needed.
