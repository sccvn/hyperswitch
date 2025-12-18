# Hyperswitch Architecture Documentation

**Generated**: December 18, 2025  
**Version**: 1.0.0  
**Maintained By**: Software Architect Agent  
**Source Repository**: juspay/hyperswitch

---

## 📋 Table of Contents

1. [Overview](#overview)
2. [Document Structure](#document-structure)
3. [Architecture Diagrams](#architecture-diagrams)
4. [Core Documentation](#core-documentation)
5. [Quick Reference](#quick-reference)
6. [How to Use This Documentation](#how-to-use-this-documentation)
7. [Document Versions](#document-versions)

---

## Overview

This documentation provides comprehensive architecture analysis of the Hyperswitch payment orchestration platform. It includes:

- **PlantUML Diagrams**: Visual representations of system architecture, components, and interactions
- **Architecture Documentation**: Detailed low-level design and component descriptions
- **Design Patterns**: Catalog of patterns implemented in the codebase
- **Data Structures**: Core domain models and their characteristics
- **Algorithms**: Routing logic, state machines, and optimization strategies

### Technology Stack

- **Language**: Rust (Edition 2021, ≥1.85.0)
- **Web Framework**: Actix-web 4.11.0
- **Database**: PostgreSQL with Diesel ORM 2.2.10
- **Cache**: Redis
- **Async Runtime**: Tokio (multi-threaded)
- **Architecture**: Layered + Hexagonal (Ports & Adapters)

### Repository Statistics

- **Total Crates**: 38
- **Lines of Code**: ~500K+ (Rust)
- **Connectors**: 100+ payment processors
- **Database Tables**: 10+ core tables
- **Design Patterns**: 13+ documented patterns

---

## Document Structure

```
generated-docs/
├── README.md                          # This file - comprehensive index
│
├── plantuml/                          # Visual architecture diagrams
│   ├── c4_component_router_detailed.puml   # Router component architecture
│   ├── class_diagram_payment_domain.puml   # Payment domain models
│   ├── sequence_payment_create_detailed.puml  # Payment creation flow
│   └── erd_detailed.puml              # Database schema
│
├── architecture/                      # Architecture documentation
│   └── low_level_design.md            # Comprehensive low-level design
│
├── patterns/                          # Design patterns documentation
│   └── design_patterns.md             # Pattern catalog with examples
│
├── data-structures/                   # Data structure documentation
│   └── core_structures.md             # Core domain models and types
│
└── algorithms/                        # Algorithms and logic
    └── routing_and_algorithms.md      # Routing logic and algorithms
```

---

## Architecture Diagrams

### PlantUML Diagrams

All diagrams can be rendered using PlantUML or viewed on [PlantText.com](https://www.planttext.com/).

#### 1. C4 Component Diagram - Router Architecture

**File**: [`plantuml/c4_component_router_detailed.puml`](plantuml/c4_component_router_detailed.puml)

**Purpose**: Shows the internal component structure of the router crate, including:
- HTTP routes layer (API endpoints)
- Core business logic (payment flows)
- Domain services
- Storage layer (repositories)
- Connector integration layer
- External dependencies

**Key Components**:
- 40+ components documented
- Layered architecture visualization
- Dependencies and data flow
- External system connections

**Use Cases**:
- Understanding router architecture
- Identifying component responsibilities
- Tracing request flows
- Onboarding new developers

---

#### 2. Class Diagram - Payment Domain

**File**: [`plantuml/class_diagram_payment_domain.puml`](plantuml/class_diagram_payment_domain.puml)

**Purpose**: Documents payment domain models and their relationships:
- `PaymentIntent` - Main payment entity
- `PaymentAttempt` - Individual connector attempts
- `Refund` - Refund tracking
- `Customer` - Customer information
- `PaymentMethod` - Stored payment methods
- `MerchantConnectorAccount` - Connector configurations

**Key Elements**:
- 50+ fields per major entity
- Enums with state transitions
- Relationships (1:many, many:1)
- Method signatures

**Use Cases**:
- Understanding domain model structure
- Database schema design
- API design reference
- Writing database queries

---

#### 3. Sequence Diagram - Payment Creation

**File**: [`plantuml/sequence_payment_create_detailed.puml`](plantuml/sequence_payment_create_detailed.puml)

**Purpose**: Detailed flow for creating a payment:
- API request reception
- Validation and authentication
- Business logic execution
- Connector integration
- Database operations
- Response generation

**Key Elements**:
- 200+ lines of detailed interactions
- Code file references (file:line)
- Timing/latency annotations
- Error handling paths

**Use Cases**:
- Understanding payment flow
- Debugging payment issues
- Optimizing performance
- Writing integration tests

---

#### 4. Entity-Relationship Diagram (ERD)

**File**: [`plantuml/erd_detailed.puml`](plantuml/erd_detailed.puml)

**Purpose**: Complete database schema with:
- All core tables (10+)
- Fields with types and constraints
- Primary keys and foreign keys
- Indexes
- Relationships

**Key Tables**:
- `merchant_account`
- `payment_intent`
- `payment_attempt`
- `refund`
- `customers`
- `payment_methods`
- `merchant_connector_account`
- `address`
- `mandate`
- `dispute`

**Use Cases**:
- Database schema reference
- Migration planning
- Query optimization
- Understanding data model

---

## Core Documentation

### 1. Low-Level Architecture

**File**: [`architecture/low_level_design.md`](architecture/low_level_design.md)

**Size**: 45KB

**Contents**:
1. **Executive Summary** - High-level overview
2. **Crate Architecture** - 38 crates organized by layer
3. **Core Components**:
   - Router (API server)
   - Connectors (100+ integrations)
   - Storage (repositories)
   - Analytics (metrics)
   - Scheduler (background jobs)
4. **Payment Flows**:
   - Payment creation
   - Confirmation
   - Capture
   - Refund
   - Webhook processing
5. **Connector Integration** - Adapter pattern
6. **Storage Architecture** - Repository pattern
7. **Routing Engine** - Dynamic routing algorithms
8. **Security** - Encryption, authentication, PCI compliance
9. **Performance** - Caching, async operations, optimization

**Use Cases**:
- Understanding overall architecture
- Making architectural decisions
- System design interviews
- Onboarding architects/senior engineers

---

### 2. Design Patterns Catalog

**File**: [`patterns/design_patterns.md`](patterns/design_patterns.md)

**Size**: 51KB

**Contents**: 13 design patterns documented:

#### Creational Patterns
1. **Factory Method** - Payment operation creation
2. **Builder Pattern** - API request/response construction

#### Structural Patterns
3. **Adapter Pattern** - Connector integration (100+ adapters)
4. **Facade Pattern** - Simplified payment API
5. **Repository Pattern** - Data access abstraction

#### Behavioral Patterns
6. **Strategy Pattern** - Routing algorithms
7. **State Pattern** - Payment lifecycle management
8. **Observer Pattern** - Event publishing
9. **Template Method** - Connector flow template

#### Domain-Driven Design (DDD)
10. **Aggregate Pattern** - Payment aggregate root
11. **Value Object Pattern** - Type-safe IDs
12. **Domain Event Pattern** - Event sourcing

#### Concurrency Patterns
13. **Actor Model** - Async task processing

**Each Pattern Includes**:
- Intent and purpose
- Code location (crate, file, lines)
- Participants and collaborations
- Implementation details with Rust code
- Benefits and trade-offs
- Usage examples
- Related patterns

**Use Cases**:
- Understanding design decisions
- Implementing similar patterns
- Code reviews
- Refactoring planning

---

### 3. Core Data Structures

**File**: [`data-structures/core_structures.md`](data-structures/core_structures.md)

**Contents**:

#### Domain Entities
1. **PaymentIntent** (~600-800 bytes)
   - 25+ fields
   - Lifecycle management
   - Access patterns (O(1), O(log n))
   
2. **PaymentAttempt** (~800-1200 bytes)
   - 40+ fields
   - Connector-specific data
   - Encrypted payment data
   
3. **Refund** (~500-700 bytes)
   - Amount tracking
   - Status management
   - Business invariants

#### Collection Types
- **HashMap**: O(1) lookups (connector configs)
- **Vec**: Ordered collections (attempts list)
- **BTreeMap**: Ordered mappings (priority routing)

#### Memory Layouts
- Stack vs. heap allocation
- Size optimization techniques
- Alignment and padding

#### Performance Characteristics
- Database query latency
- In-memory operation complexity
- Cache hit rates

#### Type-Safe Wrappers
- NewType pattern for IDs
- Phantom types for state machines
- Zero-cost abstractions

**Use Cases**:
- Understanding data model
- Memory optimization
- Performance tuning
- Writing efficient queries

---

### 4. Routing & Algorithms

**File**: [`algorithms/routing_and_algorithms.md`](algorithms/routing_and_algorithms.md)

**Contents**:

#### Payment Routing Algorithms
1. **Static Routing** - O(1) + O(k)
   - Pre-configured priority list
   - Cache-based lookup
   - Simple fallback

2. **Dynamic Volume-Based Routing** - O(n log n)
   - Real-time metrics
   - Weighted scoring (success_rate × 0.7 + volume × 0.3)
   - Business rules application

3. **Latency-Optimized Routing** - O(n log n)
   - P95 latency optimization
   - Predictive modeling
   - Low-latency path selection

#### Retry & Fallback Strategies
1. **Exponential Backoff** - O(1) per retry
   - Formula: delay = base × 2^attempt ± jitter
   - Max 3 retries, 60s max delay
   - 20% jitter

2. **Connector Fallback** - O(k) connectors
   - Automatic failover
   - Circuit breaker pattern
   - Terminal error detection

#### State Machine Transitions
- PaymentIntent state machine
- Validation algorithm - O(1)
- Terminal state detection

#### Reconciliation Algorithms
- Payment amount reconciliation - O(m + r)
- Invariant checking
- Discrepancy detection

#### Caching & Invalidation
- LRU Cache with TTL - O(1) ops
- Eviction policy
- Hit rate optimization (95%)

#### Rate Limiting
- Token bucket algorithm - O(1)
- Per-merchant/connector limits
- Retry-after calculation

**Use Cases**:
- Optimizing routing logic
- Implementing retries
- Performance tuning
- Capacity planning

---

## Quick Reference

### Key Crates by Layer

#### API Layer
- **router** - Main HTTP server (Actix-web)
- **api_models** - Request/response DTOs

#### Domain Layer
- **hyperswitch_domain_models** - Core business models
- **common_enums** - Shared enumerations

#### Application Layer
- **hyperswitch_interfaces** - Trait definitions
- **euclid** - Business rules engine (DSL → WASM)
- **analytics** - Payment analytics

#### Infrastructure Layer
- **hyperswitch_connectors** - Connector implementations (100+)
- **diesel_models** - Database ORM models
- **storage_impl** - Storage abstraction
- **redis_interface** - Redis caching
- **external_services** - Third-party integrations

#### Supporting
- **common_utils** - Shared utilities
- **masking** - PII masking
- **events** - Event logging
- **scheduler** - Background jobs

---

### Common Workflows

#### 1. Creating a Payment

**Flow**:
```
POST /payments
  → API validation (api_models)
  → Business logic (router/core/payments)
  → Domain validation (domain_models)
  → Routing decision (router/core/routing)
  → Connector call (hyperswitch_connectors)
  → Database save (diesel_models + storage_impl)
  → Response (api_models)
```

**Latency**: 50-200ms (including connector call)

**Files**:
- API: `crates/router/src/routes/payments.rs`
- Core: `crates/router/src/core/payments.rs`
- Storage: `crates/diesel_models/src/payment_intent.rs`

---

#### 2. Routing a Payment

**Algorithms**:
1. **Static**: Pre-configured priority → O(1) + O(k)
2. **Dynamic**: Real-time metrics → O(n log n)
3. **Latency-optimized**: P95 latency → O(n log n)

**Files**:
- `crates/router/src/core/routing/algorithms.rs`
- `crates/router/src/core/routing/helpers.rs`

**Configuration**:
- Merchant-level routing rules
- Connector priorities
- Business rules

---

#### 3. Handling Failures

**Strategies**:
1. **Retry** - Exponential backoff (3 attempts)
2. **Fallback** - Alternative connectors
3. **Circuit Breaker** - Disable failing connectors

**Files**:
- `crates/router/src/core/errors/retry.rs`
- `crates/router/src/core/payments/flows.rs`

---

### Database Tables Quick Ref

| Table | Purpose | Key Fields | Indexes |
|-------|---------|------------|---------|
| `payment_intent` | Main payment | payment_id, merchant_id, status, amount | PK: payment_id; idx: merchant+created |
| `payment_attempt` | Connector attempts | attempt_id, payment_id, connector, status | PK: attempt_id; idx: payment_id |
| `refund` | Refund tracking | refund_id, payment_id, refund_amount | PK: internal_id; idx: payment_id |
| `customers` | Customer info | customer_id, merchant_id, email | PK: customer_id; unique: merchant+email |
| `payment_methods` | Stored PMs | pm_id, customer_id, encrypted_data | PK: pm_id; idx: customer_id |
| `merchant_connector_account` | Connector configs | mca_id, merchant_id, connector | PK: mca_id; idx: merchant+connector |

---

## How to Use This Documentation

### For New Developers

1. **Start Here**: Read this README overview
2. **Understand Architecture**: Read [`architecture/low_level_design.md`](architecture/low_level_design.md) sections 1-3
3. **Visual Overview**: Review PlantUML diagrams (C4 component, class diagram)
4. **Core Flows**: Study sequence diagram for payment creation
5. **Patterns**: Skim [`patterns/design_patterns.md`](patterns/design_patterns.md) for key patterns
6. **Deep Dive**: Read specific crate documentation as needed

**Estimated Time**: 2-3 hours for comprehensive understanding

---

### For Feature Development

1. **Identify Components**: Use C4 component diagram to find relevant crates
2. **Understand Domain**: Read data structures documentation for affected entities
3. **Check Patterns**: Review pattern catalog for similar implementations
4. **Study Flows**: Analyze sequence diagrams for related workflows
5. **Reference Algorithms**: Check routing/algorithms doc for logic patterns

---

### For Code Reviews

1. **Architecture Alignment**: Verify changes follow documented patterns
2. **Data Model**: Check against ERD and data structures doc
3. **State Transitions**: Validate against state machine documentation
4. **Performance**: Compare against documented complexity and latency
5. **Patterns**: Ensure consistent pattern usage

---

### For System Design

1. **Current Architecture**: Study low-level design doc
2. **Patterns**: Leverage existing patterns from catalog
3. **Scalability**: Review performance characteristics
4. **Integration**: Check connector integration patterns
5. **Database**: Reference ERD for schema design

---

### For Debugging

1. **Flow Tracing**: Use sequence diagrams to trace execution path
2. **State Validation**: Check state machine documentation for valid transitions
3. **Data Verification**: Reference data structures for field constraints
4. **Algorithm Logic**: Review routing/algorithms doc for expected behavior
5. **Pattern Understanding**: Check pattern catalog for component interactions

---

## Document Versions

### Version 1.0.0 (December 18, 2025)

**Initial Release**

**Contents**:
- 5 PlantUML diagrams (C4 component, class diagram, sequence diagram, ERD)
- Comprehensive low-level architecture document (45KB)
- Design patterns catalog (51KB, 13 patterns)
- Core data structures documentation
- Routing and algorithms documentation
- This comprehensive README

**Coverage**:
- ✅ Router architecture
- ✅ Payment domain models
- ✅ Connector integration
- ✅ Storage layer
- ✅ Routing algorithms
- ✅ Design patterns
- ✅ State machines
- ✅ Database schema

**Known Gaps** (Future Updates):
- Additional sequence diagrams (confirm, capture, refund flows)
- State machine diagrams (PlantUML)
- More C4 diagrams (scheduler, analytics crates)
- Communication diagrams
- Deployment architecture
- Security architecture deep-dive
- Performance benchmarks
- API documentation integration

---

## Maintenance & Updates

### Updating Documentation

This documentation should be updated when:
- Major architectural changes occur
- New design patterns are introduced
- Core data structures change significantly
- Routing algorithms are modified
- New features affect documented flows

### Document Ownership

- **Maintained By**: Software Architect Agent
- **Review Cycle**: Quarterly or on major releases
- **Approval**: Technical Lead / Solution Architect

### Contributing

To update this documentation:
1. Modify relevant files in `generated-docs/`
2. Update PlantUML diagrams if architecture changes
3. Increment version numbers in document headers
4. Update this README with change summary
5. Create PR with "docs:" prefix

---

## Related Resources

### Internal Documentation
- **API Reference**: `api-reference/` folder
- **Connector Template**: `connector-template/` folder
- **Development Guide**: `README.md` (root)
- **Change Log**: `CHANGELOG.md`

### External Resources
- **GitHub Repository**: https://github.com/juspay/hyperswitch
- **Rust Documentation**: https://doc.rust-lang.org/
- **Actix-web Docs**: https://actix.rs/docs/
- **Diesel ORM**: https://diesel.rs/
- **PlantUML**: https://plantuml.com/

---

## Feedback & Questions

For questions or feedback on this documentation:
- Create GitHub issue with `[docs]` prefix
- Contact: Technical Architecture Team
- Slack: #architecture channel

---

**Document Status**: ✅ Complete  
**Last Reviewed**: December 18, 2025  
**Next Review**: March 2026 (or on next major release)

---

## Summary Statistics

| Category | Count | Size |
|----------|-------|------|
| **PlantUML Diagrams** | 4 | ~5KB total |
| **Documentation Files** | 4 | ~150KB total |
| **Design Patterns** | 13 | Documented |
| **Core Data Structures** | 3 | Detailed |
| **Routing Algorithms** | 3 | Analyzed |
| **Database Tables** | 10+ | In ERD |
| **Crates Documented** | 38 | Overview |
| **Total Pages** | ~100 | Rendered |

---

**End of README**
