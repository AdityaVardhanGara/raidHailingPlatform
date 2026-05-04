# Low-Level Design: Ride Service

## Service Overview

**Purpose:** Orchestrates ride request handling, validation, fare estimation, and dispatch initiation.

**Characteristics:**
- Stateless (no persistence in hot path)
- Low latency (<100ms p95)
- High throughput (1000 requests/sec per instance)
- Orchestrator pattern (delegates to specialized services)

---

## Component Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                      Ride Service                            │
│                                                               │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐      │
│  │   Request    │  │  Validation  │  │   Pricing    │      │
│  │   Handler    │→ │   Engine     │→ │   Client     │      │
│  └──────────────┘  └──────────────┘  └──────────────┘      │
│         │                                     │               │
│         ↓                                     ↓               │
│  ┌──────────────┐                    ┌──────────────┐      │
│  │   Dispatch   │                    │    Event     │      │
│  │   Client     │                    │   Publisher  │      │
│  └──────────────┘                    └──────────────┘      │
│         │                                     │               │
└─────────┼─────────────────────────────────────┼─────────────┘
          │                                     │
          ↓                                     ↓
   Dispatch Service                        Kafka
```

### Components

#### 1. Request Handler
- HTTP endpoint exposure
- Request deserialization
- Trace context propagation
- Response serialization

#### 2. Validation Engine
- Schema validation (pickup, destination, tier)
- Business rules (service area, user eligibility, payment method)
- Idempotency check (deduplicate retry requests)

#### 3. Pricing Client
- Fare estimation (base + surge multiplier)
- Surge multiplier fetch (by geo-cell)
- Fallback to base fare if pricing service unavailable

#### 4. Dispatch Client
- Dispatch request submission
- Timeout handling (1s)
- Retry logic (1 retry with 100ms backoff)

#### 5. Event Publisher
- Kafka event emission (`ride.requested`, `ride.failed`)
- Fire-and-forget (non-blocking)
- Dead-letter queue for failed publishes

---

## Data Flow

### Happy Path: Ride Request

```
1. User App → API Gateway → Ride Service
   POST /v1/rides
   {
     "rider_id": "u123",
     "pickup": {"lat": 37.7749, "lng": -122.4194},
     "destination": {"lat": 37.8044, "lng": -122.2712},
     "tier": "economy",
     "payment_method_id": "pm_456"
   }

2. Ride Service (Request Handler)
   - Generate ride_id (UUID)
   - Extract trace_id from headers
   - Log: "Ride request received"

3. Ride Service (Validation Engine)
   - Check idempotency key (Redis: GET idempotency:{key})
   - Validate schema (pickup/destination required, tier in [economy, premium, luxury])
   - Check service area (geo-fence lookup)
   - Validate user eligibility (not banned, payment method valid)

4. Ride Service (Pricing Client)
   - Calculate geo-cell (H3 index at resolution 7)
   - Fetch surge multiplier (HTTP GET /v1/pricing/surge?cell={h3_index})
   - Estimate fare:
     fare_estimate = base_fare(distance, tier) * surge_multiplier
   - Cache response (Redis: SET fare:{ride_id} {fare_estimate} EX 300)

5. Ride Service (Dispatch Client)
   - Submit dispatch request:
     POST /v1/dispatch/match
     {
       "ride_id": "r789",
       "rider_id": "u123",
       "pickup": {...},
       "tier": "economy",
       "max_wait_time_sec": 120
     }
   - Timeout: 1s
   - Response:
     {
       "status": "MATCHING",
       "estimated_wait_sec": 90
     }

6. Ride Service (Event Publisher)
   - Publish to Kafka:
     Topic: ride.requested
     Key: ride_id
     Value: {
       "ride_id": "r789",
       "rider_id": "u123",
       "pickup": {...},
       "destination": {...},
       "tier": "economy",
       "fare_estimate": 12.50,
       "surge_multiplier": 1.2,
       "timestamp": "2026-05-04T10:30:45Z"
     }

7. Ride Service → API Gateway → User App
   HTTP 202 Accepted
   {
     "ride_id": "r789",
     "status": "MATCHING",
     "fare_estimate": 12.50,
     "estimated_wait_sec": 90
   }
```

**Latency Breakdown:**
- Request validation: 10ms
- Pricing fetch: 30ms (with cache)
- Dispatch submission: 50ms
- Event publish: 5ms (async)
- **Total: ~95ms**

### Error Paths

#### 1. Invalid Request

```
- Validation fails (missing pickup, invalid tier)
- HTTP 400 Bad Request
- No Kafka event
- Log: "Ride request validation failed"
```

#### 2. User Not Eligible

```
- User banned or payment method invalid
- HTTP 403 Forbidden
- Kafka event: ride.rejected (reason: "user_ineligible")
```

#### 3. Pricing Service Down

```
- Circuit breaker open or timeout
- Fallback: Use base fare without surge (surge_multiplier = 1.0)
- Log: "Pricing service unavailable, using base fare"
- HTTP 202 (degraded mode)
```

#### 4. Dispatch Service Timeout

```
- Dispatch request times out (>1s)
- Retry once (with new idempotency key)
- If retry fails:
  - HTTP 503 Service Unavailable
  - Kafka event: ride.failed (reason: "dispatch_timeout")
  - User sees: "High demand, please try again"
```

---

## API Specification

### POST /v1/rides

**Request:**
```json
{
  "rider_id": "string (UUID)",
  "pickup": {
    "lat": "float (-90 to 90)",
    "lng": "float (-180 to 180)",
    "address": "string (optional)"
  },
  "destination": {
    "lat": "float",
    "lng": "float",
    "address": "string (optional)"
  },
  "tier": "enum (economy, premium, luxury)",
  "payment_method_id": "string",
  "promo_code": "string (optional)",
  "scheduled_time": "ISO8601 (optional, for future rides)"
}
```

**Headers:**
```
Authorization: Bearer {jwt_token}
X-Idempotency-Key: {uuid}
X-Trace-Id: {trace_id}
```

**Response (202 Accepted):**
```json
{
  "ride_id": "string (UUID)",
  "status": "MATCHING",
  "fare_estimate": "float",
  "surge_multiplier": "float",
  "estimated_wait_sec": "int",
  "created_at": "ISO8601"
}
```

**Error Responses:**

- **400 Bad Request:** Invalid schema
  ```json
  {
    "error": "validation_error",
    "message": "Invalid tier. Must be one of: economy, premium, luxury",
    "field": "tier"
  }
  ```

- **403 Forbidden:** User not eligible
  ```json
  {
    "error": "forbidden",
    "message": "Payment method declined. Please update your payment details."
  }
  ```

- **429 Too Many Requests:** Rate limit exceeded
  ```json
  {
    "error": "rate_limit_exceeded",
    "message": "Too many requests. Please try again in 60 seconds.",
    "retry_after_sec": 60
  }
  ```

- **503 Service Unavailable:** Dispatch unavailable
  ```json
  {
    "error": "service_unavailable",
    "message": "High demand in your area. Please try again shortly.",
    "retry_after_sec": 10
  }
  ```

### GET /v1/rides/{ride_id}

**Response (200 OK):**
```json
{
  "ride_id": "r789",
  "status": "ASSIGNED | MATCHING | CANCELLED | EXPIRED",
  "rider_id": "u123",
  "driver_id": "d456 (if assigned)",
  "pickup": {...},
  "destination": {...},
  "fare_estimate": 12.50,
  "created_at": "2026-05-04T10:30:45Z",
  "updated_at": "2026-05-04T10:31:20Z"
}
```

### POST /v1/rides/{ride_id}/cancel

**Response (200 OK):**
```json
{
  "ride_id": "r789",
  "status": "CANCELLED",
  "cancellation_fee": 0.0,
  "cancelled_at": "2026-05-04T10:32:00Z"
}
```

---

## Event Schemas

### ride.requested

**Topic:** `ride.requested`
**Partition Key:** `ride_id`

```json
{
  "event_id": "uuid",
  "event_type": "ride.requested",
  "timestamp": "ISO8601",
  "version": "1.0",
  "data": {
    "ride_id": "uuid",
    "rider_id": "uuid",
    "pickup": {
      "lat": 37.7749,
      "lng": -122.4194,
      "address": "123 Market St, San Francisco, CA"
    },
    "destination": {
      "lat": 37.8044,
      "lng": -122.2712,
      "address": "456 Broadway, Oakland, CA"
    },
    "tier": "economy",
    "payment_method_id": "pm_456",
    "fare_estimate": 12.50,
    "surge_multiplier": 1.2,
    "promo_code": null,
    "scheduled_time": null,
    "region": "us-west-1"
  }
}
```

### ride.failed

**Topic:** `ride.failed`
**Partition Key:** `ride_id`

```json
{
  "event_id": "uuid",
  "event_type": "ride.failed",
  "timestamp": "ISO8601",
  "version": "1.0",
  "data": {
    "ride_id": "uuid",
    "rider_id": "uuid",
    "reason": "dispatch_timeout | validation_failed | user_ineligible",
    "error_message": "Dispatch service unavailable",
    "retry_allowed": true
  }
}
```

### ride.cancelled

**Topic:** `ride.cancelled`
**Partition Key:** `ride_id`

```json
{
  "event_id": "uuid",
  "event_type": "ride.cancelled",
  "timestamp": "ISO8601",
  "version": "1.0",
  "data": {
    "ride_id": "uuid",
    "rider_id": "uuid",
    "cancelled_by": "rider | driver | system",
    "reason": "user_request | no_drivers_available | timeout",
    "cancellation_fee": 0.0
  }
}
```

---

## Data Models

### In-Memory Models (Go structs)

```go
type RideRequest struct {
    RiderID          string       `json:"rider_id" validate:"required,uuid"`
    Pickup           Location     `json:"pickup" validate:"required"`
    Destination      Location     `json:"destination" validate:"required"`
    Tier             RideTier     `json:"tier" validate:"required,oneof=economy premium luxury"`
    PaymentMethodID  string       `json:"payment_method_id" validate:"required"`
    PromoCode        *string      `json:"promo_code,omitempty"`
    ScheduledTime    *time.Time   `json:"scheduled_time,omitempty"`
}

type Location struct {
    Lat     float64 `json:"lat" validate:"required,min=-90,max=90"`
    Lng     float64 `json:"lng" validate:"required,min=-180,max=180"`
    Address *string `json:"address,omitempty"`
}

type RideTier string

const (
    TierEconomy RideTier = "economy"
    TierPremium RideTier = "premium"
    TierLuxury  RideTier = "luxury"
)

type RideResponse struct {
    RideID           string    `json:"ride_id"`
    Status           string    `json:"status"`
    FareEstimate     float64   `json:"fare_estimate"`
    SurgeMultiplier  float64   `json:"surge_multiplier"`
    EstimatedWaitSec int       `json:"estimated_wait_sec"`
    CreatedAt        time.Time `json:"created_at"`
}
```

### Redis Cache Keys

```
# Idempotency tracking
idempotency:{idempotency_key} -> {ride_id}
TTL: 24 hours

# Fare estimation cache
fare:{ride_id} -> {"estimate": 12.50, "surge": 1.2}
TTL: 5 minutes

# User eligibility cache
user:eligible:{user_id} -> {"is_eligible": true, "payment_valid": true}
TTL: 1 hour
```

---

## Scaling

### Horizontal Scaling

**Trigger:** CPU > 70% or p95 latency > 200ms

**Autoscaling Policy:**
```yaml
minReplicas: 20
maxReplicas: 200
metrics:
- type: Resource
  resource:
    name: cpu
    target:
      type: Utilization
      averageUtilization: 70
- type: Pods
  pods:
    metric:
      name: http_request_latency_p95
    target:
      type: AverageValue
      averageValue: 200ms
```

**Load Balancing:** Round-robin (stateless service)

### Vertical Scaling

**Instance Size:**
- **CPU:** 4 vCPU
- **Memory:** 8GB
- **Network:** 10 Gbps

**Capacity per Instance:**
- 100 requests/sec
- 6000 requests/min
- 10k concurrent connections

### Regional Distribution

| Region | Instances | Peak RPS |
|--------|-----------|----------|
| US-East | 40 | 4000 |
| US-West | 30 | 3000 |
| EU-West | 25 | 2500 |
| AP-South | 20 | 2000 |
| AP-Southeast | 15 | 1500 |

---

## Resilience

### Retry Policy

**Dispatch Client:**
```go
retryPolicy := retry.Policy{
    MaxAttempts:      2,
    InitialBackoff:   100 * time.Millisecond,
    MaxBackoff:       500 * time.Millisecond,
    BackoffMultiplier: 2.0,
    Jitter:           0.2,
    RetryableErrors: []error{
        errors.Timeout,
        errors.ServiceUnavailable,
    },
}
```

**Event Publisher:**
```go
// Fire-and-forget with DLQ
retryPolicy := retry.Policy{
    MaxAttempts:      3,
    InitialBackoff:   50 * time.Millisecond,
    MaxBackoff:       200 * time.Millisecond,
    OnFailure: func(event Event) {
        dlq.Push(event) // Dead-letter queue
    },
}
```

### Circuit Breaker

**Pricing Client:**
```yaml
failureRateThreshold: 50%
slowCallRateThreshold: 80%
slowCallDurationThreshold: 200ms
waitDurationInOpenState: 10s
permittedNumberOfCallsInHalfOpenState: 5
slidingWindowSize: 100
```

**Fallback:**
```go
func (c *PricingClient) GetSurgeMultiplier(ctx context.Context, geoCell string) (float64, error) {
    multiplier, err := c.httpClient.Get(ctx, "/v1/pricing/surge", geoCell)
    if err != nil {
        log.Warn("Pricing service unavailable, using base fare")
        return 1.0, nil // Fallback to no surge
    }
    return multiplier, nil
}
```

### Timeouts

| Operation | Timeout |
|-----------|---------|
| HTTP request handling | 5s |
| Pricing service call | 200ms |
| Dispatch service call | 1s |
| Redis operation | 50ms |
| Kafka publish | 100ms |

### Backpressure

**Rate Limiting:**
```yaml
# Per user (via API Gateway)
user_rate_limit: 10 requests/min

# Per region (via Ride Service)
region_rate_limit: 10000 requests/sec
```

**Queue Management:**
```go
// In-memory request queue
requestQueue := queue.New(maxSize: 5000)

if requestQueue.IsFull() {
    return errors.ServiceUnavailable("System at capacity")
}
```

---

## Monitoring

### Key Metrics

```
# Latency (histogram)
ride_service_request_duration_seconds{endpoint, status}

# Throughput (counter)
ride_service_requests_total{endpoint, status}

# Error rate (counter)
ride_service_errors_total{endpoint, error_type}

# Dependency health (gauge)
ride_service_dependency_available{service="pricing|dispatch"}

# Cache performance (counter)
ride_service_cache_hits_total
ride_service_cache_misses_total
```

### Alerts

**Critical (P0):**
- `ride_service_request_duration_seconds{p95} > 0.5s` for 2 minutes
- `ride_service_errors_total / ride_service_requests_total > 0.05` for 3 minutes

**Warning (P1):**
- `ride_service_request_duration_seconds{p95} > 0.3s` for 5 minutes
- `ride_service_cache_hit_rate < 0.8` for 10 minutes

### Logging

**Log Levels:**

```go
// ERROR: Dispatch submission failed after retries
log.Error("Failed to submit dispatch request",
    "ride_id", rideID,
    "error", err,
    "attempts", 2)

// WARN: Circuit breaker opened for pricing service
log.Warn("Pricing service circuit breaker opened",
    "service", "pricing",
    "failure_rate", 0.52)

// INFO: Ride request processed successfully
log.Info("Ride request processed",
    "ride_id", rideID,
    "rider_id", riderID,
    "latency_ms", 95)

// DEBUG: Cache miss for surge multiplier
log.Debug("Cache miss for surge multiplier",
    "geo_cell", geoCell)
```

---

## Testing Strategy

### Unit Tests

**Coverage Target:** 90%

**Key Test Cases:**
1. Request validation (valid/invalid schemas)
2. Fare calculation (with/without surge)
3. Idempotency key handling (duplicate requests)
4. Error handling (pricing down, dispatch timeout)
5. Fallback behaviors (circuit breaker open)

### Integration Tests

**Test Scenarios:**
1. End-to-end ride request flow (mock dependencies)
2. Kafka event publishing verification
3. Redis cache hit/miss scenarios
4. Timeout and retry behaviors

### Load Tests

**Tool:** k6

**Test Plan:**
```javascript
export let options = {
  stages: [
    { duration: '2m', target: 1000 },  // Ramp up
    { duration: '5m', target: 1000 },  // Sustained load
    { duration: '2m', target: 2000 },  // Peak load
    { duration: '1m', target: 0 },     // Ramp down
  ],
  thresholds: {
    'http_req_duration': ['p(95)<200'],
    'http_req_failed': ['rate<0.01'],
  },
};

export default function() {
  http.post('https://api.example.com/v1/rides', JSON.stringify({
    rider_id: __VU,
    pickup: { lat: 37.7749, lng: -122.4194 },
    destination: { lat: 37.8044, lng: -122.2712 },
    tier: 'economy',
    payment_method_id: 'pm_test',
  }), {
    headers: {
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${token}`,
    },
  });
  sleep(1);
}
```

---

## Security

### Input Validation

```go
// Strict schema validation
validator := validator.New()
if err := validator.Struct(rideRequest); err != nil {
    return errors.BadRequest("Invalid request", err)
}

// SQL injection prevention (none, no DB queries)
// NoSQL injection prevention (parameterized Redis commands)

// XSS prevention (API only, no HTML rendering)
```

### Authentication

```go
// JWT validation
token := extractToken(r.Header.Get("Authorization"))
claims, err := jwt.Verify(token, publicKey)
if err != nil {
    return errors.Unauthorized("Invalid token")
}

// Extract user_id from claims
userID := claims["sub"].(string)
```

### Rate Limiting

```go
// Per-user rate limiting (via API Gateway)
if !rateLimiter.Allow(userID) {
    return errors.TooManyRequests("Rate limit exceeded")
}
```

### Data Privacy

```
# PII handling
- Phone numbers: hashed in logs
- Addresses: lat/lng only (no street addresses in logs)
- Payment methods: token IDs only (never full card numbers)
```

---

## Dependencies

### External Services

| Service | SLA | Timeout | Fallback |
|---------|-----|---------|----------|
| Pricing Service | 99.9% | 200ms | Base fare (surge=1.0) |
| Dispatch Service | 99.95% | 1s | Queue for retry |
| UUID Service | 99.99% | 50ms | Local UUID generation |

### Infrastructure

- **Redis:** 99.99% (AWS ElastiCache)
- **Kafka:** 99.95% (MSK)
- **API Gateway:** 99.99% (AWS ALB)

---

## Deployment

### Container Specification

```dockerfile
FROM golang:1.22-alpine AS builder
WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN go build -o ride-service cmd/ride-service/main.go

FROM alpine:3.19
RUN apk --no-cache add ca-certificates
COPY --from=builder /app/ride-service /usr/local/bin/
EXPOSE 8080
ENTRYPOINT ["ride-service"]
```

### Kubernetes Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ride-service
spec:
  replicas: 20
  selector:
    matchLabels:
      app: ride-service
  template:
    metadata:
      labels:
        app: ride-service
    spec:
      containers:
      - name: ride-service
        image: ride-service:v1.2.3
        ports:
        - containerPort: 8080
        env:
        - name: PRICING_SERVICE_URL
          value: "http://pricing-service:8080"
        - name: DISPATCH_SERVICE_URL
          value: "http://dispatch-service:8080"
        - name: REDIS_URL
          valueFrom:
            secretKeyRef:
              name: redis-credentials
              key: url
        - name: KAFKA_BROKERS
          value: "kafka-1:9092,kafka-2:9092,kafka-3:9092"
        resources:
          requests:
            cpu: 2000m
            memory: 4Gi
          limits:
            cpu: 4000m
            memory: 8Gi
        livenessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /ready
            port: 8080
          initialDelaySeconds: 10
          periodSeconds: 5
```

---

## Future Enhancements

1. **ML-based fare prediction:** Train model on historical ride data
2. **Smart retry logic:** Adaptive backoff based on load
3. **A/B testing framework:** Feature flag integration for experiments
4. **GraphQL API:** Consolidated endpoint for mobile apps
5. **WebSocket support:** Real-time ride status updates without polling

---

## References

- HLD Overview: `HLD_Overview.md`
- API Documentation: `APIs_and_Events.md`
- Dispatch Service LLD: `LLD_Dispatch_Service.md`
- Data Models: `Data_Models.md`
