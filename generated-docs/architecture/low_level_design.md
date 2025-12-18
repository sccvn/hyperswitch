# Hyperswitch Low-Level Architecture

**Date**: December 18, 2025  
**Version**: 1.0.0  
**Status**: Extracted from codebase analysis

---

## Table of Contents

1. [Executive Summary](#executive-summary)
2. [Crate Architecture](#crate-architecture)
3. [Core Components](#core-components)
4. [Payment Flow Architecture](#payment-flow-architecture)
5. [Connector Integration Architecture](#connector-integration-architecture)
6. [Storage Architecture](#storage-architecture)
7. [Routing Engine Architecture](#routing-engine-architecture)
8. [Security Architecture](#security-architecture)
9. [Performance Considerations](#performance-considerations)
10. [References](#references)

---

## Executive Summary

Hyperswitch is a payment orchestration platform built in Rust, designed as a high-performance, extensible payment router supporting 100+ payment processors. The architecture follows a modular, crate-based design with clear separation of concerns.

### Key Architectural Characteristics

- **Language**: Rust (Edition 2021, ≥1.85.0)
- **Web Framework**: Actix-web 4.11.0 (async, actor-based)
- **Database**: PostgreSQL with Diesel ORM 2.2.10
- **Cache**: Redis for session management and configuration
- **Async Runtime**: Tokio (multi-threaded)
- **Architecture Style**: Layered + Hexagonal (Ports & Adapters)
- **Workspace**: 38 crates with explicit dependencies

### Performance Targets

- **Latency**: P95 < 500ms (including connector calls)
- **Throughput**: 10,000+ requests/second per instance
- **Availability**: 99.99% uptime SLA
- **Scalability**: Horizontal scaling via stateless design

---

## Crate Architecture

Hyperswitch follows a workspace-based architecture with 38 specialized crates organized into logical layers.

See [generated-docs/plantuml/c4_component_router_detailed.puml](../plantuml/c4_component_router_detailed.puml) for visual representation.

### Layer Organization

```
┌─────────────────────────────────────────────────┐
│           API Layer (router, api_models)        │
├─────────────────────────────────────────────────┤
│     Domain Layer (hyperswitch_domain_models)    │
├─────────────────────────────────────────────────┤
│  Connector Layer (hyperswitch_connectors, interfaces) │
├─────────────────────────────────────────────────┤
│    Storage Layer (diesel_models, storage_impl)  │
├─────────────────────────────────────────────────┤
│   Infrastructure (redis, external_services)     │
└─────────────────────────────────────────────────┘
```

---

## Core Components

### 1. Router (Main Server)

**Location**: `crates/router/src/`  
**Binary**: `crates/router/src/bin/router.rs`

The router is the main HTTP server handling all payment APIs.

#### Key Modules

**routes/** - HTTP endpoint handlers
- `payments.rs` - Payment CRUD operations
- `refunds.rs` - Refund operations
- `customers.rs` - Customer management
- `payment_methods.rs` - Payment method management
- `webhooks.rs` - Webhook ingestion

**core/** - Business logic orchestration
- `payments.rs` - Payment lifecycle management
- `refunds.rs` - Refund processing
- `routing/` - Connector selection engine
- `fraud_check.rs` - FRM integration
- `authentication.rs` - 3DS authentication

**connector/** - Connector dispatch
- Instantiates connector implementations
- Routes requests to appropriate connector
- Handles response transformations

**db/** - Database access layer
- Repository pattern implementation
- Transaction management
- Connection pooling

### 2. Payment Core Logic

**File**: `crates/router/src/core/payments.rs`

#### Primary Functions

**`payments_core_create()`**
- Creates PaymentIntent with initial status
- Validates merchant account
- Returns client_secret for frontend

**`payments_confirm()`**
- Validates payment_method
- Executes routing algorithm
- Calls connector authorize API
- Handles 3DS redirects
- Updates payment status

**`payments_capture()`**
- Captures authorized funds
- Supports partial capture
- Updates amount_captured

**`payments_cancel()`**
- Voids authorization
- Releases hold on funds

### 3. Routing Engine

**Location**: `crates/router/src/core/routing/`

#### Algorithm Types

**Static Routing** (`static_routing.rs`)
```rust
pub struct StaticRoutingAlgorithm {
    pub rules: Vec<RoutingRule>,
}

impl RoutingAlgorithm for StaticRoutingAlgorithm {
    fn get_connector(
        &self,
        context: &RoutingContext,
    ) -> Result<Vec<Connector>> {
        // Evaluate rules top-to-bottom
        // Return first matching connector(s)
    }
}
```

**Dynamic Routing** (`dynamic_routing.rs`)
- ML-based connector selection
- Success rate optimization
- Latency minimization
- Cost optimization

**Volume Routing** (`volume_routing.rs`)
- Percentage-based distribution
- Load balancing across connectors

#### Routing Context

```rust
pub struct RoutingContext {
    pub merchant_id: String,
    pub payment_method: PaymentMethod,
    pub currency: Currency,
    pub amount: i64,
    pub customer_country: Option<CountryAlpha2>,
}
```

### 4. Connector Integration

**Interface**: `crates/hyperswitch_interfaces/src/connector_integration_interface.rs`

#### Core Trait

```rust
#[async_trait]
pub trait ConnectorIntegration<Flow, Req, Res> {
    fn build_request(...) -> Result<Option<Request>>;
    fn handle_response(...) -> Result<RouterData>;
    fn get_error_response(...) -> Result<ErrorResponse>;
}
```

#### Connector Structure

Each connector (e.g., Stripe) implements:
- **mod.rs** - Main connector logic
- **transformers.rs** - Request/Response transformations
- **tests.rs** - Unit tests

Example: `crates/hyperswitch_connectors/src/connectors/stripe/`

---

## Payment Flow Architecture

### Authorize + Capture Flow

```
┌──────────┐
│  Merchant│
└────┬─────┘
     │ POST /payments
     ▼
┌────────────────┐
│ Create Intent  │ → PaymentIntent(RequiresPaymentMethod)
│ Create Attempt │ → PaymentAttempt(Started)
└────┬───────────┘
     │ POST /payments/:id/confirm
     ▼
┌────────────────┐
│ Routing Engine │ → Select Connector(s)
└────┬───────────┘
     │
     ▼
┌────────────────┐
│ Call Connector │ → Stripe.authorize()
│   (Stripe)     │
└────┬───────────┘
     │
     ├─Success─→ RequiresCapture (manual) or Succeeded (automatic)
     ├─3DS────→ RequiresCustomerAction + redirect_url
     └─Fail───→ Failed, try fallback
     │
     ▼
┌────────────────┐
│Customer Action │ (if 3DS)
└────┬───────────┘
     │ Webhook /webhooks/stripe
     ▼
┌────────────────┐
│ Update Status  │ → Continue to capture
└────┬───────────┘
     │ POST /payments/:id/capture (if manual)
     ▼
┌────────────────┐
│ Capture Funds  │ → Stripe.capture()
│ Update Status  │ → Succeeded
└────────────────┘
```

See [generated-docs/plantuml/sequence_payment_create_detailed.puml](../plantuml/sequence_payment_create_detailed.puml) for detailed sequence diagram.

---

## Connector Integration Architecture

### Adapter Pattern

Every connector implements the adapter pattern to unify disparate payment processor APIs.

```
┌──────────────┐
│ Router Core  │
└──────┬───────┘
       │ RouterData
       ▼
┌──────────────────────┐
│ Connector Interface  │ (Trait)
└──────┬───────────────┘
       │
       ├──→ Stripe Adapter ──→ Stripe API
       ├──→ Adyen Adapter ──→ Adyen API
       ├──→ PayPal Adapter ──→ PayPal API
       └──→ ... (100+ connectors)
```

### Transformation Layer

**transformers.rs** - Bidirectional transformations

```rust
// RouterData → Connector API Request
pub fn to_connector_request(
    router_data: &RouterData,
) -> Result<ConnectorRequest> {
    // Map fields, convert types, apply transformations
}

// Connector API Response → RouterData
pub fn from_connector_response(
    router_data: &RouterData,
    response: ConnectorResponse,
) -> Result<RouterData> {
    // Parse response, map status, extract IDs
}
```

---

## Storage Architecture

### Database Schema

See [generated-docs/plantuml/erd_detailed.puml](../plantuml/erd_detailed.puml) for complete ERD.

#### Core Tables

**payment_intent**
- Primary key: `payment_id` (v1) / `id` (v2)
- Tracks payment lifecycle
- 50+ fields including status, amount, currency

**payment_attempt**
- Primary key: `attempt_id` (v1) / `id` (v2)
- Foreign key: `payment_id`
- Stores connector-specific data
- Mutable during processing

**refund**
- Primary key: `internal_reference_id`
- Foreign keys: `payment_id`, `attempt_id`
- Independent lifecycle

**customers**
- Primary key: `customer_id`
- PII fields encrypted

**payment_methods**
- Primary key: `payment_method_id`
- Foreign keys: `customer_id`, `merchant_id`
- Card data encrypted or tokenized

### Repository Pattern

**Interface**: `crates/storage_impl/`

```rust
#[async_trait]
pub trait PaymentIntentInterface {
    async fn insert_payment_intent(...) -> Result<PaymentIntent>;
    async fn find_payment_intent_by_payment_id(...) -> Result<PaymentIntent>;
    async fn update_payment_intent(...) -> Result<PaymentIntent>;
}
```

**Implementation**: Diesel ORM + PostgreSQL

### Caching Layer

**Redis Keys**:
- `merchant:account:{merchant_id}` - TTL: 1 hour
- `routing:config:{merchant_id}` - TTL: 5 minutes
- `payment:session:{payment_id}` - TTL: 30 minutes
- `idempotency:{key}` - TTL: 24 hours

---

## Routing Engine Architecture

### Decision Algorithm

```
Input: PaymentRequest
  ├─ merchant_id
  ├─ payment_method
  ├─ currency
  ├─ amount
  └─ customer_country

Step 1: Fetch Routing Config
  └─ Cache check → DB fallback

Step 2: Filter Connectors
  ├─ Payment method support
  ├─ Currency support
  ├─ Amount limits
  ├─ Geo-restrictions
  └─ Connector status

Step 3: Apply Algorithm
  ├─ Static: Priority order
  ├─ Dynamic: Score calculation
  └─ Volume: Distribution %

Step 4: Merchant Preferences
  ├─ Preferred connectors
  ├─ Blocked connectors
  └─ Cost rules

Output: [Primary, Fallback1, Fallback2]
```

### Fallback Strategy

```rust
for connector in selected_connectors {
    match call_connector(connector, payment_data).await {
        Ok(response) => return Ok(response),
        Err(e) if e.is_retryable() => continue,
        Err(e) => return Err(e),
    }
}
```

---

## Security Architecture

### Authentication

- **API Keys**: Bearer token authentication
- **Publishable Key** (`pk_`): Client-side, limited scope
- **Secret Key** (`sk_`): Server-side, full access
- **JWT Tokens**: For user sessions

### Encryption

**At Rest**:
- AES-256-GCM for PII fields
- Keys stored in AWS KMS
- Column-level encryption in database

**In Transit**:
- TLS 1.3 for all connections
- mTLS for inter-service communication

### PII Masking

```rust
use masking::Secret;

pub struct PaymentRequest {
    pub card_number: Secret<String>, // ***1234
    pub cvv: Secret<String>,         // ***
    pub amount: i64,                 // Not masked
}
```

---

## Performance Considerations

### Latency Targets

```
P50: ~150ms
P95: ~500ms
P99: ~1000ms
```

### Optimization Techniques

1. **Connection Pooling** - Reuse DB/Redis connections
2. **Caching** - Aggressive caching of merchant config
3. **Async I/O** - Tokio runtime for non-blocking operations
4. **Query Optimization** - Indexed queries, no N+1
5. **Request Batching** - Batch analytics events

### Database Indexes

Critical indexes for performance:
```sql
CREATE INDEX idx_payment_intent_merchant_created 
ON payment_intent(merchant_id, created_at DESC);

CREATE UNIQUE INDEX idx_payment_attempt_connector_txn 
ON payment_attempt(connector_transaction_id);

CREATE INDEX idx_refund_payment 
ON refund(payment_id);
```

---

## References

### Diagrams

- [C4 Component Diagram](../plantuml/c4_component_router_detailed.puml)
- [Class Diagram - Payment Domain](../plantuml/class_diagram_payment_domain.puml)
- [Sequence Diagram - Payment Create](../plantuml/sequence_payment_create_detailed.puml)
- [State Machine - Payment Intent](../plantuml/state_machine_payment_intent.puml)
- [State Machine - Payment Attempt](../plantuml/state_machine_payment_attempt.puml)
- [ERD - Database Schema](../plantuml/erd_detailed.puml)

### Code References

- **Main Entry**: `crates/router/src/bin/router.rs`
- **Payment Core**: `crates/router/src/core/payments.rs`
- **Routing Engine**: `crates/router/src/core/routing/`
- **Connectors**: `crates/hyperswitch_connectors/src/connectors/`
- **Domain Models**: `crates/hyperswitch_domain_models/src/`
- **DB Models**: `crates/diesel_models/src/`

---

**Document Version**: 1.0.0  
**Last Updated**: December 18, 2025  
**Maintained By**: Software Architect Agent
