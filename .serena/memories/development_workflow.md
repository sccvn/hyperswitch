# Hyperswitch Development Workflow & Contribution Guide

## Contribution Philosophy

Hyperswitch is maintained as a community-driven project with enterprise backing (Juspay). The project welcomes contributions at all levels, from documentation fixes to feature implementations.

### Core Values
- **Transparency**: Code is open source by default
- **Community First**: Roadmap shaped by contributors
- **Quality**: High standards for reliability and security
- **No Vendor Lock-in**: Modular architecture allows picking components

## Getting Started

### Prerequisites
- **Rust**: 1.85.0 or later
- **PostgreSQL**: 12+ (for database)
- **Redis**: 6+ (for caching)
- **Docker** (recommended for local setup)
- **Just** or **Make** (task runners)

### Local Development Setup
```bash
# Option 1: Automated setup (recommended)
git clone https://github.com/juspay/hyperswitch.git
cd hyperswitch
scripts/setup.sh
# Follow prompts for deployment profile

# Option 2: Manual setup
git clone https://github.com/juspay/hyperswitch.git
cd hyperswitch

# Install dependencies
cargo build --all-features

# Setup database
just resurrect                    # Create fresh DB
just migrate                      # Run migrations

# Start server
just run
```

## Issue & Feature Workflows

### Reporting a Bug

1. **Search Existing Issues**: Check [GitHub Issues](https://github.com/juspay/hyperswitch/issues)
2. **Create Issue**: Use bug report template
3. **Provide Details**:
   - Rust version (`rustc --version`)
   - OS and version
   - Minimal reproducible example
   - Expected vs actual behavior
   - Relevant logs/error messages

4. **Wait for Triage**: Maintainers will classify with labels:
   - **Priority**: P-high, P-medium, P-low
   - **Difficulty**: E-easy, E-medium, E-hard
   - **Area**: A-core, A-connector-integration, A-framework, etc.
   - **Status**: S-awaiting-triage, S-in-progress, S-blocked, etc.

### Requesting a Feature

1. **Check Discussions**: See [GitHub Discussions](https://github.com/juspay/hyperswitch/discussions)
2. **Start Discussion**: Create new discussion for feature request
3. **Describe Use Case**: Why is this feature needed?
4. **Suggest Implementation**: If you have ideas
5. **Get Feedback**: Community discussion helps refine ideas

### Contributing Code

## Commit Message Convention

Hyperswitch uses Angular-style commit messages:

```
<type>(<scope>): <short summary>

<optional body (20+ chars)>

<optional footer with issue reference>
```

### Commit Types
- `feat`: New feature
- `fix`: Bug fix
- `perf`: Performance improvement
- `refactor`: Code restructuring (no functional change)
- `test`: Adding or updating tests
- `docs`: Documentation changes
- `chore`: Maintenance, dependencies, formatting
- `ci`: CI/CD configuration changes
- `build`: Build system changes

### Commit Scopes
- **Core Crates**: `router`, `masking`, `router_derive`, `router_env`, `openapi`, `postman`, `migrations`, `config`, `changelog`
- **Connector-related**: `connector`
- **Empty**: For changes across multiple crates

### Examples
```
feat(router): implement 3D Secure payment flow

This implements the 3D Secure authentication flow for credit card
payments. Includes request validation, provider routing, and
result handling.

Fixes #1234

---

fix(masking): handle null values in card masking

Prevent NullPointerException when masking cards with missing fields.

Closes #1235

---

test: add integration tests for payment retry logic

---

chore: update dependencies
```

## Pull Request Process

### Before Creating PR

1. **Run Pre-commit Checks** (REQUIRED)
   ```bash
   just precommit  # Format + clippy
   cargo test --all-features
   ```

2. **Update Migrations** (if DB changes)
   ```bash
   diesel migration generate migration_name
   # Edit up.sql and down.sql
   just migrate  # Test migrations work
   ```

3. **Add Tests**
   - Unit tests in same file as code
   - Integration tests if applicable
   - Test coverage for new functionality

4. **Update Documentation**
   - Doc comments for public APIs
   - Update CHANGELOG.md if user-facing change
   - Update docs/ if necessary

### Creating the PR

1. **Use PR Template**: GitHub auto-fills template from `.github/PULL_REQUEST_TEMPLATE.md`

2. **Fill Out Details**:
   ```markdown
   ## Summary
   - What does this PR do?
   - Why is it needed?

   ## Test Plan
   - [ ] Added unit tests
   - [ ] Tested manually with connector X
   - [ ] Verified database migrations

   ## Breaking Changes
   (If applicable)

   ## Related Issues
   Fixes #1234
   ```

3. **Assign Labels**: Add relevant labels for categorization

4. **Request Reviewers**: Tag relevant code owners

### Code Review Process

- **Community Review**: Any community member can review
- **Owner Review**: At least one code owner must approve
- **Feedback Loop**: Respond to comments constructively
- **Don't Rebase After Opening**: Just add incremental commits
- **Address All Comments**: Resolve conversations before merge

### Handling Review Feedback

**Do's**:
- ✅ Respond to all comments
- ✅ Ask clarifying questions if feedback unclear
- ✅ Make requested changes in follow-up commits
- ✅ Be respectful and professional
- ✅ Request re-review after changes

**Don'ts**:
- ❌ Force push after opening PR
- ❌ Dismiss feedback without engagement
- ❌ Take criticism personally
- ❌ Rebase extensively (causes confusion)

## Code Ownership

Different areas have designated owners who review PRs:

- **Core Router**: router team
- **Connectors**: connector integration team
- **Storage**: storage team
- **API Models**: API design team
- **Utilities**: core utilities team

See: `CODEOWNERS` file in repository

## Merge Process

Once approved:
1. Maintainer squashes commits with updated message
2. Final commit follows semantic versioning
3. Auto-merged if CI passes
4. Release notes generated from commit messages

## Development Phases (SPARC)

Hyperswitch follows SPARC methodology:

### 1. Specification Phase
- Analyze requirements
- Define acceptance criteria
- Design API contracts
- Identify edge cases

### 2. Pseudocode Phase
- Design algorithm
- Define data structures
- Plan flow logic
- Document approach

### 3. Architecture Phase
- System design
- Database schema
- API endpoints
- Error handling strategy

### 4. Refinement Phase (TDD)
- Write tests first
- Implement feature
- Refactor for clarity
- Ensure test coverage

### 5. Completion Phase
- Integration testing
- Performance validation
- Documentation
- Release preparation

## Testing Requirements

### Test Coverage
- **Unit Tests**: Required for all business logic
- **Integration Tests**: Required for API endpoints
- **Target Coverage**: 80%+ for new code

### Test Types
```rust
// Unit test
#[test]
fn test_validate_amount_with_zero_fails() { }

// Async test
#[tokio::test]
async fn test_process_payment_succeeds() { }

// Integration test (in tests/ directory)
#[tokio::test]
async fn test_end_to_end_payment_flow() { }
```

### Running Tests
```bash
cargo test --all-features                    # All tests
cargo test --test '*'                        # Integration tests only
cargo test test_payment                      # Specific test
cargo test -- --nocapture                    # Show output
```

## Documentation Requirements

### Code Comments
- **Module-level**: Explain purpose and structure
- **Public APIs**: Complete doc comments with examples
- **Complex Logic**: Explain "why" not "what"

### Example
```rust
/// Processes a payment transaction through the specified processor.
///
/// This function validates the payment, routes it to the appropriate
/// payment processor, and returns the result.
///
/// # Arguments
///
/// * `payment` - The payment to process
/// * `processor_id` - ID of the processor to use
///
/// # Returns
///
/// A `PaymentResponse` containing the result of the transaction.
///
/// # Errors
///
/// Returns `PaymentError::InvalidAmount` if amount is zero.
/// Returns `PaymentError::ProcessorNotFound` if processor_id invalid.
pub async fn process_payment(payment: Payment, processor_id: &str) 
    -> Result<PaymentResponse, PaymentError> {
    // Implementation
}
```

### Changelog Updates
If your PR is user-facing, update `CHANGELOG.md`:
```markdown
## [Unreleased]

### Added
- New payment retry logic for failed transactions

### Fixed
- Bug in card validation for certain card types

### Changed
- Updated connector API to v2.0
```

## Integration with External Services

When adding connector integrations:

1. **Implement Trait**: Create connector struct implementing `PaymentConnector`
2. **Add Connector Config**: Add configuration in `config/`
3. **Update Feature Flag**: Add feature for conditional compilation
4. **Add Tests**: Test with sandbox credentials
5. **Document**: Add connector documentation
6. **Update Template**: Reference `connector-template/` for new connectors

## Security Considerations

### No Hardcoded Secrets
- ❌ Never commit API keys, passwords, or tokens
- ✅ Use environment variables
- ✅ Use Vault/KMS for production

### PII Protection
- Use `masking` crate for card numbers, SSNs
- Never log sensitive data
- Sanitize error messages

### Error Handling
- Don't expose internal details in error responses
- Use `error-stack` for error context
- Log errors server-side, not to client

## Performance Guidelines

### Optimization
- Profile before optimizing
- Use benchmarks to measure improvements
- Keep it simple when performance sufficient

### Database
- Add indexes for frequently queried columns
- Write efficient queries
- Test with realistic data volumes

### Caching
- Use Redis for frequently accessed data
- Set appropriate TTLs
- Invalidate cache on updates

## Common Workflows

### Adding a New Connector
1. Create connector struct in `hyperswitch_connectors/src/connectors/`
2. Implement `PaymentConnector` trait
3. Add configuration template in `config/`
4. Add feature flag in `Cargo.toml`
5. Add tests with sandbox credentials
6. Add documentation
7. Update OpenAPI spec

### Adding a Database Field
1. Create migration: `diesel migration generate add_field_name`
2. Write SQL in `migrations/*/up.sql` and `down.sql`
3. Update Diesel model in `diesel_models/`
4. Update API models if exposed
5. Run `just migrate`
6. Test serialization/deserialization
7. Write tests

### Adding an API Endpoint
1. Define request/response in `api_models/src/`
2. Add handler in `router/src/routes/`
3. Add business logic in `router/src/services/`
4. Update error handling
5. Add tests (unit + integration)
6. Document endpoint
7. Update OpenAPI spec

## Getting Help

- **Slack**: https://inviter.co/hyperswitch-slack
- **Discord**: https://discord.gg/wJZ7DVW8mm
- **GitHub Discussions**: https://github.com/juspay/hyperswitch/discussions
- **Issues**: https://github.com/juspay/hyperswitch/issues

## Recognition

All contributors are credited in:
- GitHub contributors page
- Release notes
- CHANGELOG.md (for significant contributions)

Join the community and help shape the future of payment infrastructure! 🚀
