# Suggested Commands for Hyperswitch Development

This is a quick reference for the most useful commands when developing Hyperswitch.

## Setup & Installation

```bash
# Clone the repository
git clone https://github.com/juspay/hyperswitch.git
cd hyperswitch

# One-click local setup with Docker
scripts/setup.sh

# Or manual setup
cargo build --release --features release
```

## Daily Development Commands

### Pre-commit (REQUIRED before every commit)
```bash
# Format + lint + basic compile check
just precommit

# Equivalent to:
cargo +nightly fmt --all
cargo clippy --all-features --all-targets -- -D warnings
```

### Testing
```bash
# Run all tests
cargo test --all-features

# Run specific test
cargo test --all-features test_payment_creation

# Run with output
cargo test --all-features -- --nocapture

# Use nextest for faster parallel execution
cargo nextest run
```

### Code Formatting
```bash
# Auto-format code (requires nightly Rust)
cargo +nightly fmt --all

# Check formatting without changing
cargo +nightly fmt --all -- --check
```

### Compilation Checks
```bash
# Fast check without full build
cargo check

# With all features
cargo check --all-features

# Lint check (clippy)
cargo clippy --all-features --all-targets -- -D warnings
```

### Building
```bash
# Development build
just build

# Optimized release build
just build-release

# Build V2 version
just build_v2

# Build specific crate
cargo build --package router

# Build with custom features
cargo build --features "v2,revenue_recovery"
```

### Running
```bash
# Run router server (development)
just run

# Run router server (release optimized)
cargo run --release --features release

# Run V2 version
just run_v2

# Run scheduler (background job processor)
cargo run --package router --bin scheduler

# Run with debug logging
RUST_LOG=debug cargo run
```

### Documentation
```bash
# Generate and open documentation
just doc

# Generate and open with private items
cargo doc --all-features --all-targets --document-private-items --open

# Generate without opening
cargo doc --all-features --all-targets
```

## Database Management

```bash
# Run database migrations (V1)
just migrate

# Run database migrations (V2)
just migrate_v2

# Run compatible migrations (works for both V1 & V2)
just migrate_v2_compatible

# Reset database (drop and recreate)
just resurrect

# Check migration status
diesel migration list

# Custom database setup
DB_USER=testuser DB_PASSWORD=testpass just migrate
```

## Feature-Based Development

```bash
# V1 (default)
cargo build

# V2 only
cargo build --no-default-features --features "v2,stripe,dummy_connector"

# With additional features
cargo build --features "frm,recon,revenue_recovery"

# Production/release build
cargo build --release --features release

# Check which features are available
cargo metadata --all-features --format-version 1 --no-deps | jq '.packages[] | select(.name=="router") | .features'
```

## Git Workflow

```bash
# Before committing: run pre-commit checks
just precommit

# View uncommitted changes
git diff

# View staged changes
git diff --staged

# Commit with message following convention
git commit -m "feat(router): add payment retry logic"

# See recent commits
git log --oneline -10

# Check current branch
git branch -v
```

## Debugging & Troubleshooting

### Enable Debug Logging
```bash
# Set log level
export RUST_LOG=debug
export LOG_LEVEL=debug

# Run with verbose output
cargo run -- --log-level debug

# Specific module debugging
export RUST_LOG=router::payments=debug
```

### Memory & Performance
```bash
# Check compilation with all feature combinations
cargo hack check --workspace --each-feature

# Build with incremental compilation
export CARGO_INCREMENTAL=1

# Use sccache for faster rebuilds
cargo install sccache
export RUSTC_WRAPPER=sccache
```

### Problem Solving
```bash
# Clean build artifacts
cargo clean

# Clean and rebuild
cargo clean && cargo build

# Check for issues
cargo check --all-features

# Verify migrations
diesel migration list

# Reset database if corrupted
just resurrect
```

## Testing & CI

```bash
# Run CI checks locally
scripts/ci-checks.sh

# OR with just
just ci_hack

# Run integration tests
cargo test --all-features --test '*'

# Generate code coverage
RUSTFLAGS="-Cinstrument-coverage" cargo build --bin=router --package=router
grcov . --source-dir . --output-type html --binary-path ./target/debug
```

## Docker & Deployment

```bash
# Build Docker image
docker build -t hyperswitch:latest .

# Run with docker-compose
docker-compose up -d

# Extended development environment
docker-compose -f docker-compose-development.yml up -d

# View logs
docker-compose logs -f router

# Stop services
docker-compose down
```

## Code Generation

```bash
# Build euclid WASM library
just euclid-wasm dummy_connector

# Full WASM build with custom features
just euclid-wasm "stripe,adyen,dummy_connector"

# OpenAPI spec (auto-generated, check openapi/ directory)
# Specification is in: openapi/spec.json
```

## Development Workflow Steps

### Starting a New Feature
```bash
# 1. Update database schema (if needed)
diesel migration generate add_new_field
# Edit migrations/*/up.sql and down.sql

# 2. Add database model
# Edit diesel_models/src/...

# 3. Format and check
just precommit

# 4. Run migrations
just migrate

# 5. Implement feature

# 6. Test thoroughly
cargo test --all-features

# 7. Final checks before commit
just precommit
git diff --staged
```

### Creating a Pull Request
```bash
# 1. Ensure all checks pass
just precommit
cargo test --all-features

# 2. Commit with semantic message
git commit -m "feat(payments): add retry logic for failed transactions"

# 3. Push to your fork
git push origin feature/retry-logic

# 4. Create PR from GitHub UI
# Include description following template in .github/PULL_REQUEST_TEMPLATE.md
```

## Environment Variables

```bash
# Database
export DATABASE_URL="postgresql://user:password@localhost:5432/hyperswitch_db"

# Redis
export REDIS_URL="redis://localhost:6379"

# Logging
export LOG_LEVEL="debug"
export RUST_LOG="router=debug,hyperswitch_connectors=debug"

# API Keys (for testing specific connectors)
export STRIPE_API_KEY="sk_test_..."
export ADYEN_API_KEY="..."
```

## Performance Profiling

```bash
# Build with profiling info
cargo build --release

# Run with perf (Linux)
perf record -g ./target/release/router
perf report

# Generate flamegraph
cargo install flamegraph
cargo flamegraph

# Check binary size
cargo build --release
ls -lh target/release/router
```

## Quick Command Aliases

Add to your shell profile (`.bashrc`, `.zshrc`, etc.):

```bash
alias hs-precommit="just precommit"
alias hs-test="cargo test --all-features"
alias hs-run="just run"
alias hs-build="just build"
alias hs-fmt="cargo +nightly fmt --all"
alias hs-db-reset="just resurrect"
alias hs-check="cargo check --all-features"
alias hs-clippy="cargo clippy --all-features --all-targets -- -D warnings"
```

## Common Tasks Matrix

| Task | Command | V1 | V2 | Notes |
|------|---------|----|----|-------|
| Build | `just build` | ✅ | ❌ | Use `just build_v2` for V2 |
| Test | `cargo test --all-features` | ✅ | ❌ | Features conflict |
| Format | `cargo +nightly fmt --all` | ✅ | ✅ | Same for both |
| Lint | `cargo clippy` | ✅ | ✅ | Use `just clippy_v2` for V2 |
| Run | `just run` | ✅ | ❌ | Use `just run_v2` for V2 |
| Migrate | `just migrate` | ✅ | ❌ | Use `just migrate_v2` for V2 |
