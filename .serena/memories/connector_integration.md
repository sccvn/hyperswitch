# Hyperswitch Connector Integration Guide

## What is a Connector?

A **Connector** is an integration with a payment processor (like Stripe, Adyen, PayPal, etc.). Each connector implements the `PaymentConnector` trait to handle payment operations.

## Supported Connectors (Examples)

The project integrates with 100+ payment processors:
- **Credit Cards**: Stripe, Adyen, Square, Paypal, Worldpay
- **Wallets**: Apple Pay, Google Pay, Samsung Pay
- **Local Methods**: UPI, BNPL (Klarna), Bank transfers
- **Regional**: Various country-specific processors
- **Dummy**: Test connector for development

## Connector Architecture

### Trait-Based Design
```rust
#[async_trait]
pub trait PaymentConnector: Send + Sync {
    async fn authorize(&self, auth_request: AuthRequest) -> Result<AuthResponse>;
    async fn capture(&self, capture_request: CaptureRequest) -> Result<CaptureResponse>;
    async fn void(&self, void_request: VoidRequest) -> Result<VoidResponse>;
    async fn refund(&self, refund_request: RefundRequest) -> Result<RefundResponse>;
    async fn psync(&self, psync_request: PsyncRequest) -> Result<PsyncResponse>;
}
```

### Directory Structure
```
hyperswitch_connectors/
├── src/
│   ├── connectors/
│   │   ├── stripe.rs                # Stripe implementation
│   │   ├── adyen.rs                 # Adyen implementation
│   │   ├── paypal.rs                # PayPal implementation
│   │   └── [80+ more connectors]
│   ├── core.rs                      # Core connector logic
│   ├── errors.rs                    # Connector errors
│   └── lib.rs
├── tests/
│   └── connectors/
│       ├── stripe.rs
│       └── [tests for each connector]
```

## Creating a New Connector

### Step 1: Create Connector Module
```rust
// hyperswitch_connectors/src/connectors/new_processor.rs

use async_trait::async_trait;
use serde::{Deserialize, Serialize};

#[derive(Debug, Clone)]
pub struct NewProcessor;

#[async_trait]
impl PaymentConnector for NewProcessor {
    async fn authorize(
        &self,
        auth_request: AuthRequest,
    ) -> CustomResult<AuthResponse, errors::ConnectorError> {
        // Implementation
    }

    async fn capture(
        &self,
        capture_request: CaptureRequest,
    ) -> CustomResult<CaptureResponse, errors::ConnectorError> {
        // Implementation
    }
    
    // Implement other methods...
}
```

### Step 2: Define Request/Response Types
```rust
#[derive(Debug, Serialize, Deserialize)]
pub struct NewProcessorAuthRequest {
    pub amount: u64,
    pub currency: String,
    pub card: Card,
    pub merchant_reference: String,
}

#[derive(Debug, Serialize, Deserialize)]
pub struct NewProcessorAuthResponse {
    pub transaction_id: String,
    pub status: TransactionStatus,
    pub message: Option<String>,
}
```

### Step 3: Add to Connector Registry
```rust
// hyperswitch_connectors/src/connectors/mod.rs

pub mod new_processor;

// In register_connector function:
match connector_name {
    "new_processor" => Box::new(new_processor::NewProcessor),
    // ... other connectors
}
```

### Step 4: Add Feature Flag
```toml
# Cargo.toml
[features]
new_processor = []
default = ["stripe", "adyen", "new_processor"]
```

### Step 5: Add Configuration
```toml
# config/development.toml

[connectors.new_processor]
api_key = "sk_test_..."
api_url = "https://api.newprocessor.com"
secondary_api_url = "https://secondary.newprocessor.com"
api_version = "2023-01-01"
```

## Common Implementation Patterns

### HTTP Request Building
```rust
impl NewProcessor {
    fn build_authorize_request(
        &self,
        auth_request: AuthRequest,
        _conf: &Self::Config,
    ) -> CustomResult<Request, errors::ConnectorError> {
        let url = self.get_auth_url()?;
        let body = NewProcessorAuthRequest::from(auth_request);
        
        RequestBuilder::new()
            .url(&url)
            .method(Method::Post)
            .attach_default_headers()
            .json_body(body)
            .build()
    }
}
```

### Response Handling
```rust
impl NewProcessor {
    fn handle_authorize_response(
        &self,
        resp: Response,
    ) -> CustomResult<AuthResponse, errors::ConnectorError> {
        let response: NewProcessorAuthResponse = resp.response.parse_struct("NewProcessorAuthResponse")?;
        
        Ok(AuthResponse {
            status: response.status.into(),
            connector_transaction_id: response.transaction_id,
            auth_type: None,
            connector_meta_data: None,
            connector_wallets_details: None,
        })
    }
}
```

### Error Mapping
```rust
fn map_error(
    &self,
    error: NewProcessorErrorResponse,
) -> errors::ConnectorError {
    match error.error_code {
        "INVALID_REQUEST" => {
            errors::ConnectorError::InvalidRequestData {
                message: error.message,
                connector: "NewProcessor",
            }
        }
        "AUTH_FAILED" => {
            errors::ConnectorError::AuthenticationFailed
        }
        _ => errors::ConnectorError::UnknownError,
    }
}
```

## Webhook Handling

### Implement Webhook Trait
```rust
#[async_trait]
impl WebhookConnector for NewProcessor {
    fn verify_webhook_signature(
        &self,
        body: &[u8],
        signature: &str,
        secret: &str,
    ) -> CustomResult<bool, errors::ConnectorError> {
        // Verify HMAC signature
        let expected_signature = hmac::sign(body, secret);
        Ok(signature == expected_signature)
    }

    fn get_webhook_resource_object(
        &self,
        body: &[u8],
    ) -> CustomResult<Box<dyn WebhookResourceObject>, errors::ConnectorError> {
        let webhook: NewProcessorWebhook = serde_json::from_slice(body)?;
        Ok(Box::new(webhook))
    }
}
```

## Testing Connectors

### Unit Tests
```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn test_build_authorize_request() {
        let auth_request = create_test_auth_request();
        let connector = NewProcessor;
        let req = connector.build_authorize_request(auth_request, &config)
            .expect("Failed to build request");
        
        assert_eq!(req.method, Method::Post);
        assert!(req.url.contains("authorize"));
    }

    #[tokio::test]
    async fn test_handle_authorize_response() {
        let response_body = r#"{
            "transaction_id": "12345",
            "status": "success"
        }"#;
        
        let connector = NewProcessor;
        let response = connector.handle_authorize_response(response_body)
            .expect("Failed to parse response");
        
        assert_eq!(response.connector_transaction_id, "12345");
    }
}
```

### Integration Tests
```rust
#[tokio::test]
async fn test_authorize_success() {
    let processor = NewProcessor::new();
    let auth_request = AuthRequest {
        amount: 1000,
        currency: Currency::USD,
        card: test_card(),
        merchant_reference: "test_123".to_string(),
    };
    
    let response = processor.authorize(&auth_request)
        .await
        .expect("Authorization failed");
    
    assert_eq!(response.status, TransactionStatus::Authorized);
}
```

## Connector Configuration

### Environment Variables
```bash
# Set API credentials
export NEW_PROCESSOR_API_KEY="sk_test_..."
export NEW_PROCESSOR_API_URL="https://api.newprocessor.com"
export NEW_PROCESSOR_WEBHOOK_SECRET="secret123"
```

### Configuration File
```toml
# config/production.toml
[connectors.new_processor]
api_key = "${NEW_PROCESSOR_API_KEY}"
api_url = "${NEW_PROCESSOR_API_URL}"
webhook_secret = "${NEW_PROCESSOR_WEBHOOK_SECRET}"
timeout = 30  # seconds
retry_attempts = 3
```

## Common Connector Operations

### Payment Authorization
- Validates payment details
- Authorizes amount with processor
- Returns transaction ID and status

### Payment Capture
- Captures previously authorized payment
- May be partial or full amount
- Returns confirmation

### Payment Void
- Cancels authorized but uncaptured payment
- Available within time window (typically 24 hours)

### Refund
- Returns funds to customer
- Can be partial or full
- Requires captured payment

### Periodic Sync (PSYnc)
- Queries connector for current payment status
- Used for reconciliation
- Handles webhook losses

## Adding Connector Support for New Payment Method

### Card Payments
```rust
pub struct CardAuth {
    pub card_number: String,
    pub expiry_month: String,
    pub expiry_year: String,
    pub cvv: String,
}
```

### Wallets
```rust
pub struct WalletAuth {
    pub wallet_type: WalletType,  // Apple Pay, Google Pay, etc.
    pub token: String,
}
```

### Bank Transfers
```rust
pub struct BankTransferAuth {
    pub account_number: String,
    pub bank_code: String,
    pub account_holder_name: String,
}
```

## Error Handling Best Practices

1. **Map Processor Errors to ConnectorError**
   ```rust
   match processor_error.code {
       "INSUFFICIENT_FUNDS" => ConnectorError::ProcessorUnavailable,
       "DECLINED" => ConnectorError::TransactionFailed,
       _ => ConnectorError::UnknownError,
   }
   ```

2. **Preserve Error Context**
   - Log full error response for debugging
   - Return sanitized error to client

3. **Retry Logic**
   - Idempotent operations (use merchant_reference)
   - Exponential backoff for rate limits
   - Max 3 retries typically

## Connector Documentation Requirements

1. **README.md** in connector module
   - API version supported
   - Supported payment methods
   - Configuration requirements
   - Testing instructions

2. **API Reference Link**
   - Link to processor API documentation
   - Authentication method
   - Sandbox URL and test credentials

3. **Known Limitations**
   - Unsupported features
   - Regional restrictions
   - Rate limits

## Useful Resources

- **Connector Template**: `connector-template/` directory
- **Example Connector**: `hyperswitch_connectors/src/connectors/stripe.rs`
- **Test Utilities**: `test_utils/` crate for helpers
- **API Documentation**: Each processor's API docs

## Integration Checklist

Before submitting PR:

- ✅ Connector implements `PaymentConnector` trait
- ✅ All payment methods tested (Auth, Capture, Refund, etc.)
- ✅ Error handling for all response scenarios
- ✅ Request/response types properly serialized
- ✅ Configuration documented
- ✅ Unit tests (80%+ coverage)
- ✅ Integration tests with sandbox credentials
- ✅ Webhook handling tested
- ✅ Documentation updated
- ✅ Feature flag added
- ✅ Pre-commit checks pass: `just precommit`

## Connector Maintenance

### Regular Updates
- Monitor API changes from processor
- Update error handling for new error codes
- Keep test credentials valid in sandbox

### Performance Monitoring
- Track connector response times
- Monitor success rates
- Alert on threshold changes

### Security
- Rotate API keys regularly
- Never log sensitive data
- Use secure configurations in production
