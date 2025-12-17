---
title: "Hyperswitch Copilot Multi-Agent Orchestrator Instructions"
version: "1.0.0"
model: "HIVE"
topology: "Hierarchical"
primary_language: "Rust"
secondary_languages: ["TypeScript", "JavaScript"]
entry_point_analysis: 
  - "Cargo.toml"
  - "package.json"
  - "docker-compose.yml"
  - "Dockerfile"
  - "justfile"
  - "Makefile"
artifacts:
  - "docs/repo_summary.md"
  - "docs/architecture/"
  - "plantuml/"
  - "tests/"
  - "cypress-tests/"
  - "cypress-tests-v2/"
allowed_tools: ['runCommands', 'search', 'edit', 'runTasks', 'testFailure', 'fetch', 'serena/*']
policies:
  - security_scan: 'run cargo-audit / snyk test / cargo-deny'
  - diagrams: 'plantuml for architecture visualization'
  - test: 'TDD with unit & integration tests; BDD scenarios for features'
  - code_quality: 'cargo clippy, cargo fmt, nextest runner'
  - performance: 'k6 load tests, benchmark tests'
---

# Hyperswitch Multi-Agent System Orchestrator

## High-Level Flow

### 1. Repository Discovery & Indexing
- Scan workspace structure and build project index
- Extract runtime dependencies from `Cargo.toml` (Rust workspace with 40+ crates)
- Identify build systems: Cargo, Make, Just, Docker, Nix
- Map API endpoints from router crate
- Index connector implementations (100+ payment processors)

### 2. Static Analysis & Architecture Inference
- Analyze crate dependencies and module boundaries
- Identify domain models, API layers, storage implementations
- Detect patterns: Repository, Factory, Strategy, Adapter
- Extract trait hierarchies and implementations
- Map database models via Diesel ORM

### 3. Generate PlantUML Diagrams
- **C4 Context**: External systems (payment processors, databases, Redis, Kafka)
- **C4 Container**: Main components (Router, Scheduler, Drainer, Analytics)
- **C4 Component**: Internal crate architecture
- **Sequence Diagrams**: Payment flow, authentication, routing logic
- **ERD**: Database schema from Diesel models
- **State Machines**: Payment status, order lifecycle

### 4. Feature Backlog Generation
- Analyze open GitHub issues and TODOs in code
- Extract feature requests from connector-template/
- Review CHANGELOG.md for upcoming features
- Prioritize based on labels and milestones

### 5. Feature Planning (BDD → Tasks)
- Convert feature requests to Gherkin scenarios
- Define acceptance criteria with NFRs (latency, throughput)
- Break down into tasks: API, service, repository, tests, migrations
- Estimate complexity and dependencies

### 6. Implementation (TDD)
- Write failing tests first (unit, integration, contract)
- Implement minimal code to pass tests
- Run `cargo test`, `cargo nextest run`
- Ensure compliance with SOLID, KISS principles
- Create PR with test coverage reports

### 7. Automated Code Review & Security
- Run `cargo clippy` for linting
- Run `cargo fmt --check` for formatting
- Static analysis with `cargo-audit`, `cargo-deny`
- Security scan with Snyk (if available)
- Check test coverage with `cargo tarpaulin` or `cargo-llvm-cov`

### 8. Integration & Performance Testing
- Run Cypress E2E tests (`cypress-tests/`, `cypress-tests-v2/`)
- Execute integration tests with Testcontainers
- Load testing with k6 scripts (`loadtest/`)
- API testing with Postman collections (`postman/`)

### 9. Deployment & Monitoring
- Build Docker images
- Run database migrations (Diesel)
- Deploy to environment (local/staging/prod)
- Monitor with Grafana, Prometheus, Loki
- Verify health checks and readiness probes

### 10. Report, Iterate, Deploy
- Generate summary report with metrics
- Update documentation
- Create release notes
- Tag version and publish

---

## Project-Specific Context

### Technology Stack
- **Primary**: Rust (Edition 2021, ≥1.85.0)
- **Web Framework**: Actix-web 4.11.0
- **Database**: PostgreSQL with Diesel ORM 2.2.10
- **Cache**: Redis with custom interface
- **Async Runtime**: Tokio (multi-threaded)
- **Testing**: cargo-nextest, Testcontainers, Cypress
- **Monitoring**: Grafana, Prometheus, Loki, Tempo

### Key Crates
- `router`: Main payment routing server
- `hyperswitch_connectors`: Payment processor adapters (100+ connectors)
- `hyperswitch_interfaces`: Trait definitions
- `hyperswitch_domain_models`: Core business models
- `api_models`: Request/response DTOs
- `diesel_models`: Database ORM models
- `storage_impl`: Storage backends
- `scheduler`: Task scheduling
- `analytics`: Payment analytics
- `euclid`: Business logic DSL (compiles to WASM)

### Development Commands
```bash
# Format code
cargo +nightly fmt

# Lint
cargo clippy --all-features --all-targets

# Run tests
cargo nextest run
cargo test --workspace

# Build
cargo build --release

# Run locally
docker-compose up -d
cargo run -- -f ./config/docker_compose.toml

# Database migrations
diesel migration run --config-file diesel.toml
```

### Feature Flags
- `v1` / `v2`: API versions (mutually exclusive)
- `release`: Production build
- `oltp` / `olap`: Database optimization modes
- `frm`: Fraud Risk Management
- `payouts`: Payout functionality
- `recon`: Reconciliation
- `dynamic_routing`: Advanced routing

### Quality Gates
- ✅ Test coverage: ≥80% for core modules
- ✅ Cyclomatic complexity: <15 per function
- ✅ No `unsafe` code without justification
- ✅ All public APIs documented
- ✅ All clippy warnings resolved
- ✅ Security audit passed (cargo-audit, cargo-deny)
- ✅ Performance benchmarks within thresholds

---

## Agent Coordination Model

### Primary Agents
1. **Product Owner / Business Analyst** - Feature intake, requirements, BDD scenarios
2. **Solution Architect** - Architecture design, PlantUML diagrams, pattern analysis
3. **Software Engineer** - TDD implementation, code quality, refactoring
4. **UI/UX Designer** - Frontend design, user flows (for web client SDK)
5. **Manual Tester** - Exploratory testing, test case design
6. **Automation Tester** - Integration tests, E2E tests, load tests
7. **Security Analyst** - Security scanning, vulnerability remediation
8. **DevOps Engineer** - CI/CD, deployment, monitoring

### Communication Protocol
- Agents share context via structured JSON artifacts
- Each agent produces documented outputs in standardized formats
- Handoffs between agents include: status, blockers, next steps
- All agents reference PlantUML diagrams and architecture docs

### Decision Authority
- **Architect**: Technology choices, design patterns, module boundaries
- **Product Owner**: Feature priorities, acceptance criteria, NFRs
- **Engineer**: Implementation details, refactoring, testing strategies
- **Security**: Vulnerability severity, remediation approaches
- **DevOps**: Deployment strategies, infrastructure changes

---

## Output Artifacts (Required)

### Documentation
- `docs/repo_summary.md` - Human-readable project summary
- `docs/architecture/overview.md` - Architecture narrative
- `docs/architecture/patterns.md` - Design patterns catalog
- `docs/architecture/connectors.md` - Connector integration guide
- `docs/architecture/deployment.md` - Deployment guide

### Diagrams (PlantUML)
- `plantuml/c4_context.puml` - System context
- `plantuml/c4_container.puml` - Container architecture
- `plantuml/c4_component.puml` - Component structure
- `plantuml/sequence_payment_flow.puml` - Payment processing
- `plantuml/sequence_auth_flow.puml` - Authentication flow
- `plantuml/erd.puml` - Database schema
- `plantuml/state_payment.puml` - Payment state machine

### Tests
- `tests/unit/**/*.rs` - Unit tests
- `tests/integration/**/*.rs` - Integration tests
- `cypress-tests/` - E2E tests (legacy)
- `cypress-tests-v2/` - E2E tests (v2)
- `loadtest/` - Performance tests (k6)
- `postman/` - API tests

### Reports
- `reports/coverage/` - Test coverage
- `reports/security/` - Security scan results
- `reports/performance/` - Load test results
- `reports/quality/` - Code quality metrics

### CI/CD
- `.github/workflows/` - GitHub Actions pipelines
- Quality gates enforced at PR level
- Automated testing, linting, security scanning

---

## Usage Guidelines

### For New Features
1. Agent: **Product Owner** → Create BDD feature file with scenarios
2. Agent: **Architect** → Design component interactions, update diagrams
3. Agent: **Engineer** → Implement with TDD, create PR
4. Agent: **Automation Tester** → Add integration & E2E tests
5. Agent: **Security Analyst** → Run security scans
6. Agent: **DevOps** → Deploy to staging, run smoke tests

### For Bug Fixes
1. Agent: **Manual Tester** → Reproduce issue, document steps
2. Agent: **Engineer** → Write failing test, fix bug
3. Agent: **Automation Tester** → Add regression test
4. Agent: **DevOps** → Deploy hotfix

### For Refactoring
1. Agent: **Architect** → Identify patterns, propose changes
2. Agent: **Engineer** → Refactor with existing tests as safety net
3. Agent: **Automation Tester** → Verify no regressions
4. Agent: **Product Owner** → Approve changes

---

## Best Practices

### Code Organization
- Follow existing crate structure
- Keep modules focused and cohesive
- Use trait abstractions for extensibility
- Document public APIs with rustdoc

### Testing Strategy
- Unit tests: Business logic, pure functions
- Integration tests: API endpoints, database operations
- E2E tests: Full user flows
- Contract tests: Connector implementations
- Performance tests: Critical paths (payment processing, routing)

### Security
- No hardcoded secrets (use environment variables)
- Validate all inputs
- Sanitize outputs
- Use masking crate for PII
- Run security audits regularly

### Performance
- Profile critical paths
- Use async/await efficiently
- Implement caching strategically
- Monitor database query performance
- Set appropriate timeouts

### Documentation
- Keep README.md updated
- Document architecture decisions (ADRs)
- Maintain API documentation
- Update PlantUML diagrams with changes
- Write clear commit messages (conventional commits)

---

## Integration Points

### External Services
- Payment processors (Stripe, PayPal, Adyen, etc.)
- AWS services (KMS, S3)
- Email providers
- Kafka/Redis for events
- Monitoring (Grafana, Prometheus)

### API Versions
- V1: Current stable API
- V2: Next-gen API (in development)
- Compatibility layer for migration

### Database
- PostgreSQL primary database
- Redis for caching and session management
- Diesel ORM with migrations
- OLTP vs OLAP optimization modes

---

## Continuous Improvement

### Metrics Tracking
- Test coverage trends
- Security vulnerability count
- Performance benchmark results
- Code quality scores
- Deployment frequency
- Mean time to recovery (MTTR)

### Learning Loop
- Capture patterns from implementations
- Document anti-patterns and lessons learned
- Update templates and guidelines
- Share knowledge across agents
- Improve automation based on repetitive tasks

---

## License & Compliance
- **License**: Apache 2.0
- **Compliance**: PCI-DSS for payment data
- **Security**: Regular audits, vulnerability management
- **Privacy**: GDPR, PII masking and encryption

---

**Generated**: 2025-12-17  
**Maintained by**: Multi-Agent System Orchestrator  
**Version**: 1.0.0
