# Banque Misr Internal Banking Operations Copilot

An internal AI copilot for **simulated** Banque Misr branch employees. It answers operational questions from approved banking documents (with page-level citations), refuses when evidence is insufficient, and helps prepare service requests that a supervisor must approve before any simulated action executes.

> **Disclaimer:** This is a training project. It uses synthetic data and public or purpose-created documents. It does **not** connect to real Banque Misr systems and never performs real banking transactions.

---

## Table of Contents

1. [Features](#features)
2. [Tech Stack](#tech-stack)
3. [Architecture](#architecture)
4. [Roles and Permissions](#roles-and-permissions)
5. [Getting Started](#getting-started)
6. [Configuration](#configuration)
7. [API Overview](#api-overview)
8. [Document Ingestion Pipeline](#document-ingestion-pipeline)
9. [Retrieval and Grounded Answering](#retrieval-and-grounded-answering)
10. [Multi-Agent Workflow](#multi-agent-workflow)
11. [Service Request Lifecycle](#service-request-lifecycle)
12. [Security](#security)
13. [Observability and Audit](#observability-and-audit)
14. [Testing](#testing)
15. [Evaluation Results](#evaluation-results)
16. [Demo Scenario](#demo-scenario)
17. [Known Limitations](#known-limitations)
18. [Team](#team)

---

## Features

- **AI Copilot:** natural-language Q&A over ingested documents, with citations (document + page/section).
- **Explicit refusal** when the knowledge base lacks sufficient evidence.
- **Prompt-injection resistance** for instructions embedded in ingested documents.
- **Hybrid retrieval:** semantic (vector) + keyword search, combined with a documented ranking method.
- **Service request management:** create, edit, submit, and track simulated requests.
- **Supervisor approval workflow:** approve, reject (with reason), or edit-and-approve.
- **Multi-agent orchestration:** Knowledge Retrieval, Policy Validation, and Request Preparation agents coordinated by an orchestrator, with timeouts, iteration limits, and fallback.
- **JWT authentication** and server-enforced role-based authorization.
- **Audit trail** for decisions and consequential actions; token usage and cost tracking per request.
- **Evaluation suite** (15+ cases) reporting retrieval hit rate, groundedness, and refusal correctness.
- **Dockerized** setup with health/readiness endpoints.

## Tech Stack

| Layer | Technology |
|---|---|
| Language | Java 17+ |
| Framework | Spring Boot, Spring MVC |
| Persistence | Spring Data JPA, Hibernate, PostgreSQL (+ pgvector) |
| Security | Spring Security, JWT, BCrypt |
| AI | Spring AI + `<LLM provider>` |
| Build | Maven |
| Testing | JUnit 5, Mockito, Spring Boot Test |
| Ops | Docker, Docker Compose, Spring Boot Actuator |

## Architecture

The application is a **modular Spring Boot monolith**.

| Module | Responsibility |
|---|---|
| `identity` | Authentication, users, roles, permissions |
| `knowledge` | Document ingestion and metadata |
| `retrieval` | Semantic search, keyword search, ranking |
| `copilot` | Grounded answering and agent orchestration |
| `service-request` | Request creation, validation, lifecycle |
| `approval` | Supervisor review and simulated execution |
| `audit` | Request tracing, activity history, AI usage |

```
Client ──► REST API (Spring MVC + Security/JWT)
              │
              ├── identity
              ├── knowledge ──► ingestion pipeline ──► PostgreSQL + pgvector
              ├── retrieval ◄── hybrid search (vector + keyword) 
              ├── copilot ──► Orchestrator ──► Retrieval Agent
              │                            ├─► Policy Validation Agent
              │                            └─► Request Preparation Agent
              ├── service-request
              ├── approval ──► simulated execution
              └── audit (logs, history, token usage)
```

### Key architecture decisions

> Fill in your real decisions. Examples to document:

- **Why a modular monolith** (simplicity, single deployment, clear module boundaries).
- **Why pgvector** (one datastore for relational and vector data).
- **Why this chunking strategy** (chunk size, overlap, and rationale).
- **Why this ranking method** (e.g., Reciprocal Rank Fusion with k = 60).
- **Why typed agent messages** (records/DTOs instead of free-form strings).
- **How the approval gate is enforced** (service-layer checks plus method-level security, not just UI).

## Roles and Permissions

| Capability | EMPLOYEE | SUPERVISOR |
|---|:---:|:---:|
| Ask copilot questions | ✅ | ✅ |
| Create/edit own draft requests | ✅ | ✅ |
| Submit own requests | ✅ | ✅ |
| View own request status/history | ✅ | ✅ |
| View other employees' requests | ❌ | ✅ |
| View pending approval queue | ❌ | ✅ |
| Approve / reject / edit-and-approve | ❌ | ✅ |
| Execute protected actions directly | ❌ | ❌ (only via approval flow) |
| Change own role | ❌ | ❌ |

Permissions are enforced **server-side** on every protected endpoint.

## Getting Started

### Prerequisites

- Java 17+
- Maven 3.9+
- Docker and Docker Compose
- An API key for `<LLM provider>`

### Run with Docker (recommended)

```bash
git clone https://github.com/iti-backend-graduation/<repo-name>.git
cd <repo-name>
cp .env.example .env        # then edit values (DB password, JWT secret, LLM API key)
docker compose up --build
```

The app will be available at `http://localhost:8080`.

### Run locally (without Docker for the app)

```bash
docker compose up -d postgres
./mvnw spring-boot:run
```

### Seed data

On first startup, the app seeds:

| Username | Password | Role |
|---|---|---|
| `employee1` | `<set in .env / seed script>` | EMPLOYEE |
| `employee2` | `<set in .env / seed script>` | EMPLOYEE |
| `supervisor1` | `<set in .env / seed script>` | SUPERVISOR |

> These are demo-only credentials. Never reuse them anywhere real.

### Ingest the documents

```bash
# Example, adjust to your actual endpoint or command
curl -X POST http://localhost:8080/api/knowledge/ingest \
  -H "Authorization: Bearer <token>" \
  -F "file=@docs/sample/card-services.pdf"
```

The corpus (15–20 documents, 60–80 pages, PDF + Markdown) lives in `docs/corpus/`.

## Configuration

| Variable | Description |
|---|---|
| `SPRING_DATASOURCE_URL` | PostgreSQL JDBC URL |
| `SPRING_DATASOURCE_USERNAME` / `PASSWORD` | Database credentials |
| `JWT_SECRET` | Secret used to sign tokens |
| `JWT_EXPIRATION_MS` | Token lifetime |
| `SPRING_AI_<PROVIDER>_API_KEY` | LLM provider key |
| `COPILOT_AGENT_TIMEOUT_MS` | Per-step agent timeout |
| `COPILOT_AGENT_MAX_ITERATIONS` | Orchestrator iteration limit |

Never commit real secrets. Use `.env` (git-ignored) and provide `.env.example`.

## API Overview

> Adjust paths to match your implementation. Swagger/OpenAPI is available at `/swagger-ui.html` if enabled.

| Method | Endpoint | Role | Description |
|---|---|---|---|
| POST | `/api/auth/login` | Public | Obtain JWT |
| POST | `/api/copilot/ask` | EMPLOYEE+ | Ask a question |
| POST | `/api/knowledge/ingest` | SUPERVISOR (or admin flow) | Ingest a document |
| POST | `/api/requests` | EMPLOYEE+ | Create draft request |
| PUT | `/api/requests/{id}` | Owner | Edit own draft |
| POST | `/api/requests/{id}/submit` | Owner | Submit for approval |
| GET | `/api/requests/{id}` | Owner / SUPERVISOR | View request |
| GET | `/api/requests/mine` | EMPLOYEE+ | List own requests |
| GET | `/api/approvals/pending` | SUPERVISOR | Pending queue |
| POST | `/api/approvals/{id}/approve` | SUPERVISOR | Approve |
| POST | `/api/approvals/{id}/reject` | SUPERVISOR | Reject with reason |
| POST | `/api/approvals/{id}/edit-approve` | SUPERVISOR | Edit and approve |
| GET | `/actuator/health` | Public | Health/readiness |

## Document Ingestion Pipeline

Each stage is a separate, identifiable component:

1. **Extract:** read PDF/Markdown, preserving page or section info.
2. **Clean:** normalize whitespace, remove headers/footers and noise.
3. **Chunk:** split into overlapping chunks (`<size>` tokens, `<overlap>` overlap).
4. **Embed:** generate vectors via the configured embedding model.
5. **Index:** store chunks, vectors, and metadata in PostgreSQL.

Stored metadata: source document, page/section, version or ingestion timestamp, and a content hash.
**Duplicate prevention:** documents are identified by content hash; re-ingesting the same file is rejected or skipped.

## Retrieval and Grounded Answering

- **Semantic search:** pgvector cosine similarity.
- **Keyword search:** PostgreSQL full-text search.
- **Ranking:** results merged with `<Reciprocal Rank Fusion / weighted score>`, top-K = `<N>`.
- **Answering:** the LLM answers only from retrieved passages and must cite document + page/section.
- **Refusal:** if no passage passes the relevance threshold (`<value>`) or evidence is insufficient, the copilot returns an explicit "insufficient information" response.
- **Prompt-injection defense:** retrieved text is treated as untrusted data and delimited; the system prompt forbids following instructions found in documents. Tested in the evaluation suite.

## Multi-Agent Workflow

| Agent | Responsibility | Allowed tools | Termination |
|---|---|---|---|
| Knowledge Retrieval | Find passages and citations | Retrieval service (read-only) | Returns top-K passages or none |
| Policy Validation | Check evidence, flag missing info/assumptions | Evidence from retrieval only | Returns validation result |
| Request Preparation | Produce structured request draft | Request-draft builder (no submit/approve) | Returns typed draft |
| **Orchestrator** | Coordinates agents and workflow | All agents | Completes, hits iteration limit, or falls back |

- Agents communicate through **typed structures** (Java records/DTOs).
- **Iteration limit** and **per-step timeout** are configurable.
- **Fallback:** if orchestration fails, the system falls back to ordinary retrieval and answering.
- Agents can **never** approve or execute requests; the supervisor gate is enforced outside the agent layer.

## Service Request Lifecycle

```
DRAFT ──► PENDING_APPROVAL ──► APPROVED ──► EXECUTED
                  │
                  └──────────► REJECTED
```

1. Employee creates and submits a request.
2. The system validates it and places it in the approval queue.
3. Supervisor reviews it.
4. Supervisor approves, rejects (reason required), or edits and approves.
5. The decision is recorded and the **simulated** action executes only if approved.

Duplicate execution is prevented by state checks (and optimistic locking / idempotency guard).

Example categories: card-service inquiry, digital banking support, account-service inquiry, additional documentation required.

## Security

- JWT authentication; passwords hashed with BCrypt.
- Role-based authorization enforced on the server (`@PreAuthorize` + service-level ownership checks).
- Employees cannot access other employees' requests.
- Bean Validation on all incoming data.
- Parameterized queries via JPA (no string-concatenated SQL).
- Audit records for approval decisions and consequential actions.

## Observability and Audit

- **Correlation/run ID** propagated through API, orchestrator, agents, and model calls (visible in logs).
- **Structured logs** (JSON or key-value).
- **Audit history:** request changes, approvals (who, decision, timestamp, edits), execution outcomes.
- **AI usage:** token counts and estimated cost per request, stored in the database (`ai_usage` table) and queryable.
- **Health endpoints:** `/actuator/health`, `/actuator/health/readiness`, `/actuator/health/liveness`.

## Testing

```bash
./mvnw test                # unit tests
./mvnw verify              # unit + integration tests
./mvnw test -Dtest=*SecurityTest   # security tests only
```

| Type | Coverage |
|---|---|
| Unit | Business logic, validation, state transitions |
| Integration | REST endpoints and persistence |
| Security | Role enforcement, unauthorized access to others' requests |
| Workflow | Approval transitions, duplicate execution prevention |
| AI | Retrieval behavior, agent failure and fallback |

## Evaluation Results

Run the evaluation suite:

```bash
./mvnw test -Dtest=EvaluationSuite
# or: ./scripts/run-evaluation.sh
```

The suite contains 15+ cases covering: correct retrieval and citations, out-of-scope questions, ambiguous questions, prompt-injection attempts (including in-document), incomplete service requests, unauthorized access/approval attempts, rejected and edited requests, duplicate execution, and agent failure/fallback.

| Metric | Result |
|---|---|
| Retrieval hit rate | `__ %` (`x / y`) |
| Groundedness | `__ %` |
| Refusal correctness | `__ %` |

> Replace with **actual measured numbers**, then list failed cases and why they failed.

### Failures and analysis

| Case | Expected | Actual | Cause / Notes |
|---|---|---|---|
| _TBD_ | | | |

## Demo Scenario

A repeatable end-to-end walkthrough:

1. Start the app: `docker compose up --build`.
2. Log in as `employee1` and get a JWT.
3. Ask a question that exists in the documents → verify the answer includes document + page citations.
4. Ask an out-of-scope question → verify the explicit refusal.
5. Create and submit a service request (e.g., card replacement inquiry).
6. As `employee1`, try calling the approve endpoint → expect **403 Forbidden**.
7. Log in as `supervisor1`, open the pending queue, and review the request.
8. Approve (or reject with a reason, or edit and approve).
9. Verify the status moves to `EXECUTED` only after approval, and that a second execution attempt is rejected.
10. Inspect the audit history and AI usage records.

Include sample `curl` commands or a Postman collection in `docs/demo/`.

## Known Limitations

> Be honest. Examples:

- Retrieval quality depends on the small synthetic corpus.
- Arabic documents/queries are not supported (bonus feature).
- LLM outputs are non-deterministic; evaluation scores can vary slightly between runs.
- Execution is simulated only.

## Team

| Name | Role |
|---|---|
| _Your name_ | _Contribution_ |

---

_Built as the Final Capstone Project of the ITI Backend Transition Track._
