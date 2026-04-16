# Loan Origination Workflow Comparison: Camunda vs Temporal

This repository contains two parallel implementations of the same loan origination workflow, demonstrating the differences between **Camunda 8 (Zeebe)** and **Temporal** for orchestrating business processes.

## Table of Contents
- [Overview](#overview)
- [Loan Flow Logic](#loan-flow-logic)
- [Architecture Comparison](#architecture-comparison)
- [Technology Stack](#technology-stack)
- [Implementation Details](#implementation-details)
- [Pros and Cons](#pros-and-cons)
- [When to Use Which](#when-to-use-which)
- [Key Differences](#key-differences)
- [Running the Projects](#running-the-projects)

---

## Overview

Both POCs implement the same loan origination workflow with the following business logic:
1. **Credit Check**: Evaluate applicant's creditworthiness
2. **Decision Gateway**: Route based on credit score
   - **Approved** (score ≥ 750): Generate contract and complete
   - **Rejected** (score < 600 or outside manual review range): Immediately reject
   - **Manual Review** (600 ≤ score < 680): Requires human intervention
3. **Manual Review Process**:
   - Wait for underwriter decision with 7-day timeout
   - If approved by underwriter, loop back to credit check
   - If timeout expires, auto-reject the application
4. **Contract Generation**: For approved loans, generate loan contract

### Projects
- **[camunda-loan-poc](./camunda-loan-poc/)**: Full Spring Boot application with REST API, PostgreSQL persistence, and Camunda 8 orchestration
- **[temporal-loan-poc](./temporal-loan-poc/)**: Temporal workflow implementation with activity-based architecture

---

## Loan Flow Logic

### Visual Flow

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
           │          │        │
           │     Approved   Timeout
           │          │        │
           │          └────────┤
           │                   │
           ▼                   ▼
    ┌─────────────┐      ┌────────┐
    │  Generate   │      │ Auto   │
    │  Contract   │      │ Reject │
    └──────┬──────┘      └────────┘
           │
           ▼
       ┌────────┐
       │Complete│
       └────────┘
```

### Decision Logic

| Credit Score Range | Decision | Next Step |
|--------------------|----------|-----------|
| ≥ 750 | APPROVED | Generate Contract → Complete |
| 600-679 | MANUAL_REVIEW | Wait for underwriter (7 days) → Re-check |
| < 600 | REJECTED | End process |

---

## Architecture Comparison

### Camunda 8 Architecture

```
┌─────────────────────────────────────────────────────────┐
│                    REST API Layer                        │
│  Spring Boot Controllers + DTOs                          │
└─────────────────────────────────────────────────────────┘
                          ↓
┌─────────────────────────────────────────────────────────┐
│                   Domain Services                        │
│  • LoanService (orchestration)                           │
│  • RolloutService (version selection)                    │
│  • StateTransitionService (audit)                        │
└─────────────────────────────────────────────────────────┘
                          ↓
┌─────────────────────────────────────────────────────────┐
│              Zeebe Workflow Engine (BPMN)                │
│  • Visual workflow definition (XML)                      │
│  • Process instance state management                     │
│  • Job distribution to workers                           │
└─────────────────────────────────────────────────────────┘
                          ↓
┌─────────────────────────────────────────────────────────┐
│                    Zeebe Job Workers                     │
│  • CreditCheckWorker (@JobWorker annotation)             │
│  • ContractGenerationWorker                              │
│  • DocVerificationWorker (v2)                            │
└─────────────────────────────────────────────────────────┘
                          ↓
┌─────────────────────────────────────────────────────────┐
│                    PostgreSQL Database                   │
│  • loan_applications (business data)                     │
│  • state_transitions (audit trail)                       │
└─────────────────────────────────────────────────────────┘
```

### Temporal Architecture

```
┌─────────────────────────────────────────────────────────┐
│                  Workflow Definition                     │
│  LoanApplicationWorkflowImpl (Kotlin/Java code)          │
│  • Pure code-based workflow logic                        │
│  • Deterministic execution                               │
└─────────────────────────────────────────────────────────┘
                          ↓
┌─────────────────────────────────────────────────────────┐
│                   Activity Interface                     │
│  Activity stubs with retry/timeout policies              │
│  • CreditCheckActivity                                   │
│  • DecisionActivity                                      │
│  • ContractGenerationActivity                            │
└─────────────────────────────────────────────────────────┘
                          ↓
┌─────────────────────────────────────────────────────────┐
│                  Activity Implementations                │
│  Business logic executed by workers                      │
│  • Stateless, idempotent operations                      │
│  • Can be scaled independently                           │
└─────────────────────────────────────────────────────────┘
                          ↓
┌─────────────────────────────────────────────────────────┐
│                   Temporal Server                        │
│  • Event history storage                                 │
│  • Workflow state reconstruction                         │
│  • Signal/query handling                                 │
└─────────────────────────────────────────────────────────┘
```

---



## Implementation Details

### Camunda 8 Implementation

#### BPMN Process Definition
```xml
<bpmn:process id="loan-process-v1" name="Loan Process V1">
  <bpmn:serviceTask id="credit_check" name="Credit Check">
    <zeebe:taskDefinition type="credit-check" retries="3"/>
  </bpmn:serviceTask>

  <bpmn:exclusiveGateway id="Gateway_Decision" name="Credit decision?">
    <bpmn:outgoing>Flow_Approved</bpmn:outgoing>
    <bpmn:outgoing>Flow_Rejected</bpmn:outgoing>
    <bpmn:outgoing>Flow_Manual_Review</bpmn:outgoing>
  </bpmn:exclusiveGateway>

  <bpmn:userTask id="manual_underwriting" name="Manual Underwriting"/>

  <bpmn:boundaryEvent id="Event_Timeout" name="7 day timeout">
    <bpmn:timerEventDefinition>
      <bpmn:timeDuration>P7D</bpmn:timeDuration>
    </bpmn:timerEventDefinition>
  </bpmn:boundaryEvent>
</bpmn:process>
```

#### Worker Implementation
```kotlin
@Component
class CreditCheckWorker(private val loanService: LoanService) {

    @JobWorker(type = "credit-check", timeout = 60000L, maxJobsActive = 10)
    fun performCreditCheck(
        @Variable loanApplicationId: Long,
        @Variable applicantId: String
    ): Map<String, Any> {
        val creditScore = simulateCreditScore(applicantId)
        val decision = when {
            creditScore >= 750 -> CreditDecision.Approved
            creditScore >= 600 && creditScore < 680 -> CreditDecision.ManualReview
            else -> CreditDecision.Rejected
        }

        return mapOf(
            "creditDecision" to decision.value,
            "creditScore" to creditScore
        )
    }
}
```

#### Key Features
- **Visual BPMN Designer**: Camunda Modeler for process design
- **Operate UI**: Real-time monitoring and incident management
- **Version Coexistence**: Multiple process versions running simultaneously
- **User Tasks**: Built-in support for manual interventions
- **Timers**: Native BPMN timer events for timeouts

### Temporal Implementation

#### Workflow Implementation
```kotlin
class LoanApplicationWorkflowImpl : LoanApplicationWorkflow {

    override fun processApplication(application: LoanApplication): LoanResult {
        var creditScore = 0
        var manualReviewAttempts = 0

        while (manualReviewAttempts < maxManualReviewAttempts) {
            // Credit check with automatic retries
            val creditResult = creditCheckActivity.checkCredit(application)
            creditScore = creditResult.score

            // Make decision
            val decision = decisionActivity.makeDecision(application, creditScore)

            when (decision) {
                Decision.APPROVED -> {
                    val contract = contractGenerationActivity.generateContract(application)
                    return LoanResult(/* ... */, contractId = contract.contractId)
                }

                Decision.REJECTED -> {
                    return LoanResult(/* rejected */)
                }

                Decision.MANUAL_REVIEW -> {
                    manualReviewAttempts++

                    // Wait for signal with 7-day timeout
                    try {
                        Workflow.await(Duration.ofDays(7)) { /* signal */ }
                        continue // Loop back to credit check
                    } catch (e: Exception) {
                        return LoanResult(/* timeout rejected */)
                    }
                }
            }
        }
    }
}
```

#### Activity Implementation
```kotlin
class CreditCheckActivityImpl : CreditCheckActivity {

    override fun checkCredit(application: LoanApplication): CreditCheckResult {
        // Simulate credit check with 30% failure rate for retry demo
        if (Random.nextDouble() < 0.3) {
            throw RuntimeException("Credit service unavailable")
        }

        val creditScore = calculateCreditScore(application)
        return CreditCheckResult(score = creditScore, reportId = UUID.randomUUID())
    }
}
```

#### Key Features
- **Code-Based Workflows**: Type-safe, refactorable workflow definitions
- **Event Sourcing**: Complete workflow history preserved
- **Versioning**: Workflow.getVersion() for in-flight migrations
- **Signals**: External communication mechanism for manual reviews
- **Timers**: Workflow.await() with timeout for time-based logic

---

## Pros and Cons

### Camunda 8 (Zeebe)

#### Pros ✅
1. **Visual Process Design**
   - BPMN 2.0 standard provides universal business-technical communication
   - Non-developers can understand and validate workflows
   - Camunda Modeler offers drag-and-drop process design

2. **Rich Ecosystem**
   - Operate UI for real-time process monitoring
   - Incident management and manual job retries
   - Optimize for process analytics and reporting
   - Tasklist for user task management

3. **Enterprise Features**
   - User tasks with forms and assignments
   - Timer events, message events, error handling
   - Process instance migration tools
   - Multi-tenancy support

4. **Standards-Based**
   - BPMN 2.0 industry standard
   - Portable process definitions
   - Extensive documentation and community

5. **Separation of Concerns**
   - Process logic separate from business logic
   - Workers can be written in any language (polyglot)
   - Easy to involve business analysts in process design

6. **Built-in Audit**
   - Complete process history in Operate
   - Out-of-the-box compliance reporting

#### Cons ❌
1. **XML Overhead**
   - BPMN files can be verbose
   - Requires separate tooling (Camunda Modeler)
   - Merge conflicts in XML can be challenging

2. **Infrastructure Complexity**
   - Requires Zeebe broker, Elasticsearch, Operate
   - More moving parts to manage in production
   - Higher resource footprint

3. **Learning Curve**
   - BPMN 2.0 specification learning required
   - Understanding Zeebe-specific extensions
   - Different debugging approach (Operate UI vs IDE)

4. **Coupling to BPMN**
   - Complex conditional logic can be awkward in BPMN
   - Sometimes requires workarounds for non-standard patterns

5. **Versioning Complexity**
   - Process version management requires careful planning
   - In-flight instance migration can be complex

### Temporal

#### Pros ✅
1. **Code-First Approach**
   - Workflows are just code (Kotlin, Java, Go, Python, etc.)
   - Use familiar IDE, debugging, and testing tools
   - Type safety and compile-time checks
   - Easy refactoring with IDE support

2. **Developer Experience**
   - No XML or external process definitions
   - Local development without external dependencies
   - Standard unit testing frameworks
   - Rich language features (loops, conditionals, error handling)

3. **Powerful Versioning**
   - Workflow.getVersion() for in-flight workflow evolution
   - Gradual rollout of new versions
   - No separate process migration tools needed

4. **Event Sourcing**
   - Complete deterministic workflow replay
   - Perfect audit trail out of the box
   - Time-travel debugging capabilities

5. **Scalability**
   - Highly scalable architecture
   - Efficient handling of long-running workflows
   - Activity retries with exponential backoff

6. **Flexibility**
   - Easy to implement complex business logic
   - Dynamic workflow behavior
   - Rich control flow (loops, recursion, etc.)

#### Cons ❌
1. **No Visual Designer**
   - No graphical process modeling tool
   - Harder for non-developers to understand workflows
   - Business analysts cannot directly design processes

2. **Less Accessible to Business**
   - Requires programming knowledge to understand workflows
   - No BPMN standard for documentation
   - Harder to get business sign-off on process design

3. **Limited UI Tooling**
   - Temporal UI is more technical
   - Less polished than Camunda Operate
   - No built-in user task management UI

4. **Learning Curve**
   - Unique concepts (determinism, workflow replay)
   - Must understand event sourcing model
   - Different mental model than traditional orchestration

5. **No Built-in User Tasks**
   - Manual interventions require signal handling
   - No out-of-the-box task assignment/routing
   - Need custom UI for human tasks

6. **Debugging Complexity**
   - Deterministic constraints can be tricky
   - Cannot use random values or current time directly
   - Requires understanding of Temporal's execution model

---

## When to Use Which

### Choose Camunda 8 When:

✅ **Business involvement is critical**
- Non-technical stakeholders need to understand and validate processes
- Process design is a collaborative activity with business analysts
- Compliance and audit require visual process documentation

✅ **You need rich UI tooling**
- Real-time process monitoring is essential
- Manual incident resolution is required
- User task management with forms and assignments

✅ **Standard BPMN processes**
- Workflows align well with BPMN patterns
- Industry standard compliance is required
- Process portability across platforms

✅ **Enterprise features matter**
- Multi-tenancy requirements
- Complex organizational structures
- Extensive reporting and analytics needs

✅ **Example Use Cases:**
- Loan origination (this POC!)
- Insurance claims processing
- Order fulfillment
- Employee onboarding
- Regulatory compliance workflows

### Choose Temporal When:

✅ **Developer-centric organization**
- Engineering team owns workflow logic
- Code reviews are preferred over visual reviews
- Strong development practices (CI/CD, testing)

✅ **Complex business logic**
- Dynamic, data-driven workflows
- Heavy conditional logic and loops
- Need for algorithmic decision-making

✅ **Microservices orchestration**
- Service-to-service coordination
- Saga pattern implementation
- Distributed transactions

✅ **Long-running workflows**
- Workflows spanning months or years
- Need perfect audit trail
- Time-travel debugging requirements

✅ **Rapid iteration needed**
- Fast-changing business requirements
- Need quick deployments
- Preference for code over configuration

✅ **Example Use Cases:**
- E-commerce order processing
- Payment processing workflows
- Data pipeline orchestration
- Background job scheduling
- Distributed saga orchestration

### Hybrid Scenarios

Consider a hybrid approach when:
- Different teams have different preferences (business-critical flows in Camunda, technical flows in Temporal)
- Migration strategy from one platform to another
- Using both for different domains within the same organization

---

## Key Differences

### 1. Process Definition

| Aspect | Camunda 8 | Temporal |
|--------|-----------|----------|
| **Format** | BPMN 2.0 XML | Code (Kotlin, Java, Go, etc.) |
| **Tooling** | Camunda Modeler | IDE (IntelliJ, VSCode) |
| **Visualization** | Built-in diagram | Must be generated separately |
| **Version Control** | XML diffs | Standard code diffs |
| **Refactoring** | Manual XML editing | IDE refactoring tools |

### 2. Execution Model

| Aspect | Camunda 8 | Temporal |
|--------|-----------|----------|
| **State Management** | Zeebe broker | Event history |
| **Replay** | Not applicable | Deterministic replay |
| **Debugging** | Operate UI | Standard debugging + replay |
| **Failure Handling** | Incidents in Operate | Activity retries |

### 3. Manual Reviews / User Tasks

| Aspect | Camunda 8 | Temporal |
|--------|-----------|----------|
| **Implementation** | BPMN User Task | Signal handling |
| **UI** | Built-in Tasklist | Custom implementation |
| **Assignment** | Native support | Custom logic |
| **Forms** | Camunda Forms | Custom UI |

### 4. Timeouts

| Aspect | Camunda 8 | Temporal |
|--------|-----------|----------|
| **Implementation** | BPMN Timer Boundary Event | `Workflow.await(Duration)` |
| **Configuration** | XML: `<timeDuration>P7D</timeDuration>` | Code: `Duration.ofDays(7)` |
| **Visibility** | Clear in BPMN diagram | Must read code |

### 5. Retries

| Aspect | Camunda 8 | Temporal |
|--------|-----------|----------|
| **Configuration** | `@JobWorker(retries="3")` | `RetryOptions.setMaximumAttempts(3)` |
| **Backoff** | Job metadata | `RetryOptions.setBackoffCoefficient()` |
| **Manual Retry** | Operate UI button | Re-execute workflow |

### 6. Versioning

| Aspect | Camunda 8 | Temporal |
|--------|-----------|----------|
| **Strategy** | Multiple BPMN versions | `Workflow.getVersion()` |
| **Migration** | Process instance migration | Automatic replay with version |
| **Rollout** | Start new instances on new version | Code-based routing |
| **In-flight** | Must migrate or complete old version | Continues on old code path |

### 7. Monitoring

| Aspect | Camunda 8 | Temporal |
|--------|-----------|----------|
| **UI** | Camunda Operate (rich) | Temporal UI (technical) |
| **Process View** | Visual BPMN diagram | Event history |
| **Incidents** | Dedicated incident management | Workflow failures |
| **Metrics** | Built-in dashboards | Prometheus/custom |

### 8. Development Experience

| Aspect | Camunda 8 | Temporal |
|--------|-----------|----------|
| **Local Dev** | Docker Compose (Zeebe + deps) | Temporal server (lightweight) |
| **Testing** | `zeebe-process-test-extension` | Standard unit tests |
| **Debugging** | Logs + Operate UI | IDE debugger + replay |
| **Hot Reload** | Deploy new BPMN | Standard code hot reload |

---

## Comparison Summary

### Quick Decision Matrix

| Criteria | Camunda 8 | Temporal |
|----------|:---------:|:--------:|
| **Visual process design** |   ⭐⭐⭐⭐⭐   | ⭐ |
| **Business stakeholder accessibility** |   ⭐⭐⭐⭐⭐   | ⭐⭐ |
| **Developer experience** |    ⭐⭐⭐    | ⭐⭐⭐⭐⭐ |
| **Complex business logic** |    ⭐⭐⭐    | ⭐⭐⭐⭐⭐ |
| **UI tooling** |   ⭐⭐⭐⭐⭐   | ⭐⭐⭐ |
| **Learning curve** |    ⭐⭐     | ⭐⭐⭐ |
| **Debugging** |   ⭐⭐⭐⭐    | ⭐⭐⭐⭐⭐ |
| **Versioning flexibility** |    ⭐⭐⭐    | ⭐⭐⭐⭐⭐ |
| **Audit trail** |   ⭐⭐⭐⭐⭐   | ⭐⭐⭐⭐⭐ |
| **Infrastructure complexity** |    ⭐⭐⭐    | ⭐⭐⭐⭐ |
| **Enterprise features** |   ⭐⭐⭐⭐⭐   | ⭐⭐⭐ |
| **Rapid iteration** |    ⭐⭐⭐    | ⭐⭐⭐⭐⭐ |

### Final Recommendation

**Choose Camunda 8** if:
- Business process management is a core capability
- Visual process documentation is required
- Non-technical stakeholders are deeply involved
- You need comprehensive UI tooling out-of-the-box

**Choose Temporal** if:
- You're a developer-centric organization
- Workflows contain complex, dynamic logic
- You prefer code-based approaches
- Rapid iteration and strong type safety matter
