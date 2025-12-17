---
language: "Rust"
edition: "2021"
min_version: "1.85.0"
framework: "Actix-web"
orm: "Diesel"
use_case: "Hyperswitch payment platform core development"
---

# Rust Core Development Prompt

## Language Context

You are developing **Hyperswitch**, a payment orchestration platform written in Rust. The codebase follows Rust best practices, emphasizes type safety, uses async/await extensively, and prioritizes performance and correctness.

## Core Principles

### 1. Type Safety & Zero-Cost Abstractions
```rust
// Use newtypes for domain concepts
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct PaymentId(String);

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct MerchantId(String);

// Compiler prevents mixing different ID types
fn find_payment(payment_id: PaymentId) -> Result<Payment> {
    // Cannot accidentally pass MerchantId here
}
```

### 2. Error Handling with error-stack
```rust
use error_stack::{Result, ResultExt};

pub fn process_payment(
    payment_data: PaymentData
) -> Result<PaymentResponse, errors::PaymentError> {
    let validated = payment_data
        .validate()
        .change_context(errors::PaymentError::ValidationFailed)
        .attach_printable("Payment validation failed")?;
    
    let connector = select_connector(&validated)
        .change_context(errors::PaymentError::ConnectorSelectionFailed)
        .attach_printable_lazy(|| format!("No connector available for {:?}", validated))?;
    
    connector
        .authorize(validated)
        .await
        .change_context(errors::PaymentError::ProcessingFailed)
}
```

### 3. Async/Await with Tokio
```rust
use tokio::join;

#[async_trait]
pub trait PaymentProcessor {
    async fn authorize(&self, req: PaymentRequest) -> Result<PaymentResponse>;
    async fn capture(&self, payment_id: &str) -> Result<CaptureResponse>;
}

// Parallel operations
async fn load_payment_context(payment_id: &str) -> Result<PaymentContext> {
    let (payment, merchant, customer) = tokio::try_join!(
        db.find_payment(payment_id),
        db.find_merchant_by_payment(payment_id),
        db.find_customer_by_payment(payment_id),
    )?;
    
    Ok(PaymentContext { payment, merchant, customer })
}
```

### 4. Traits for Extensibility
```rust
// Define behavior contracts
pub trait Connector: Send + Sync {
    fn name(&self) -> &'static str;
    fn supports_payment_method(&self, pm: &PaymentMethodType) -> bool;
}

pub trait PaymentConnector: Connector {
    async fn authorize(&self, req: &AuthorizeRequest) -> ConnectorResult<AuthorizeResponse>;
    async fn capture(&self, req: &CaptureRequest) -> ConnectorResult<CaptureResponse>;
    async fn void(&self, req: &VoidRequest) -> ConnectorResult<VoidResponse>;
}

// Implement for each connector
pub struct Stripe {
    api_key: Secret<String>,
    base_url: String,
}

#[async_trait]
impl PaymentConnector for Stripe {
    async fn authorize(&self, req: &AuthorizeRequest) -> ConnectorResult<AuthorizeResponse> {
        // Stripe-specific implementation
    }
}
```

## Hyperswitch-Specific Patterns

### 1. Connector Integration Pattern
```rust
// crates/hyperswitch_connectors/src/connectors/example.rs

use hyperswitch_interfaces::api::{self, ConnectorIntegration};
use hyperswitch_domain_models::router_data::RouterData;

pub struct ExampleConnector;

impl<Flow, Request, Response> ConnectorIntegration<Flow, Request, Response> 
for ExampleConnector 
where
    Self: ConnectorIntegrationAny<Flow, Request, Response>,
{
    fn get_headers(
        &self,
        req: &RouterData<Flow, Request, Response>,
        _connectors: &Connectors,
    ) -> CustomResult<Vec<(String, Maskable<String>)>, errors::ConnectorError> {
        let mut headers = vec![
            ("Content-Type".to_string(), "application/json".to_string().into()),
        ];
        
        let auth = ExampleAuth::try_from(&req.connector_auth_type)?;
        headers.push(("Authorization".to_string(), format!("Bearer {}", auth.api_key).into()));
        
        Ok(headers)
    }

    fn get_url(
        &self,
        req: &RouterData<Flow, Request, Response>,
        _connectors: &Connectors,
    ) -> CustomResult<String, errors::ConnectorError> {
        Ok(format!("{}/payments", self.base_url(&req.connector)))
    }

    fn build_request(
        &self,
        req: &RouterData<Flow, Request, Response>,
        _connectors: &Connectors,
    ) -> CustomResult<Option<Request>, errors::ConnectorError> {
        let connector_req = ExamplePaymentRequest::try_from(req)?;
        
        Ok(Some(
            RequestBuilder::new()
                .method(Method::Post)
                .url(&self.get_url(req, _connectors)?)
                .attach_default_headers()
                .headers(self.get_headers(req, _connectors)?)
                .body(RequestContent::Json(Box::new(connector_req)))
                .build(),
        ))
    }

    fn handle_response(
        &self,
        data: &RouterData<Flow, Request, Response>,
        res: HttpResponse,
    ) -> CustomResult<RouterData<Flow, Request, Response>, errors::ConnectorError> {
        let response: ExamplePaymentResponse = res
            .response
            .parse_struct("ExamplePaymentResponse")
            .change_context(errors::ConnectorError::ResponseDeserializationFailed)?;

        RouterData::try_from(ResponseRouterData {
            response,
            data: data.clone(),
            http_code: res.status_code,
        })
    }
}
```

### 2. Database Access with Diesel
```rust
// crates/diesel_models/src/payment_intent.rs

use diesel::{Insertable, Queryable, Identifiable};
use crate::schema::payment_intent;

#[derive(Clone, Debug, Identifiable, Queryable)]
#[diesel(table_name = payment_intent)]
pub struct PaymentIntent {
    #[diesel(column_name = payment_id)]
    pub id: String,
    pub merchant_id: String,
    pub status: PaymentStatus,
    pub amount: i64,
    pub currency: Currency,
    pub created_at: time::PrimitiveDateTime,
    pub modified_at: time::PrimitiveDateTime,
}

#[derive(Clone, Debug, Insertable)]
#[diesel(table_name = payment_intent)]
pub struct PaymentIntentNew {
    pub payment_id: String,
    pub merchant_id: String,
    pub status: PaymentStatus,
    pub amount: i64,
    pub currency: Currency,
}

// Repository implementation
// crates/storage_impl/src/payments.rs

#[async_trait]
impl PaymentIntentInterface for Store {
    async fn insert_payment_intent(
        &self,
        payment: PaymentIntentNew,
    ) -> StorageResult<PaymentIntent> {
        let conn = self.get_conn().await?;
        
        conn.transaction_async(|conn| async move {
            diesel::insert_into(payment_intent::table)
                .values(&payment)
                .get_result::<PaymentIntent>(conn)
                .await
        })
        .await
        .map_err(|e| StorageError::DatabaseError(e.into()))
    }

    async fn find_payment_intent_by_id(
        &self,
        payment_id: &str,
    ) -> StorageResult<PaymentIntent> {
        let conn = self.get_conn().await?;
        
        payment_intent::table
            .filter(payment_intent::payment_id.eq(payment_id))
            .first::<PaymentIntent>(&conn)
            .await
            .map_err(|e| match e {
                diesel::result::Error::NotFound => StorageError::ValueNotFound,
                e => StorageError::DatabaseError(e.into()),
            })
    }
}
```

### 3. API Handler Pattern (Actix-web)
```rust
// crates/router/src/routes/payments.rs

use actix_web::{web, HttpRequest, HttpResponse};
use api_models::payments::{PaymentsCreateRequest, PaymentsResponse};

pub async fn payments_create(
    state: web::Data<AppState>,
    req: HttpRequest,
    json_payload: web::Json<PaymentsCreateRequest>,
) -> HttpResponse {
    let flow = Flow::PaymentsCreate;
    
    Box::pin(api::server_wrap(
        flow,
        state.into_inner(),
        &req,
        json_payload.into_inner(),
        |state, auth, req| {
            payments::payments_create_core(state, auth.merchant_account, req)
        },
        &auth::ApiKeyAuth,
    ))
    .await
}

// Core business logic
// crates/router/src/core/payments.rs

pub async fn payments_create_core(
    state: AppState,
    merchant_account: MerchantAccount,
    req: PaymentsCreateRequest,
) -> RouterResponse<PaymentsResponse> {
    // Validate request
    let validated_req = req.validate()?;
    
    // Create payment intent
    let payment = domain::PaymentIntent::create_from_request(
        &merchant_account,
        validated_req,
    )?;
    
    // Store in database
    let stored_payment = state
        .store
        .insert_payment_intent(payment.clone())
        .await?;
    
    // Select connector
    let connector = routing::select_connector(&state, &stored_payment).await?;
    
    // Process payment
    let result = connector
        .authorize(AuthorizeRequest::from(stored_payment))
        .await?;
    
    // Update payment status
    state
        .store
        .update_payment_intent(
            &payment.id,
            PaymentIntentUpdate::StatusUpdate {
                status: result.status,
                connector_payment_id: result.connector_payment_id,
            },
        )
        .await?;
    
    Ok(api::ApplicationResponse::Json(
        PaymentsResponse::from(result)
    ))
}
```

### 4. Testing Patterns
```rust
// Unit tests
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn test_payment_id_generation() {
        let id = PaymentId::generate();
        assert_eq!(id.0.len(), 32);
    }

    #[tokio::test]
    async fn test_connector_selection() {
        let state = mock_app_state();
        let payment = mock_payment_intent();
        
        let connector = routing::select_connector(&state, &payment)
            .await
            .unwrap();
        
        assert_eq!(connector.name(), "stripe");
    }
}

// Integration tests
// tests/integration/payments_test.rs

#[actix_web::test]
async fn test_payment_creation_flow() {
    let app = spawn_test_app().await;
    let client = reqwest::Client::new();
    
    let response = client
        .post(&format!("{}/payments", app.address))
        .header("api-key", &app.test_merchant.api_key)
        .json(&json!({
            "amount": 1000,
            "currency": "USD",
            "payment_method": "card",
            "payment_method_data": {
                "card": {
                    "number": "4242424242424242",
                    "exp_month": "12",
                    "exp_year": "2025",
                    "cvc": "123"
                }
            }
        }))
        .send()
        .await
        .expect("Failed to execute request");
    
    assert_eq!(response.status(), 201);
    
    let body: PaymentsResponse = response
        .json()
        .await
        .expect("Failed to parse response");
    
    assert_eq!(body.status, PaymentStatus::Succeeded);
}
```

## Code Quality Standards

### 1. Clippy Configuration
```toml
# Cargo.toml
[lints.clippy]
# Deny
unwrap_used = "deny"
expect_used = "deny"
panic = "deny"
todo = "deny"

# Warn
missing_docs_in_private_items = "warn"
```

### 2. Running Checks
```bash
# Format
cargo +nightly fmt

# Lint
cargo clippy --all-features --all-targets -- -D warnings

# Test
cargo nextest run
cargo test --doc

# Check all features
cargo check --all-features
cargo check --no-default-features
```

### 3. Documentation
```rust
/// Creates a new payment intent for the given merchant.
///
/// # Arguments
///
/// * `merchant_account` - The merchant account creating the payment
/// * `request` - The payment creation request
///
/// # Returns
///
/// Returns a `PaymentIntent` on success, or a `PaymentError` on failure.
///
/// # Errors
///
/// This function will return an error if:
/// - The request validation fails
/// - The database operation fails
/// - The merchant is not authorized
///
/// # Example
///
/// ```rust
/// let payment = create_payment_intent(merchant, request).await?;
/// assert_eq!(payment.status, PaymentStatus::Created);
/// ```
pub async fn create_payment_intent(
    merchant_account: MerchantAccount,
    request: PaymentsCreateRequest,
) -> Result<PaymentIntent, PaymentError> {
    // Implementation
}
```

## Performance Considerations

### 1. Avoid Unnecessary Clones
```rust
// Bad
fn process(data: Vec<Payment>) -> Vec<PaymentResponse> {
    data.into_iter()
        .map(|payment| transform(payment.clone())) // Unnecessary clone
        .collect()
}

// Good
fn process(data: Vec<Payment>) -> Vec<PaymentResponse> {
    data.into_iter()
        .map(transform) // Consume by value
        .collect()
}
```

### 2. Use References Where Possible
```rust
// Good - borrows instead of taking ownership
fn validate_payment(payment: &Payment) -> Result<(), ValidationError> {
    if payment.amount <= 0 {
        return Err(ValidationError::InvalidAmount);
    }
    Ok(())
}
```

### 3. Batch Database Operations
```rust
// Bad - N+1 queries
for payment_id in payment_ids {
    let payment = db.find_payment(&payment_id).await?;
    payments.push(payment);
}

// Good - Single query
let payments = db
    .find_payments_by_ids(&payment_ids)
    .await?;
```

## Security Practices

### 1. Use masking Crate for PII
```rust
use masking::Secret;

pub struct CardData {
    pub number: Secret<String>,
    pub cvv: Secret<String>,
    pub exp_month: String,
    pub exp_year: String,
}

// Logs as: CardData { number: "***", cvv: "***", ... }
```

### 2. Constant-Time Comparisons
```rust
use subtle::ConstantTimeEq;

fn verify_api_key(provided: &str, expected: &str) -> bool {
    provided.as_bytes()
        .ct_eq(expected.as_bytes())
        .into()
}
```

---

**Version**: 1.0.0  
**Last Updated**: 2025-12-17  
**Applies to**: Hyperswitch v1.x, v2.x
