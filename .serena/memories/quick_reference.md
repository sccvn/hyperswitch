# Hyperswitch Quick Reference Guide

## Project Identity
- **Name**: Hyperswitch
- **Type**: Open-source, modular payments infrastructure
- **Language**: Rust (Edition 2021, Rust 1.85.0+)
- **License**: Apache 2.0
- **Repository**: https://github.com/juspay/hyperswitch
- **Maintained By**: Juspay (400+ enterprise customers)

## 🚀 Quick Start

```bash
# Clone and setup (5 minutes)
git clone https://github.com/juspay/hyperswitch.git
cd hyperswitch
scripts/setup.sh

# Or manual setup
cargo build --release --features release
just migrate
just run

# Access at http://localhost:8080
```

## 📁 Key Directories
- **`crates/`** - 40+ Rust crates (modular architecture)
- **`router/`** - Main payment processing engine
- **`config/`** - Configuration files
- **`migrations/`** - Database migrations (V1 & V2)
- **`docker/`** - Container configs
- **`docs/`** - Documentation

## 🛠️ Essential Commands

| Command | Purpose |
|---------|---------|
| `just precommit` | Format + lint (REQUIRED before commit) |
| `cargo test --all-features` | Run all tests |
| `just build` | Build debug binary |
| `just build-release` | Build optimized release |
| `just run` | Run router server |
| `just migrate` | Run database migrations |
| `just resurrect` | Reset database |
| `cargo +nightly fmt --all` | Format code |

## 📦 Tech Stack Summary

**Core:**
- **Web Framework**: Actix-web 4.11
- **Database**: PostgreSQL + Diesel ORM 2.2
- **Cache**: Redis
- **Async**: Tokio (multi-threaded)
- **Serialization**: Serde/JSON

**Security:**
- `ring` for cryptography
- `argon2` for password hashing
- `x509-parser` for certificates
- PII protection with `masking` crate

**Payment Processing:**
- 100+ connector integrations
- Trait-based extensible architecture
- Support for cards, wallets, BNPL, bank transfers, UPI

## 🎯 Code Style Rules

**MUST DO:**
- ✅ Use `error-stack` for error handling
- ✅ Use `tracing` macros for logging (not `println!`)
- ✅ Write doc comments for public APIs
- ✅ Add unit tests (target 80%+ coverage)
- ✅ Run `just precommit` before commits

**NEVER DO:**
- ❌ Use `unsafe` code (forbidden by clippy)
- ❌ Use `unwrap()`, `expect()`, `panic!()` in production code
- ❌ Hardcode secrets or API keys
- ❌ Force push after opening PR
- ❌ Commit without running precommit checks

## 📝 Commit Message Format
```
<type>(<scope>): <description>

<optional body explaining why>

Fixes #123
```

**Types**: `feat`, `fix`, `perf`, `refactor`, `test`, `docs`, `chore`, `ci`

## 🧪 Testing
```bash
# Run all tests
cargo test --all-features

# Run specific test
cargo test test_name

# Run with output
cargo test -- --nocapture

# Generate coverage
RUSTFLAGS="-Cinstrument-coverage" cargo build --bin=router
grcov . --source-dir . --output-type html --binary-path ./target/debug
```

## 🔌 Key Traits & Patterns

**PaymentConnector** - Implement to add new payment processor
**WebhookConnector** - Implement for webhook handling
**StorageInterface** - Implement custom storage backend
**ErrorResponse** - Standardized error formatting

## 🌍 API Basics

**Main Endpoints:**
- `POST /payments` - Create payment
- `POST /payments/{id}/confirm` - Confirm payment
- `POST /payments/{id}/capture` - Capture authorized payment
- `POST /payments/{id}/refund` - Refund payment
- `GET /payments/{id}` - Get payment status

## 📊 Feature Flags

**Mutually Exclusive:**
- `v1` (default) - V1 API
- `v2` - V2 API (new architecture)

**Optional:**
- `frm` - Fraud Risk Management
- `recon` - Reconciliation
- `revenue_recovery` - Revenue recovery features
- `release` - Production build (includes everything)

## 🗄️ Database

**Key Tables (auto-generated from Diesel models):**
- `payments` - Payment transactions
- `payment_attempts` - Individual payment attempts
- `payment_methods` - Stored payment methods
- `refunds` - Refund transactions
- `webhooks` - Webhook events
- `customers` - Customer data

**Tools:**
- Migration tool: Diesel CLI
- Migrations: `migrations/` and `v2_migrations/` directories

## 🔐 Security

**Secrets Management:**
- Never hardcode API keys
- Use environment variables or Vault
- Mask PII using `masking` crate

**PII Protection:**
```rust
use masking::Secret;

#[derive(Serialize, Deserialize)]
pub struct Card {
    #[serde(serialize_with = "masking::serialize_masked")]
    pub number: Secret<String>,
}
```

## 📚 Memory Files Available

This project has created 8 comprehensive memory files:

1. **project_overview.md** - Project description, philosophy, features
2. **tech_stack.md** - Dependencies, versions, architecture
3. **code_structure.md** - Directory layout, crate organization
4. **build_commands.md** - Build, test, format commands
5. **code_style_conventions.md** - Style guide, naming, patterns
6. **suggested_commands.md** - Most useful commands reference
7. **development_workflow.md** - How to contribute, commit conventions
8. **connector_integration.md** - How to add payment processors
9. **quick_reference.md** - This file!

## 🚨 Common Pitfalls

1. **Feature Conflicts**: V1 and V2 can't both be enabled
   ```bash
   # Wrong: will fail
   cargo build --features v1,v2
   
   # Right: pick one
   cargo build --features v1
   ```

2. **Clippy Warnings**: Must fix all warnings before commit
   ```bash
   # Check for issues
   cargo clippy --all-features -- -D warnings
   ```

3. **Formatting**: Must use nightly Rust
   ```bash
   cargo +nightly fmt --all
   ```

4. **Database Connection**: Migrations fail without PostgreSQL
   ```bash
   # Use Docker
   docker-compose up -d
   ```

## 📞 Getting Help

- **Documentation**: https://docs.hyperswitch.io/
- **GitHub Discussions**: https://github.com/juspay/hyperswitch/discussions
- **Slack**: https://inviter.co/hyperswitch-slack
- **Discord**: https://discord.gg/wJZ7DVW8mm
- **Issues**: https://github.com/juspay/hyperswitch/issues

## ✅ Pre-Commit Checklist

Before creating a pull request:
- [ ] Run `just precommit` (format + clippy)
- [ ] All tests pass: `cargo test --all-features`
- [ ] No hardcoded secrets or configuration
- [ ] All public APIs have doc comments
- [ ] Error handling uses `error-stack`
- [ ] Logging uses `tracing` macros
- [ ] Commit message follows convention
- [ ] No `TODO`, `unwrap()`, or `panic!()` without docs

## 🎓 Learning Path

1. **Read**: `project_overview.md` (what is this project?)
2. **Explore**: `code_structure.md` (where is everything?)
3. **Setup**: Use `scripts/setup.sh` (get it running locally)
4. **Learn**: `code_style_conventions.md` (how to write Rust here?)
5. **Contribute**: `development_workflow.md` (how to create PRs?)
6. **Reference**: `suggested_commands.md` (quick command lookup)

## 📈 Architecture Highlights

**Microservices Pattern:**
- Router (main server) - handles payment flows
- Scheduler (background service) - processes deferred tasks
- Drainer - consumes Redis streams

**Storage Abstraction:**
- Trait-based design
- Pluggable implementations (PostgreSQL, Redis)
- Multi-database support

**Connector Pattern:**
- Trait-based connector interface
- Pluggable payment processors
- Webhook handling

**Type Safety:**
- Strong typing prevents bugs
- Newtype pattern for domain concepts
- Compile-time guarantees

---

**Last Updated**: 2025-12-17
**Rust Version**: 1.85.0+
**Project Status**: Active Development
