# CLAUDE.md — Agentic Product Engineering Constitution

> Place this file at the root of every repository.
> Claude CLI, Claude Code, and Claude API read it automatically — no prompt needed.
> Every code generation, architecture review, and AI-assisted decision in this codebase inherits these rules.

---

**Version:** 2.0.0 — Agentic Edition
**Last Updated:** 2026
**Audience:** All Engineers, Architects, AI / LLM Teams
**Review Cadence:** Quarterly — Staff Engineer approval required
**Status:** Living Standard — PRs welcome
**By:** [Vishal Mishra](https://www.linkedin.com/in/vishalkmishra)
---

## Table of Contents

- [Core Mandate](#core-mandate)
- [Part 1 — Foundational Principles](#part-1--foundational-principles)
- [Part 2 — The 12-Factor App](#part-2--the-12-factor-app)
- [Part 3 — Architecture Patterns](#part-3--architecture-patterns)
- [Part 4 — Agentic App Architecture](#part-4--agentic-app-architecture)
- [Part 5 — Scalability](#part-5--scalability)
- [Part 6 — Security Architecture](#part-6--security-architecture)
- [Part 7 — Testing Strategy](#part-7--testing-strategy)
- [Part 8 — CI/CD & DevOps](#part-8--cicd--devops)
- [Part 9 — Code Quality Standards](#part-9--code-quality-standards)
- [Part 10 — Agentic-Specific Standards](#part-10--agentic-specific-standards)
- [Architectural Decision Checklist](#architectural-decision-checklist)
- [Canonical References](#canonical-references)

---

## Core Mandate

> **"Complexity is the enemy of reliability. Simplicity is the ultimate sophistication."**

Every architectural decision must answer three questions:

1. **Can it scale** to 10M+ concurrent users without re-architecture?
2. **Can a new engineer understand it** in under 30 minutes?
3. **Can it fail gracefully** without cascading disasters?

---

## Part 1 — Foundational Principles

### 1.1 DRY — Don't Repeat Yourself

Every piece of knowledge must have a **single, unambiguous, authoritative representation** in the system.
Duplication of logic = duplication of bugs. Fix a bug in one place, not five.

**Apply to:** business logic, configuration, schema definitions, validation rules, API contracts.

```ts
// ❌ Bad: validateEmail() duplicated in auth.ts, checkout.ts, profile.ts
// ✅ Good:
import { validateEmail } from "@myapp/shared/validators";
```

### 1.2 KISS — Keep It Simple, Stupid

- The simplest solution that correctly solves the problem is always the best.
- Every abstraction has a cost — only add it when the benefit exceeds the cost.
- If you need a comment to explain what code *does*, rewrite the code.
- Prefer 10 lines of obvious code over 3 lines of clever code.

### 1.3 YAGNI — You Aren't Gonna Need It

- Never implement something based on speculation about future needs.
- Build for today's requirements. Refactor when new requirements arrive.
- Premature optimisation and premature abstraction are equally dangerous.

> **Rule:** If there is no current ticket with a real use case, do not build it.

### 1.4 SOLID Principles

| Principle | Rule | Violation Signal |
|---|---|---|
| **Single Responsibility** | A class/module does ONE thing | `UserService` that also sends emails and writes logs |
| **Open / Closed** | Open for extension, closed for modification | `if type == "new"` scattered everywhere |
| **Liskov Substitution** | Subtypes must be substitutable for base types | Overriding methods that break parent contracts |
| **Interface Segregation** | No client forced to depend on methods it doesn't use | Fat interfaces with 20+ methods |
| **Dependency Inversion** | Depend on abstractions, not concretions | `new MySQLDatabase()` hardcoded inside a service |

### 1.5 Separation of Concerns (SoC)

| Layer | Responsibility | Example |
|---|---|---|
| **Presentation** | What to show | HTTP serialisation, UI components |
| **Application** | Orchestrate use cases | Command handlers, application services |
| **Domain** | Business rules | Entities, value objects, domain events |
| **Infrastructure** | I/O & external systems | DB, queues, third-party APIs |

Layers communicate **inward only** (Clean Architecture Dependency Rule).
Infrastructure depends on Application. Application depends on Domain. Domain depends on nothing.

### 1.6 Loose Coupling & High Cohesion

- **Loose Coupling:** Components know as little as possible about each other. Communicate through interfaces, events, and contracts — never through internal implementation details.
- **High Cohesion:** Things that change together live together. Group by domain/feature (`/orders/`) not by technical layer (`/controllers/`).

```ts
// ❌ Bad: PaymentService imports UserRepository & EmailService directly
// ✅ Good:
eventBus.emit("payment.completed", { orderId, amount, userId });
// UserService and EmailService react independently — zero coupling
```

### 1.7 Law of Demeter (Principle of Least Knowledge)

- A module should only talk to its **immediate friends**, never to strangers.
- Avoid deep method chaining: `order.getCustomer().getAddress().getCity()` — each link is a hidden dependency.
- Each unit should have limited knowledge about other units.

---

## Part 2 — The 12-Factor App

All services **MUST** comply with every factor. No exceptions. Compliance is verified in CI.

| # | Factor | Implementation Requirement |
|---|---|---|
| **I** | Codebase | One repo per service tracked in Git. One codebase → many deploys (dev/staging/prod). |
| **II** | Dependencies | Explicitly declare ALL dependencies. Zero reliance on system-level packages. |
| **III** | Config | ALL config in env vars. Zero secrets in code. Use Vault / AWS SSM in prod. |
| **IV** | Backing Services | DB, cache, queue as attached resources via URL. Swap MySQL → Postgres without code change. |
| **V** | Build / Release / Run | Strict separation: `build` → `release` → `run`. Immutable releases — never mutate. |
| **VI** | Processes | Services are **stateless**. No sticky sessions. All state in Redis or DB. |
| **VII** | Port Binding | Services self-contained and expose via port. No runtime webserver injection. |
| **VIII** | Concurrency | Scale out via process model. Horizontal, not vertical. One process type per concern. |
| **IX** | Disposability | Fast startup < 5s, graceful SIGTERM shutdown. Crash-safe with queue ack patterns. |
| **X** | Dev/Prod Parity | Dev, staging, prod kept identical. Same DB engine. Same queue. Docker everywhere. |
| **XI** | Logs | Services write to **stdout** as event streams. Never manage log files. |
| **XII** | Admin Processes | DB migrations and one-off scripts run as isolated one-off processes. Never manual prod SSH. |

---

## Part 3 — Architecture Patterns

### 3.1 Clean / Hexagonal Architecture (Ports & Adapters)

```
┌─────────────────────────────────────────────┐
│           Frameworks & Drivers              │  ← HTTP, CLI, Tests
│  ┌───────────────────────────────────────┐  │
│  │         Interface Adapters            │  │  ← Controllers, Presenters, Gateways
│  │  ┌─────────────────────────────────┐  │  │
│  │  │       Application Layer         │  │  │  ← Use Cases, Application Services
│  │  │  ┌───────────────────────────┐  │  │  │
│  │  │  │      Domain Layer         │  │  │  │  ← Entities, Value Objects, Domain Events
│  │  │  │  (Pure Business Logic)    │  │  │  │  ← Zero infrastructure imports
│  │  │  └───────────────────────────┘  │  │  │
│  │  └─────────────────────────────────┘  │  │
│  └───────────────────────────────────────┘  │
└─────────────────────────────────────────────┘
```

- **Ports:** Interfaces defined by the domain (e.g., `IUserRepository`).
- **Adapters:** Implementations (e.g., `PostgresUserRepository`, `InMemoryUserRepository`).
- **Testability:** Domain and use cases are 100% unit-testable with zero infrastructure.

### 3.2 Event-Driven Architecture (EDA) + CQRS

- Services communicate via **immutable, versioned domain events**.
- Event schema: `{ eventId, eventType, aggregateId, version, timestamp, payload, metadata }`.
- **CQRS — Write path:** Commands → Domain → Events → Write Store.
- **CQRS — Read path:** Events → Projections → Read-optimised Store → Queries.
- Never mix read and write models.
- Use Event Sourcing for audit-critical domains: finance, healthcare, legal.

### 3.3 Microservices Design Rules

- **Size Rule:** A service is ownable by a team of 2–8 engineers.
- **Bounded Contexts:** Each service owns its data. No shared databases between services.
- **API Contracts:** All inter-service communication via versioned APIs or typed events.
- **Saga Pattern:** For distributed transactions — Choreography (events) or Orchestration (saga orchestrator).
- **Anti-Corruption Layer (ACL):** Always wrap legacy or third-party systems.

### 3.4 Resilience Patterns

| Pattern | Rule |
|---|---|
| **Circuit Breaker** | Stop calling a failing service. Fail fast. Auto-recover after timeout. |
| **Retry + Backoff** | Exponential backoff with jitter. Max 3–5 retries. Never retry `400` responses. |
| **Bulkhead** | Isolate failures. Separate thread pools per downstream dependency. |
| **Timeout** | Every external call has a deadline. Never wait forever. |
| **Fallback** | Degrade gracefully. Serve cached or default data on failure. |
| **Rate Limiting** | Token bucket per user/IP. Protect all ingress points. |
| **Idempotency** | Every write operation must be safely re-executable with the same result. |

### 3.5 Data Architecture — Polyglot Persistence

| Database | Use Case |
|---|---|
| PostgreSQL | Transactional, relational data |
| MongoDB | Flexible schema, hierarchical documents |
| Redis | Hot data, sessions, pub/sub, rate limit counters |
| Kafka | High-throughput event streaming, audit logs |
| Elasticsearch | Full-text search, faceted filtering |
| pgvector / Pinecone | Embeddings, semantic search, RAG |
| InfluxDB | Metrics, time-series, IoT |

- **Database per Service:** No shared DB schemas across service boundaries.
- **Read Replicas:** All analytical queries hit replicas. Never the primary for reads.
- Prefer eventual consistency. Use sagas for distributed consistency requirements.

---

## Part 4 — Agentic App Architecture

> **The Agentic Difference:**
> Traditional apps execute deterministic code. Agentic apps use LLMs as reasoning engines that autonomously plan, call tools, reflect on results, and loop until a goal is achieved.
> Every principle in this section exists to make that loop **safe, observable, and cost-controlled.**

### 4.1 System Layers

```
┌─────────────────────────────────────────────────────────────────┐
│  INPUT LAYER                                                    │
│  STT (Whisper/Deepgram) │ Text (API/UI/CLI) │ Vision/Docs      │
│  Webhooks / Cron / Event Triggers                               │
├─────────────────────────────────────────────────────────────────┤
│  MESSAGE / EVENT BUS  (async, loosely coupled)                  │
├─────────────────────────────────────────────────────────────────┤
│  ORCHESTRATION LAYER                                            │
│  Agent Orchestrator                                             │
│    ├─ Context Manager   (token budget, trimming)                │
│    ├─ Memory Router     (short-term / long-term)                │
│    └─ Prompt Builder    (system, few-shot, tools)               │
├─────────────────────────────────────────────────────────────────┤
│  LLM GATEWAY  (provider-agnostic)                               │
│  Rate limiting · Retries · Cost tracking · Model fallback       │
│    ├─ LLM Model Call    (Claude / GPT / Gemini / local)         │
│    ├─ Streaming Handler (SSE / WebSocket)                       │
│    └─ Tool-Call Parser  → dispatch to Tool Layer                │
├─────────────────────────────────────────────────────────────────┤
│  TOOL LAYER  (parallel / sequential execution)                  │
│  Web Search │ Code Exec │ Vector DB │ APIs/MCP │ File I/O       │
│                    ↑ tool results feed back to LLM              │
├─────────────────────────────────────────────────────────────────┤
│  MEMORY LAYER                                                   │
│  Working Memory (context) │ Episodic (Postgres) │ Semantic (pgvector) │
│                    ↑ write-back from every session              │
├─────────────────────────────────────────────────────────────────┤
│  OUTPUT LAYER                                                   │
│  Safety Filter │ TTS │ Text Stream │ Action Outputs │ Observability │
└─────────────────────────────────────────────────────────────────┘
```

### 4.2 STT — Speech-to-Text Standards

**Approved providers:** OpenAI Whisper (self-hosted), Deepgram Nova-2, Azure Cognitive Speech.

```ts
interface STTResult {
  transcript:  string;
  confidence:  number;          // 0–1. Reject < 0.70, clarify on < 0.85
  language:    string;          // BCP-47 tag, auto-detected
  words:       WordTimestamp[]; // required for alignment / diarisation
  isFinal:     boolean;         // streaming partial vs final segment
  durationMs:  number;
}
```

**Rules:**
- Stream audio — do not buffer the entire audio file before transcribing.
- Always return word-level timestamps for downstream alignment.
- Language detection must be automatic with fallback to user locale preference.
- On confidence < 0.85: return partial result and request clarification from the user.

### 4.3 TTS — Text-to-Speech Standards

**Approved providers:** ElevenLabs (expressive), OpenAI TTS-1-HD, Azure Neural TTS.

```ts
interface TTSConfig {
  voiceId:   string;   // pinned voice ID — never "default"
  model:     string;   // pinned model version
  speed:     number;   // 0.75–1.25
  streaming: boolean;  // always true in production
  ssml:      boolean;  // always true for agent outputs
  cacheKey?: string;   // cache repeated short phrases (< 20 chars)
}
```

**Rules:**
- Stream audio output via chunked transfer — never wait for full synthesis before playing.
- Voice selection must be configurable per agent role (assistant vs narrator vs alert).
- SSML markup required for: pauses, emphasis, numbers, dates, abbreviations.
- Always provide a text fallback — never TTS-only output paths.
- Cache synthesised audio for repeated phrases: greetings, confirmations, errors.

### 4.4 LLM Gateway Design

```ts
interface LLMCallConfig {
  model:          string;        // e.g. "claude-sonnet-4-20250514" — always pinned
  temperature:    number;        // 0 = deterministic, 0.7 = creative
  maxTokens:      number;        // always cap — never unbounded
  timeoutMs:      number;        // 30_000 default, 120_000 for complex tasks
  retryPolicy:    RetryPolicy;   // retry 429 / 500, never retry 400
  fallbackModel?: string;        // cheaper / faster model on primary failure
  cache?:         CacheConfig;   // semantic or exact-match caching
  stream:         boolean;       // true by default in all production paths
}
```

**Rules:**
- Provider-agnostic: abstract over Anthropic, OpenAI, Gemini, Ollama behind one interface.
- **Never use `"latest"` or unversioned model aliases in production.**
- Implement automatic model fallback: primary fails → cheaper backup model.
- Cost tracking per request, per user, per feature. Alert on anomalies > 2σ of baseline.
- Semantic cache: hash prompt embeddings — identical semantic intent → cached response.

### 4.5 Agent Design Patterns

#### ReAct (Reason + Act)

```
Thought      →  identify what information or action is needed
Action       →  call a tool with typed, validated parameters
Observation  →  receive result, validate, update context
Loop         →  repeat until goal achieved OR budget exhausted
Answer       →  structured final response with citations
```

Every reasoning step is logged and auditable. Observation is always fed back into the context window.

#### Planner–Executor–Critic

- **Planner:** High-level LLM that decomposes the goal into a DAG of subtasks.
- **Executor:** Specialist sub-agents or tools that execute each subtask (in parallel where safe).
- **Critic / Evaluator:** Reviews output quality, flags failures, triggers re-planning.

#### Multi-Agent Orchestration

```
User Request
  └─ Orchestrator Agent
       ├─ Research Agent   →  web search, document retrieval, fact verification
       ├─ Code Agent       →  code generation, execution, test writing
       ├─ Data Agent       →  SQL, analytics, data visualisation
       └─ Writer Agent     →  drafting, summarisation, editing
  └─ Synthesiser Agent  →  merge outputs  →  Final Response
```

- Each agent has a **single, well-defined role** — never multi-role agents.
- Agents communicate via **structured message passing**, not shared mutable state.
- Orchestrator is stateless — state lives in a persistent task graph (Temporal / LangGraph).

### 4.6 Memory Architecture

| Memory Type | Storage | Use Case | TTL |
|---|---|---|---|
| **Working Memory** | LLM Context Window | Current task state, conversation turn | Per-session |
| **Episodic Memory** | PostgreSQL / MongoDB | Past interactions, decisions, audit trail | Long-term |
| **Semantic Memory** | pgvector / Pinecone | Knowledge base, documents, facts | Persistent |
| **Procedural Memory** | Tool Registry / Code | How to perform tasks | Permanent |
| **Cache Layer** | Redis | Repeated queries, embeddings, TTS audio | TTL-based |

### 4.7 Tool Design Standards

```ts
interface AgentTool {
  name:        string;        // snake_case verb-noun: search_web, execute_code
  description: string;        // tells the LLM exactly when and why to use this tool
  parameters:  JSONSchema;    // strictly typed; all required fields explicit
  execute(p):  Promise<ToolResult>;
  timeoutMs:   number;        // always set a deadline
  retryPolicy: RetryPolicy;   // idempotent tools may retry; others must not
  rateLimit:   RateLimit;     // prevent runaway tool usage by the agent
  reversible:  boolean;       // false = requires human confirmation before execution
  cost?:       CostEstimate;  // used for budget-aware routing decisions
}
```

- Every tool must be: deterministic where possible, idempotent, rate-limited, observable.
- Prefer many small, composable tools over few large monolithic tools.
- **Irreversible actions (delete, send, pay, deploy) MUST require explicit human confirmation.**
- Every `ToolResult` carries: `{ success, data, error, durationMs, costUsd }`.

### 4.8 Context Window Management

```ts
// Context assembly order — most important first:
const context = [
  systemPrompt,        // agent role, constraints, output format, tone
  fewShotExamples,     // 2–5 curated demonstrations of correct behaviour
  retrievedMemory,     // top-k chunks by cosine similarity from vector DB
  toolSchemas,         // only tools relevant to the current task
  conversationHistory, // trimmed to token budget (oldest first eviction)
  currentTask,         // the actual user request
];
```

- **Budget:** Track token usage per turn. Alert at 70% capacity. Hard stop at 90%.
- **Summarisation:** Auto-summarise history via a dedicated summarisation step when approaching the limit.
- **Relevance:** Vector search to inject only relevant memory, never all memory.
- **Eviction:** LRU + relevance-based pruning. Recent + relevant always beats old + irrelevant.

### 4.9 Agentic Safety Architecture

> ⚠️ Every agentic system must implement all seven layers. Missing any single layer is a production incident waiting to happen.

| Safety Layer | Implementation |
|---|---|
| **Sandboxing** | Code execution in isolated containers (gVisor / Firecracker). Zero host filesystem access. |
| **Least Privilege** | Agents request minimum permissions. Scope tokens to required resources only. |
| **Action Budget** | Cap per run: max tool calls, max spend ($), max wall-clock time. Hard limits enforced server-side. |
| **Guardrails** | Input/output classifiers run before every LLM call and after every tool result. |
| **Audit Log** | Every decision, tool call, and result logged immutably with `trace_id`. Non-repudiable. |
| **Kill Switch** | Global circuit breaker to halt all agent execution instantly. Accessible to on-call ops. |
| **Human-in-the-Loop** | Irreversible actions pause and require explicit human approval before proceeding. |

### 4.10 Prompt Architecture

- **System Prompt = Contract:** Defines agent persona, capabilities, constraints, and output format. Treat it as a typed API — version it and review changes formally.
- **Structured Outputs:** Always use JSON schema or function calling for programmatic output. Never parse free-form LLM text with regex.
- **Prompt Versioning:** Treat prompts as code. Version in Git. A/B test via feature flags. Rollback on regression.
- **Prompt Injection Defence:** Sanitise all user-controlled inputs before injection. Delimit system content from user content explicitly.
- **Temperature Discipline:**
  - `0.0` for deterministic / extraction tasks
  - `0.3–0.5` for analysis and reasoning
  - `0.7–1.0` for creative tasks only

### 4.11 Multi-Model Strategy

| Task Type | Model Tier | Rationale |
|---|---|---|
| Simple extraction / classification | Fast/small (Haiku, GPT-4o-mini) | Low cost, low latency |
| Reasoning / planning | Mid-tier (Sonnet, GPT-4o) | Balance quality vs cost |
| Complex multi-step analysis | Frontier (Opus, o1) | Quality critical, cost secondary |
| Embeddings / RAG | Embedding model (text-embedding-3) | Optimised for semantic similarity |
| Code generation / review | Code-specialised (Sonnet, GPT-4o) | Strong tool use and code quality |

### 4.12 Agentic Anti-Patterns — Never Do These

> Every item below has caused real production incidents.

- ❌ Infinite agent loops with no termination condition or budget cap.
- ❌ Parsing LLM output with regex instead of structured output / function calling.
- ❌ Storing secrets, PII, or credentials inside prompts or context windows.
- ❌ Granting agents write access to production systems without human approval gates.
- ❌ Using `"latest"` model aliases in production — breaks silently on provider updates.
- ❌ Running LLM-generated code outside a sandboxed container.
- ❌ Trusting tool input sourced from user-provided or web content (prompt injection).
- ❌ No cost cap — a runaway agent can generate thousands of dollars in minutes.

---

## Part 5 — Scalability

### 5.1 Scalability Tiers

| User Range | Architecture |
|---|---|
| 0 – 10K | Modular monolith + PostgreSQL + Redis. Keep it simple. |
| 10K – 100K | Add CDN, read replicas, async background jobs, connection pooling. |
| 100K – 1M | Extract bottleneck services. Add API Gateway + message queues. |
| 1M – 10M | Full microservices. Multi-region. CQRS. Kafka. DB sharding. |
| 10M+ | Cell-based architecture. Global load balancing. Data mesh. |

### 5.2 Caching Strategy

| Level | Technology | Latency | Use Case |
|---|---|---|---|
| L1 | In-process (Map / LRU) | < 1ms | Hot computation results |
| L2 | Redis Cluster | < 1ms | Sessions, rate limits, hot data |
| L3 | CDN (Cloudflare / CloudFront) | 5–50ms | Static assets, cacheable API responses |
| L4 | Read Replica | 1–5ms | Heavy analytical read queries |

- **Cache-aside:** App checks cache first; on miss loads from DB and populates cache.
- **TTL discipline:** Every cache entry has a TTL. No eternal cache entries.
- **Event-driven invalidation:** Emit domain events on mutations → invalidate synchronously.

### 5.3 API Design for Scale

```
REST Standards:
  GET    /resources          → list (cursor-based pagination only)
  GET    /resources/:id      → single resource
  POST   /resources          → create (201 + Location header)
  PATCH  /resources/:id      → partial update (idempotent with If-Match)
  DELETE /resources/:id      → soft delete (204)

Cursor pagination (offset breaks at scale):
  { data: [...], nextCursor: "opaque_token", hasMore: true }

Required headers on all mutation endpoints:
  Idempotency-Key: <uuid>
  X-RateLimit-Limit / X-RateLimit-Remaining / X-RateLimit-Reset

Versioning: /api/v1/ in URL path. Never break existing versions.
```

### 5.4 Observability — The Three Pillars

| Pillar | Stack | Key Metrics |
|---|---|---|
| **Metrics** | Prometheus + Grafana | RED per service: Rate, Errors, Duration |
| **Tracing** | OpenTelemetry + Tempo/Jaeger | Distributed trace across every service hop |
| **Logging** | Structured JSON → Loki / Datadog | Every log line carries `trace_id` |

**SLO targets:**
- API p99 latency < 200ms
- Error rate < 0.1%
- Availability > 99.95%

Every team owns a service dashboard. On-call engineer diagnoses any issue in < 5 minutes.

---

## Part 6 — Security Architecture

> **Zero Trust Mandate:** "Never trust, always verify."
> Every request must be authenticated and authorised regardless of network origin. Assume breach at all times.

### 6.1 Security Layers

| Layer | Technology | Responsibility |
|---|---|---|
| Perimeter | Cloudflare WAF + DDoS Shield | Block malicious traffic at edge |
| API Gateway | Kong / AWS API Gateway | Auth, rate limiting, input validation |
| Service Mesh | Istio / Linkerd | mTLS, network policies, retries |
| Application | OWASP Top 10 controls | Input sanitisation, output encoding |
| Data | AES-256 + TLS 1.3 | Encryption at rest and in transit |
| Secrets | HashiCorp Vault / AWS SSM | Rotated credentials. **Never in `.env` in prod.** |

### 6.2 Authentication & Authorisation

- **AuthN:** OAuth 2.0 + OIDC (Auth0 / Cognito / Keycloak). JWTs expire in 15 minutes.
- **AuthZ:** RBAC for coarse-grained, ABAC / OPA for fine-grained policy enforcement.
- **Short-lived credentials:** Refresh tokens rotated on use. API keys scoped to minimum required.
- **Audit Trail:** Every auth event logged with IP, device fingerprint, `user_id`, timestamp.

### 6.3 Agent-Specific Security

- **Prompt injection:** Treat all user input as untrusted. Delimit system content from user content explicitly in every prompt.
- **Tool authorisation:** Each tool has an explicit permission list. Agents cannot self-grant permissions.
- **Exfiltration prevention:** Monitor and block unusual data volume in tool call outputs.
- **Model output scanning:** Classify every LLM output before surfacing to the user or downstream systems.

---

## Part 7 — Testing Strategy

### 7.1 The Testing Pyramid

```
              /\
             /  \    E2E Tests (5%)
            /    \   Playwright / Cypress — happy path user journeys
           /──────\
          /        \  Integration Tests (25%)
         /          \ Supertest / TestContainers — service + DB + queue
        /────────────\
       /              \ Unit Tests (70%)
      /                \ Jest / Vitest / Pytest — pure domain logic, use cases
     /──────────────────\
```

### 7.2 Testing Rules

| Test Type | Rule |
|---|---|
| **Unit** | No I/O, no network. Pure functions. < 10ms each. Run on every commit. |
| **Integration** | Real DB (TestContainers), real queue. Run on every PR. |
| **E2E** | Full user journey. Run on every deploy to staging. |
| **Contract** | Pact for consumer-driven contracts between services. |
| **Performance** | k6 / Gatling — weekly against staging. SLOs as pass/fail gate. |
| **LLM Evals** | Automated eval suite against golden examples. Track regression per deploy. |

- **Coverage:** 80% minimum overall, 90%+ on domain layer. Coverage is a floor, not a target.
- **LLM Evals:** Use LangSmith / Braintrust / custom harness. Track: accuracy, latency, cost, hallucination rate.

---

## Part 8 — CI/CD & DevOps

### 8.1 Pipeline Stages (Every Commit)

```
Code Push
  → [Lint + Type Check]      < 30s
  → [Unit Tests]             < 2 min
  → [Build + SAST Scan]      < 3 min   (Snyk, SonarQube)
  → [Integration Tests]      < 10 min  (TestContainers)
  → [Deploy → Staging]
  → [Smoke Tests + E2E]      < 5 min
  → [LLM Eval Suite]         < 5 min   (agentic pipelines only)
  → [Deploy → Production]    Blue/Green or Canary
```

### 8.2 Deployment Strategies

- **Blue/Green:** Instant switch, instant rollback. Higher infrastructure cost.
- **Canary:** 1% → 10% → 50% → 100%. Monitor SLOs at each stage. Auto-rollback on breach.
- **Feature Flags:** Decouple deployment from release. Ship dark, enable per user segment.
- **GitOps:** All infra in Terraform + Helm. Applied via ArgoCD. No manual `kubectl` in prod.

### 8.3 Infrastructure as Code Rules

- All infrastructure defined in Git. PRs for every infra change. Peer review required.
- `terraform plan` output included in every PR. No surprises.
- Immutable infrastructure: never SSH into servers. Replace, never patch.
- Environments are cattle, not pets.

---

## Part 9 — Code Quality Standards

### 9.1 Naming Conventions

```
Functions      → verbs:             getUserById(), processPayment()
Variables      → nouns:             userAccount, paymentResult
Booleans       → is / has / can:    isValid, hasPermission, canRetry
Constants      → SCREAMING_SNAKE:   MAX_RETRY_COUNT, DEFAULT_TIMEOUT_MS
Files          → kebab-case:        user-repository.ts, payment-service.ts
Classes        → PascalCase:        UserRepository, PaymentService
Events         → past-tense noun:   UserRegistered, PaymentFailed
Agent tools    → verb-noun:         search_web, execute_code, read_file
```

### 9.2 Function Design Rules

- One function, one thing. If the name needs "and", split it.
- Max **20 lines** per function. If longer, extract a well-named helper.
- Max **3 parameters**. Beyond that, use a named params object / struct.
- No side effects in query functions. Commands may have side effects.
- Return early to avoid nesting. Guard clauses over nested if-else trees.

### 9.3 Error Handling

```ts
// Typed domain errors — never throw raw strings
class PaymentFailedError extends DomainError {
  constructor(
    public readonly reason: PaymentFailureReason,
    public readonly transactionId: string,
  ) {
    super(`Payment failed: ${reason} [tx: ${transactionId}]`);
  }
}

// Result type for expected failures — no exceptions for flow control
type Result<T, E> = { ok: true; value: T } | { ok: false; error: E };

// Every error must carry: type, message, context, trace_id
// NEVER expose stack traces or internals to API clients
```

---

## Part 10 — Agentic-Specific Standards

### 10.1 Observability Schema — Every LLM Call

Log every LLM interaction with this exact schema. No exceptions.

```json
{
  "trace_id":       "uuid-v4",
  "agent_id":       "research-agent-v2",
  "run_id":         "uuid-v4",
  "step":           3,
  "model":          "claude-sonnet-4-20250514",
  "input_tokens":   1240,
  "output_tokens":  380,
  "latency_ms":     1843,
  "tool_calls":     ["search_web", "summarise"],
  "cost_usd":       0.0023,
  "cached":         false,
  "success":        true,
  "guardrail_hit":  false,
  "confidence":     0.94
}
```

**Latency SLOs:** p50 < 2s · p95 < 8s · p99 < 20s
**Cost SLOs:** Alert when per-user daily cost exceeds 2σ of rolling baseline.
**Human review queue:** Flag outputs with confidence < 0.80 for async review.

### 10.2 Agentic Workflow Orchestration

- Prefer **DAG (Directed Acyclic Graph)** execution over sequential chains.
- Use **LangGraph / Temporal / Prefect** for complex multi-step workflows.
- Every task node must be: retryable, observable, independently deployable.
- **Dead Letter Queue (DLQ):** Failed tasks are captured here. Never silently dropped.
- **Checkpoint:** Persist agent state at every major step. Resume from failure point on crash.

```
TaskGraph (DAG example):

  [Fetch Data]  ──→  [Validate]  ──→  [Enrich]  ──→  [Generate Report]
                                          ↑
  [Web Search]  ──→  [Analyse]  ─────────┘
```

### 10.3 Agentic Workflow Patterns

```ts
// Pattern: budget-enforced agent run
const run = await agent.execute({
  task:          userRequest,
  maxToolCalls:  50,
  maxSpendUsd:   5.00,
  maxWallTimeMs: 600_000,   // 10 minutes hard limit
  onBudgetExhausted: "return_partial",  // never "throw"
  requireHumanApproval: ["delete_*", "send_*", "deploy_*"],
});
```

---

## Architectural Decision Checklist

Before merging any significant architectural change, every item must be answered affirmatively.
This checklist is enforced in PR review.

### General Architecture

- [ ] Does it violate DRY? Is there any duplication of logic or configuration?
- [ ] Are services loosely coupled via events or well-defined interfaces?
- [ ] Is all config in environment variables (12-Factor III)?
- [ ] Is the service stateless — no in-process session state?
- [ ] Does it have circuit breakers and timeouts on every external call?
- [ ] Is it horizontally scalable without re-architecture?
- [ ] Are there retry and idempotency guarantees on all write operations?
- [ ] Is it fully observable: metrics, distributed traces, structured logs?
- [ ] Does it degrade gracefully when a downstream dependency fails?
- [ ] Are secrets managed securely — not in code, not in `.env` in prod?
- [ ] Is the database schema backward-compatible with the previous deployed version?
- [ ] Is there a documented rollback plan with a tested procedure?
- [ ] Are tests written at the correct level of the pyramid?

### Agentic / LLM-Specific

- [ ] Are all LLM model versions pinned — no `"latest"` aliases anywhere?
- [ ] Is there a hard cost cap per agent run (in USD)?
- [ ] Is there a hard tool call limit per agent run?
- [ ] Is there a hard wall-clock time limit per agent run?
- [ ] Are all tool calls rate-limited and fully observable?
- [ ] Are irreversible actions gated behind human confirmation?
- [ ] Is all agent code execution sandboxed in an isolated container?
- [ ] Is the context window token budget tracked and enforced?
- [ ] Are prompts versioned in Git and A/B tested via feature flags?
- [ ] Is there a kill switch to halt all agent execution instantly?
- [ ] Are LLM evals part of the CI pipeline with regression tracking?
- [ ] Is STT confidence thresholded — with a fallback to text input?
- [ ] Is TTS output streamed — not buffered — for low-latency voice?
- [ ] Is prompt injection mitigated for all user-controlled and external content?

---

## Canonical References

| Domain | Reference |
|---|---|
| Clean Code | *Clean Code* — Robert C. Martin |
| Architecture | *Clean Architecture* — Robert C. Martin |
| DDD | *Domain-Driven Design* — Eric Evans |
| Microservices | *Building Microservices* — Sam Newman |
| Data Systems | *Designing Data-Intensive Applications* — Martin Kleppmann |
| Resilience | *Release It!* — Michael Nygard |
| 12-Factor | https://12factor.net |
| SRE | *Site Reliability Engineering* — Google |
| Security | OWASP Top 10 + CIS Benchmarks |
| Agents | Anthropic Agent Patterns + LangGraph Documentation |
| LLM Evals | Braintrust Docs + RAGAS Framework |
| STT | OpenAI Whisper Docs + Deepgram API Reference |
| TTS | ElevenLabs API Reference + OpenAI TTS Docs |

---

*This document is a living standard. All engineers are expected to know it, apply it, and improve it.*
*PRs to this file require Staff Engineer review.*

---

**Version:** 2.0.0 — Agentic Edition | **Owner:** Architecture Guild | **2026**
