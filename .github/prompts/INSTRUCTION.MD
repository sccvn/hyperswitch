# Technology-Specific Prompts Directory

This directory contains language and framework-specific prompt files that guide agents in writing idiomatic, high-quality code for Hyperswitch.

## Available Prompts

### 1. Rust Core Development (`rust_core.prompt.md`)

**Use for**: Core Hyperswitch backend development

**Covers**:
- Rust 2021 edition best practices
- Actix-web API patterns
- Diesel ORM usage
- Async/await with Tokio
- Error handling with error-stack
- Type safety & newtypes
- Connector integration patterns
- Testing strategies (unit, integration, TDD)
- Performance optimization
- Security practices (PII masking, constant-time comparisons)

**When to use**:
- Implementing new connectors
- Adding API endpoints
- Database schema changes
- Payment processing logic
- Business domain models

### 2. TypeScript/Node.js (Coming Soon)

**Use for**: Web SDK, client-side integrations

**Will cover**:
- TypeScript best practices
- React/Vue component patterns
- SDK development
- Browser compatibility
- Webpack/Vite configuration

### 3. SQL/Diesel Migrations (Coming Soon)

**Use for**: Database schema evolution

**Will cover**:
- Diesel migration patterns
- Schema design principles
- Index optimization
- Data migration strategies

## Using Technology Prompts

### 1. Reference in Agent Context
```bash
@workspace /agent software-engineer --context rust_core "Implement Adyen connector"
```

### 2. Include in Code Generation
```bash
@workspace "Generate a new Rust connector for PayPal following rust_core.prompt.md patterns"
```

### 3. Code Review with Context
```bash
@workspace /review --style rust_core "Review this connector implementation"
```

## Prompt Structure

Each prompt file follows this structure:

```markdown
---
language: "Language name"
framework: "Primary framework"
use_case: "When to use"
---

# Title

## Language Context
[Overview of language and project-specific context]

## Core Principles
[Key principles with code examples]

## [Project]-Specific Patterns
[Patterns specific to Hyperswitch]

## Code Quality Standards
[Linting, formatting, testing standards]

## Performance Considerations
[Optimization techniques]

## Security Practices
[Security best practices]
```

## Creating New Prompts

### Template
```markdown
---
language: "YourLanguage"
framework: "YourFramework"
min_version: "X.Y.Z"
use_case: "Description"
---

# [Language] [Framework] Development Prompt

## Language Context
[Describe the language, ecosystem, and project usage]

## Core Principles
### 1. Principle Name
```code
// Example
```

### 2. Another Principle
```code
// Example
```

## [Project]-Specific Patterns
[Document project-specific patterns]

## Code Quality Standards
[Define quality standards]

## Testing Strategies
[Testing approaches]

## See Also
- Related prompts
- Documentation links
```

### Steps to Add a New Prompt
1. Create `[language]_[framework].prompt.md`
2. Add YAML frontmatter
3. Document core principles with examples
4. Include project-specific patterns
5. Define quality standards
6. Add to this README

## Integration with Agents

### Software Engineer Agent
Automatically uses prompts based on:
- File extension (`.rs` → rust_core.prompt.md)
- Crate location (`crates/router/` → rust_core.prompt.md)
- Explicit context flag

### Solution Architect Agent
Uses prompts to:
- Validate design patterns
- Recommend language-appropriate solutions
- Generate idiomatic code examples

### Automation Tester Agent
Uses prompts to:
- Generate test fixtures
- Write idiomatic test code
- Follow testing conventions

## Examples

### Rust Development
```bash
# Generate new connector
@workspace /agent software-engineer "Create new connector for Stripe Connect following rust_core patterns"

# Refactor existing code
@workspace /refactor --style rust_core crates/router/src/core/payments.rs "Apply SOLID principles"

# Code review
@workspace /review --style rust_core "Check this PR for Rust best practices"
```

### TypeScript Development (when available)
```bash
# Generate SDK component
@workspace /agent software-engineer "Create payment widget following typescript_sdk patterns"
```

## Quality Metrics

Each prompt defines quality standards:
- **Rust**: cargo clippy (0 warnings), cargo fmt, 80% coverage
- **TypeScript**: ESLint, Prettier, Jest coverage
- **SQL**: Migration validation, index analysis

## Maintenance

### Updating Prompts
- Review quarterly for new language features
- Update based on team feedback
- Incorporate lessons learned from code reviews
- Add new patterns as they emerge

### Versioning
- Prompts are versioned with the project
- Breaking changes documented in CHANGELOG
- Backward compatibility maintained when possible

## See Also

- `../agents/` - Agent definitions that use these prompts
- `../instructions/` - Orchestration patterns
- `../copilot-instructions.md` - Main configuration
- `../../docs/architecture/` - Architecture documentation

---

**Maintained by**: Multi-Agent System Orchestrator  
**Version**: 1.0.0  
**Last Updated**: 2025-12-17
