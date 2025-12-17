# Agent Instructions Directory

This directory contains agent definition files for the Hyperswitch multi-agent system.

## Available Agents

### 1. Product Owner / Business Analyst Agent
**File**: `product-owner-business_analyst.agent.md`

**Responsibilities**:
- Feature backlog management
- BDD scenario creation (Gherkin)
- Acceptance criteria definition
- User story mapping
- NFR specification

**Key Outputs**:
- `docs/features/*.feature` - BDD scenarios
- `docs/backlog.md` - Prioritized backlog
- `docs/requirements/*.md` - Requirements docs

### 2. Solution Architect Agent
**File**: `solution-architect.agent.md`

**Responsibilities**:
- Architecture design and documentation
- PlantUML diagram generation
- Pattern detection and recommendation
- Technology evaluation
- Dependency analysis

**Key Outputs**:
- `plantuml/*.puml` - Architecture diagrams (C4, sequence, ERD, state)
- `docs/architecture/*.md` - Architecture documentation
- ADRs (Architecture Decision Records)

### 3. Software Engineer Agent
**File**: `software-engineer.agent.md`

**Responsibilities**:
- TDD implementation
- Code refactoring
- Unit & integration testing
- Code review
- SOLID principles adherence

**Key Outputs**:
- `crates/**/*.rs` - Rust source code
- `tests/**/*.rs` - Test files
- `migrations/**/*.sql` - Database migrations

### 4. Automation Tester Agent
**File**: `automation-tester.agent.md`

**Responsibilities**:
- Integration testing
- End-to-end testing (Cypress)
- API testing (Postman)
- Load testing (k6)
- CI/CD test integration

**Key Outputs**:
- `tests/integration/**/*.rs` - Integration tests
- `cypress-tests/**/*.ts` - E2E tests
- `loadtest/**/*.js` - Performance tests
- `postman/**/*.json` - API test collections

## Agent Communication Protocol

Agents communicate via structured artifacts:

```
Product Owner → Solution Architect
  Input: Feature requirements, BDD scenarios
  Output: Technical design, architecture diagrams

Solution Architect → Software Engineer
  Input: Technical design, component specs
  Output: Implementation questions, code

Software Engineer → Automation Tester
  Input: Implemented features, APIs
  Output: Test results, bugs found

Automation Tester → Product Owner
  Input: Test results
  Output: Feature validation, acceptance
```

## Using Agents

### Invoke a Specific Agent
```bash
# In GitHub Copilot Chat
@workspace /agent product-owner "Create BDD scenarios for Apple Pay support"
@workspace /agent solution-architect "Design architecture for payment retry logic"
@workspace /agent software-engineer "Implement Adyen connector with TDD"
@workspace /agent automation-tester "Create integration tests for refund flow"
```

### Multi-Agent Workflow
```bash
# Full feature development workflow
@workspace /workflow new-feature "Support Google Pay for Checkout.com"

# This will orchestrate:
# 1. Product Owner: Create requirements & BDD scenarios
# 2. Solution Architect: Design components & API
# 3. Software Engineer: Implement with TDD
# 4. Automation Tester: Create integration & E2E tests
```

## Agent Metadata

Each agent file contains YAML frontmatter:

```yaml
---
role: "Agent Name"
authority: "Decision-making scope"
domain: "Domain expertise"
language_focus: ["Primary languages"]
capabilities:
  - capability1
  - capability2
allowed_tools: ['tool1', 'tool2']
outputs:
  - "path/to/output"
example_prompt: "Example usage"
---
```

## Quality Gates

Agents enforce quality gates at each stage:

- **Product Owner**: Requirements complete, acceptance criteria clear
- **Solution Architect**: Design reviewed, patterns validated
- **Software Engineer**: Tests pass, coverage ≥80%, clippy clean
- **Automation Tester**: Integration tests pass, load tests meet NFRs

## Customization

To create a new agent:

1. Copy an existing agent file
2. Update the YAML frontmatter
3. Define responsibilities and outputs
4. Provide example workflows
5. Add to this README

## See Also

- `..instructions/` - Orchestration topology patterns
- `../prompts/` - Technology-specific prompts
- `../copilot-instructions.md` - Main orchestrator configuration
