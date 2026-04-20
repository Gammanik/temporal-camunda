# Loan Origination Workflow: Camunda vs Temporal Comparison

This repository demonstrates the **same loan origination workflow** implemented in both **Camunda 8 (Zeebe)** and **Temporal**, showcasing the key differences between BPMN-based and code-first workflow orchestration.

## 📸 UI Screenshots

**See the UIs in action**: [UI-SCREENSHOTS.md](UI-SCREENSHOTS.md) - Visual comparison of Camunda Operate vs Temporal Web UI

---

## Overview

Both implementations handle the same business logic:

1. **Credit Check** → Evaluate applicant's creditworthiness
2. **Decision Gateway**:
   - **Score ≥ 750**: Auto-approve → Generate contract
   - **Score 600-679**: Manual review → Wait up to 7 days for underwriter decision
   - **Score < 600**: Auto-reject
3. **Contract Generation** → For approved loans

### Projects

- **[camunda-loan-poc/](./camunda-loan-poc/)** - BPMN-based workflow with Spring Boot, PostgreSQL, and Camunda 8
- **[temporal-loan-poc/](./temporal-loan-poc/)** - Code-first workflow with Temporal's activity-based architecture
- **[restate-loan-poc/](./restate-loan-poc/)** - Durable execution with Restate's event sourcing and RPC-style programming

---

## Workflow Visualization

```
┌─────────────┐
│   Start     │
└──────┬──────┘
       │
       ▼
┌─────────────────┐
│  Credit Check   │ ◄─────────┐
│  (Retry: 3x)    │            │
└──────┬──────────┘            │
       │                       │
       ▼                       │
┌──────────────────┐           │
│ Decision Gateway │           │
└──────┬───────────┘           │
       │                       │
   ┌───┴───┬───────────┬───────┘
   │       │           │
   ▼       ▼           ▼
┌──────┐ ┌──────┐ ┌──────────────────┐
│REJECT│ │APPROVE│ │ MANUAL REVIEW    │
│      │ │       │ │ (7 day timeout)  │
└──────┘ └───┬───┘ └────────┬─────────┘
           │              │
           │          ┌───┴────┐
           │     Approved   Timeout
           │          │        │
           ▼          └────────┤
    ┌─────────────┐            ▼
    │  Generate   │      ┌────────┐
    │  Contract   │      │ Reject │
    └──────┬──────┘      └────────┘
           │
           ▼
       ┌────────┐
       │Complete│
       └────────┘
```

**Detailed State Machine**: See [docs/credit-check-fsm.mmd](docs/credit-check-fsm.mmd) for the full finite state machine diagram including KYC, Bureau Pull, Income Verification, and Risk Scoring steps.

---

## Quick Comparison

| Aspect | Camunda 8 | Temporal | Restate |
|--------|-----------|----------|---------|
| **Process Definition** | BPMN 2.0 XML | Kotlin/Java code | Kotlin/Java code |
| **Tooling** | Visual modeler | IDE (IntelliJ, VSCode) | IDE (IntelliJ, VSCode) |
| **Business Accessibility** | ⭐⭐⭐⭐⭐ Non-technical can understand | ⭐⭐ Requires programming knowledge | ⭐⭐ Requires programming knowledge |
| **Developer Experience** | ⭐⭐⭐ XML + Code | ⭐⭐⭐⭐⭐ Pure code with type safety | ⭐⭐⭐⭐⭐ Pure code with coroutines |
| **UI Monitoring** | Rich (Operate, Tasklist, Optimize) | Technical (Event history) | Admin API (JSON) |
| **Manual Tasks** | Built-in user tasks with forms | Signals + custom UI | Durable promises + custom UI |
| **Versioning** | BPMN version deployment | `Workflow.getVersion()` in code | Code versioning |
| **Debugging** | Operate UI + logs | IDE debugger + deterministic replay | IDE debugger + event replay |
| **Learning Curve** | BPMN standard | Temporal concepts (determinism, replay) | Event sourcing + async/await |
| **Infrastructure** | Zeebe + Elasticsearch + Operate | Temporal server + PostgreSQL + workers | Single binary (restate-server) |
| **Setup Time** | ~1 day (with license issues) | ~4 hours | ~1 hour |
| **External Dependencies** | PostgreSQL, Elasticsearch, Zeebe | PostgreSQL, Temporal server cluster | None (embedded storage) |
| **Licensing** | Enterprise (8.6+) for Zeebe in production | MIT (self-host) / Cloud (vendor) | BSL → OSS after 4 years |
| **State Management** | Zeebe broker state | Event history (external DB) | Native event sourcing (embedded) |
| **Operational Overhead** | High (multiple services) | Medium (cluster + DB) | Low (single process) |

---

## Architecture Comparison

### Camunda 8: BPMN-Driven Architecture

```
REST API (Spring Boot)
    ↓
Domain Services
    ↓
Zeebe Engine (BPMN XML Process Definition)
    ↓
Job Workers (@JobWorker annotated classes)
    ↓
PostgreSQL (Business Data)
```

**Key characteristics**:
- Process logic separated from business logic
- Visual BPMN diagrams as source of truth
- Zeebe distributes jobs to workers
- Operate UI for monitoring and incident management

### Temporal: Code-First Architecture

```
Workflow Definition (Kotlin/Java code)
    ↓
Activity Stubs (with retry policies)
    ↓
Activity Implementations (business logic)
    ↓
Temporal Server (event history + state)
```

**Key characteristics**:
- Workflow IS code - no separate definition files
- Type-safe, compile-time checked
- Event sourcing with deterministic replay
- Temporal UI for event history inspection

---

## Implementation Examples

### Camunda: BPMN + Worker

**BPMN Process Definition** (`loan-process.bpmn`):
```xml
<bpmn:serviceTask id="credit_check" name="Credit Check">
  <zeebe:taskDefinition type="credit-check" retries="3"/>
</bpmn:serviceTask>

<bpmn:exclusiveGateway id="Gateway_Decision">
  <bpmn:outgoing>Flow_Approved</bpmn:outgoing>
  <bpmn:outgoing>Flow_Manual_Review</bpmn:outgoing>
  <bpmn:outgoing>Flow_Rejected</bpmn:outgoing>
</bpmn:exclusiveGateway>

<bpmn:userTask id="manual_underwriting" name="Manual Underwriting"/>
```

**Worker Implementation**:
```kotlin
@Component
class CreditCheckWorker {
    @JobWorker(type = "credit-check", timeout = 60000L)
    fun performCreditCheck(
        @Variable loanApplicationId: Long
    ): Map<String, Any> {
        val score = simulateCreditScore()
        return mapOf(
            "creditScore" to score,
            "creditDecision" to determineDecision(score)
        )
    }
}
```

### Temporal: Pure Code

**Workflow Implementation**:
```kotlin
class LoanApplicationWorkflowImpl : LoanApplicationWorkflow {

    private val creditCheckActivity = Workflow.newActivityStub(
        CreditCheckActivity::class.java,
        ActivityOptions.newBuilder()
            .setStartToCloseTimeout(Duration.ofMinutes(1))
            .setRetryOptions(RetryOptions.newBuilder()
                .setMaximumAttempts(3)
                .build())
            .build()
    )

    override fun processApplication(app: LoanApplication): LoanResult {
        var attempts = 0

        while (attempts < MAX_MANUAL_REVIEWS) {
            // Credit check with auto-retry
            val creditResult = creditCheckActivity.checkCredit(app)

            when (val decision = makeDecision(creditResult.score)) {
                Decision.APPROVED -> {
                    val contract = contractActivity.generate(app)
                    return LoanResult.approved(contract)
                }
                Decision.REJECTED -> {
                    return LoanResult.rejected()
                }
                Decision.MANUAL_REVIEW -> {
                    attempts++
                    try {
                        // Wait for signal or 7-day timeout
                        Workflow.await(Duration.ofDays(7)) {
                            approvalReceived
                        }
                        continue // Loop back to credit check
                    } catch (e: TimeoutException) {
                        return LoanResult.rejectedByTimeout()
                    }
                }
            }
        }
    }
}
```

---

## Pros & Cons Summary

### Camunda 8 ✅❌

**Pros:**
- ✅ **Visual BPMN diagrams** - Universal business-technical communication
- ✅ **Rich UI ecosystem** - Operate (monitoring), Tasklist (user tasks), Optimize (analytics)
- ✅ **Business analyst friendly** - Non-developers can design and validate workflows
- ✅ **Built-in user tasks** - Forms, assignments, routing out-of-the-box
- ✅ **Industry standard** - BPMN 2.0 compliance and portability
- ✅ **Enterprise features** - Multi-tenancy, extensive audit, compliance reporting

**Cons:**
- ❌ **License change (8.6+, Oct 2024)** - Zeebe requires [Enterprise license for production use](https://camunda.com/blog/2024/10/camunda-8-6-launch/) (Community Edition removed)
- ❌ **Spring integration issues** - [spring-zeebe-starter compatibility problems](https://github.com/camunda-community-hub/spring-zeebe/issues) require downgrading to 8.5.x for stable development
- ❌ **XML overhead** - Verbose BPMN files, merge conflicts can be painful
- ❌ **Infrastructure complexity** - Zeebe + Elasticsearch + Operate
- ❌ **Learning curve** - Must learn BPMN standard and Zeebe specifics
- ❌ **Complex logic can be awkward** - BPMN not ideal for algorithmic workflows
- ❌ **Debugging** - Different from standard IDE debugging

**Best fit:** Human-in-the-loop BPM, compliance-owned workflows with business stakeholder involvement. For pure machine-to-machine orchestration, this is overkill.

### Temporal ✅❌

**Pros:**
- ✅ **Code-first** - Workflows are just code (Kotlin, Java, Go, Python, TypeScript)
- ✅ **Type safety** - Compile-time checks, IDE refactoring, autocomplete
- ✅ **Developer experience** - Standard debugging, testing, version control
- ✅ **Flexible logic** - Loops, conditionals, complex algorithms are natural
- ✅ **Event sourcing** - Perfect audit trail with time-travel debugging
- ✅ **Powerful versioning** - `Workflow.getVersion()` for in-flight evolution

**Cons:**
- ❌ **Deterministic replay requirement** - Any non-deterministic code (`LocalDateTime.now()`, `Random()`, external calls inside workflow) causes `NonDeterministicException` during replay. This is a significant mental shift for Java/Kotlin teams accustomed to standard programming.
- ❌ **Infrastructure complexity** - Requires Temporal Server + PostgreSQL + worker processes for self-hosting, or Temporal Cloud (vendor lock + ongoing costs)
- ❌ **Versioning complexity** - `getVersion()` patching becomes cumbersome with 10+ workflow versions; managing multiple in-flight version branches is error-prone
- ❌ **No visual designer** - No graphical process modeling
- ❌ **Business accessibility** - Requires programming knowledge
- ❌ **No built-in user tasks** - Must implement signals + custom UI
- ❌ **Technical UI** - Less polished than Camunda Operate

### Restate ✅❌

**Pros:**
- ✅ **Single binary deployment** - Zero external dependencies (no separate database required)
- ✅ **Minimal operational overhead** - One process vs. Temporal cluster or Camunda stack
- ✅ **Code-first workflows** - Write durable workflows as regular async functions with `ctx.run()`
- ✅ **Virtual Objects for stateful entities** - Maps naturally to domain objects like `LoanApplication`
- ✅ **Durable promises** - First-class coordination primitive for manual approvals and timeouts
- ✅ **Event sourcing built-in** - Complete audit trail without external event store
- ✅ **No BPMN lock-in** - Pure code, no XML or visual modeling required
- ✅ **HA clustering (v1.2+)** - Production-ready high availability since February 2025
- ✅ **Strong engineering pedigree** - Team includes ex-Apache Flink committers (Stephan Ewen et al.), $7M seed funding

**Cons:**
- ❌ **Younger ecosystem** - Project launched 2023, less battle-tested than Temporal (2019) or Camunda (2008+)
- ❌ **Smaller community** - Fewer Stack Overflow answers, community plugins, and third-party integrations
- ❌ **BSL license** - Business Source License converts to Apache 2.0 after 4 years (not immediate OSS like MIT)
- ❌ **Limited polyglot support** - Primarily Java/Kotlin, TypeScript; not as broad as Temporal's 5+ SDKs
- ❌ **No visual designer** - No graphical process modeling (like Temporal)
- ❌ **Maturity risk** - Fewer production deployments and edge case discoveries compared to incumbents

**Best fit:** Machine-to-machine orchestration, microservices coordination, developer-owned workflows where operational simplicity and architectural elegance matter more than visual tooling or maximal ecosystem maturity.

---

## When to Use Which

### Choose Camunda 8 when:

✅ **Business process management is a core capability**
- Non-technical stakeholders need to understand and approve workflows
- Process design involves business analysts and compliance officers
- Visual documentation is required for audits and regulations

✅ **Rich UI tooling is critical**
- Need out-of-the-box monitoring, incident management, and analytics
- User tasks with forms, assignments, and routing
- Real-time process visualization for operations teams

✅ **Standard BPMN workflows**
- Workflows align well with BPMN patterns
- Portability across BPMN-compliant platforms is valuable
- Industry standard compliance matters

**Example use cases:**
- Loan origination (this POC!)
- Insurance claims processing
- Regulatory compliance workflows
- Employee onboarding
- Order fulfillment with manual approvals

---

### Choose Temporal when:

✅ **Developer-centric organization**
- Engineering team owns workflow logic
- Code reviews and CI/CD are preferred
- Strong testing culture

✅ **Complex, dynamic business logic**
- Algorithmic decision-making
- Data-driven workflows with heavy conditionals
- Need for loops, recursion, complex state management

✅ **Microservices orchestration**
- Service-to-service coordination (saga pattern)
- Distributed transactions
- Long-running background jobs

✅ **Rapid iteration required**
- Fast-changing business requirements
- Need quick deployments without BPMN modeling
- Preference for code over visual configuration

**Example use cases:**
- E-commerce order processing
- Payment processing pipelines
- Data pipeline orchestration
- Distributed saga coordination
- Background job scheduling

---

### Choose Restate when:

✅ **Operational simplicity is paramount**
- Small team that can't afford dedicated infrastructure engineers
- Want to avoid managing separate database, queue, and workflow engine
- Prefer single-binary deployment over distributed clusters

✅ **Developer-owned workflows with minimal ops**
- Engineering team owns both application code and orchestration logic
- Need durable execution without Temporal's infrastructure burden
- Value architectural simplicity and fast iteration

✅ **Stateful service orchestration**
- Natural fit for entity-based workflows (orders, loans, accounts)
- Virtual Objects map cleanly to domain models
- Need direct RPC-style service invocation vs. activity stubs

✅ **Startup or greenfield projects**
- Can tolerate younger ecosystem in exchange for simpler architecture
- Team comfortable with bleeding-edge tech (ex-Flink pedigree reduces risk)
- Want to avoid licensing surprises (BSL is explicit)

**Example use cases:**
- Loan origination (this POC!)
- Order processing and fulfillment
- Account lifecycle management
- Microservices saga orchestration
- Scheduled job workflows with state

---

## Using Both Together: Trade-offs

Some organizations run **both Camunda and Temporal** (or Restate) simultaneously:

**Potential architecture:**
- Camunda for customer-facing, compliance-heavy workflows (visual BPMN for audits)
- Temporal/Restate for backend machine-to-machine orchestration (code-first reliability)

**Reality check:**
- ❌ **Double infrastructure cost** - Run and maintain two separate orchestration platforms
- ❌ **Double operational overhead** - Two monitoring systems, two deployment pipelines, two on-call rotations
- ❌ **Team expertise dilution** - Developers must learn both systems, slows velocity
- ❌ **Integration complexity** - Coordinating workflows across platforms adds failure modes

**Verdict for lending:** For most lending use cases (including this POC), running both is overkill. Pick one based on your organization's culture and constraints. If business stakeholders need BPMN, choose Camunda. If developer velocity and simplicity matter most, choose Restate. If you need battle-tested maturity, choose Temporal.

---

## Recommendation

**For the loan origination workflow in this POC, we recommend Restate.**

### Why Restate?

✅ **Minimal operational overhead** - Single binary vs. Temporal cluster (PostgreSQL + server + workers) or Camunda stack (Zeebe + Elasticsearch + Operate)

✅ **Code-first without BPMN lock-in** - Pure Kotlin/Java workflows, no XML ceremony, same developer experience as Temporal but simpler deployment

✅ **Native stateful entities** - Virtual Objects map naturally to `LoanApplication` state machine; cleaner than Temporal's activity stubs or Camunda's external state management

✅ **No licensing landmines** - BSL is explicit and converts to OSS after 4 years; no surprise Enterprise requirements like Camunda 8.6+

✅ **Architectural simplicity reduces risk** - Fewer moving parts = fewer failure modes, easier debugging, faster onboarding

### Main Risk: Maturity

❌ **Younger project** - Restate launched in 2023 vs. Temporal (2019) / Camunda (2008+). Smaller community, fewer production war stories.

**Mitigation:** The team's pedigree (Apache Flink committers) and architectural simplicity itself reduce risk. A simpler system with strong fundamentals beats a complex mature system for most teams.

### When NOT to Choose Restate

- **Business stakeholders require BPMN** → Choose Camunda
- **Need maximum ecosystem maturity** → Choose Temporal
- **Require extensive polyglot SDKs** → Choose Temporal (Go, Python, PHP, .NET vs. Restate's Java/Kotlin/TypeScript)

---

## Key Differences at a Glance

| Feature | Camunda 8 | Temporal | Restate |
|---------|-----------|----------|---------|
| **Process definition** | BPMN 2.0 XML | Code (multiple languages) | Code (Kotlin/Java/TS) |
| **Visual designer** | ✅ Camunda Modeler | ❌ No | ❌ No |
| **State management** | Zeebe broker | Event history (event sourcing) | Event sourcing (embedded) |
| **Manual tasks** | Built-in user tasks | Custom signals + UI | Durable promises + UI |
| **Timeouts** | BPMN timer events (`P7D`) | `Workflow.await(Duration.ofDays(7))` | `ctx.promise().await()` |
| **Retries** | `@JobWorker(retries=3)` | `RetryOptions.setMaximumAttempts(3)` | Automatic via `ctx.run()` |
| **Versioning** | Deploy new BPMN versions | `Workflow.getVersion()` in code | Code versioning |
| **Monitoring UI** | Rich (Operate, Optimize) | Technical (Event History) | Admin API (JSON) |
| **Debugging** | Operate UI + logs | IDE debugger + replay | IDE debugger + event replay |
| **Testing** | `zeebe-process-test` | Standard JUnit/TestNG | Standard JUnit/TestNG |
| **Learning curve** | BPMN standard | Determinism + event sourcing | Event sourcing + async/await |
| **Deployment** | Multi-service cluster | Server cluster + workers | Single binary |

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

