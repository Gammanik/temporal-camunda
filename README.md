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

---

## Quick Comparison

| Aspect | Camunda 8 | Temporal |
|--------|-----------|----------|
| **Process Definition** | BPMN 2.0 XML | Kotlin/Java code |
| **Tooling** | Visual modeler | IDE (IntelliJ, VSCode) |
| **Business Accessibility** | ⭐⭐⭐⭐⭐ Non-technical can understand | ⭐⭐ Requires programming knowledge |
| **Developer Experience** | ⭐⭐⭐ XML + Code | ⭐⭐⭐⭐⭐ Pure code with type safety |
| **UI Monitoring** | Rich (Operate, Tasklist, Optimize) | Technical (Event history) |
| **Manual Tasks** | Built-in user tasks with forms | Signals + custom UI |
| **Versioning** | BPMN version deployment | `Workflow.getVersion()` in code |
| **Debugging** | Operate UI + logs | IDE debugger + deterministic replay |
| **Learning Curve** | BPMN standard | Temporal concepts (determinism, replay) |
| **Infrastructure** | Zeebe + Elasticsearch + Operate | Temporal server (lighter) |

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
- ❌ **XML overhead** - Verbose BPMN files, merge conflicts can be painful
- ❌ **Infrastructure complexity** - Zeebe + Elasticsearch + Operate
- ❌ **Learning curve** - Must learn BPMN standard and Zeebe specifics
- ❌ **Complex logic can be awkward** - BPMN not ideal for algorithmic workflows
- ❌ **Debugging** - Different from standard IDE debugging

### Temporal ✅❌

**Pros:**
- ✅ **Code-first** - Workflows are just code (Kotlin, Java, Go, Python, TypeScript)
- ✅ **Type safety** - Compile-time checks, IDE refactoring, autocomplete
- ✅ **Developer experience** - Standard debugging, testing, version control
- ✅ **Flexible logic** - Loops, conditionals, complex algorithms are natural
- ✅ **Event sourcing** - Perfect audit trail with time-travel debugging
- ✅ **Powerful versioning** - `Workflow.getVersion()` for in-flight evolution

**Cons:**
- ❌ **No visual designer** - No graphical process modeling
- ❌ **Business accessibility** - Requires programming knowledge
- ❌ **No built-in user tasks** - Must implement signals + custom UI
- ❌ **Technical UI** - Less polished than Camunda Operate
- ❌ **Determinism constraints** - Cannot use random values or `System.currentTimeMillis()` directly
- ❌ **Learning curve** - Unique concepts (deterministic replay, workflow constraints)

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

## Using Both Together: Hybrid Approach

**Many fintech companies use BOTH Camunda and Temporal in the same application.** This hybrid approach leverages the strengths of each platform:

### Why Combine Them?

| Workflow Type | Best Tool | Reason |
|---------------|-----------|--------|
| **Customer-facing processes** | Camunda | Visual documentation for compliance, business stakeholder involvement |
| **Backend orchestration** | Temporal | Complex logic, microservice coordination, developer-owned |
| **Approval workflows** | Camunda | Built-in user tasks, forms, and tasklist UI |
| **Payment processing** | Temporal | Deterministic execution, perfect audit trail, retry logic |
| **Loan origination** | Camunda | Visual BPMN for regulatory compliance |
| **Settlement workflows** | Temporal | Complex calculations, event sourcing, time-travel debugging |

### Real-World Hybrid Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                     Fintech Application                      │
├─────────────────────────────────────────────────────────────┤
│                                                               │
│  Customer Journey (Camunda)                                  │
│  ┌─────────────────────────────────────────────────────┐    │
│  │ • Account opening (BPMN with KYC approvals)         │    │
│  │ • Loan application (manual underwriting steps)      │    │
│  │ • Dispute resolution (customer service tasks)       │    │
│  └─────────────────────────────────────────────────────┘    │
│                           │                                  │
│                           ▼                                  │
│                   [REST API / Events]                        │
│                           │                                  │
│                           ▼                                  │
│  Backend Processes (Temporal)                                │
│  ┌─────────────────────────────────────────────────────┐    │
│  │ • Payment processing (retry + idempotency)          │    │
│  │ • Transaction settlement (complex calculations)     │    │
│  │ • Fraud detection pipeline (ML model orchestration) │    │
│  │ • Account reconciliation (scheduled jobs)           │    │
│  └─────────────────────────────────────────────────────┘    │
│                                                               │
└─────────────────────────────────────────────────────────────┘
```

### Example: Loan Workflow Hybrid

**Camunda handles**:
- Initial application submission
- Credit check approval decision
- Manual underwriting (user tasks with forms)
- Final contract generation and signing
- Compliance audit trail with BPMN visualization

**Temporal handles** (triggered by Camunda):
- Third-party credit bureau API calls (with sophisticated retry logic)
- Payment disbursement to customer bank account
- Automated payment collection (monthly, with exponential backoff)
- Default detection and collection workflows
- Settlement reconciliation

### Communication Patterns

**Camunda → Temporal**:
```kotlin
// In Camunda worker, start a Temporal workflow
@JobWorker(type = "initiate-payment")
fun initiatePayment(@Variable loanId: Long) {
    val temporalClient = WorkflowClient.newInstance(/*...*/)
    val paymentWorkflow = temporalClient.newWorkflowStub(
        PaymentWorkflow::class.java,
        WorkflowOptions.newBuilder()
            .setWorkflowId("payment-$loanId")
            .build()
    )
    WorkflowClient.start { paymentWorkflow.processPayment(loanId) }
}
```

**Temporal → Camunda** (via message correlation):
```kotlin
// In Temporal activity, signal Camunda process
override fun notifyPaymentComplete(loanId: Long, status: PaymentStatus) {
    zeebeClient.newPublishMessageCommand()
        .messageName("payment-completed")
        .correlationKey(loanId.toString())
        .variables(mapOf("paymentStatus" to status))
        .send()
}
```

### Benefits of Hybrid Approach

✅ **Best of both worlds**:
- Use Camunda where business visibility and user tasks matter
- Use Temporal where complex logic and reliability are critical

✅ **Team specialization**:
- Business analysts work on Camunda processes
- Backend engineers own Temporal workflows

✅ **Scalability**:
- Camunda for moderate volume, user-facing processes
- Temporal for high-volume, automated backend flows

✅ **Gradual adoption**:
- Start with one platform
- Add the other as needs evolve
- No need to choose just one

### Trade-offs

❌ **Infrastructure complexity**: Running and maintaining both platforms
❌ **Operational overhead**: Two monitoring UIs, two deployment processes
❌ **Team expertise**: Developers need to learn both systems
❌ **Integration complexity**: Ensuring reliable communication between platforms

---

## Key Differences at a Glance

| Feature | Camunda 8 | Temporal |
|---------|-----------|----------|
| **Process definition** | BPMN 2.0 XML | Code (multiple languages) |
| **Visual designer** | ✅ Camunda Modeler | ❌ No |
| **State management** | Zeebe broker | Event history (event sourcing) |
| **Manual tasks** | Built-in user tasks | Custom signals + UI |
| **Timeouts** | BPMN timer events (`P7D`) | `Workflow.await(Duration.ofDays(7))` |
| **Retries** | `@JobWorker(retries=3)` | `RetryOptions.setMaximumAttempts(3)` |
| **Versioning** | Deploy new BPMN versions | `Workflow.getVersion()` in code |
| **Monitoring UI** | Rich (Operate, Optimize) | Technical (Event History) |
| **Debugging** | Operate UI + logs | IDE debugger + replay |
| **Testing** | `zeebe-process-test` | Standard JUnit/TestNG |
| **Learning curve** | BPMN standard | Determinism + event sourcing |

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

---

## Learn More

- **Camunda 8 Docs**: https://docs.camunda.io/
- **Temporal Docs**: https://docs.temporal.io/
- **BPMN 2.0 Spec**: https://www.omg.org/spec/BPMN/2.0/
- **UI Screenshots**: [UI-SCREENSHOTS.md](UI-SCREENSHOTS.md)

---

## Conclusion

Both Camunda and Temporal are powerful workflow orchestration platforms with different philosophies:

- **Camunda** excels at business process management with visual BPMN, rich UI tooling, and stakeholder collaboration
- **Temporal** shines for developer-centric, code-first workflows with complex logic and strong reliability guarantees

**The best choice depends on your organization's culture, use case, and requirements.** Many successful companies use both in a hybrid architecture, applying each where it provides the most value.

This POC demonstrates that the **same business workflow** can be implemented in both platforms, giving you hands-on experience to make an informed decision.
