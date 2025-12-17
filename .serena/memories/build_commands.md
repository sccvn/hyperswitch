# Hyperswitch Build & Development Commands

## Task Runner
Project uses **`just`** as primary task runner (justfile). Also supports **Make**.

### Quick Reference

| Task | Command | Purpose |
|------|---------|---------|
| List tasks | `just list` | Show available recipes |
| Format code | `just fmt` | Run Rust formatter (nightly) |
| Check code | `just check` | Compile check |
| Clippy lint | `just clippy` | Run linter (v1 features) |
| Clippy V2 | `just clippy_v2` | Run linter (v2 features) |
| Build | `just build` | Build binaries |
| Build release | `just build-release` | Optimized build with release features |
| Build V2 | `just build_v2` | Build V2 version |
| Run server | `just run` | Run router server |
| Run V2 | `just run_v2` | Run V2 version |
| Documentation | `just doc` | Generate docs |
| Pre-commit | `just precommit` | Format + clippy (required before commits) |
| Migrate DB | `just migrate` | Run database migrations (V1) |
| Migrate V2 | `just migrate_v2` | Run database migrations (V2) |
| Reset DB | `just resurrect` | Drop and recreate database |
| WASM build | `just euclid-wasm [features]` | Build euclid as WASM library |

## Cargo Commands (Direct)

### Basic Commands
```bash
cargo check                          # Check without building
cargo build                          # Build debug binary
cargo build --release               # Build release binary
cargo run                            # Build and run
cargo test --all-features           # Run all tests
cargo doc --all-features --open     # Generate and open docs
```

### Formatting & Linting
```bash
cargo +nightly fmt --all            # Format all code (requires nightly)
cargo +nightly fmt --all -- --check # Check formatting without changing
cargo clippy --all-features --all-targets -- -D warnings  # Run clippy with all features
cargo clippy --all-targets          # Run clippy on v1 features
```

### Testing
```bash
cargo test --all-features           # Run all tests
cargo nextest run                   # Use nextest (parallel test runner)
cargo test --doc                    # Run doctests
```

### Feature Combinations

**IMPORTANT**: V1 and V2 are mutually exclusive features.

```bash
# V1 (default)
cargo check --features "default"
cargo build --no-default-features --features "v2,other_features"

# Get all features automatically
cargo metadata --all-features --format-version 1 --no-deps | jq ...
```

### Code Coverage
```bash
# 1. Install prerequisites
rustup component add llvm-tools-preview
cargo install grcov

# 2. Build with coverage
RUSTFLAGS="-Cinstrument-coverage" cargo build --bin=router --package=router

# 3. Run tests (in another terminal)
# Run cypress tests or manual tests

# 4. Generate HTML report
LLVM_PROFILE_FILE="coverage.profraw" target/debug/router
grcov . --source-dir . --output-type html --binary-path ./target/debug
cd html && python3 -m http.server 8000

# VSCode integration
grcov . -s . -t lcov --output-path lcov.info --binary-path ./target/debug --keep-only "crates/*"
```

## Make Commands (Alternative)

| Command | Purpose |
|---------|---------|
| `make check` | Check compilation |
| `make build` | Build debug binary |
| `make fmt` | Format code |
| `make clippy` | Run clippy linter |
| `make test` | Run tests |
| `make nextest` | Run tests with nextest |
| `make precommit` | Format + clippy + test |
| `make doc` | Generate documentation |
| `make euclid-wasm` | Build euclid WASM library |
| `make hack` | Run cargo hack feature check |

## Database Setup

### Using Just
```bash
# Configure database URL
DB_USER=myuser DB_PASSWORD=mypass DB_NAME=hyperswitch_db just migrate

# Reset database
just resurrect

# Migrate V2
just migrate_v2

# Migrate compatible migrations
just migrate_v2_compatible
```

### Using Diesel CLI
```bash
# Install diesel CLI
cargo install diesel_cli --no-default-features --features postgres

# Run migrations
diesel migration run --database-url "postgresql://user:pass@localhost/hyperswitch_db"

# Check migration status
diesel migration list --database-url "postgresql://user:pass@localhost/hyperswitch_db"
```

## Docker Commands

### One-Click Local Setup
```bash
# Clone and setup
git clone --depth 1 --branch latest https://github.com/juspay/hyperswitch
cd hyperswitch
scripts/setup.sh

# Follow prompts for deployment profile:
# - Standard: App server + Control Center
# - Full: Includes monitoring + schedulers
# - Minimal: Standalone App server
```

### Manual Docker
```bash
# Build Docker image
docker build -t hyperswitch:latest .

# Run with docker-compose
docker-compose up -d

# Development with extended setup
docker-compose -f docker-compose-development.yml up -d

# View logs
docker-compose logs -f router
```

## Development Workflow

### Before Committing
```bash
# Required pre-commit checks
just precommit

# Or manually:
cargo +nightly fmt --all
cargo clippy --all-features --all-targets -- -D warnings
cargo test --all-features
```

### Building for Release
```bash
# Optimized build with production features
just build-release

# Or with make
make precommit && cargo build --release --features release
```

### Running with Custom Config
```bash
# Set environment variables
export DATABASE_URL="postgresql://user:pass@host/db"
export REDIS_URL="redis://localhost:6379"
export LOG_LEVEL="debug"

# Run router
cargo run --package router --bin router

# Run scheduler
cargo run --package router --bin scheduler
```

## CI/CD Commands

### Local CI Simulation
```bash
# Run all CI checks locally
scripts/ci-checks.sh

# Or with just
just ci_hack
```

### Code Generation
```bash
# Generate OpenAPI spec (auto-generated)
# Located in: openapi/spec.json

# WASM bindings
just euclid-wasm dummy_connector
```

## Environment Variables

| Variable | Default | Purpose |
|----------|---------|---------|
| `DATABASE_URL` | - | PostgreSQL connection string |
| `REDIS_URL` | `redis://localhost` | Redis connection |
| `LOG_LEVEL` | `info` | Logging level |
| `RUST_LOG` | - | Tracing filter |
| `RUSTFLAGS` | - | Compiler flags |

## Troubleshooting

### Feature Conflicts
```bash
# If features conflict, check enabled features
cargo metadata --format-version 1 | jq '.packages[] | select(.name=="router") | .features'
```

### Slow Compilation
```bash
# Use incremental compilation
export CARGO_INCREMENTAL=1

# Use sccache
cargo install sccache
export RUSTC_WRAPPER=sccache
```

### Database Issues
```bash
# Reset database completely
just resurrect hyperswitch_db

# Check migration status
diesel migration list
```
