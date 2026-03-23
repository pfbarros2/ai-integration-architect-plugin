# Enterprise Security Patterns for AI Integration

Security reference for AI integrations handling enterprise data.

## Table of Contents
1. Authentication & Authorization Patterns
2. Data Classification & Handling
3. Audit Logging
4. Compliance Checklists
5. Common Vulnerabilities in AI Integrations

---

## 1. Authentication & Authorization Patterns

### For MCP Servers (Claude connecting to enterprise systems)

**Option A: Service Account (simplest)**
- MCP server uses a dedicated service account with minimal permissions
- Pros: Simple setup, no per-user auth flow
- Cons: All actions appear as one user in audit logs; can't enforce per-user permissions
- When to use: Internal tools, read-only integrations, prototyping

**Option B: OAuth2 with user delegation**
- MCP server initiates OAuth2 flow; user authorizes access to their account
- The server stores refresh tokens and acts on behalf of the user
- Pros: Per-user permissions, proper audit trail
- Cons: More complex, requires token management
- When to use: Production integrations touching user-specific data

**Option C: API key per integration**
- Each MCP server instance gets a scoped API key
- Pros: Simple, easy to rotate
- Cons: No per-user granularity
- When to use: System-to-system integrations, SaaS APIs

### For API Gateways (enterprise systems calling AI)

```
Client → API Gateway
           │
           ├── Validate API key or JWT
           ├── Check rate limits (per-client)
           ├── Verify IP allowlist (if applicable)
           └── Forward to LLM service
```

Always validate authentication before touching the LLM — even failed auth attempts should be logged but should not incur LLM costs.

### Secret management rules
- Never store secrets in code, config files, or environment variable defaults in Dockerfiles
- Use a secrets manager: AWS Secrets Manager, HashiCorp Vault, GCP Secret Manager, Azure Key Vault, or Doppler
- Rotate API keys on a schedule (90 days recommended) and immediately on suspected compromise
- Use separate keys for development, staging, and production
- Log when secrets are accessed (but never log the secret values themselves)

---

## 2. Data Classification & Handling

Before building any integration, classify the data that will flow through the AI.

### Classification levels
| Level | Examples | Handling rules |
|-------|----------|----------------|
| **Public** | Marketing content, public docs | No special handling needed |
| **Internal** | Internal wikis, meeting notes, org charts | Encrypt in transit; restrict to authenticated users |
| **Confidential** | Customer data, financials, HR records | Encrypt in transit and at rest; access logging required; data minimization |
| **Restricted** | PII, health records, payment data, trade secrets | All of the above + explicit approval for AI processing; consider on-prem / VPC deployment |

### Data flow audit
For each integration, document:
1. What data enters the AI system (inputs)
2. What data the AI sends to external systems (outputs)
3. Where data is stored (even temporarily — logs, caches, vector DBs)
4. Who can access each data store
5. How long data is retained
6. Whether data crosses geographic boundaries (relevant for GDPR, data sovereignty)

### Minimization principles
- Send only the data the AI needs, not entire records. If the AI needs a customer's order history to answer a question, don't also send their payment details.
- Truncate or summarize large datasets before sending to the LLM.
- Avoid storing LLM conversation logs that contain sensitive data unless required for compliance.
- If using RAG, consider whether the vector database needs the full text or just embeddings.

---

## 3. Audit Logging

Every AI-initiated action on an enterprise system must be logged. This is non-negotiable for compliance and debugging.

### What to log
```json
{
  "timestamp": "2026-03-23T14:30:00Z",
  "event_type": "ai_action",
  "action": "create_ticket",
  "target_system": "jira",
  "target_resource": "PROJECT-123",
  "initiated_by": "claude-agent",
  "on_behalf_of": "user@company.com",
  "input_summary": "Create bug ticket for login timeout issue",
  "result": "success",
  "result_id": "PROJ-456",
  "risk_level": "medium",
  "approval_status": "auto_approved",
  "session_id": "sess_abc123",
  "model_used": "claude-sonnet-4-6",
  "tokens_used": 2340
}
```

### What NOT to log
- Full LLM prompts containing sensitive data (log a hash or summary instead)
- API keys, tokens, or credentials
- Full customer records (log IDs/references only)

### Retention
- Keep audit logs for at least 1 year (or as required by your compliance framework)
- Store in append-only, tamper-evident storage
- Ensure logs are queryable for incident investigation

---

## 4. Compliance Checklists

### SOC 2 considerations for AI integrations
- [ ] Access controls: AI agents use least-privilege service accounts
- [ ] Change management: Changes to prompts, tools, and models go through version control and review
- [ ] Monitoring: Continuous monitoring of AI actions with alerting on anomalies
- [ ] Incident response: Runbook exists for "AI took an incorrect action" scenarios
- [ ] Vendor management: LLM provider's SOC 2 report reviewed and accepted

### GDPR considerations
- [ ] Data processing agreement (DPA) with LLM provider covers your use case
- [ ] Right to erasure: Can you delete a user's data from vector DBs and logs?
- [ ] Data minimization: Only necessary data is sent to the LLM
- [ ] Cross-border transfers: Know where your LLM provider processes data
- [ ] Purpose limitation: AI processing is limited to the stated purpose
- [ ] Transparency: Users know when they're interacting with AI

### HIPAA considerations (if health data involved)
- [ ] Business Associate Agreement (BAA) with LLM provider
- [ ] PHI never stored in vector DBs without encryption
- [ ] Access logging for all PHI-touching AI actions
- [ ] De-identification applied where possible before AI processing
- [ ] Minimum necessary standard applied to all data flows

---

## 5. Common Vulnerabilities in AI Integrations

### Prompt injection via enterprise data
**Risk**: Malicious content in enterprise systems (e.g., a support ticket containing "ignore previous instructions") can manipulate the AI agent.
**Mitigation**: Treat all data from enterprise systems as untrusted input. Use clear system prompt boundaries. Validate AI outputs before executing actions.

### Over-privileged service accounts
**Risk**: The MCP server's service account has admin access "for convenience" — if compromised, an attacker (or a confused AI) can do anything.
**Mitigation**: Create purpose-built service accounts with only the permissions the integration needs. Read-only where possible.

### Unbounded AI actions
**Risk**: An AI agent in a loop takes hundreds of actions before anyone notices (e.g., creating duplicate tickets, sending mass emails).
**Mitigation**: Implement action budgets (max N actions per session), require human approval for bulk operations, and alert on unusual action volume.

### Data leakage through AI responses
**Risk**: AI includes sensitive data from one user's context in another user's response (especially in shared deployments).
**Mitigation**: Ensure session isolation. Don't share conversation context across users. Clear context between sessions.

### Stale or poisoned knowledge bases
**Risk**: RAG pipeline serves outdated or incorrect information because ingestion failed or source data was corrupted.
**Mitigation**: Monitor ingestion health. Add freshness metadata to chunks. Alert when the knowledge base hasn't been updated within the expected window.
