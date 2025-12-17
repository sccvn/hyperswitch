# Hyperswitch Code Style & Conventions

## Rust Code Style

### Formatting
- **Tool**: `cargo +nightly fmt` (uses unstable features)
- **Configuration**: `.rustfmt.toml` in project root
- **Line Length**: Standard Rust conventions (default ~100 chars based on rustfmt)

### Clippy Linting
- **Lint Level**: `-D warnings` (deny all warnings in clippy)
- **Configuration**: `.clippy.toml` in project root
- **Disabled Lints**: 
  - `option_map_unit_fn`: Allowed in workspace
  - `todo`: Allowed (but warned in code reviews)
  - `diverging_sub_expression`: Allowed for V2

### Edition & Rust Version
- **Edition**: 2021
- **Minimum Rust Version**: 1.85.0
- **Nightly Toolchain**: Required for formatting only (`cargo +nightly fmt`)

## Unsafe Code
- **Policy**: Forbidden (`unsafe_code = "forbid"`)
- No unsafe blocks allowed unless explicitly discussed and justified

## Linting Rules (Clippy)

### Warnings
```
as_conversions
cloned_instead_of_copied
dbg_macro (debug prints should use tracing::debug!)
expect_used (prefer ? operator or Result handling)
fn_params_excessive_bools (max 3 bool params)
index_refutable_slice
indexing_slicing
large_futures
missing_panics_doc
mod_module_files
out_of_bounds_indexing
panic (use error-stack or thiserror)
panic_in_result_fn
panicking_unwrap (use ? operator)
print_stderr / print_stdout (use tracing::* instead)
todo (document why)
trivially_copy_pass_by_ref
unimplemented (avoid)
unnecessary_self_imports
unreachable
unwrap_in_result_fn (use ? operator)
unwrap_used (use ? operator)
use_self (use Self in impl blocks)
wildcard_dependencies
```

## Naming Conventions

### Files & Modules
- **snake_case** for file names (e.g., `payment_methods.rs`)
- **One concept per file**: Keep files focused
- **Module organization**: By feature or functional domain

### Variables & Functions
- **snake_case** for variables and functions
- **Descriptive names**: Avoid abbreviations
- **Verb-based functions**: `process_payment()`, `validate_input()`, `is_active()`

### Constants
- **SCREAMING_SNAKE_CASE** for constants
- **Module-level**: Define close to usage
- **Avoid magic numbers**: Use named constants instead

### Traits & Types
- **PascalCase** for traits, structs, enums, types
- **Generic type parameters**: Single uppercase letters or descriptive names (T, E, S, Req, Res)

### Booleans
- **Prefix with verbs**: `is_active`, `has_value`, `should_retry`
- **Avoid negation in names**: `is_valid` not `is_not_invalid`

## Error Handling

### Error Strategy
- **Primary Pattern**: `error-stack` crate for rich error context
- **Custom Errors**: Use `thiserror` derive macros
- **Result Type**: Use `Result<T, Error>` (from error-stack)
- **Never Panic in Production**: Use proper error handling
- **Forbidden**: `unwrap()`, `expect()`, `panic!()` in business logic

### Example Pattern
```rust
use error_stack::{Result, Report};
use thiserror::Error;

#[derive(Error, Debug)]
pub enum PaymentError {
    #[error("Invalid payment amount")]
    InvalidAmount,
    #[error("Processor unavailable")]
    ProcessorError,
}

pub fn process_payment(amount: u64) -> Result<PaymentId, PaymentError> {
    // Return errors with context
    if amount == 0 {
        return Err(Report::new(PaymentError::InvalidAmount));
    }
    Ok(payment_id)
}
```

## Async/Await

### Async Functions
- Use `async` keyword for I/O-bound operations
- **Async Traits**: Use `#[async_trait]` from `async-trait` crate
- **Tokio Integration**: Use tokio utilities for concurrency

### Example
```rust
#[async_trait]
pub trait PaymentProcessor {
    async fn process(&self, payment: Payment) -> Result<PaymentResponse>;
}

pub async fn handle_payment_request(req: PaymentRequest) -> Result<PaymentResponse> {
    let payment = validate_payment(&req)?;
    processor.process(payment).await
}
```

## Documentation

### Doc Comments
- **Module-level**: `//!` comments at top of file
- **Public items**: `///` comments for all public types, traits, functions
- **Visibility**: Document public API completely

### Example
```rust
/// Processes a payment transaction.
///
/// This function validates the payment amount, routes it to the appropriate
/// processor, and returns the transaction result.
///
/// # Arguments
///
/// * `payment` - The payment details to process
///
/// # Returns
///
/// Returns a `PaymentResponse` on success or a `PaymentError` on failure.
///
/// # Errors
///
/// Returns `PaymentError::InvalidAmount` if the amount is zero or negative.
pub async fn process_payment(payment: Payment) -> Result<PaymentResponse, PaymentError> {
    // ...
}
```

## Testing

### Test Organization
- **Unit Tests**: In same file with `#[cfg(test)]` module
- **Integration Tests**: In `tests/` directory
- **Test Naming**: `test_<function>_<scenario>_<expected_result>`

### Example
```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn test_validate_amount_with_zero_should_fail() {
        let result = validate_amount(0);
        assert!(result.is_err());
    }

    #[tokio::test]
    async fn test_process_payment_with_valid_data_should_succeed() {
        let payment = create_test_payment();
        let result = process_payment(payment).await;
        assert!(result.is_ok());
    }
}
```

## Feature Flags

### Feature Organization
- **Mutually Exclusive**: `v1` and `v2` can't both be enabled
- **Dependent Features**: Declare dependencies in `Cargo.toml` under `[features]`
- **Compile Optimization**: Use features for optional functionality

### Example
```toml
[features]
default = ["v1", "stripe"]
v1 = ["dependency_v1"]
v2 = ["dependency_v2"]
release = ["stripe", "email", "aws_kms"]
```

## Trait Design

### Common Patterns
- **Extensibility**: Use traits for connectors, storage, auth
- **Dependency Injection**: Accept trait objects rather than concrete types
- **Builder Pattern**: For complex object construction

### Example
```rust
pub trait PaymentConnector: Send + Sync {
    async fn authorize(&self, request: AuthRequest) -> Result<AuthResponse>;
    async fn capture(&self, request: CaptureRequest) -> Result<CaptureResponse>;
}

pub struct PaymentRouter {
    connectors: HashMap<String, Arc<dyn PaymentConnector>>,
}
```

## Database Models

### Diesel Models
- **Location**: `diesel_models/` crate
- **Naming**: Model names match table names (snake_case in DB)
- **Struct Naming**: Singular form (Payment not Payments)

### Example
```rust
// In diesel_models/src/payment.rs
#[derive(Queryable, Insertable, Selectable)]
#[diesel(table_name = payments)]
pub struct Payment {
    pub payment_id: String,
    pub amount: i64,
    pub status: PaymentStatus,
}

#[derive(Insertable)]
#[diesel(table_name = payments)]
pub struct NewPayment {
    pub payment_id: String,
    pub amount: i64,
}
```

## Serialization

### Serde
- **Derives**: `#[derive(Serialize, Deserialize)]`
- **Custom Fields**: Use `#[serde(rename)]` for API compatibility
- **Defaults**: Use `#[serde(default)]` for optional fields

### Example
```rust
#[derive(Serialize, Deserialize, Debug)]
pub struct PaymentRequest {
    #[serde(rename = "payment_id")]
    pub id: String,
    
    #[serde(default)]
    pub metadata: Option<Map<String, Value>>,
    
    #[serde(skip_serializing_if = "Option::is_none")]
    pub description: Option<String>,
}
```

## Type Safety

### Newtype Pattern
- Use for domain concepts (avoid primitive type confusion)
- Provides type safety without runtime cost

### Example
```rust
#[derive(Clone, Debug)]
pub struct PaymentId(String);

#[derive(Clone, Debug)]
pub struct CustomerId(String);

// Type system prevents confusing payment_id with customer_id
fn charge_customer(customer: CustomerId, payment: PaymentId) { }
```

## Common Patterns

### Result Handling
```rust
// Prefer this
fn operation() -> Result<Value> {
    let x = risky_call()?;
    Ok(x)
}

// Over panic
fn operation_bad() -> Value {
    risky_call().unwrap()
}
```

### Optional Handling
```rust
// Use and_then for chaining
value.and_then(|v| other_operation(v))

// Use map for transformations
value.map(|v| v * 2)

// Use ok_or for conversion
optional_value.ok_or(some_error)
```

### Logging
```rust
// Use tracing macros, not println!
tracing::debug!("Processing payment: {:?}", payment);
tracing::info!("Payment processed: {}", payment_id);
tracing::warn!("Retry attempt {}: {}", attempt, reason);
tracing::error!("Payment failed: {}", error);
```

## Pre-commit Checklist

Before committing code:
1. ✅ Run `just precommit` (format + clippy)
2. ✅ All tests pass: `cargo test --all-features`
3. ✅ No `TODO`, `unwrap()`, or `panic!()` without documentation
4. ✅ All public APIs have doc comments
5. ✅ Error handling uses `error-stack` or `thiserror`
6. ✅ No hardcoded secrets or configuration
