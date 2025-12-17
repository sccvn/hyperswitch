# Hyperswitch Code Structure

## Directory Organization

```
hyperswitch/
├── crates/                          # Rust crates (workspace)
│   ├── router/                      # Main payment router server
│   ├── hyperswitch_connectors/      # Payment processor integrations
│   ├── hyperswitch_interfaces/      # Trait definitions for extensibility
│   ├── hyperswitch_domain_models/   # Core business domain models
│   ├── api_models/                  # Request/response DTOs
│   ├── diesel_models/               # Database models (ORM)
│   ├── common_enums/                # Shared enums across crates
│   ├── common_utils/                # Utility functions and helpers
│   ├── common_types/                # Shared type definitions
│   ├── storage_impl/                # Storage backend implementations
│   ├── redis_interface/             # Redis wrapper interface
│   ├── external_services/           # AWS KMS, S3, email, etc.
│   ├── scheduler/                   # Task scheduling service
│   ├── masking/                     # PII protection utilities
│   ├── cards/                       # Card handling and validation
│   ├── router_derive/               # Procedural macros for router
│   ├── router_env/                  # Environment, logging, config
│   ├── euclid/                      # Business logic DSL
│   ├── euclid_wasm/                 # WASM compilation of euclid
│   ├── analytics/                   # Analytics engine
│   ├── payment_link/                # Payment link generation
│   ├── payment_methods/             # Payment method management
│   ├── subscriptions/               # Subscription management
│   ├── drainer/                     # Redis stream consumer
│   ├── test_utils/                  # Testing utilities
│   ├── openapi/                     # OpenAPI spec generation
│   ├── smithy/                      # Code generation tools
│   └── [35 more specialized crates]
│
├── migrations/                       # V1 database migrations (Diesel)
├── v2_migrations/                    # V2 database migrations
├── v2_compatible_migrations/         # Migrations compatible with both V1 & V2
│
├── config/                           # Startup configuration files
├── docker/                           # Docker and container configs
├── docker-compose.yml               # Local development setup
├── docker-compose-development.yml   # Extended development setup
│
├── scripts/                          # Automation and utility scripts
│   └── setup.sh                      # One-click local setup
│
├── docs/                             # Documentation
│   ├── CONTRIBUTING.md              # Contribution guidelines
│   ├── imgs/                         # Images and diagrams
│
├── monitoring/                       # Grafana & Loki configs
├── postman/                          # Postman API collections
├── cypress-tests/                    # E2E tests (legacy)
├── cypress-tests-v2/                # E2E tests (v2)
├── loadtest/                         # Performance testing
│
├── proto/                            # Protocol buffer definitions
├── connector-template/               # Boilerplate for new connectors
│
├── Cargo.toml                        # Workspace manifest
├── Cargo.lock                        # Dependency lock
├── justfile                          # Just task runner commands
├── Makefile                          # Make commands
├── diesel.toml                       # V1 Diesel config
├── diesel_v2.toml                    # V2 Diesel config
├── flake.nix                         # Nix development environment
│
└── CLAUDE.md                         # Claude Code configuration

```

## Key Crate Purposes

### Core Crates
- **router**: Main application server, payment flow orchestration
- **hyperswitch_connectors**: Adapter implementations for 100+ payment processors
- **hyperswitch_interfaces**: Trait definitions for connectors, storage, auth, etc.
- **hyperswitch_domain_models**: Domain-driven design models (Payments, Customers, etc.)

### API Crates
- **api_models**: Request/response structures for REST API
- **diesel_models**: Database entity models
- **common_types**: Type definitions used across multiple crates
- **common_enums**: Shared enumerations

### Infrastructure Crates
- **storage_impl**: Database and cache implementations
- **redis_interface**: Abstraction over Redis operations
- **external_services**: AWS, email, KMS integrations
- **router_env**: Logging, configuration, environment setup

### Processing Crates
- **scheduler**: Deferred task execution and cron jobs
- **drainer**: Processes Redis streams and persists to database
- **analytics**: Payment analytics and reporting

### Security Crates
- **masking**: PII protection (card numbers, SSN, etc.)
- **cards**: Card validation and type detection

### Specialized Crates
- **euclid**: DSL for business logic (rules engine)
- **euclid_wasm**: WASM compilation of euclid
- **payment_link**: Payment link generation
- **payment_methods**: Payment method storage and retrieval
- **subscriptions**: Subscription management
- **smithy**: Code generation from specifications

## API Structure Pattern
Typical API endpoint structure follows:
1. **Models** - Request/Response DTOs in `api_models`
2. **Handlers** - HTTP handlers in `router/src/routes`
3. **Services** - Business logic in `router/src/services`
4. **Core** - Core flows in `router/src/core`
5. **Database** - ORM models in `diesel_models`
6. **Tests** - Integration tests in respective modules

## Configuration Files
- `config/`: Application configuration templates
- `.env`: Environment variables (not in repo)
- `docker-compose*.yml`: Container orchestration
- `flake.nix`: Nix development environment

## Migration System
- Managed by Diesel CLI
- Separate migration directories for V1 and V2
- Compatible migrations in `v2_compatible_migrations/`
- Prefix system: V1 migrations have no prefix, V2 compatible get `8` prefix, V2-only get `9` prefix
