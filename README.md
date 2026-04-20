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
| **Process Definition** | BPMN 2.0 XML | Code | Code |
| **Developer Experience** | XML + workers | Pure code, type-safe | Pure code |
| **Infrastructure** | Zeebe + Elasticsearch + Operate | Server + PostgreSQL + workers | Single binary |
| **Operational Overhead** | High | Medium | Low |
| **Licensing** | Enterprise for production (8.6+) | MIT (self-host) / Cloud | BSL → Apache 2.0 after 4 years |
| **Key Challenge** | License cost + integration issues | Deterministic replay + versioning | Younger ecosystem |
| **Best For** | Business analyst–owned workflows | Battle-tested maturity | Operational simplicity |

### Key Findings

**Camunda 8** — Not recommended for engineering-owned workflows.
- Enterprise license required for production
- BPMN overhead pays off when business analysts own workflows; unrealized value otherwise

**Temporal** — Viable but operationally heavy.
- Requires server cluster + PostgreSQL + worker processes
- Deterministic replay contract is a persistent source of bugs
- Versioning accumulates complexity after multiple revisions

**Restate** — Recommended.
- Single binary, no external dependencies
- Code-first with `ctx.run` at durability points
- Virtual Objects for stateful entities
- Built by ex-Apache Flink team; HA clustering available
- BSL license converts to Apache 2.0 after 4 years

---

## Recommended Architecture

Keep the workflow engine as an infrastructure concern behind a port:

```
┌─────────────────────────────────────────────────────────┐
│                        Domain                           │
│  ┌─────────────────────────────────────────────────┐   │
│  │  LoanApplication (aggregate)                     │   │
│  │  ApplicationState + FSM guards                   │   │
│  │  Zero framework dependencies                     │   │
│  └─────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────┘
                          ↕
┌─────────────────────────────────────────────────────────┐
│                     Application                         │
│  ┌─────────────────────────────────────────────────┐   │
│  │  LoanApplicationService (use-cases)              │   │
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
│  │ PostgreSQL   │  │Restate adapter│  │ NATS adapter │  │
│  │ adapter      │  │ (orchestrator)│  │ (events)     │  │
│  └──────────────┘  └──────────────┘  └──────────────┘  │
└─────────────────────────────────────────────────────────┘
```

**Key principles:**
- **Source of truth** = PostgreSQL
- **Restate** = Thin orchestrator calling back into internal API
- **FSM guards & invariants** = Domain layer, not Restate
- **Reversible decision** = Swapping engines affects only workflow adapter. Domain, persistence, REST API, tests untouched.

