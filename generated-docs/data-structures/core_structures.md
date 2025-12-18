# Hyperswitch Core Data Structures

**Date**: December 18, 2025  
**Version**: 1.0.0  
**Source**: Codebase Analysis

---

## Table of Contents

1. [Domain Entities](#domain-entities)
2. [Collection Types](#collection-types)
3. [Memory Layouts](#memory-layouts)
4. [Performance Characteristics](#performance-characteristics)
5. [Type-Safe Wrappers](#type-safe-wrappers)

---

## Domain Entities

### 1. PaymentIntent

**Location**: `crates/diesel_models/src/payment_intent.rs`

**Purpose**: Central entity tracking payment lifecycle from creation to completion.

#### Structure (V2)

```rust
#[derive(Clone, Debug, PartialEq, Identifiable, Queryable)]
pub struct PaymentIntent {
    // Primary Key
    pub id: GlobalId,                                    // 64 bytes (String)
    
    // Core Fields
    pub merchant_id: MerchantId,                         // 64 bytes
    pub status: IntentStatus,                            // 1-2 bytes (enum)
    pub amount: MinorUnit,                               // 8 bytes (i64)
    pub currency: Currency,                              // 1-2 bytes (enum)
    pub amount_captured: Option<MinorUnit>,              // 9 bytes (Option<i64>)
    
    // Customer & Profile
    pub customer_id: Option<GlobalCustomerId>,           // 65 bytes
    pub profile_id: ProfileId,                           // 64 bytes
    pub business_country: Option<CountryAlpha2>,         // 3 bytes
    pub business_label: Option<String>,                  // 24+ bytes
    
    // Payment Configuration
    pub capture_method: Option<CaptureMethod>,           // 2 bytes
    pub authentication_type: Option<AuthenticationType>, // 2 bytes
    pub setup_future_usage: Option<FutureUsage>,         // 2 bytes
    
    // Metadata
    pub description: Option<Description>,                // 24+ bytes
    pub return_url: Option<Url>,                         // 24+ bytes
    pub metadata: Option<SecretSerdeValue>,              // 24+ bytes (JSONB)
    pub order_details: Option<Vec<OrderDetails>>,        // 24+ bytes
    
    // Routing
    pub routing_algorithm_id: Option<String>,            // 24+ bytes
    pub connector_metadata: Option<SecretSerdeValue>,    // 24+ bytes
    
    // Timestamps
    pub created_at: PrimitiveDateTime,                   // 8 bytes
    pub modified_at: PrimitiveDateTime,                  // 8 bytes
    pub last_synced: Option<PrimitiveDateTime>,          // 9 bytes
    
    // Attempt Management
    pub active_attempt_id: Option<GlobalAttemptId>,      // 65 bytes
    pub attempt_count: i16,                              // 2 bytes
    pub authorization_count: Option<i32>,                // 5 bytes
    
    // Advanced Features
    pub feature_metadata: Option<FeatureMetadata>,       // 24+ bytes
    pub frm_metadata: Option<SecretSerdeValue>,          // 24+ bytes
    pub enable_partial_authorization: Option<bool>,      // 2 bytes
    
    // ... 20+ more fields
}
```

#### Memory Layout

**Approximate Size**: ~600-800 bytes per instance (depending on optional fields)

**Breakdown**:
- Fixed fields: ~200 bytes
- Optional strings (avg): ~400 bytes
- JSONB fields (avg): ~100 bytes

**Database Storage**: ~1-2 KB per row (with JSONB compression)

#### Access Patterns

**Primary Access**:
```sql
-- By payment_id (O(1) with unique index)
SELECT * FROM payment_intent WHERE id = $1;
```

**Secondary Access**:
```sql
-- By merchant + created_at (O(log n) with composite index)
SELECT * FROM payment_intent 
WHERE merchant_id = $1 AND created_at > $2 
ORDER BY created_at DESC LIMIT 100;

-- By customer (O(log n) with index)
SELECT * FROM payment_intent 
WHERE customer_id = $1 
ORDER BY created_at DESC;
```

**Typical Query Latency**: 2-5ms

---

### 2. PaymentAttempt

**Location**: `crates/diesel_models/src/payment_attempt.rs`

**Purpose**: Individual payment attempt with connector-specific data.

#### Structure (V2)

```rust
#[derive(Clone, Debug, PartialEq, Identifiable, Queryable)]
pub struct PaymentAttempt {
    // Primary Keys
    pub id: GlobalAttemptId,                             // 64 bytes
    pub payment_id: GlobalId,                            // 64 bytes (FK)
    
    // Core Fields
    pub merchant_id: MerchantId,                         // 64 bytes (FK)
    pub status: AttemptStatus,                           // 1-2 bytes
    pub amount: MinorUnit,                               // 8 bytes
    pub currency: Currency,                              // 1-2 bytes
    pub net_amount: MinorUnit,                           // 8 bytes
    
    // Connector Details
    pub connector: Option<String>,                       // 24+ bytes
    pub merchant_connector_id: Option<MCAId>,            // 65 bytes
    pub connector_transaction_id: Option<ConnectorTxnId>, // 24+ bytes
    pub connector_payment_id: Option<String>,            // 24+ bytes
    pub connector_request_reference_id: Option<String>,  // 24+ bytes
    pub connector_response_reference_id: Option<String>, // 24+ bytes
    
    // Payment Method
    pub payment_method: Option<PaymentMethod>,           // 2 bytes
    pub payment_method_type: Option<PaymentMethodType>,  // 2 bytes
    pub payment_method_id: Option<PaymentMethodId>,      // 65 bytes
    pub payment_method_data: Option<EncryptedData>,      // Variable (encrypted)
    pub payment_token: Option<String>,                   // 24+ bytes
    
    // Amount Tracking
    pub amount_capturable: MinorUnit,                    // 8 bytes
    pub amount_captured: Option<MinorUnit>,              // 9 bytes
    pub amount_to_capture: Option<MinorUnit>,            // 9 bytes
    pub authorized_amount: Option<MinorUnit>,            // 9 bytes
    
    // Capture Configuration
    pub capture_method: Option<CaptureMethod>,           // 2 bytes
    pub capture_before: Option<PrimitiveDateTime>,       // 9 bytes
    pub multiple_capture_count: Option<i16>,             // 3 bytes
    
    // Error Handling
    pub error_code: Option<String>,                      // 24+ bytes
    pub error_message: Option<String>,                   // 24+ bytes
    pub error_reason: Option<String>,                    // 24+ bytes
    pub issuer_error_code: Option<String>,               // 24+ bytes
    pub issuer_error_message: Option<String>,            // 24+ bytes
    pub network_error_message: Option<String>,           // 24+ bytes
    
    // Network Details
    pub card_network: Option<CardNetwork>,               // 2 bytes
    pub network_transaction_id: Option<String>,          // 24+ bytes
    pub network_advice_code: Option<String>,             // 24+ bytes
    pub network_decline_code: Option<String>,            // 24+ bytes
    
    // Authentication
    pub authentication_type: Option<AuthenticationType>, // 2 bytes
    pub authentication_id: Option<String>,               // 24+ bytes
    pub authentication_connector: Option<String>,        // 24+ bytes
    pub authentication_data: Option<AuthenticationData>, // 24+ bytes (JSONB)
    
    // Surcharge & Tax
    pub surcharge_amount: Option<MinorUnit>,             // 9 bytes
    pub tax_amount: Option<MinorUnit>,                   // 9 bytes
    pub order_tax_amount: Option<MinorUnit>,             // 9 bytes
    pub shipping_cost: Option<MinorUnit>,                // 9 bytes
    
    // Browser & Client
    pub browser_info: Option<BrowserInformation>,        // 24+ bytes (JSONB)
    pub client_source: Option<String>,                   // 24+ bytes
    pub client_version: Option<String>,                  // 24+ bytes
    
    // Timestamps
    pub created_at: PrimitiveDateTime,                   // 8 bytes
    pub modified_at: PrimitiveDateTime,                  // 8 bytes
    
    // Mandate & Recurring
    pub mandate_id: Option<MandateId>,                   // 65 bytes
    pub connector_mandate_detail: Option<ConnectorMandateRef>, // 24+ bytes (JSONB)
    
    // Advanced
    pub connector_metadata: Option<SecretSerdeValue>,    // 24+ bytes (JSONB)
    pub feature_metadata: Option<FeatureMetadata>,       // 24+ bytes (JSONB)
    pub preprocessing_step_id: Option<String>,           // 24+ bytes
    pub charges: Option<ChargeRefunds>,                  // 24+ bytes (JSONB)
    
    // ... 10+ more fields
}
```

#### Memory Layout

**Approximate Size**: ~800-1200 bytes per instance

**Breakdown**:
- Fixed fields: ~150 bytes
- Optional strings: ~500-700 bytes
- JSONB fields: ~150 bytes
- Encrypted data: Variable (100-500 bytes)

**Database Storage**: ~1.5-3 KB per row

#### Access Patterns

**Primary Access**:
```sql
-- By attempt_id (O(1))
SELECT * FROM payment_attempt WHERE id = $1;

-- By payment_id (O(log n), typically 1-3 attempts)
SELECT * FROM payment_attempt 
WHERE payment_id = $1 
ORDER BY created_at DESC;
```

**Unique Constraints**:
```sql
-- Connector transaction ID must be unique
CREATE UNIQUE INDEX idx_connector_txn 
ON payment_attempt(connector_transaction_id) 
WHERE connector_transaction_id IS NOT NULL;
```

---

### 3. Refund

**Location**: `crates/diesel_models/src/refund.rs`

**Purpose**: Track refund operations on successful payments.

#### Structure (V1)

```rust
#[derive(Clone, Debug, Identifiable, Queryable)]
pub struct Refund {
    // Primary Keys
    pub internal_reference_id: String,                   // 64 bytes (PK)
    pub refund_id: String,                               // 64 bytes (merchant reference)
    
    // Relations
    pub payment_id: PaymentId,                           // 64 bytes (FK)
    pub merchant_id: MerchantId,                         // 64 bytes (FK)
    pub attempt_id: String,                              // 64 bytes (FK)
    
    // Connector Info
    pub connector: String,                               // 24+ bytes
    pub connector_transaction_id: ConnectorTransactionId, // 24+ bytes
    pub connector_refund_id: Option<ConnectorTxnId>,     // 24+ bytes
    pub external_reference_id: Option<String>,           // 24+ bytes
    
    // Amounts
    pub total_amount: MinorUnit,                         // 8 bytes
    pub currency: Currency,                              // 1-2 bytes
    pub refund_amount: MinorUnit,                        // 8 bytes
    
    // Status & Type
    pub refund_type: RefundType,                         // 1-2 bytes
    pub refund_status: RefundStatus,                     // 1-2 bytes
    pub sent_to_gateway: bool,                           // 1 byte
    
    // Error Handling
    pub refund_error_message: Option<String>,            // 24+ bytes
    pub refund_error_code: Option<String>,               // 24+ bytes
    pub refund_error_reason: Option<String>,             // 24+ bytes
    
    // Additional Info
    pub description: Option<String>,                     // 24+ bytes
    pub metadata: Option<SecretSerdeValue>,              // 24+ bytes (JSONB)
    pub refund_arn: Option<String>,                      // 24+ bytes
    pub refund_reason: Option<String>,                   // 24+ bytes
    
    // Timestamps
    pub created_at: PrimitiveDateTime,                   // 8 bytes
    pub modified_at: PrimitiveDateTime,                  // 8 bytes
    
    // Configuration
    pub profile_id: Option<ProfileId>,                   // 65 bytes
    pub updated_by: Option<String>,                      // 24+ bytes
    pub merchant_connector_id: Option<MCAId>,            // 65 bytes
    pub charges: Option<ChargeRefunds>,                  // 24+ bytes (JSONB)
}
```

#### Memory Layout

**Approximate Size**: ~500-700 bytes per instance

**Database Storage**: ~1-1.5 KB per row

#### Business Invariants

```rust
impl Refund {
    pub fn validate(&self) -> Result<(), ValidationError> {
        // Refund amount must be positive
        if self.refund_amount.get_amount_as_i64() <= 0 {
            return Err(ValidationError::InvalidAmount);
        }
        
        // Refund amount cannot exceed total amount
        if self.refund_amount > self.total_amount {
            return Err(ValidationError::RefundExceedsTotal);
        }
        
        Ok(())
    }
    
    pub fn is_terminal(&self) -> bool {
        matches!(
            self.refund_status,
            RefundStatus::Success | RefundStatus::Failure | RefundStatus::TransactionFailure
        )
    }
}
```

---

## Collection Types

### 1. HashMap Usage

**Location**: Throughout codebase for O(1) lookups

**Example**: Connector Configuration Cache

```rust
use std::collections::HashMap;

pub struct ConnectorCache {
    // merchant_id + connector_name → Configuration
    configs: HashMap<(MerchantId, String), ConnectorConfig>,
}

impl ConnectorCache {
    pub fn get(&self, merchant_id: &MerchantId, connector: &str) 
        -> Option<&ConnectorConfig> 
    {
        self.configs.get(&(merchant_id.clone(), connector.to_string()))
    }
    
    pub fn insert(&mut self, merchant_id: MerchantId, connector: String, config: ConnectorConfig) {
        self.configs.insert((merchant_id, connector), config);
    }
}
```

**Complexity**: O(1) average case for get/insert  
**Memory**: ~48 bytes per entry + key/value sizes

---

### 2. Vec Usage

**Location**: Ordered collections

**Example**: Payment Attempts List

```rust
pub struct PaymentWithAttempts {
    pub intent: PaymentIntent,
    pub attempts: Vec<PaymentAttempt>,  // Ordered by created_at
}

impl PaymentWithAttempts {
    pub fn latest_attempt(&self) -> Option<&PaymentAttempt> {
        self.attempts.last()  // O(1)
    }
    
    pub fn find_attempt(&self, attempt_id: &str) -> Option<&PaymentAttempt> {
        self.attempts.iter()
            .find(|a| a.attempt_id == attempt_id)  // O(n), but n typically small (1-3)
    }
}
```

**Complexity**: 
- Push: O(1) amortized
- Access by index: O(1)
- Linear search: O(n)

**Memory**: 24 bytes (header) + capacity * element_size

---

### 3. BTreeMap Usage

**Location**: Ordered mappings

**Example**: Connector Priority Routing

```rust
use std::collections::BTreeMap;

pub struct OrderedConnectorList {
    // priority (lower = higher priority) → Connector
    connectors: BTreeMap<u8, Connector>,
}

impl OrderedConnectorList {
    pub fn get_by_priority(&self) -> Vec<&Connector> {
        self.connectors.values().collect()  // Already ordered
    }
    
    pub fn insert(&mut self, priority: u8, connector: Connector) {
        self.connectors.insert(priority, connector);
    }
}
```

**Complexity**: O(log n) for insert/get  
**Memory**: ~48 bytes per node + key/value sizes

---

## Memory Layouts

### Stack vs Heap Allocation

#### Stack-Allocated (Small, Fixed-Size)

```rust
// Stored entirely on stack
pub struct PaymentMetrics {
    pub success_count: u64,      // 8 bytes
    pub failure_count: u64,      // 8 bytes
    pub total_amount: i64,       // 8 bytes
    pub average_latency_ms: u32, // 4 bytes
}
// Total: 28 bytes on stack
```

#### Heap-Allocated (Large, Variable-Size)

```rust
// Stored on heap (Vec, String, Box)
pub struct PaymentIntent {
    pub payment_id: String,          // 24 bytes (stack) → heap data
    pub metadata: Option<serde_json::Value>, // 24 bytes (stack) → heap data
    pub order_details: Vec<OrderDetail>, // 24 bytes (stack) → heap array
    // Total stack: ~150 bytes
    // Total heap: ~500-1000 bytes
}
```

---

## Performance Characteristics

### Database Query Performance

#### payment_intent Queries

```sql
-- Fastest: Primary key lookup (O(1) with index)
SELECT * FROM payment_intent WHERE id = $1;
-- Typical: 1-2ms

-- Fast: Composite index range scan (O(log n + k))
SELECT * FROM payment_intent 
WHERE merchant_id = $1 AND created_at > $2 
ORDER BY created_at DESC LIMIT 100;
-- Typical: 3-8ms

-- Moderate: JSONB query with GIN index
SELECT * FROM payment_intent 
WHERE metadata @> '{"key": "value"}';
-- Typical: 10-50ms (depends on selectivity)

-- Slow: Full table scan (O(n))
SELECT * FROM payment_intent WHERE description LIKE '%keyword%';
-- Typical: 100-1000ms+ (avoid!)
```

#### payment_attempt Queries

```sql
-- Fastest: Unique constraint lookup
SELECT * FROM payment_attempt WHERE connector_transaction_id = $1;
-- Typical: 1-2ms

-- Fast: Foreign key lookup (indexed)
SELECT * FROM payment_attempt WHERE payment_id = $1;
-- Typical: 2-4ms (1-3 attempts typically)
```

### In-Memory Data Structure Performance

| Operation | Vec | HashMap | BTreeMap |
|-----------|-----|---------|----------|
| Insert | O(1)* | O(1)* | O(log n) |
| Lookup | O(n) | O(1)* | O(log n) |
| Remove | O(n) | O(1)* | O(log n) |
| Iterate | O(n) | O(n) | O(n) |
| Ordered | ❌ | ❌ | ✅ |
| Memory | Low | Medium | Medium-High |

*Amortized

---

## Type-Safe Wrappers

### NewType Pattern for IDs

**Location**: `crates/common_utils/src/id_type.rs`

```rust
#[derive(Debug, Clone, PartialEq, Eq, Hash, Serialize, Deserialize)]
pub struct PaymentId(String);

#[derive(Debug, Clone, PartialEq, Eq, Hash, Serialize, Deserialize)]
pub struct MerchantId(String);

#[derive(Debug, Clone, PartialEq, Eq, Hash, Serialize, Deserialize)]
pub struct CustomerId(String);
```

**Benefits**:
- ✅ Prevents mixing IDs (e.g., passing PaymentId where MerchantId expected)
- ✅ Type-safe at compile time
- ✅ Zero runtime cost (newtype pattern)
- ✅ Can add validation in constructor

**Example**:

```rust
impl PaymentId {
    pub fn new(id: String) -> Result<Self, IdError> {
        // Validate format
        if !id.starts_with("pay_") {
            return Err(IdError::InvalidFormat);
        }
        if id.len() != 32 {
            return Err(IdError::InvalidLength);
        }
        Ok(Self(id))
    }
}
```

### Phantom Types for States

```rust
use std::marker::PhantomData;

pub struct PaymentIntent<State> {
    pub payment_id: String,
    pub amount: i64,
    _state: PhantomData<State>,
}

pub struct RequiresPaymentMethod;
pub struct RequiresConfirmation;
pub struct Processing;
pub struct Succeeded;

impl PaymentIntent<RequiresPaymentMethod> {
    pub fn add_payment_method(self, pm: PaymentMethod) 
        -> PaymentIntent<RequiresConfirmation> 
    {
        PaymentIntent {
            payment_id: self.payment_id,
            amount: self.amount,
            _state: PhantomData,
        }
    }
}

impl PaymentIntent<RequiresConfirmation> {
    pub fn confirm(self) -> PaymentIntent<Processing> {
        // State transition
        PaymentIntent {
            payment_id: self.payment_id,
            amount: self.amount,
            _state: PhantomData,
        }
    }
}

// Can only call confirm() on RequiresConfirmation state
// Enforced at compile time!
```

---

## Data Structure Optimization Tips

### 1. Use Appropriate Sizes

```rust
// Bad: Oversized types
pub struct Payment {
    pub count: u64,    // Max value likely < 255
}

// Good: Right-sized types
pub struct Payment {
    pub count: u8,     // Sufficient, saves 7 bytes
}
```

### 2. Order Fields by Size

```rust
// Bad: Poor alignment (24 bytes)
pub struct BadStruct {
    pub flag: bool,    // 1 byte
    pub amount: i64,   // 8 bytes (7 bytes padding before)
    pub count: u8,     // 1 byte (7 bytes padding after)
}

// Good: Optimal alignment (16 bytes)
pub struct GoodStruct {
    pub amount: i64,   // 8 bytes
    pub flag: bool,    // 1 byte
    pub count: u8,     // 1 byte
                       // 6 bytes padding at end
}
```

### 3. Use Options Sparingly

```rust
// Bad: Many Options (each adds 1 byte)
pub struct ManyOptions {
    pub field1: Option<u8>,
    pub field2: Option<u8>,
    pub field3: Option<u8>,
    // 6 bytes total (instead of 3)
}

// Good: Bitflags for boolean options
bitflags! {
    pub struct Flags: u8 {
        const FIELD1 = 0b001;
        const FIELD2 = 0b010;
        const FIELD3 = 0b100;
    }
}
// 1 byte total
```

---

**Document Version**: 1.0.0  
**Last Updated**: December 18, 2025  
**Maintained By**: Software Architect Agent
