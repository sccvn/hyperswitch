# Hyperswitch Design Patterns Catalog

**Date**: December 18, 2025  
**Version**: 1.0.0  
**Extracted From**: Hyperswitch Codebase Analysis

---

## Table of Contents

1. [Creational Patterns](#creational-patterns)
2. [Structural Patterns](#structural-patterns)
3. [Behavioral Patterns](#behavioral-patterns)
4. [Concurrency Patterns](#concurrency-patterns)
5. [Domain-Driven Design Patterns](#domain-driven-design-patterns)

---

## Creational Patterns

### 1. Factory Pattern

#### Pattern: Connector Factory

**Intent**: Create connector instances based on connector name without exposing instantiation logic.

**Location**: `crates/hyperswitch_interfaces/src/connector_integration_interface.rs`

**Implementation**:

```rust
pub enum ConnectorEnum {
    Stripe,
    Adyen,
    Checkout,
    PayPal,
    // ... 100+ variants
}

impl ConnectorEnum {
    pub fn new() -> Box<dyn ConnectorIntegration> {
        match self {
            Self::Stripe => Box::new(Stripe::new()),
            Self::Adyen => Box::new(Adyen::new()),
            Self::Checkout => Box::new(Checkout::new()),
            Self::PayPal => Box::new(PayPal::new()),
            // ... more connectors
        }
    }
}
```

**Usage**:

```rust
let connector_name = "stripe";
let connector_enum = ConnectorEnum::from_str(connector_name)?;
let connector = connector_enum.new();
```

**Benefits**:
- ✅ Decouples client code from specific connector classes
- ✅ Single responsibility for connector creation
- ✅ Easy to add new connectors

**Trade-offs**:
- ⚠️ Large match statement grows with connectors
- ⚠️ Compile-time overhead for 100+ variants

---

### 2. Builder Pattern

#### Pattern: Request Builder

**Intent**: Construct complex request objects step-by-step with fluent API.

**Location**: `crates/api_models/src/payments.rs`

**Implementation**:

```rust
#[derive(Builder)]
#[builder(pattern = "owned")]
pub struct PaymentsRequest {
    pub amount: i64,
    pub currency: Currency,
    #[builder(default)]
    pub customer_id: Option<String>,
    #[builder(default)]
    pub payment_method_data: Option<PaymentMethodData>,
    #[builder(default)]
    pub return_url: Option<String>,
    #[builder(default)]
    pub metadata: Option<Metadata>,
    // ... 30+ fields
}
```

**Usage**:

```rust
let request = PaymentsRequestBuilder::default()
    .amount(5000)
    .currency(Currency::USD)
    .customer_id(Some("cust_123".to_string()))
    .payment_method_data(Some(card_data))
    .build()?;
```

**Benefits**:
- ✅ Improves readability for complex objects
- ✅ Enforces required fields at compile-time
- ✅ Provides defaults for optional fields

**Macro**: Uses `derive_builder` crate

---

## Structural Patterns

### 1. Adapter Pattern

#### Pattern: Connector Adapter

**Intent**: Convert the interface of each payment processor to a unified interface expected by the router.

**Location**: `crates/hyperswitch_connectors/src/connectors/*/mod.rs`

**Interface (Target)**:

```rust
#[async_trait]
pub trait PaymentAuthorize: ConnectorCommon {
    fn build_request(
        &self,
        req: &PaymentAuthorizeRouterData,
    ) -> Result<Option<Request>>;
    
    fn handle_response(
        &self,
        data: &PaymentAuthorizeRouterData,
        res: Response,
    ) -> Result<PaymentAuthorizeRouterData>;
}
```

**Adapter Implementation (Stripe)**:

```rust
impl PaymentAuthorize for Stripe {
    fn build_request(
        &self,
        req: &PaymentAuthorizeRouterData,
    ) -> Result<Option<Request>> {
        // Adapt RouterData → Stripe-specific format
        let stripe_req = StripePaymentIntentRequest {
            amount: req.request.amount,
            currency: req.request.currency.to_string().to_lowercase(),
            payment_method: extract_pm(&req.request.payment_method_data)?,
            confirm: true,
        };
        
        Ok(Some(RequestBuilder::new()
            .method(Method::Post)
            .url(&self.base_url().join("v1/payment_intents"))
            .headers(self.get_headers(req)?)
            .json(&stripe_req)
            .build()))
    }
    
    fn handle_response(
        &self,
        data: &PaymentAuthorizeRouterData,
        res: Response,
    ) -> Result<PaymentAuthorizeRouterData> {
        // Adapt Stripe response → RouterData
        let stripe_res: StripePaymentIntentResponse = res.json()?;
        
        Ok(PaymentAuthorizeRouterData {
            status: map_stripe_status(stripe_res.status)?,
            response: Ok(PaymentAuthorizeResponse {
                connector_transaction_id: stripe_res.id,
                // ... map fields
            }),
            ..data.clone()
        })
    }
}
```

**Benefits**:
- ✅ Unifies 100+ different connector APIs
- ✅ Router core doesn't know connector-specific details
- ✅ Easy to add new connectors without changing router

**Trade-offs**:
- ⚠️ Transformation overhead
- ⚠️ Need to maintain mappings for each connector

---

### 2. Facade Pattern

#### Pattern: Payments Core Facade

**Intent**: Provide simplified interface to complex payment processing subsystem.

**Location**: `crates/router/src/core/payments.rs`

**Facade**:

```rust
pub struct PaymentsCore {
    state: AppState,
}

impl PaymentsCore {
    // Simplified interface
    pub async fn create_payment(
        &self,
        merchant_account: MerchantAccount,
        request: PaymentsRequest,
    ) -> Result<PaymentsResponse> {
        // Orchestrates:
        // 1. Validation
        // 2. Intent creation
        // 3. Attempt creation
        // 4. Routing
        // 5. Connector call
        // 6. Status update
        // 7. Analytics
        // 8. Webhooks
        
        // ... complex orchestration
    }
}
```

**Behind the Facade**:
- `ValidationService` - Input validation
- `PaymentIntentService` - Intent CRUD
- `PaymentAttemptService` - Attempt CRUD
- `RoutingEngine` - Connector selection
- `ConnectorDispatcher` - Connector calls
- `AnalyticsPublisher` - Event publishing
- `WebhookManager` - Webhook dispatch

**Benefits**:
- ✅ Hides complexity from route handlers
- ✅ Single point of entry for payment operations
- ✅ Easy to modify internal implementation

---

### 3. Repository Pattern

#### Pattern: Storage Abstraction

**Intent**: Abstract data access logic from business logic.

**Location**: `crates/storage_impl/src/`

**Repository Interface**:

```rust
#[async_trait]
pub trait PaymentIntentInterface {
    async fn insert_payment_intent(
        &self,
        payment_intent: PaymentIntentNew,
    ) -> Result<PaymentIntent, StorageError>;
    
    async fn find_payment_intent_by_payment_id(
        &self,
        payment_id: &str,
        merchant_id: &str,
    ) -> Result<PaymentIntent, StorageError>;
    
    async fn update_payment_intent(
        &self,
        this: PaymentIntent,
        payment_intent_update: PaymentIntentUpdate,
    ) -> Result<PaymentIntent, StorageError>;
}
```

**Implementation (PostgreSQL)**:

```rust
pub struct PaymentIntentRepository {
    db_pool: Pool<ConnectionManager<PgConnection>>,
}

#[async_trait]
impl PaymentIntentInterface for PaymentIntentRepository {
    async fn insert_payment_intent(
        &self,
        payment_intent: PaymentIntentNew,
    ) -> Result<PaymentIntent, StorageError> {
        let conn = self.db_pool.get().await?;
        
        diesel::insert_into(payment_intent::table)
            .values(&payment_intent)
            .get_result(&conn)
            .await
            .map_err(|e| StorageError::from(e))
    }
    
    // ... other methods
}
```

**Benefits**:
- ✅ Business logic independent of storage technology
- ✅ Easy to switch from PostgreSQL to other databases
- ✅ Enables caching layer (Redis) without changing interface
- ✅ Testable with mock repositories

---

## Behavioral Patterns

### 1. Strategy Pattern

#### Pattern: Routing Strategy

**Intent**: Define family of routing algorithms, encapsulate each one, make them interchangeable.

**Location**: `crates/router/src/core/routing/`

**Strategy Interface**:

```rust
#[async_trait]
pub trait RoutingAlgorithm {
    async fn get_connector(
        &self,
        context: &RoutingContext,
    ) -> Result<Vec<Connector>>;
}
```

**Concrete Strategies**:

**Static Routing**:
```rust
pub struct StaticRoutingAlgorithm {
    pub rules: Vec<RoutingRule>,
}

#[async_trait]
impl RoutingAlgorithm for StaticRoutingAlgorithm {
    async fn get_connector(
        &self,
        context: &RoutingContext,
    ) -> Result<Vec<Connector>> {
        // Evaluate rules top-to-bottom
        for rule in &self.rules {
            if rule.matches(context) {
                return Ok(rule.connectors.clone());
            }
        }
        Err(RoutingError::NoMatchingRule)
    }
}
```

**Dynamic Routing**:
```rust
pub struct DynamicRoutingAlgorithm {
    pub analytics: AnalyticsClient,
}

#[async_trait]
impl RoutingAlgorithm for DynamicRoutingAlgorithm {
    async fn get_connector(
        &self,
        context: &RoutingContext,
    ) -> Result<Vec<Connector>> {
        // Fetch success rates
        let stats = self.analytics.get_connector_stats(context).await?;
        
        // Calculate scores
        let mut scored: Vec<(Connector, f64)> = context
            .eligible_connectors()
            .map(|c| {
                let score = stats.get(&c)
                    .map(|s| s.success_rate * 0.7 + s.volume * 0.3)
                    .unwrap_or(0.0);
                (c, score)
            })
            .collect();
        
        // Sort by score
        scored.sort_by(|a, b| b.1.partial_cmp(&a.1).unwrap());
        
        Ok(scored.into_iter().map(|(c, _)| c).collect())
    }
}
```

**Volume Routing**:
```rust
pub struct VolumeRoutingAlgorithm {
    pub distribution: HashMap<Connector, u8>, // Percentage
}

#[async_trait]
impl RoutingAlgorithm for VolumeRoutingAlgorithm {
    async fn get_connector(
        &self,
        context: &RoutingContext,
    ) -> Result<Vec<Connector>> {
        // Weighted random selection
        let rand: u8 = rand::random();
        let mut cumulative = 0;
        
        for (connector, percentage) in &self.distribution {
            cumulative += percentage;
            if rand < cumulative {
                return Ok(vec![connector.clone()]);
            }
        }
        
        Err(RoutingError::NoConnectorSelected)
    }
}
```

**Context**:

```rust
pub struct RoutingEngine {
    algorithm: Box<dyn RoutingAlgorithm>,
}

impl RoutingEngine {
    pub async fn route(
        &self,
        context: &RoutingContext,
    ) -> Result<Vec<Connector>> {
        self.algorithm.get_connector(context).await
    }
}
```

**Benefits**:
- ✅ Easy to add new routing algorithms
- ✅ Algorithm can be changed at runtime
- ✅ Each algorithm is independently testable

---

### 2. State Pattern

#### Pattern: Payment State Machine

**Intent**: Allow payment object to alter its behavior when its internal state changes.

**Location**: `crates/diesel_models/src/payment_intent.rs`

**State Enum**:

```rust
#[derive(Debug, Clone, PartialEq, Serialize, Deserialize)]
pub enum IntentStatus {
    RequiresPaymentMethod,
    RequiresConfirmation,
    RequiresCustomerAction,
    Processing,
    RequiresCapture,
    PartiallyCaptured,
    PartiallyCapturedAndCapturable,
    Succeeded,
    Failed,
    Cancelled,
    RequiresMerchantAction,
}
```

**State Transition Logic**:

```rust
impl PaymentIntent {
    pub fn can_transition_to(&self, new_status: IntentStatus) -> bool {
        use IntentStatus::*;
        
        match (&self.status, new_status) {
            // Initial state transitions
            (RequiresPaymentMethod, RequiresConfirmation) => true,
            (RequiresPaymentMethod, Cancelled) => true,
            
            // Confirmation transitions
            (RequiresConfirmation, Processing) => true,
            (RequiresConfirmation, Cancelled) => true,
            
            // Processing transitions
            (Processing, RequiresCustomerAction) => true,
            (Processing, Succeeded) => true,
            (Processing, Failed) => true,
            (Processing, RequiresCapture) => true,
            (Processing, RequiresMerchantAction) => true,
            
            // Customer action transitions
            (RequiresCustomerAction, Processing) => true,
            (RequiresCustomerAction, Failed) => true,
            (RequiresCustomerAction, Cancelled) => true,
            
            // Capture transitions
            (RequiresCapture, Succeeded) => true,
            (RequiresCapture, PartiallyCaptured) => true,
            (RequiresCapture, Cancelled) => true,
            
            // Terminal states (no transitions)
            (Succeeded, _) => false,
            (Failed, _) => false,
            (Cancelled, _) => false,
            
            _ => false,
        }
    }
    
    pub fn transition_to(
        &mut self,
        new_status: IntentStatus,
    ) -> Result<(), PaymentError> {
        if !self.can_transition_to(new_status) {
            return Err(PaymentError::InvalidStateTransition {
                from: self.status.clone(),
                to: new_status,
            });
        }
        
        self.status = new_status;
        self.modified_at = Utc::now().naive_utc();
        
        Ok(())
    }
}
```

**Benefits**:
- ✅ Encapsulates state-specific behavior
- ✅ Prevents invalid state transitions
- ✅ State transitions are explicit and documented

See [state_machine_payment_intent.puml](../plantuml/state_machine_payment_intent.puml) for visual representation.

---

### 3. Observer Pattern

#### Pattern: Event Publishing

**Intent**: Define one-to-many dependency where payment events notify multiple subscribers.

**Location**: `crates/events/src/event_logger.rs`

**Subject (Event Publisher)**:

```rust
pub struct EventPublisher {
    subscribers: Vec<Box<dyn EventSubscriber>>,
}

impl EventPublisher {
    pub async fn publish(&self, event: PaymentEvent) {
        for subscriber in &self.subscribers {
            subscriber.on_event(&event).await;
        }
    }
}
```

**Observer Interface**:

```rust
#[async_trait]
pub trait EventSubscriber: Send + Sync {
    async fn on_event(&self, event: &PaymentEvent);
}
```

**Concrete Observers**:

**Analytics Subscriber**:
```rust
pub struct AnalyticsSubscriber {
    analytics_client: AnalyticsClient,
}

#[async_trait]
impl EventSubscriber for AnalyticsSubscriber {
    async fn on_event(&self, event: &PaymentEvent) {
        match event {
            PaymentEvent::Created { payment_id, amount, .. } => {
                self.analytics_client.track_payment_created(*amount).await;
            }
            PaymentEvent::Succeeded { payment_id, connector, .. } => {
                self.analytics_client.track_payment_success(connector).await;
            }
            // ... more events
            _ => {}
        }
    }
}
```

**Webhook Subscriber**:
```rust
pub struct WebhookSubscriber {
    webhook_client: WebhookClient,
}

#[async_trait]
impl EventSubscriber for WebhookSubscriber {
    async fn on_event(&self, event: &PaymentEvent) {
        if let Some(webhook_url) = event.merchant_webhook_url() {
            self.webhook_client.send(webhook_url, event).await;
        }
    }
}
```

**Audit Log Subscriber**:
```rust
pub struct AuditLogSubscriber {
    audit_repo: AuditLogRepository,
}

#[async_trait]
impl EventSubscriber for AuditLogSubscriber {
    async fn on_event(&self, event: &PaymentEvent) {
        let audit_entry = AuditLogEntry::from_event(event);
        self.audit_repo.insert(audit_entry).await;
    }
}
```

**Benefits**:
- ✅ Loosely coupled event producers and consumers
- ✅ Easy to add new subscribers
- ✅ Publishers don't need to know about subscribers

---

### 4. Template Method Pattern

#### Pattern: Connector Integration Template

**Intent**: Define skeleton of connector integration algorithm, letting subclasses override specific steps.

**Location**: `crates/hyperswitch_interfaces/src/api.rs`

**Abstract Template**:

```rust
#[async_trait]
pub trait ConnectorCommon {
    // Template method (cannot be overridden)
    async fn execute_connector_flow<Flow, Req, Res>(
        &self,
        router_data: &RouterData<Flow, Req, Res>,
    ) -> Result<RouterData<Flow, Req, Res>>
    where
        Self: ConnectorIntegration<Flow, Req, Res>,
    {
        // Step 1: Build request (abstract - must implement)
        let request = self.build_request(router_data)?;
        
        // Step 2: Get headers (can be overridden)
        let headers = self.get_headers(router_data)?;
        
        // Step 3: Make HTTP call (concrete - provided)
        let response = self.call_connector_api(request, headers).await?;
        
        // Step 4: Handle response (abstract - must implement)
        let result = self.handle_response(router_data, response)?;
        
        // Step 5: Log (concrete - provided)
        self.log_connector_call(&result).await;
        
        Ok(result)
    }
    
    // Abstract methods (must implement)
    fn build_request(...) -> Result<Request>;
    fn handle_response(...) -> Result<RouterData>;
    
    // Hook methods (can override)
    fn get_headers(...) -> Result<Headers> {
        // Default implementation
    }
    
    // Concrete methods (cannot override)
    async fn call_connector_api(...) -> Result<Response> {
        // Provided implementation
    }
    
    async fn log_connector_call(...) {
        // Provided implementation
    }
}
```

**Benefits**:
- ✅ Reuses common connector logic
- ✅ Enforces consistent flow
- ✅ Subclasses only implement what's unique

---

## Concurrency Patterns

### 1. Actor Model

#### Pattern: Tokio Tasks as Actors

**Intent**: Encapsulate state and behavior in asynchronous tasks that communicate via messages.

**Location**: `crates/scheduler/src/workflows/`

**Example**: Background Job Processor

```rust
pub struct JobProcessor {
    receiver: mpsc::Receiver<Job>,
    db: Arc<Database>,
}

impl JobProcessor {
    pub fn spawn(rx: mpsc::Receiver<Job>, db: Arc<Database>) -> JoinHandle<()> {
        let mut processor = Self {
            receiver: rx,
            db,
        };
        
        tokio::spawn(async move {
            while let Some(job) = processor.receiver.recv().await {
                processor.process_job(job).await;
            }
        })
    }
    
    async fn process_job(&self, job: Job) {
        // Process job
    }
}
```

**Usage**:

```rust
let (tx, rx) = mpsc::channel(100);
let handle = JobProcessor::spawn(rx, db.clone());

// Send jobs
tx.send(job).await?;
```

**Benefits**:
- ✅ No shared mutable state (message passing)
- ✅ Natural concurrency model
- ✅ Fault isolation

---

### 2. Worker Pool Pattern

#### Pattern: Actix-web Worker Pool

**Intent**: Process requests concurrently using pool of worker threads.

**Location**: `crates/router/src/lib.rs`

**Configuration**:

```rust
HttpServer::new(move || {
    App::new()
        .app_data(state.clone())
        .configure(routes::payments::config)
        .configure(routes::refunds::config)
        // ... more routes
})
.workers(num_cpus::get() * 2)  // Worker pool size
.bind(("0.0.0.0", 8080))?
.run()
.await
```

**Benefits**:
- ✅ Efficient CPU utilization
- ✅ Request isolation
- ✅ Automatic load balancing

---

## Domain-Driven Design Patterns

### 1. Aggregate Pattern

#### Pattern: Payment Aggregate

**Intent**: Treat PaymentIntent + PaymentAttempts as single unit of consistency.

**Aggregate Root**: `PaymentIntent`  
**Entities**: `PaymentAttempt`, `Refund`

**Invariants**:
- Payment attempts belong to exactly one intent
- Active attempt ID must reference valid attempt
- Amount captured <= Amount authorized
- Refund amount <= Amount captured

**Location**: `crates/hyperswitch_domain_models/src/payments/`

```rust
pub struct PaymentAggregate {
    pub intent: PaymentIntent,
    pub attempts: Vec<PaymentAttempt>,
    pub refunds: Vec<Refund>,
}

impl PaymentAggregate {
    pub fn add_attempt(&mut self, attempt: PaymentAttempt) -> Result<()> {
        // Validate invariants
        if attempt.payment_id != self.intent.payment_id {
            return Err(DomainError::InvalidAggregate);
        }
        
        self.attempts.push(attempt);
        self.intent.attempt_count += 1;
        
        Ok(())
    }
    
    pub fn capture(&mut self, amount: i64) -> Result<()> {
        // Validate invariants
        if amount > self.intent.amount {
            return Err(DomainError::CaptureExceedsAuthorization);
        }
        
        if self.intent.status != IntentStatus::RequiresCapture {
            return Err(DomainError::InvalidStatus);
        }
        
        self.intent.amount_captured = Some(amount);
        self.intent.status = IntentStatus::Succeeded;
        
        Ok(())
    }
}
```

---

### 2. Value Object Pattern

#### Pattern: Money (Amount + Currency)

**Intent**: Model concepts without identity as immutable value objects.

**Location**: `crates/common_utils/src/types.rs`

```rust
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub struct Money {
    pub amount: MinorUnit, // Amount in smallest currency unit
    pub currency: Currency,
}

impl Money {
    pub fn new(amount: i64, currency: Currency) -> Self {
        Self {
            amount: MinorUnit::new(amount),
            currency,
        }
    }
    
    pub fn add(&self, other: &Money) -> Result<Money, MoneyError> {
        if self.currency != other.currency {
            return Err(MoneyError::CurrencyMismatch);
        }
        
        Ok(Money {
            amount: self.amount + other.amount,
            currency: self.currency,
        })
    }
}
```

**Benefits**:
- ✅ Type-safe money handling
- ✅ Prevents currency mixing
- ✅ Immutable (no accidental modification)

---

## Pattern Summary Matrix

| Pattern | Type | Location | Primary Benefit |
|---------|------|----------|-----------------|
| Factory | Creational | `hyperswitch_interfaces` | Connector instantiation |
| Builder | Creational | `api_models` | Complex object construction |
| Adapter | Structural | `hyperswitch_connectors` | Unified connector interface |
| Facade | Structural | `router/core` | Simplified payment API |
| Repository | Structural | `storage_impl` | Data access abstraction |
| Strategy | Behavioral | `router/core/routing` | Interchangeable routing algorithms |
| State | Behavioral | `diesel_models` | Payment lifecycle management |
| Observer | Behavioral | `events` | Event-driven architecture |
| Template Method | Behavioral | `hyperswitch_interfaces` | Connector integration flow |
| Actor Model | Concurrency | `scheduler` | Isolated concurrent tasks |
| Worker Pool | Concurrency | `router` | Request processing |
| Aggregate | DDD | `domain_models` | Consistency boundary |
| Value Object | DDD | `common_utils` | Immutable domain concepts |

---

**Document Version**: 1.0.0  
**Last Updated**: December 18, 2025  
**Maintained By**: Software Architect Agent
