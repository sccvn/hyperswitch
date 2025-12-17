---
topology: "Hierarchical (Hive)"
model: "Queen-led with specialized workers"
use_case: "Structured feature development with clear authority"
coordination: "Top-down with feedback loops"
---

# Hierarchical (Hive) Orchestration Pattern

## Overview

The Hive pattern follows a hierarchical structure where a **Queen agent** (Product Owner/Architect) coordinates **Worker agents** (Engineers, Testers) through clear command chains.

```
                    Queen (Product Owner)
                           |
          +----------------+----------------+
          |                                 |
    Architect Agent                   DevOps Agent
          |                                 |
     +----+----+                       +----+----+
     |         |                       |         |
 Engineer   Engineer               Tester     Tester
```

## When to Use

✅ **Use Hierarchical/Hive when:**
- Feature requirements are well-defined upfront
- Clear authority structure is needed
- Compliance requires approval gates
- Team prefers structured workflows
- Complex features need coordination

❌ **Avoid when:**
- Requirements are highly exploratory
- Rapid iteration is priority
- Team prefers autonomous work
- Small, simple tasks

## Roles & Authority

### Queen Layer (Strategic)
**Product Owner Agent**
- Authority: Final say on requirements, priorities, and acceptance
- Responsibilities: Define features, create BDD scenarios, prioritize backlog
- Outputs: User stories, acceptance criteria, NFRs

### Lieutenant Layer (Tactical)
**Solution Architect Agent**
- Authority: Technical design decisions, pattern selection
- Responsibilities: Design architecture, create diagrams, evaluate tech
- Outputs: PlantUML diagrams, ADRs, component specs

**DevOps Agent**
- Authority: Infrastructure and deployment decisions
- Responsibilities: CI/CD, monitoring, deployment strategies
- Outputs: Pipeline configs, infrastructure as code

### Worker Layer (Execution)
**Software Engineer Agent**
- Authority: Implementation details, refactoring within design
- Responsibilities: TDD implementation, code quality
- Outputs: Source code, unit tests, migrations
- Reports to: Solution Architect

**Automation Tester Agent**
- Authority: Test strategies, test data management
- Responsibilities: Integration tests, E2E tests, load tests
- Outputs: Test suites, test reports
- Reports to: Product Owner (for acceptance), DevOps (for CI/CD)

## Communication Protocol

### Downward (Command)
```yaml
Product Owner → Architect:
  - Feature requirements
  - BDD scenarios
  - Acceptance criteria
  - Business constraints

Architect → Engineers:
  - Technical design
  - Component specifications
  - Patterns to follow
  - API contracts

DevOps → Testers:
  - Environment configurations
  - CI/CD requirements
  - Deployment schedules
```

### Upward (Feedback)
```yaml
Engineers → Architect:
  - Implementation challenges
  - Design clarifications
  - Refactoring proposals
  - Technical debt reports

Testers → DevOps:
  - Environment issues
  - CI/CD failures
  - Performance bottlenecks

Architect → Product Owner:
  - Technical feasibility
  - Effort estimates
  - Risk assessments
  - Alternative approaches
```

## Workflow Example: New Payment Method

### Phase 1: Requirements (Queen Layer)
```gherkin
# Product Owner creates feature

Feature: Support Google Pay for Checkout.com
  As a merchant
  I want to accept Google Pay via Checkout.com
  So that I can offer more payment options

  Scenario: Successful Google Pay payment
    Given a merchant with Checkout.com configured
    And the customer has Google Pay enabled
    When the customer selects Google Pay
    And authorizes the payment
    Then the payment is processed via Checkout.com
    And the response status is "succeeded"
```

**Handoff**: Product Owner → Solution Architect  
**Artifact**: `docs/features/google-pay-checkoutcom.feature`

### Phase 2: Design (Lieutenant Layer)
```
# Solution Architect creates design

1. Component Design:
   - New GooglePayWalletData struct
   - Extend CheckoutComConnector trait
   - Add GooglePay transformer

2. API Changes:
   - Add google_pay to PaymentMethodData enum
   - Extend /payments request schema

3. Sequence Diagram: google-pay-flow.puml

4. Estimated Effort: 8 story points
```

**Handoff**: Architect → Software Engineer  
**Artifact**: `docs/architecture/google-pay-design.md`, `plantuml/google-pay-flow.puml`

### Phase 3: Implementation (Worker Layer)
```rust
// Software Engineer implements with TDD

// Step 1: Write failing test
#[tokio::test]
async fn test_checkoutcom_google_pay_authorize() {
    // Test implementation
}

// Step 2: Implement minimal code
impl ConnectorIntegration<Authorize> for CheckoutCom {
    // Implementation
}

// Step 3: Refactor
// Step 4: Submit PR
```

**Handoff**: Engineer → Automation Tester  
**Artifact**: PR with code + unit tests

### Phase 4: Testing (Worker Layer)
```typescript
// Automation Tester creates E2E tests

describe('Google Pay - Checkout.com', () => {
  it('should process Google Pay payment', () => {
    // Cypress test implementation
  });
});
```

**Handoff**: Tester → Product Owner  
**Artifact**: Test results, coverage report

### Phase 5: Acceptance (Queen Layer)
```markdown
# Product Owner validates

✅ Acceptance Criteria Met:
- Google Pay payments process via Checkout.com
- Success rate: 99.5% (meets >99% NFR)
- Latency p95: 180ms (meets <200ms NFR)
- Test coverage: 92%

**Decision: APPROVED for deployment**
```

## Decision-Making Framework

| Decision Type | Authority | Escalation Path |
|--------------|-----------|-----------------|
| Feature Priority | Product Owner | Stakeholders |
| Architecture | Solution Architect | Tech Lead |
| Implementation Details | Software Engineer | Architect |
| Test Strategy | Automation Tester | DevOps |
| Deployment | DevOps | Product Owner |

## Quality Gates

### Gate 1: Requirements Review
- **Owner**: Product Owner
- **Checklist**:
  - [ ] BDD scenarios complete
  - [ ] Acceptance criteria clear
  - [ ] NFRs specified
  - [ ] Dependencies identified

### Gate 2: Design Review
- **Owner**: Solution Architect
- **Checklist**:
  - [ ] Components defined
  - [ ] Patterns validated
  - [ ] API contracts documented
  - [ ] PlantUML diagrams created

### Gate 3: Code Review
- **Owner**: Software Engineer + Architect
- **Checklist**:
  - [ ] Tests pass (unit + integration)
  - [ ] Coverage ≥80%
  - [ ] Clippy warnings: 0
  - [ ] Follows design patterns

### Gate 4: Testing Review
- **Owner**: Automation Tester
- **Checklist**:
  - [ ] Integration tests pass
  - [ ] E2E tests pass
  - [ ] Load tests meet NFRs
  - [ ] Security scan clean

### Gate 5: Acceptance Review
- **Owner**: Product Owner
- **Checklist**:
  - [ ] All acceptance criteria met
  - [ ] NFRs validated
  - [ ] User documentation complete
  - [ ] Approved for deployment

## Benefits of Hierarchical Pattern

✅ **Clear accountability**: Each layer owns specific decisions  
✅ **Quality gates**: Structured reviews at each phase  
✅ **Compliance-friendly**: Audit trail of approvals  
✅ **Predictable**: Well-defined handoffs and timelines  
✅ **Scalable**: Can add more workers without chaos

## Challenges

⚠️ **Bottlenecks**: Queen/Lieutenant can become blockers  
⚠️ **Slower**: Multiple approval layers add latency  
⚠️ **Less flexible**: Harder to adapt to changing requirements  
⚠️ **Communication overhead**: Formal handoffs take time

## Metrics

- **Cycle time**: Time from requirements → deployment
- **Handoff time**: Time between layers
- **Rework rate**: % of features returned for changes
- **Gate pass rate**: % passing each gate on first attempt

---

**Status**: Active  
**Recommended for**: Large features, compliance-critical work  
**Version**: 1.0.0
