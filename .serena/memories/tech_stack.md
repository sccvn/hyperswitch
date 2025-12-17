# Hyperswitch Technology Stack

## Core Technologies
- **Language**: Rust (Edition 2021, Minimum: 1.85.0)
- **Async Runtime**: Tokio (multi-threaded)
- **Web Framework**: Actix-web 4.11.0
- **Database**: PostgreSQL with Diesel ORM 2.2.10
- **Cache**: Redis (with custom interface wrapper)
- **Configuration**: TOML-based with Cargo features

## Key Dependencies

### Web & HTTP
- `actix-web`: HTTP server framework
- `actix-http`: HTTP primitives
- `actix-cors`: CORS support
- `actix-multipart`: Multipart request handling
- `actix-rt`: Async runtime

### Database & Storage
- `diesel`: Database ORM with PostgreSQL support
- `bb8`: Connection pooling
- `async-bb8-diesel`: Async connection management

### Serialization & Data
- `serde`: Serialization framework
- `serde_json`: JSON support
- `serde_qs`: Query string serialization
- `serde_with`: Custom serialization helpers
- `serde_path_to_error`: Better error context

### Security & Cryptography
- `ring`: Cryptographic operations
- `sha2`: SHA-2 hashing
- `blake3`: BLAKE3 hashing
- `hkdf`: Key derivation
- `openssl`: OpenSSL bindings
- `argon2`: Password hashing
- `x509-parser`: X.509 certificate parsing
- `jsonwebtoken`: JWT handling
- `josekit`: JOSE/JWE support

### HTTP Client
- `reqwest`: HTTP client (with rustls-tls, gzip, multipart)

### API & Protocol
- `prost-types`: Protocol Buffer types (v2 only)
- `rdkafka`: Kafka integration for events
- `utoipa`: OpenAPI documentation generation

### Utilities
- `uuid`: UUID generation
- `nanoid`: Nanoid generation
- `base64`: Base64 encoding/decoding
- `hex`: Hex encoding/decoding
- `url`: URL parsing
- `regex`: Regular expressions
- `time`: Date/time handling (with serde support)
- `tokio`: Async runtime and utilities
- `bytes`: Byte manipulation
- `futures`: Future utilities
- `async-trait`: Async trait support
- `error-stack`: Error context handling
- `thiserror`: Error derive macros
- `tracing-futures`: Tracing integration

### Development Tools
- `cargo+nightly fmt`: Code formatting (nightly required)
- `cargo clippy`: Linting
- `cargo nextest`: Next-gen test runner
- `wasm-pack`: WASM compilation
- `diesel CLI`: Migration management

## Feature Flags

### Default Features
`v1`, `kv_store`, `stripe`, `oltp`, `olap`, `accounts_cache`, `dummy_connector`, `payouts`, `payout_retry`, `retry`, `frm`, `tls`, `partial-auth`

### Major Feature Sets
- `v1` / `v2` - API versions (mutually exclusive)
- `release` - Production build (includes email, AWS KMS, S3, dynamic routing)
- `oltp` / `olap` - Database optimization modes
- `frm` - Fraud Risk Management
- `payouts` - Payout functionality
- `recon` - Reconciliation support
- `revenue_recovery` - Revenue recovery features
- `encryption_service` - End-to-end encryption
- `dynamic_routing` - Advanced routing logic
- `email` - Email notifications
- `tokenization_v2` - Payment method tokenization (v2)

## Build Artifacts
- Main Binary: `router` - Payments router server
- Secondary Binary: `scheduler` - Task scheduling service
- WASM Library: `euclid` - DSL for business logic (compiled to WASM)

## Crate Organization
The project uses a workspace with 40+ crates organized by function:
- Core: `router`, `hyperswitch_connectors`, `hyperswitch_interfaces`
- Domain: `hyperswitch_domain_models`, `diesel_models`, `common_types`
- API: `api_models`, `openapi`
- Utilities: `common_utils`, `masking`, `cards`, `scheduler`
- Infrastructure: `storage_impl`, `redis_interface`, `external_services`
- Analytics: `analytics`
- Specialized: `euclid`, `euclid_wasm`, `subscription`, `payment_link`
