# AI Integration Architect — Claude Code Plugin

A Claude Code plugin that helps teams plan, design, and scaffold AI integration into enterprise systems.

**46% of enterprises say integrating AI into existing systems is their #1 challenge.** This plugin acts as a senior integration architect — it assesses your system landscape, recommends architecture patterns, generates production-ready code scaffolds, and produces deployment plans with security baked in.

## Installation

### Option 1: One-command install via marketplace (recommended)

```bash
claude plugin marketplace add pfbarros2/ai-integration-architect-plugin
claude plugin install ai-integration-architect
```

### Option 2: Install directly from the skill repo

```bash
claude plugin marketplace add pfbarros2/ai-integration-architect
claude plugin install ai-integration-architect
```

### Option 3: Local install

```bash
git clone https://github.com/pfbarros2/ai-integration-architect-plugin.git
claude --plugin-dir ./ai-integration-architect-plugin
```

## What it does

| Phase | Output |
|-------|--------|
| **Assess** | Maps your current systems (APIs, databases, SaaS tools) and identifies where AI adds the most value |
| **Architect** | Recommends integration patterns (MCP servers, API gateways, RAG pipelines, event-driven agents) with trade-off analysis |
| **Scaffold** | Generates working starter code — MCP servers, API connectors, RAG pipelines, middleware — with auth, rate limiting, error handling, and tests |
| **Deploy** | Produces deployment configs, monitoring setup, security checklists, cost estimates, and phased rollout plans |

## Example prompts

Once installed, just describe your integration challenge:

> "We run Shopify Plus with Salesforce CRM and Zendesk for support. Our CEO wants AI to help reduce support response times. Where do I start?"

> "I need an MCP server that connects Claude to our internal REST API. It uses OAuth2, has /employees, /projects, and /timesheets endpoints."

> "We have 5000 pages of docs across Confluence, Google Docs, and Notion. Engineers waste hours searching. I want a RAG pipeline so our AI assistant can answer questions from any internal doc."

> "Design an event-driven architecture where an AI agent triages incoming support tickets in Zendesk, looks up customer history in Salesforce, and drafts responses."

## Supported integration patterns

- **MCP Servers** — Connect Claude to any API, database, or internal tool (TypeScript + Python templates)
- **API Gateways** — Expose AI as a service for your existing systems to call, with model routing (Haiku/Sonnet/Opus by complexity)
- **RAG Pipelines** — Ingest enterprise knowledge from Confluence, SharePoint, Notion, Google Docs; chunk, embed, retrieve with reranking
- **Event-Driven Agents** — AI that responds to system events (new ticket, alert, deployment) with human-in-the-loop approval gates
- **Hybrid Architectures** — Combine patterns for complex workflows

## Security built in

Every output includes enterprise security by default:

- Least-privilege service accounts and scoped API keys
- Audit logging for all AI-initiated actions
- Data classification guidance (Public → Internal → Confidential → Restricted)
- Compliance checklists for SOC 2, GDPR, and HIPAA
- Prompt injection mitigation strategies

## Plugin structure

```
ai-integration-architect-plugin/
├── .claude-plugin/
│   └── plugin.json
├── skills/
│   └── ai-integration-architect/
│       ├── SKILL.md
│       └── references/
│           ├── patterns.md
│           ├── scaffolds.md
│           ├── security.md
│           └── deployment.md
├── marketplace.json
└── README.md
```

## License

MIT
