# Deployment & Operations Guide

Templates and checklists for deploying AI integrations to production.

## Table of Contents
1. Deployment Checklist
2. Docker Configuration
3. Monitoring Setup
4. Cost Estimation Framework
5. Phased Rollout Plan

---

## 1. Deployment Checklist

Use this before any production deployment. Skip items that don't apply, but review each one.

### Pre-deployment
- [ ] All secrets stored in environment variables or secrets manager (never hardcoded)
- [ ] API keys use least-privilege scopes
- [ ] Rate limiting configured and tested under load
- [ ] Error handling covers all external API failure modes (timeout, 429, 500, auth expired)
- [ ] Structured logging in place with request IDs for tracing
- [ ] Health check endpoint returns meaningful status
- [ ] README documents all environment variables and setup steps
- [ ] Unit tests pass; integration tests run against staging environment
- [ ] Data sensitivity classification completed (what data flows through the AI?)
- [ ] CORS, network policies, and firewall rules configured

### At deployment
- [ ] Deploy to staging first; smoke test with real-world queries
- [ ] Monitor error rates and latency for first 30 minutes
- [ ] Verify logs are flowing to your observability platform
- [ ] Confirm rate limits are enforced correctly
- [ ] Test failover/fallback behavior

### Post-deployment
- [ ] Set up alerts for error rate spikes, latency degradation, and cost thresholds
- [ ] Document runbook for common failure scenarios
- [ ] Schedule first review (1 week out) to check AI output quality
- [ ] Set up cost monitoring dashboard

---

## 2. Docker Configuration

### MCP Server Dockerfile
```dockerfile
FROM node:22-slim AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci --production=false
COPY . .
RUN npm run build

FROM node:22-slim
WORKDIR /app
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/node_modules ./node_modules
COPY --from=builder /app/package.json ./

# Security: run as non-root
RUN addgroup --system mcp && adduser --system --ingroup mcp mcp
USER mcp

HEALTHCHECK --interval=30s --timeout=3s CMD node -e "console.log('ok')" || exit 1

CMD ["node", "dist/index.js"]
```

### docker-compose.yml (with vector DB for RAG)
```yaml
version: "3.8"

services:
  ai-gateway:
    build: .
    ports:
      - "3000:3000"
    env_file: .env
    depends_on:
      - vector-db
    restart: unless-stopped
    deploy:
      resources:
        limits:
          memory: 512M

  vector-db:
    image: qdrant/qdrant:latest
    ports:
      - "6333:6333"
    volumes:
      - qdrant_data:/qdrant/storage
    restart: unless-stopped

volumes:
  qdrant_data:
```

---

## 3. Monitoring Setup

### Key metrics to track

**For MCP Servers / API Gateways:**
| Metric | Why | Alert threshold |
|--------|-----|-----------------|
| Request latency (p50, p95, p99) | Detect degradation | p95 > 10s |
| Error rate | Catch failures | > 5% over 5 min |
| Requests per minute | Capacity planning | > 80% of rate limit |
| Token usage per request | Cost control | > 10K tokens avg |
| Auth failures | Security monitoring | > 10 per hour |

**For RAG Pipelines:**
| Metric | Why | Alert threshold |
|--------|-----|-----------------|
| Retrieval latency | User experience | p95 > 2s |
| Relevance score (avg) | Quality degradation | < 0.7 avg score |
| Empty result rate | Missing knowledge | > 20% of queries |
| Ingestion lag | Freshness | > 24h behind source |
| Embedding cost | Budget | > $X/day |

**For Event-Driven Agents:**
| Metric | Why | Alert threshold |
|--------|-----|-----------------|
| Event processing latency | SLA compliance | > 30s for P0 events |
| Dead letter queue depth | Failed processing | > 0 (alert immediately) |
| Action approval rate | AI accuracy | < 80% approved |
| Queue depth | Backlog | > 100 unprocessed |

### Logging format (structured JSON)
```json
{
  "timestamp": "2026-03-23T14:30:00.000Z",
  "level": "info",
  "service": "mcp-salesforce",
  "request_id": "req_abc123",
  "tool": "search_contacts",
  "duration_ms": 342,
  "tokens_used": 1250,
  "status": "success",
  "user": "claude-session-xyz"
}
```

---

## 4. Cost Estimation Framework

### LLM API costs (as of early 2026, approximate)
| Model | Input (per 1M tokens) | Output (per 1M tokens) | Best for |
|-------|----------------------|----------------------|----------|
| Claude Haiku 4.5 | $0.80 | $4.00 | Triage, classification, simple Q&A |
| Claude Sonnet 4.6 | $3.00 | $15.00 | Most integration tasks, code generation |
| Claude Opus 4.6 | $15.00 | $75.00 | Complex reasoning, multi-step planning |

### Cost estimation template
```
Monthly cost estimate:
├── LLM API costs
│   ├── Requests/month: ___
│   ├── Avg input tokens/request: ___
│   ├── Avg output tokens/request: ___
│   ├── Model tier distribution: ___% Haiku / ___% Sonnet / ___% Opus
│   └── Estimated monthly LLM cost: $___
│
├── Infrastructure costs
│   ├── Compute (containers/servers): $___
│   ├── Vector DB (if RAG): $___
│   ├── Message queue (if event-driven): $___
│   └── Estimated monthly infra cost: $___
│
├── Embedding costs (if RAG)
│   ├── Documents to embed: ___
│   ├── Re-embedding frequency: ___
│   └── Estimated monthly embedding cost: $___
│
└── Total estimated monthly cost: $___
```

### Cost optimization strategies
- **Model routing**: Use Haiku for 70%+ of requests, Sonnet for complex ones, Opus sparingly. Saves 60-80%.
- **Caching**: Cache identical or near-identical requests. Even 20% cache hit rate significantly reduces costs.
- **Prompt optimization**: Shorter, focused prompts = fewer input tokens. Avoid stuffing unnecessary context.
- **Batch where possible**: Batch API pricing is typically 50% cheaper than real-time.
- **Set hard limits**: Per-user, per-endpoint, and monthly spending caps to prevent cost surprises.

---

## 5. Phased Rollout Plan

### Phase 1: Proof of concept (1-2 weeks)
- Single use case, single system integration
- 3-5 internal users
- Manual monitoring
- Goal: Validate that the AI integration delivers value

### Phase 2: Pilot (2-4 weeks)
- Expand to 10-20 users or one team
- Add monitoring and alerting
- Collect feedback on AI output quality
- Iterate on prompts and tool definitions
- Goal: Confirm reliability and output quality at small scale

### Phase 3: Limited production (1-2 months)
- Open to a department or business unit
- Full observability and alerting in place
- SLA defined and monitored
- Cost tracking active
- Goal: Prove operational readiness

### Phase 4: General availability
- Available to all intended users
- Self-service onboarding
- Runbook for operations team
- Regular quality reviews (monthly)
- Continuous improvement based on usage data
