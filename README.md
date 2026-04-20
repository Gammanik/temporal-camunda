# Loan Origination Workflow: Camunda vs Temporal vs Restate

Comparison of three workflow orchestration engines for loan origination: **Camunda 8**, **Temporal**, and **Restate**.

Repository: https://github.com/Gammanik/temporal-camunda

## Overview

All three implementations handle identical business logic:

1. **Credit Check** → Evaluate creditworthiness (3 retries)
2. **Decision**: Auto-approve (≥750), Manual review (600-749, 7-day timeout), Auto-reject (<600)
3. **Contract Generation** → For approved loans

### Projects

- **[camunda-loan-poc/](./camunda-loan-poc/)** - BPMN-based (XML + workers)
- **[temporal-loan-poc/](./temporal-loan-poc/)** - Code-first (activities + deterministic replay)
- **[restate-loan-poc/](./restate-loan-poc/)** - Durable functions (event sourcing + Virtual Objects)

---

## Comparison

| Aspect | Camunda 8 | Temporal | Restate |
|--------|-----------|----------|---------|
| **Process Definition** | BPMN 2.0 XML | Code (Go, Java, Python, TS, PHP) | Code (Java, Kotlin, TS) |
| **Developer Experience** | XML + workers | Pure code, type-safe | Pure code, async/await |
| **Infrastructure** | Zeebe + Elasticsearch + Operate | Server + PostgreSQL + workers | Single binary |
| **Operational Overhead** | High | Medium | Low |
| **Licensing** | Enterprise for production (8.6+, Oct 2024) | MIT (self-host) / Cloud | BSL → Apache 2.0 after 4 years |
| **Key Challenge** | License cost ($50k–$330k/year) + integration issues | Deterministic replay footguns + versioning complexity | Younger ecosystem (2023) |
| **Best For** | Business analyst–owned workflows | Battle-tested maturity | Operational simplicity |

### Key Findings

**Camunda 8** — Not recommended for engineering-owned workflows.
- Since v8.6 (Oct 2024), Zeebe requires Enterprise license for production ($50k–$330k/year estimate)
- Hit license-related build issues in `spring-zeebe-starter` 8.6/8.7 during POC
- BPMN overhead pays off when business analysts own workflows; unrealized value otherwise

**Temporal** — Viable but operationally heavy.
- Requires Temporal Server (Frontend + History + Matching + Worker services) + PostgreSQL
- Deterministic replay contract is a persistent source of bugs (`LocalDateTime.now()`, `Math.random()` break on replay)
- Versioning via `getVersion()` accumulates `if-else` noise after 10–15 revisions

**Restate** — Recommended.
- Single binary, no external database, no separate workers
- Code-first Java SDK reads as regular functions with `ctx.run` at durability points
- Virtual Objects provide native stateful entities (per-key serialization for rate-limiting, idempotency)
- Built by ex-Apache Flink team; HA clustering since v1.2 (Feb 2025)
- BSL license converts to Apache 2.0 after 4 years

---

## Recommended Architecture

Keep the workflow engine as an infrastructure concern behind a port:

```
┌─────────────────────────────────────────────────────────┐
│                        Domain                           │
│  ┌─────────────────────────────────────────────────┐   │
│  │  LoanApplication (aggregate)                     │   │
│  │  sealed ApplicationState + FSM guards            │   │
│  │  Pure Java/Kotlin, zero framework dependencies   │   │
│  └─────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────┘
                          ↕
┌─────────────────────────────────────────────────────────┐
│                     Application                         │
│  ┌─────────────────────────────────────────────────┐   │
│  │  LoanApplicationService (transactional use-cases)│   │
│  │                                                  │   │
│  │  Ports:                                          │   │
│  │   - LoanApplicationRepository                    │   │
│  │   - WorkflowOrchestrator                         │   │
│  │   - DomainEventPublisher                         │   │
│  └─────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────┘
                          ↕
┌─────────────────────────────────────────────────────────┐
│                   Infrastructure                        │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  │
│  │ JPA adapter  │  │Restate adapter│  │ NATS adapter │  │
│  │ (PostgreSQL) │  │ (orchestrator)│  │ (events)     │  │
│  └──────────────┘  └──────────────┘  └──────────────┘  │
└─────────────────────────────────────────────────────────┘
```

**Key principles:**
- **Source of truth** = PostgreSQL (application state)
- **Restate** = Thin orchestrator calling back into LOS internal API
- **FSM guards & invariants** = Domain layer, not Restate
- **Reversible decision** = Swapping to Temporal = ~300 lines (workflow adapter + handler). Domain, persistence, REST API, tests untouched.

---

## Getting Started

### Quick Start - Camunda POC

```bash
cd camunda-loan-poc
docker-compose up -d
./gradlew bootRun
```

Access Camunda Operate: **http://localhost:8081** (demo/demo)

Submit a loan application:
```bash
curl -X POST http://localhost:8080/api/loan-applications \
  -H "Content-Type: application/json" \
  -d '{"applicantId": "CUST-001", "loanAmount": 50000, "applicantName": "John Doe"}'
```

### Quick Start - Temporal POC

```bash
cd temporal-loan-poc
docker-compose up -d
./gradlew run  # Start worker
./gradlew runClient  # Execute workflow
```

Access Temporal UI: **http://localhost:8233**

### Quick Start - Restate POC

```bash
cd restate-loan-poc
docker-compose up -d
./gradlew bootRun
```

Register services with Restate:
```bash
curl -X POST http://localhost:8080/deployments \
  -H "Content-Type: application/json" \
  -d '{"uri": "http://host.docker.internal:9080"}'
```

Submit a loan application:
```bash
curl -X POST http://localhost:9070/LoanApplicationWorkflow/${APPLICATION_ID}/run/processApplication \
  -H "Content-Type: application/json" \
  -d '{"applicationId": "APP-001", "applicantName": "Jane Smith", "amount": 50000, "income": 75000}'
```

Access Restate Admin API: **http://localhost:8080**

---

## Learn More

- **Camunda 8 Docs**: https://docs.camunda.io/
- **Temporal Docs**: https://docs.temporal.io/
- **Restate Docs**: https://docs.restate.dev/
- **BPMN 2.0 Spec**: https://www.omg.org/spec/BPMN/2.0/
- **UI Screenshots**: [UI-SCREENSHOTS.md](UI-SCREENSHOTS.md)

