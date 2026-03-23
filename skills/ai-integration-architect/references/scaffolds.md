# Code Scaffolding Templates

Templates for the most common integration scaffolds. Use these as starting points — adapt to the user's language, framework, and system.

## Table of Contents
1. MCP Server (TypeScript)
2. MCP Server (Python)
3. API Gateway / Middleware (Node.js)
4. RAG Pipeline (Python)
5. Event-Driven Agent (Python)
6. Common Utilities

---

## 1. MCP Server (TypeScript)

The most common scaffold. Generates a TypeScript MCP server that connects Claude to an external API or database.

### Project structure
```
mcp-{system-name}/
├── src/
│   ├── index.ts          # Server entry point
│   ├── tools/            # Tool definitions (one file per tool group)
│   │   ├── read.ts       # Read/query tools
│   │   └── write.ts      # Create/update/delete tools
│   ├── client.ts         # API/DB client with auth
│   ├── types.ts          # Shared types
│   └── utils/
│       ├── auth.ts       # Authentication handling
│       ├── rate-limit.ts # Rate limiting
│       └── logger.ts     # Structured logging
├── tests/
│   └── tools.test.ts     # Tool unit tests
├── package.json
├── tsconfig.json
├── .env.example          # Required environment variables
└── README.md
```

### Key file: src/index.ts
```typescript
import { McpServer } from "@modelcontextprotocol/sdk/server/mcp.js";
import { StdioServerTransport } from "@modelcontextprotocol/sdk/server/stdio.js";
import { registerReadTools } from "./tools/read.js";
import { registerWriteTools } from "./tools/write.js";
import { logger } from "./utils/logger.js";

const server = new McpServer({
  name: "mcp-{system-name}",
  version: "1.0.0",
});

// Register tool groups
registerReadTools(server);
registerWriteTools(server);

// Start server
async function main() {
  const transport = new StdioServerTransport();
  await server.connect(transport);
  logger.info("MCP server started", { name: "mcp-{system-name}" });
}

main().catch((error) => {
  logger.error("Server failed to start", { error });
  process.exit(1);
});
```

### Key file: src/tools/read.ts
```typescript
import { McpServer } from "@modelcontextprotocol/sdk/server/mcp.js";
import { z } from "zod";
import { client } from "../client.js";
import { rateLimiter } from "../utils/rate-limit.js";
import { logger } from "../utils/logger.js";

export function registerReadTools(server: McpServer) {
  server.tool(
    "{entity}_search",
    "Search for {entities} by query. Returns matching results with key fields.",
    {
      query: z.string().describe("Search query"),
      limit: z.number().optional().default(10).describe("Max results (default 10)"),
    },
    async ({ query, limit }) => {
      await rateLimiter.checkLimit("{entity}_search");
      logger.info("Tool called: {entity}_search", { query, limit });

      try {
        const results = await client.search(query, { limit });
        return {
          content: [{
            type: "text",
            text: JSON.stringify(results, null, 2),
          }],
        };
      } catch (error) {
        logger.error("{entity}_search failed", { error, query });
        return {
          content: [{
            type: "text",
            text: `Error searching {entities}: ${error.message}`,
          }],
          isError: true,
        };
      }
    }
  );

  server.tool(
    "{entity}_get",
    "Get detailed information about a specific {entity} by ID.",
    {
      id: z.string().describe("The {entity} ID"),
    },
    async ({ id }) => {
      await rateLimiter.checkLimit("{entity}_get");
      const result = await client.getById(id);
      return {
        content: [{
          type: "text",
          text: JSON.stringify(result, null, 2),
        }],
      };
    }
  );
}
```

### Key file: src/client.ts
```typescript
import { logger } from "./utils/logger.js";

// Replace with the actual API client for the target system
export class ApiClient {
  private baseUrl: string;
  private apiKey: string;

  constructor() {
    this.baseUrl = process.env.{SYSTEM}_API_URL || "";
    this.apiKey = process.env.{SYSTEM}_API_KEY || "";

    if (!this.baseUrl || !this.apiKey) {
      throw new Error(
        "Missing required environment variables: {SYSTEM}_API_URL, {SYSTEM}_API_KEY"
      );
    }
  }

  private async request(path: string, options?: RequestInit) {
    const url = `${this.baseUrl}${path}`;
    const response = await fetch(url, {
      ...options,
      headers: {
        "Authorization": `Bearer ${this.apiKey}`,
        "Content-Type": "application/json",
        ...options?.headers,
      },
    });

    if (!response.ok) {
      const body = await response.text();
      logger.error("API request failed", { url, status: response.status, body });
      throw new Error(`API error ${response.status}: ${body}`);
    }

    return response.json();
  }

  async search(query: string, opts: { limit: number }) {
    return this.request(`/search?q=${encodeURIComponent(query)}&limit=${opts.limit}`);
  }

  async getById(id: string) {
    return this.request(`/items/${encodeURIComponent(id)}`);
  }
}

export const client = new ApiClient();
```

### package.json essentials
```json
{
  "name": "mcp-{system-name}",
  "version": "1.0.0",
  "type": "module",
  "scripts": {
    "build": "tsc",
    "start": "node dist/index.js",
    "dev": "tsx src/index.ts",
    "test": "vitest"
  },
  "dependencies": {
    "@modelcontextprotocol/sdk": "^1.12.0",
    "zod": "^3.23.0"
  },
  "devDependencies": {
    "typescript": "^5.5.0",
    "tsx": "^4.16.0",
    "vitest": "^2.0.0"
  }
}
```

---

## 2. MCP Server (Python)

Same concept, Python implementation using the official MCP SDK.

### Project structure
```
mcp_{system_name}/
├── src/
│   ├── server.py         # Server entry point with tool definitions
│   ├── client.py         # API/DB client
│   ├── auth.py           # Authentication
│   └── utils.py          # Rate limiting, logging
├── tests/
│   └── test_tools.py
├── pyproject.toml
├── .env.example
└── README.md
```

### Key file: src/server.py
```python
import os
import logging
from mcp.server.fastmcp import FastMCP

logger = logging.getLogger(__name__)
mcp = FastMCP("mcp-{system-name}")

@mcp.tool()
async def search_{entities}(query: str, limit: int = 10) -> str:
    """Search for {entities} matching the query.

    Args:
        query: Search query string
        limit: Maximum number of results (default 10)
    """
    from .client import api_client
    results = await api_client.search(query, limit=limit)
    return json.dumps(results, indent=2)

@mcp.tool()
async def get_{entity}(id: str) -> str:
    """Get detailed information about a specific {entity}.

    Args:
        id: The {entity} identifier
    """
    from .client import api_client
    result = await api_client.get_by_id(id)
    return json.dumps(result, indent=2)

if __name__ == "__main__":
    mcp.run()
```

---

## 3. API Gateway / Middleware (Node.js)

For when enterprise systems need to call AI as a service.

### Project structure
```
ai-gateway/
├── src/
│   ├── index.ts            # Express/Fastify server
│   ├── routes/
│   │   ├── analyze.ts      # AI analysis endpoints
│   │   └── health.ts       # Health check
│   ├── prompts/
│   │   └── templates.ts    # Prompt templates
│   ├── middleware/
│   │   ├── auth.ts         # API key / JWT validation
│   │   ├── rate-limit.ts   # Per-client rate limiting
│   │   └── logging.ts      # Request/response logging
│   ├── services/
│   │   ├── llm.ts          # LLM client (Anthropic SDK)
│   │   ├── cache.ts        # Response caching
│   │   └── model-router.ts # Route to Haiku/Sonnet/Opus by complexity
│   └── types.ts
├── tests/
├── Dockerfile
├── docker-compose.yml
└── README.md
```

### Model routing pattern
```typescript
// services/model-router.ts
type ModelTier = "fast" | "balanced" | "powerful";

const MODEL_MAP = {
  fast: "claude-haiku-4-5-20251001",
  balanced: "claude-sonnet-4-6-20260321",
  powerful: "claude-opus-4-6-20260321",
} as const;

export function selectModel(task: {
  complexity: "low" | "medium" | "high";
  tokenBudget?: number;
  latencyRequirement?: "realtime" | "standard" | "batch";
}): string {
  if (task.latencyRequirement === "realtime" || task.complexity === "low") {
    return MODEL_MAP.fast;
  }
  if (task.complexity === "high") {
    return MODEL_MAP.powerful;
  }
  return MODEL_MAP.balanced;
}
```

---

## 4. RAG Pipeline (Python)

Ingestion + retrieval pipeline for enterprise knowledge.

### Project structure
```
rag-pipeline/
├── ingest/
│   ├── loader.py          # Document loaders (PDF, Confluence, SharePoint, etc.)
│   ├── chunker.py         # Chunking strategies
│   ├── embedder.py        # Embedding generation
│   └── store.py           # Vector store writer
├── retrieve/
│   ├── search.py          # Similarity search + reranking
│   ├── context.py         # Context assembly for prompts
│   └── cite.py            # Source citation generator
├── config.py              # Centralized configuration
├── tests/
├── scripts/
│   ├── ingest_docs.py     # CLI: ingest a directory of documents
│   └── query.py           # CLI: test a retrieval query
└── README.md
```

### Chunking pattern
```python
# ingest/chunker.py
from dataclasses import dataclass

@dataclass
class Chunk:
    text: str
    metadata: dict  # source, page, section, timestamp

def chunk_document(
    text: str,
    source: str,
    chunk_size: int = 800,
    chunk_overlap: int = 200,
    strategy: str = "semantic",  # "semantic" | "fixed" | "paragraph"
) -> list[Chunk]:
    """Split document into chunks with metadata.

    - semantic: split by headings/sections, fall back to paragraphs
    - fixed: split by token count with overlap
    - paragraph: split by paragraph boundaries
    """
    # Implementation depends on strategy
    ...
```

---

## 5. Event-Driven Agent (Python)

AI agent that responds to system events.

### Project structure
```
event-agent/
├── src/
│   ├── consumer.py        # Event consumer (Kafka/SQS/webhook)
│   ├── router.py          # Route events to handlers
│   ├── handlers/
│   │   ├── ticket_triage.py
│   │   ├── alert_response.py
│   │   └── ...
│   ├── actions/
│   │   ├── dispatcher.py  # Execute approved actions
│   │   └── approval.py    # Human approval gate
│   ├── prompts/
│   │   └── templates.py
│   └── config.py
├── tests/
├── Dockerfile
└── README.md
```

### Approval gate pattern
```python
# actions/approval.py
from enum import Enum

class RiskLevel(Enum):
    LOW = "low"       # Auto-approve (read operations, notifications)
    MEDIUM = "medium" # Approve with audit log (updates, non-critical writes)
    HIGH = "high"     # Require human approval (deletes, financial, customer-facing)

def assess_risk(action: str, target_system: str, is_write: bool) -> RiskLevel:
    """Assess the risk level of an AI-initiated action."""
    if not is_write:
        return RiskLevel.LOW
    if target_system in HIGH_RISK_SYSTEMS:
        return RiskLevel.HIGH
    return RiskLevel.MEDIUM

async def request_approval(action: dict, risk: RiskLevel) -> bool:
    """Gate actions based on risk level."""
    if risk == RiskLevel.LOW:
        return True
    if risk == RiskLevel.MEDIUM:
        log_action(action)  # Audit trail, auto-approve
        return True
    # HIGH risk: notify human, wait for approval
    return await send_approval_request(action)
```

---

## 6. Common Utilities

### Rate limiter (reusable across all patterns)
```typescript
// utils/rate-limit.ts
class RateLimiter {
  private windows: Map<string, { count: number; resetAt: number }> = new Map();

  constructor(private maxRequests: number = 60, private windowMs: number = 60_000) {}

  async checkLimit(toolName: string): Promise<void> {
    const now = Date.now();
    const key = toolName;
    const window = this.windows.get(key);

    if (!window || now > window.resetAt) {
      this.windows.set(key, { count: 1, resetAt: now + this.windowMs });
      return;
    }

    if (window.count >= this.maxRequests) {
      throw new Error(`Rate limit exceeded for ${toolName}. Retry after ${new Date(window.resetAt).toISOString()}`);
    }

    window.count++;
  }
}

export const rateLimiter = new RateLimiter();
```

### Structured logger (reusable)
```typescript
// utils/logger.ts
export const logger = {
  info(message: string, context?: Record<string, unknown>) {
    console.log(JSON.stringify({ level: "info", message, ...context, timestamp: new Date().toISOString() }));
  },
  error(message: string, context?: Record<string, unknown>) {
    console.error(JSON.stringify({ level: "error", message, ...context, timestamp: new Date().toISOString() }));
  },
  warn(message: string, context?: Record<string, unknown>) {
    console.warn(JSON.stringify({ level: "warn", message, ...context, timestamp: new Date().toISOString() }));
  },
};
```
