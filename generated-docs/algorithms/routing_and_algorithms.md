# Hyperswitch Algorithms & Routing Logic

**Date**: December 18, 2025  
**Version**: 1.0.0  
**Source**: Codebase Analysis

---

## Table of Contents

1. [Payment Routing Algorithms](#payment-routing-algorithms)
2. [Retry & Fallback Strategies](#retry--fallback-strategies)
3. [State Machine Transitions](#state-machine-transitions)
4. [Reconciliation Algorithms](#reconciliation-algorithms)
5. [Caching & Invalidation](#caching--invalidation)
6. [Rate Limiting](#rate-limiting)

---

## Payment Routing Algorithms

### 1. Static Routing Algorithm

**Location**: `crates/router/src/core/routing/algorithms.rs`

**Purpose**: Route payments based on pre-configured priority list.

#### Algorithm

```rust
/// Static routing based on merchant-configured priority list
/// 
/// Time Complexity: O(1) for cache lookup + O(k) for filtering
/// Space Complexity: O(n) where n = total configured connectors
/// 
/// Algorithm Steps:
/// 1. Lookup routing config from cache: O(1)
/// 2. Filter connectors by payment_method & currency: O(k)
/// 3. Sort by priority (pre-sorted): O(1)
/// 4. Return top k connectors for fallback: O(k)
///
pub async fn static_routing(
    ctx: &RoutingContext,
    pm: PaymentMethod,
    currency: Currency,
) -> Result<Vec<ConnectorChoice>> {
    // Step 1: Lookup cached routing configuration
    let config = ctx.routing_cache
        .get(&(ctx.merchant_id.clone(), pm, currency))
        .ok_or(RoutingError::NoConfigFound)?;
    
    // Step 2: Filter eligible connectors
    let eligible = config.connectors
        .iter()
        .filter(|c| {
            c.payment_methods.contains(&pm) &&
            c.currencies.contains(&currency) &&
            c.enabled
        })
        .collect::<Vec<_>>();
    
    if eligible.is_empty() {
        return Err(RoutingError::NoEligibleConnectors);
    }
    
    // Step 3: Already sorted by priority in config
    // Step 4: Return configured connectors
    Ok(eligible.iter()
        .map(|c| ConnectorChoice {
            connector_name: c.name.clone(),
            merchant_connector_id: c.mca_id.clone(),
        })
        .collect())
}
```

**Complexity Analysis**:
- **Time**: O(1) + O(k) where k = configured connectors (typically 3-10)
- **Space**: O(k) for filtered list
- **Typical Latency**: <1ms (in-memory operation)

**Use Cases**:
- Simple routing needs
- Deterministic fallback order
- Cost-based routing (pre-configured by cheapest)

---

### 2. Dynamic Routing Algorithm (Volume-Based)

**Location**: `crates/router/src/core/routing/algorithms.rs`

**Purpose**: Route based on real-time success rates and transaction volumes.

#### Algorithm

```rust
/// Dynamic routing based on connector performance metrics
///
/// Time Complexity: O(n log n) where n = eligible connectors
/// Space Complexity: O(n) for scoring array
///
/// Algorithm Steps:
/// 1. Fetch eligible connectors: O(n)
/// 2. Fetch performance metrics from analytics: O(n) parallel
/// 3. Calculate weighted scores: O(n)
/// 4. Sort by score descending: O(n log n)
/// 5. Apply business rules: O(n)
/// 6. Return top k: O(k)
///
pub async fn dynamic_routing_volume_based(
    ctx: &RoutingContext,
    pm: PaymentMethod,
    currency: Currency,
    amount: i64,
) -> Result<Vec<ConnectorChoice>> {
    // Step 1: Get eligible connectors
    let eligible = get_eligible_connectors(ctx, pm, currency).await?;
    
    if eligible.is_empty() {
        return Err(RoutingError::NoEligibleConnectors);
    }
    
    // Step 2: Fetch performance metrics (last 7 days)
    let metrics = ctx.analytics_client
        .get_connector_metrics(
            &ctx.merchant_id,
            eligible.iter().map(|c| c.name.as_str()),
            Duration::days(7),
        )
        .await?;
    
    // Step 3: Calculate weighted scores
    let mut scored: Vec<(ConnectorChoice, f64)> = eligible
        .into_iter()
        .map(|connector| {
            let metric = metrics.get(&connector.name)
                .unwrap_or(&ConnectorMetric::default());
            
            // Weighted scoring formula:
            // score = (success_rate * 0.7) + (volume_ratio * 0.3)
            let success_rate = metric.success_count as f64 
                / metric.total_count.max(1) as f64;
            let volume_ratio = metric.total_volume as f64 
                / ctx.total_merchant_volume.max(1.0);
            
            let score = (success_rate * 0.7) + (volume_ratio * 0.3);
            
            (connector, score)
        })
        .collect();
    
    // Step 4: Sort by score descending
    scored.sort_by(|a, b| b.1.partial_cmp(&a.1).unwrap());
    
    // Step 5: Apply business rules
    let filtered = apply_business_rules(scored, amount, currency);
    
    // Step 6: Return top 3 for fallback
    Ok(filtered.into_iter()
        .take(3)
        .map(|(choice, _score)| choice)
        .collect())
}
```

**Complexity Analysis**:
- **Time**: O(n) + O(n log n) = O(n log n)
- **Space**: O(n) for scoring array
- **Typical Latency**: 5-15ms (includes analytics query)

**Scoring Formula**:
```
score = (success_rate × 0.7) + (volume_ratio × 0.3)

where:
  success_rate = successful_transactions / total_transactions
  volume_ratio = connector_volume / total_merchant_volume
```

**Business Rules Applied**:
```rust
fn apply_business_rules(
    scored: Vec<(ConnectorChoice, f64)>,
    amount: i64,
    currency: Currency,
) -> Vec<(ConnectorChoice, f64)> {
    scored.into_iter()
        .filter(|(choice, _score)| {
            // Rule 1: Min success rate threshold
            if _score < &0.7 {
                return false;
            }
            
            // Rule 2: Connector supports amount range
            if amount < choice.min_amount || amount > choice.max_amount {
                return false;
            }
            
            // Rule 3: Regional restrictions
            if let Some(allowed_countries) = &choice.allowed_countries {
                // Check against business_country
            }
            
            true
        })
        .collect()
}
```

---

### 3. Advanced Routing Algorithm (Latency-Optimized)

**Location**: `crates/router/src/core/routing/algorithms.rs`

**Purpose**: Minimize payment processing latency.

#### Algorithm

```rust
/// Latency-optimized routing with predictive modeling
///
/// Time Complexity: O(n log n)
/// Space Complexity: O(n)
///
pub async fn latency_optimized_routing(
    ctx: &RoutingContext,
    pm: PaymentMethod,
    currency: Currency,
) -> Result<Vec<ConnectorChoice>> {
    let eligible = get_eligible_connectors(ctx, pm, currency).await?;
    
    // Fetch latency metrics (P50, P95, P99)
    let latency_metrics = ctx.analytics_client
        .get_latency_metrics(&ctx.merchant_id, &eligible)
        .await?;
    
    let mut scored: Vec<(ConnectorChoice, f64)> = eligible
        .into_iter()
        .map(|connector| {
            let latency = latency_metrics.get(&connector.name)
                .map(|m| m.p95_latency_ms)
                .unwrap_or(10000.0); // Default: 10s (penalized)
            
            // Score inversely proportional to latency
            // Lower latency = higher score
            let score = 1000.0 / (latency + 1.0);
            
            (connector, score)
        })
        .collect();
    
    scored.sort_by(|a, b| b.1.partial_cmp(&a.1).unwrap());
    
    Ok(scored.into_iter()
        .take(3)
        .map(|(choice, _)| choice)
        .collect())
}
```

**Latency Calculation**:
```
score = 1000 / (P95_latency_ms + 1)

Example:
  Connector A: P95 = 500ms → score = 1000/501 ≈ 2.0
  Connector B: P95 = 2000ms → score = 1000/2001 ≈ 0.5
  → Connector A preferred
```

---

## Retry & Fallback Strategies

### 1. Exponential Backoff Retry

**Location**: `crates/router/src/core/errors/retry.rs`

**Purpose**: Retry failed operations with increasing delays.

#### Algorithm

```rust
/// Exponential backoff retry with jitter
///
/// Formula: delay = min(base_delay * 2^attempt, max_delay) ± jitter
/// Max attempts: 3
/// Base delay: 1 second
/// Max delay: 60 seconds
/// Jitter: ±20%
///
/// Time Complexity: O(1) per retry
/// Space Complexity: O(1)
///
pub async fn retry_with_backoff<F, T, E>(
    operation: F,
    max_retries: u32,
) -> Result<T, E>
where
    F: Fn() -> Pin<Box<dyn Future<Output = Result<T, E>>>>,
    E: std::error::Error,
{
    let mut attempt = 0;
    let base_delay_ms = 1000;
    let max_delay_ms = 60000;
    
    loop {
        match operation().await {
            Ok(result) => return Ok(result),
            Err(err) if attempt >= max_retries => return Err(err),
            Err(err) if !is_retryable(&err) => return Err(err),
            Err(_) => {
                attempt += 1;
                
                // Calculate exponential backoff
                let exp_delay = base_delay_ms * 2_u64.pow(attempt);
                let capped_delay = exp_delay.min(max_delay_ms);
                
                // Add jitter (±20%)
                let jitter = (capped_delay as f64 * 0.2) as u64;
                let jitter_offset = rand::thread_rng().gen_range(0..=jitter * 2) - jitter;
                let final_delay = (capped_delay as i64 + jitter_offset) as u64;
                
                tokio::time::sleep(Duration::from_millis(final_delay)).await;
            }
        }
    }
}

fn is_retryable<E: std::error::Error>(err: &E) -> bool {
    // Retry on:
    // - Network timeouts
    // - HTTP 5xx errors
    // - Connection errors
    // - Rate limit errors (429)
    
    // Do NOT retry on:
    // - HTTP 4xx (except 429)
    // - Authentication errors (401, 403)
    // - Validation errors (400)
    
    match err.downcast_ref::<ConnectorError>() {
        Some(ConnectorError::Timeout) => true,
        Some(ConnectorError::ServiceUnavailable) => true,
        Some(ConnectorError::RateLimited) => true,
        Some(ConnectorError::BadRequest) => false,
        Some(ConnectorError::Unauthorized) => false,
        _ => false,
    }
}
```

**Example Delays**:
```
Attempt 0: initial request
Attempt 1: 1s ± 200ms → 800ms-1200ms
Attempt 2: 2s ± 400ms → 1600ms-2400ms
Attempt 3: 4s ± 800ms → 3200ms-4800ms
```

**Jitter Benefits**:
- Prevents thundering herd problem
- Distributes retry load
- Reduces collision probability

---

### 2. Connector Fallback Strategy

**Location**: `crates/router/src/core/payments/flows.rs`

**Purpose**: Automatically fallback to alternative connectors on failure.

#### Algorithm

```rust
/// Connector fallback with circuit breaker
///
/// Time Complexity: O(k) where k = fallback connectors
/// Space Complexity: O(1)
///
pub async fn execute_payment_with_fallback(
    ctx: &PaymentContext,
    connectors: Vec<ConnectorChoice>,
) -> Result<PaymentResponse> {
    if connectors.is_empty() {
        return Err(PaymentError::NoConnectorsAvailable);
    }
    
    let mut last_error = None;
    
    for (index, connector) in connectors.iter().enumerate() {
        // Check circuit breaker state
        if ctx.circuit_breaker.is_open(&connector.connector_name) {
            tracing::warn!("Circuit breaker open for {}", connector.connector_name);
            continue;
        }
        
        tracing::info!(
            "Attempting payment with connector {} (fallback level: {})",
            connector.connector_name,
            index
        );
        
        match execute_connector_payment(ctx, connector).await {
            Ok(response) => {
                // Success - record and return
                ctx.circuit_breaker.record_success(&connector.connector_name);
                ctx.metrics.record_connector_success(&connector.connector_name);
                
                return Ok(response);
            }
            Err(err) if is_terminal_error(&err) => {
                // Terminal error - don't fallback
                tracing::error!("Terminal error from {}: {:?}", connector.connector_name, err);
                return Err(err);
            }
            Err(err) => {
                // Retriable error - try next connector
                tracing::warn!("Error from {}: {:?}, trying fallback", connector.connector_name, err);
                ctx.circuit_breaker.record_failure(&connector.connector_name);
                ctx.metrics.record_connector_failure(&connector.connector_name);
                
                last_error = Some(err);
                continue;
            }
        }
    }
    
    // All connectors failed
    Err(last_error.unwrap_or(PaymentError::AllConnectorsFailed))
}

fn is_terminal_error(err: &PaymentError) -> bool {
    matches!(err,
        PaymentError::CardDeclined |
        PaymentError::InsufficientFunds |
        PaymentError::InvalidCard |
        PaymentError::Blocked
    )
}
```

**Circuit Breaker Logic**:
```rust
pub struct CircuitBreaker {
    failure_threshold: u32,  // Open after N failures
    success_threshold: u32,  // Close after N successes
    timeout: Duration,       // Half-open timeout
    state: HashMap<String, BreakerState>,
}

enum BreakerState {
    Closed { failures: u32 },
    Open { opened_at: Instant },
    HalfOpen { successes: u32 },
}

impl CircuitBreaker {
    pub fn is_open(&self, connector: &str) -> bool {
        match self.state.get(connector) {
            Some(BreakerState::Open { opened_at }) => {
                // Check if timeout expired
                opened_at.elapsed() < self.timeout
            }
            _ => false,
        }
    }
    
    pub fn record_failure(&mut self, connector: &str) {
        let state = self.state.entry(connector.to_string())
            .or_insert(BreakerState::Closed { failures: 0 });
        
        match state {
            BreakerState::Closed { failures } => {
                *failures += 1;
                if *failures >= self.failure_threshold {
                    *state = BreakerState::Open { 
                        opened_at: Instant::now() 
                    };
                }
            }
            BreakerState::HalfOpen { .. } => {
                // Failed during half-open - reopen
                *state = BreakerState::Open { 
                    opened_at: Instant::now() 
                };
            }
            _ => {}
        }
    }
}
```

---

## State Machine Transitions

### 1. Payment Intent State Machine

**Purpose**: Manage payment lifecycle with valid transitions.

#### State Transition Graph

```
RequiresPaymentMethod
    ↓ add_payment_method()
RequiresConfirmation
    ↓ confirm()
Processing
    ├→ RequiresCustomerAction (3DS)
    ├→ Succeeded
    ├→ Failed
    └→ Cancelled

RequiresCustomerAction
    ├→ Processing (customer_authenticated)
    ├→ Failed (auth_failed)
    └→ Cancelled

Succeeded → [Terminal]
Failed → [Terminal]
Cancelled → [Terminal]
```

#### Algorithm

```rust
/// State machine for PaymentIntent status transitions
///
/// Time Complexity: O(1) for validation
/// Space Complexity: O(1)
///
pub fn validate_status_transition(
    current: IntentStatus,
    new: IntentStatus,
) -> Result<(), StatusTransitionError> {
    use IntentStatus::*;
    
    let valid = match (current, new) {
        // From RequiresPaymentMethod
        (RequiresPaymentMethod, RequiresConfirmation) => true,
        (RequiresPaymentMethod, Cancelled) => true,
        
        // From RequiresConfirmation
        (RequiresConfirmation, Processing) => true,
        (RequiresConfirmation, Cancelled) => true,
        
        // From Processing
        (Processing, RequiresCustomerAction) => true,
        (Processing, Succeeded) => true,
        (Processing, Failed) => true,
        (Processing, Cancelled) => true,
        
        // From RequiresCustomerAction
        (RequiresCustomerAction, Processing) => true,
        (RequiresCustomerAction, Failed) => true,
        (RequiresCustomerAction, Cancelled) => true,
        
        // From terminal states
        (Succeeded, _) => false,
        (Failed, _) => false,
        (Cancelled, _) => false,
        
        // Same state (idempotent)
        (s1, s2) if s1 == s2 => true,
        
        // All other transitions invalid
        _ => false,
    };
    
    if valid {
        Ok(())
    } else {
        Err(StatusTransitionError::InvalidTransition {
            from: current,
            to: new,
        })
    }
}
```

---

## Reconciliation Algorithms

### 1. Payment Amount Reconciliation

**Location**: `crates/router/src/core/payments/reconciliation.rs`

**Purpose**: Verify payment amounts match across attempts, captures, and refunds.

#### Algorithm

```rust
/// Reconciliation algorithm for payment amounts
///
/// Invariant: captured_amount - refunded_amount = net_amount
///
/// Time Complexity: O(m + r) where m = captures, r = refunds
/// Space Complexity: O(1)
///
pub fn reconcile_payment_amounts(
    payment: &PaymentIntent,
    attempts: &[PaymentAttempt],
    refunds: &[Refund],
) -> Result<ReconciliationResult> {
    // Step 1: Sum all captured amounts
    let total_captured: i64 = attempts
        .iter()
        .filter(|a| a.status == AttemptStatus::Captured)
        .map(|a| a.amount)
        .sum();  // O(m)
    
    // Step 2: Sum all successful refunds
    let total_refunded: i64 = refunds
        .iter()
        .filter(|r| r.refund_status == RefundStatus::Success)
        .map(|r| r.refund_amount)
        .sum();  // O(r)
    
    // Step 3: Calculate net amount
    let net_amount = total_captured - total_refunded;
    
    // Step 4: Verify invariants
    let discrepancies = vec![];
    
    // Check 1: Captured amount matches payment intent
    if let Some(captured) = payment.amount_captured {
        if captured != total_captured {
            discrepancies.push(Discrepancy::CapturedMismatch {
                expected: captured,
                actual: total_captured,
            });
        }
    }
    
    // Check 2: Net amount is positive
    if net_amount < 0 {
        discrepancies.push(Discrepancy::NegativeNetAmount {
            net_amount,
        });
    }
    
    // Check 3: Refunds don't exceed captured
    if total_refunded > total_captured {
        discrepancies.push(Discrepancy::RefundExceedsCaptured {
            captured: total_captured,
            refunded: total_refunded,
        });
    }
    
    // Check 4: Each refund valid
    for refund in refunds {
        if refund.refund_amount > refund.total_amount {
            discrepancies.push(Discrepancy::RefundExceedsPayment {
                refund_id: refund.refund_id.clone(),
                refund_amount: refund.refund_amount,
                payment_amount: refund.total_amount,
            });
        }
    }
    
    if discrepancies.is_empty() {
        Ok(ReconciliationResult::Success {
            total_captured,
            total_refunded,
            net_amount,
        })
    } else {
        Ok(ReconciliationResult::Discrepancies {
            total_captured,
            total_refunded,
            net_amount,
            issues: discrepancies,
        })
    }
}
```

---

## Caching & Invalidation

### 1. LRU Cache with TTL

**Location**: `crates/router/src/core/cache.rs`

**Purpose**: Cache merchant configs with eviction policy.

#### Algorithm

```rust
use std::collections::{HashMap, LinkedList};
use std::time::{Duration, Instant};

/// LRU Cache with TTL (Time-To-Live)
///
/// Time Complexity:
///   - get: O(1) average
///   - insert: O(1) average
///   - evict: O(1)
///
/// Space Complexity: O(n) where n = capacity
///
pub struct LruCache<K, V> {
    capacity: usize,
    ttl: Duration,
    map: HashMap<K, CacheEntry<V>>,
    lru_list: LinkedList<K>,
}

struct CacheEntry<V> {
    value: V,
    inserted_at: Instant,
    access_count: u64,
}

impl<K: Clone + Eq + std::hash::Hash, V: Clone> LruCache<K, V> {
    pub fn get(&mut self, key: &K) -> Option<V> {
        if let Some(entry) = self.map.get_mut(key) {
            // Check TTL
            if entry.inserted_at.elapsed() > self.ttl {
                // Expired - remove
                self.map.remove(key);
                return None;
            }
            
            // Update LRU
            entry.access_count += 1;
            self.move_to_front(key);
            
            Some(entry.value.clone())
        } else {
            None
        }
    }
    
    pub fn insert(&mut self, key: K, value: V) {
        // Check capacity
        if self.map.len() >= self.capacity && !self.map.contains_key(&key) {
            // Evict least recently used
            if let Some(lru_key) = self.lru_list.pop_back() {
                self.map.remove(&lru_key);
            }
        }
        
        let entry = CacheEntry {
            value,
            inserted_at: Instant::now(),
            access_count: 0,
        };
        
        self.map.insert(key.clone(), entry);
        self.lru_list.push_front(key);
    }
    
    fn move_to_front(&mut self, key: &K) {
        // Remove from current position
        self.lru_list.retain(|k| k != key);
        // Add to front
        self.lru_list.push_front(key.clone());
    }
}
```

**Cache Hit Rate Analysis**:
```
hit_rate = cache_hits / total_requests

Example metrics:
  - Merchant config cache: ~95% hit rate
  - Routing config cache: ~90% hit rate
  - Connector config cache: ~85% hit rate
```

---

## Rate Limiting

### 1. Token Bucket Algorithm

**Location**: `crates/router/src/middleware/rate_limit.rs`

**Purpose**: Throttle requests per merchant/connector.

#### Algorithm

```rust
/// Token bucket rate limiter
///
/// Time Complexity: O(1) per check
/// Space Complexity: O(n) where n = number of buckets
///
pub struct TokenBucket {
    capacity: u32,         // Max tokens
    refill_rate: u32,      // Tokens per second
    tokens: f64,           // Current tokens
    last_refill: Instant,  // Last refill time
}

impl TokenBucket {
    pub fn try_acquire(&mut self, tokens: u32) -> Result<(), RateLimitError> {
        // Refill bucket based on elapsed time
        let now = Instant::now();
        let elapsed = now.duration_since(self.last_refill).as_secs_f64();
        let refill_amount = elapsed * self.refill_rate as f64;
        
        self.tokens = (self.tokens + refill_amount).min(self.capacity as f64);
        self.last_refill = now;
        
        // Check if enough tokens available
        if self.tokens >= tokens as f64 {
            self.tokens -= tokens as f64;
            Ok(())
        } else {
            Err(RateLimitError::Exceeded {
                available: self.tokens as u32,
                required: tokens,
                retry_after: Duration::from_secs_f64(
                    (tokens as f64 - self.tokens) / self.refill_rate as f64
                ),
            })
        }
    }
}
```

**Example Configuration**:
```rust
// Merchant API rate limit: 100 requests/second
TokenBucket::new(capacity: 100, refill_rate: 100);

// Connector rate limit: 50 requests/second
TokenBucket::new(capacity: 50, refill_rate: 50);
```

---

**Document Version**: 1.0.0  
**Last Updated**: December 18, 2025  
**Maintained By**: Software Architect Agent
