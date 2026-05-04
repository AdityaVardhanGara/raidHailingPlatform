# High-Level Design: Multi-Region Ride-Hailing Platform

## Executive Summary

A globally distributed, multi-tenant ride-hailing platform designed to handle 300k concurrent drivers, 60k ride requests/minute, and 500k location updates/second with sub-second dispatch latency.

**Architecture Philosophy:**
- Low-latency matching via Redis + stateless services
- Strong consistency via DB-backed trip/payment services
- Scalability via event-driven architecture

---

## System Architecture Overview

### Architecture Diagram Reference
See: `system-architecture-diagram.png` (your HLD diagram)

### Core Design Principles

1. **Regional Isolation**: Each region operates independently with local writes, no cross-region sync on hot path
2. **Event-Driven**: Kafka/Pulsar backbone for decoupling and scalability
3. **Cache-First**: Redis for sub-millisecond reads on hot path
4. **Idempotency**: All APIs idempotent for flaky mobile networks
5. **Graceful Degradation**: Circuit breakers and fallbacks at every boundary

---

## Service Inventory

| Service | Latency Tier | Consistency Model | Primary Storage |
|---------|--------------|-------------------|-----------------|
| API Gateway | <50ms | Stateless | None |
| Ride Service | <100ms | Eventual | None (orchestrator) |
| **Dispatch Service** | **<500ms** | **Eventual** | **Redis** |
| Driver Service | <200ms | Eventual | Redis + DB |
| Trip Service | <500ms | Strong (ACID) | PostgreSQL |
| Pricing Service | <200ms | Eventual | Redis + DB |
| Payment Service | <2s | Strong | PostgreSQL |
| Notification Service | <5s | Best-effort | Kafka |
| Admin Service | <1s | Strong | PostgreSQL |
| UUID Service | <10ms | N/A | None (Snowflake) |

---

## Data Flow Patterns

### Critical Path: Ride Request to Assignment

```
User → API Gateway → Ride Service → Dispatch Service → Driver Service → User
                          ↓              ↓
                    Pricing Service   Redis (geo-index)
                          ↓
                    Kafka (ride.requested)
```

**Timeline:**
- T+0ms: Request received at API Gateway
- T+50ms: Ride Service validates & fetches surge multiplier
- T+100ms: Dispatch Service queries Redis geo-index (GEORADIUS)
- T+300ms: Fan-out to top N drivers via Driver Service
- T+800ms: First acceptance received
- T+1000ms: Trip record created (PENDING → ASSIGNED)

### Asynchronous Flows

1. **Location Updates:**
   ```
   Driver App → API Gateway → Driver Service → Kafka (driver.location.updated) → Redis (GEOADD)
   ```

2. **Trip Completion:**
   ```
   Driver App → Trip Service (END_TRIP) → Kafka (trip.completed)
                                              ↓
                              [Payment Service, Pricing Service, Notification Service]
   ```

---

## Scaling Strategy

### Horizontal Scaling

| Service | Scaling Trigger | Target Metric | Auto-scale Range |
|---------|----------------|---------------|------------------|
| API Gateway | CPU > 70% | RPS/instance | 10-100 pods |
| Ride Service | Latency p95 > 200ms | Requests queued | 20-200 pods |
| Dispatch Service | Redis query latency | Active requests | 50-500 pods |
| Driver Service | Kafka lag > 1000 | Events/sec | 30-300 pods |

### Vertical Scaling

- **Redis Clusters**: Sharded by geo-hash prefix (100 shards per region)
- **Kafka Partitions**: 200 partitions for `driver.location.updated` (by driver_id hash)
- **Database**: Read replicas (1 primary + 3 replicas per region)

### Regional Deployment

- **Primary Regions**: US-East, US-West, EU-West, AP-South, AP-Southeast
- **Failover**: Standby region per primary (promotes in <30s)
- **Data Residency**: User/trip data stored in home region per GDPR/DPDP

---

## Storage Architecture

### Hot Storage (Redis)

**Clusters:**
1. **Geo-Cluster**: Driver locations (GEOSPATIAL index)
   - TTL: 30 seconds
   - Eviction: LRU
   - Size: ~10GB per region

2. **State-Cluster**: Driver availability, active trips
   - TTL: varies (5min - 2hrs)
   - Persistence: AOF (append-only file)

3. **Idempotency-Cluster**: Request deduplication keys
   - TTL: 24 hours
   - Eviction: TTL-based

**Sharding Strategy:** Consistent hashing by `driver_id` or `trip_id`

### Warm Storage (PostgreSQL/CockroachDB)

**Tables:**
- `users` (riders, drivers) - 50M rows
- `trips` (historical + active) - 500M rows/year
- `payments` (transactions) - 500M rows/year
- `pricing_policies` (surge configs) - 10K rows
- `feature_flags` (admin controls) - 500 rows

**Partitioning:**
- `trips`: RANGE partition by `created_at` (monthly)
- `payments`: HASH partition by `payment_id` (64 partitions)

**Indexes:**
- `trips`: B-tree on `rider_id`, `driver_id`, `status`, `created_at`
- `payments`: B-tree on `trip_id`, `status`, `created_at`

### Cold Storage (S3/GCS)

- Trip archives (>6 months old)
- Audit logs
- ML training datasets

---

## Trade-offs & Design Decisions

### 1. Redis vs. Database for Driver Locations

**Choice:** Redis (GEOSPATIAL)

| Aspect | Redis | PostgreSQL + PostGIS |
|--------|-------|---------------------|
| Latency | <1ms | 10-50ms |
| Scale | 500k writes/sec | 50k writes/sec |
| Consistency | Eventual | Strong |
| Complexity | Low | Medium |

**Trade-off:** Accepted eventual consistency (stale locations <2s old) for 50x latency improvement.

### 2. Kafka vs. Pulsar for Event Streaming

**Choice:** Kafka

| Aspect | Kafka | Pulsar |
|--------|-------|--------|
| Maturity | High | Medium |
| Throughput | 2M msgs/sec/broker | 1.5M msgs/sec/broker |
| Ops Complexity | Medium | High |
| Multi-tenancy | Manual | Built-in |

**Trade-off:** Chose operational simplicity and ecosystem maturity over native multi-tenancy.

### 3. Synchronous vs. Asynchronous Payment Processing

**Choice:** Hybrid

- **Pre-auth (sync):** Validate payment method before dispatch (adds ~200ms)
- **Capture (async):** Post-trip via Kafka event

**Trade-off:** Accepted 200ms dispatch delay to prevent unpaid rides (99.7% success rate).

### 4. Regional vs. Global User Database

**Choice:** Regional with async replication

**Trade-off:** Users can't seamlessly cross regions (edge case: <0.1% of trips) in exchange for 10x lower latency and GDPR compliance.

---

## Resilience & Failure Handling

### Retry Policies

| Service | Initial Backoff | Max Attempts | Jitter |
|---------|----------------|--------------|--------|
| API Gateway → Ride Service | 10ms | 2 | 20% |
| Dispatch → Driver Service | 50ms | 3 | 30% |
| Payment → PSP | 1s | 5 | 50% |
| Trip Service → Database | 100ms | 3 | 10% |

**Idempotency Keys:** All state-changing requests include `X-Idempotency-Key` header (UUID, 24hr TTL).

### Backpressure Mechanisms

1. **API Gateway:**
   - Rate limiting: 100 req/sec per user, 10k req/sec per region
   - Queue depth: 1000 requests (HTTP 429 if exceeded)

2. **Kafka Consumers:**
   - Max poll records: 500
   - Pause partitions if processing lag > 5000 messages

3. **Redis:**
   - Connection pool: 500 max connections per service instance
   - Command timeout: 100ms

### Circuit Breakers

**Configuration (using Resilience4j):**

```yaml
Dispatch Service → Driver Service:
  failure_rate_threshold: 50%
  slow_call_threshold: 80%
  wait_duration_in_open_state: 10s
  permitted_calls_in_half_open: 5

Payment Service → PSP:
  failure_rate_threshold: 30%
  wait_duration_in_open_state: 60s
  permitted_calls_in_half_open: 3
```

**Fallback Behaviors:**
- **Dispatch fails:** Queue request for manual reassignment (driver app shows delayed matching)
- **Pricing fails:** Use cached base fare (disable surge)
- **Payment fails:** Allow trip to complete, retry payment async (reconciliation job)
- **Notification fails:** Store in dead-letter queue (DLQ), retry for 7 days

### Failure Modes & Recovery

#### Scenario 1: Redis Cluster Down

**Impact:** No new dispatches, existing trips unaffected

**Detection:** Health check fails, Redis commands timeout

**Recovery:**
1. Automatic failover to standby Redis cluster (<5s)
2. If standby also down:
   - Dispatch Service falls back to PostgreSQL + PostGIS (degraded latency: 10-50ms)
   - Alert on-call engineer
3. Rebuild Redis from DB + Kafka event replay (15 minutes)

**Prevention:**
- Redis AOF persistence enabled
- Snapshot every 5 minutes
- Multi-AZ deployment

#### Scenario 2: Kafka Partition Leader Failure

**Impact:** Producer lag spike, consumer lag spike

**Detection:** Producer send timeout, consumer lag metric

**Recovery:**
1. Kafka controller elects new leader from ISR (<10s)
2. Producers/consumers automatically reconnect
3. If quorum lost (ISR < min.insync.replicas):
   - Dispatch Service buffers events in-memory (max 10k events)
   - Trip Service writes to database + async event emission

**Prevention:**
- min.insync.replicas = 2
- replication.factor = 3
- unclean.leader.election = false

#### Scenario 3: Payment PSP Outage

**Impact:** Trip completion blocked

**Detection:** Circuit breaker opens (3 consecutive failures)

**Recovery:**
1. Trip Service marks trip as COMPLETED_PENDING_PAYMENT
2. Payment Service polls PSP every 5 minutes (exponential backoff to 1 hour)
3. User sees "Payment processing" message
4. Reconciliation job runs hourly to retry failed payments

**Prevention:**
- Multi-PSP setup (primary + fallback)
- Pre-auth before trip starts
- Automatic failover to fallback PSP

#### Scenario 4: Database Primary Failure

**Impact:** No new trips can be created

**Detection:** Health check fails, write queries timeout

**Recovery:**
1. Automatic failover to read replica (promotes to primary in ~30s)
2. During failover:
   - Trip Service returns HTTP 503
   - Ride Service queues requests in-memory (max 5k)
   - User sees "High demand, retrying..."
3. DNS updates to point to new primary
4. Services reconnect automatically

**Prevention:**
- PostgreSQL synchronous replication (1 primary + 1 sync standby)
- Automated failover via Patroni or cloud provider HA
- Connection pooling (PgBouncer)

---

## Observability

### Metrics (Prometheus)

**Golden Signals:**
- **Latency:** p50, p95, p99 per service/endpoint
- **Traffic:** RPS per service
- **Errors:** Error rate (%) per service
- **Saturation:** CPU, memory, queue depth

**Custom Metrics:**
- `dispatch_latency_ms` (histogram)
- `active_drivers_count` (gauge)
- `ride_requests_per_minute` (counter)
- `surge_multiplier` (gauge by geo-cell)

### Logging (ELK Stack)

**Structured Logs (JSON):**
```json
{
  "timestamp": "2026-05-04T10:30:45.123Z",
  "service": "dispatch-service",
  "trace_id": "a1b2c3d4",
  "level": "INFO",
  "message": "Driver assigned",
  "driver_id": "d123",
  "ride_id": "r456",
  "latency_ms": 342
}
```

**Log Levels:**
- ERROR: Payment failures, dispatch timeouts, DB errors
- WARN: Circuit breaker opened, retry exhausted, high latency
- INFO: State transitions (ride requested, driver assigned, trip started)
- DEBUG: Cache hits/misses, Kafka offsets

### Tracing (Jaeger)

**Trace Propagation:** OpenTelemetry context (via HTTP headers)

**Critical Traces:**
1. Ride request → Driver assignment (target: <1s)
2. Trip start → Trip end → Payment capture (target: <5s post-trip)
3. Location update → Redis update (target: <50ms)

### Alerting (PagerDuty)

**P0 (Page immediately):**
- Dispatch latency p95 > 2s for 2 minutes
- Error rate > 5% for 5 minutes
- Redis/Kafka/DB cluster down

**P1 (Alert on-call):**
- Dispatch latency p95 > 1.5s for 5 minutes
- Payment failure rate > 10% for 10 minutes

**P2 (Email):**
- Consumer lag > 10,000 messages for 15 minutes
- Cache hit rate < 80% for 30 minutes

---

## Security & Compliance

### Authentication & Authorization

- **User Auth:** JWT tokens (15min expiry, refresh tokens in DB)
- **Service-to-Service:** mTLS + API keys
- **Admin:** SSO (Okta) + MFA

### Data Protection

- **Encryption in Transit:** TLS 1.3
- **Encryption at Rest:** AES-256 (DB, S3, Redis AOF)
- **PII Masking:** Phone numbers, addresses logged as hashes

### Compliance

- **PCI DSS:** Payment data never stored, PSP tokenization
- **GDPR/DPDP:** Regional data residency, right to erasure (7-day SLA)
- **Audit Logs:** Immutable append-only logs retained for 7 years

---

## Capacity Planning

### Current Load (2026)

- 300k concurrent drivers
- 60k ride requests/min (1000/sec)
- 500k location updates/sec

### Projected Load (2027)

- 600k concurrent drivers (2x)
- 120k ride requests/min (2x)
- 1M location updates/sec (2x)

### Resource Requirements (per region, 2027)

| Component | Instances | vCPU | Memory | Storage |
|-----------|-----------|------|--------|---------|
| API Gateway | 50 | 2 | 4GB | - |
| Ride Service | 100 | 4 | 8GB | - |
| Dispatch Service | 250 | 8 | 16GB | - |
| Driver Service | 150 | 4 | 8GB | - |
| Redis Cluster | 100 nodes | 8 | 64GB | 1TB |
| Kafka Cluster | 30 brokers | 16 | 128GB | 10TB |
| PostgreSQL | 4 (1P+3R) | 32 | 256GB | 5TB |

**Estimated Cost:** $500k/month/region (AWS)

---

## Deployment Strategy

### CI/CD Pipeline

1. **Build:** Docker images pushed to ECR
2. **Test:** Unit tests (95% coverage), integration tests
3. **Deploy:** Kubernetes rolling update (10% canary, then 100%)
4. **Verify:** Automated smoke tests, metric dashboards

### Release Cadence

- **Hotfixes:** On-demand (0-2 hours)
- **Minor releases:** Twice per week (Tue/Thu)
- **Major releases:** Monthly (first Monday)

### Feature Flags

**LaunchDarkly integration:**
- `enable_surge_pricing_v2`
- `dispatch_algorithm_v3`
- `payment_provider_failover`

**Kill Switches:**
- `disable_new_rides` (emergency stop)
- `disable_surge_pricing` (PR nightmare mode)

---

## Next Steps

For detailed design of individual components:
- **Dispatch Service LLD:** See `LLD_Dispatch_Service.md`
- **Ride Service LLD:** See `LLD_Ride_Service.md`
- **APIs & Events:** See `APIs_and_Events.md`
- **Data Models:** See `Data_Models.md`

For failure scenario deep-dives:
- See `Resilience_Playbook.md`
