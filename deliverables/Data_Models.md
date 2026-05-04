# Data Models & Database Design

## Table of Contents
1. [Entity Relationship Diagram](#entity-relationship-diagram)
2. [PostgreSQL Schema](#postgresql-schema)
3. [Redis Data Structures](#redis-data-structures)
4. [Data Flow & Consistency](#data-flow--consistency)
5. [Partitioning & Sharding Strategy](#partitioning--sharding-strategy)
6. [Data Retention & Archival](#data-retention--archival)

---

## Entity Relationship Diagram

```
┌─────────────┐          ┌─────────────┐          ┌─────────────┐
│    Users    │          │   Drivers   │          │  Vehicles   │
│             │          │             │          │             │
│ user_id (PK)│◄────────┤ driver_id   │         ┌┤ vehicle_id  │
│ email       │          │ user_id(FK) ├────────┤ │ driver_id   │
│ phone       │          │ license_num │         │ │ make/model  │
│ created_at  │          │ status      │         │ │ plate       │
└─────────────┘          │ rating      │         │ └─────────────┘
       │                 └─────────────┘         │
       │                        │                │
       │                        │                │
       ├────────────────────────┴────────────────┤
       │                                          │
       ↓                                          ↓
┌─────────────┐          ┌─────────────┐   ┌─────────────┐
│    Rides    │          │    Trips    │   │  Payments   │
│             │          │             │   │             │
│ ride_id (PK)├─────────→│ trip_id (PK)├──→│ payment_id  │
│ rider_id    │          │ ride_id(FK) │   │ trip_id(FK) │
│ driver_id   │          │ driver_id   │   │ amount      │
│ pickup_lat  │          │ status      │   │ status      │
│ dest_lat    │          │ started_at  │   │ psp_txn_id  │
│ status      │          │ ended_at    │   │ created_at  │
│ tier        │          │ distance_km │   └─────────────┘
│ fare_est    │          │ fare        │
│ surge_mult  │          └─────────────┘
│ created_at  │
└─────────────┘
       │
       ↓
┌─────────────┐
│   Ratings   │
│             │
│ rating_id   │
│ trip_id(FK) │
│ rider_score │
│ driver_score│
│ created_at  │
└─────────────┘
```

---

## PostgreSQL Schema

### users

```sql
CREATE TABLE users (
    user_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email VARCHAR(255) NOT NULL UNIQUE,
    phone VARCHAR(20) NOT NULL UNIQUE,
    full_name VARCHAR(255) NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    role VARCHAR(20) NOT NULL CHECK (role IN ('rider', 'driver', 'admin')),
    status VARCHAR(20) NOT NULL DEFAULT 'active' CHECK (status IN ('active', 'suspended', 'banned')),
    profile_image_url TEXT,
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP NOT NULL DEFAULT NOW(),
    deleted_at TIMESTAMP
);

CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_users_phone ON users(phone);
CREATE INDEX idx_users_status ON users(status) WHERE deleted_at IS NULL;
```

---

### drivers

```sql
CREATE TABLE drivers (
    driver_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL UNIQUE REFERENCES users(user_id),
    license_number VARCHAR(50) NOT NULL UNIQUE,
    license_state VARCHAR(2) NOT NULL,
    license_expiry DATE NOT NULL,
    background_check_status VARCHAR(20) NOT NULL DEFAULT 'pending',
    background_check_date TIMESTAMP,
    status VARCHAR(20) NOT NULL DEFAULT 'offline' CHECK (status IN ('online', 'offline', 'busy')),
    rating DECIMAL(3,2) DEFAULT 5.00 CHECK (rating >= 0 AND rating <= 5),
    total_trips INTEGER DEFAULT 0,
    total_earnings DECIMAL(10,2) DEFAULT 0.00,
    home_region VARCHAR(50) NOT NULL,
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_drivers_user_id ON drivers(user_id);
CREATE INDEX idx_drivers_status ON drivers(status);
CREATE INDEX idx_drivers_rating ON drivers(rating);
CREATE INDEX idx_drivers_region ON drivers(home_region);
```

---

### vehicles

```sql
CREATE TABLE vehicles (
    vehicle_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    driver_id UUID NOT NULL REFERENCES drivers(driver_id),
    make VARCHAR(50) NOT NULL,
    model VARCHAR(50) NOT NULL,
    year INTEGER NOT NULL CHECK (year >= 2015 AND year <= EXTRACT(YEAR FROM NOW()) + 1),
    color VARCHAR(30) NOT NULL,
    plate_number VARCHAR(20) NOT NULL UNIQUE,
    vin VARCHAR(17) NOT NULL UNIQUE,
    registration_state VARCHAR(2) NOT NULL,
    registration_expiry DATE NOT NULL,
    insurance_policy_number VARCHAR(50) NOT NULL,
    insurance_expiry DATE NOT NULL,
    tier VARCHAR(20) NOT NULL CHECK (tier IN ('economy', 'premium', 'luxury')),
    seats INTEGER NOT NULL DEFAULT 4 CHECK (seats >= 2 AND seats <= 8),
    is_active BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_vehicles_driver_id ON vehicles(driver_id);
CREATE INDEX idx_vehicles_plate ON vehicles(plate_number);
CREATE INDEX idx_vehicles_tier ON vehicles(tier) WHERE is_active = TRUE;
```

---

### rides

```sql
CREATE TABLE rides (
    ride_id UUID PRIMARY KEY,
    rider_id UUID NOT NULL REFERENCES users(user_id),
    driver_id UUID REFERENCES drivers(driver_id),
    pickup_lat DECIMAL(10,8) NOT NULL,
    pickup_lng DECIMAL(11,8) NOT NULL,
    pickup_address TEXT,
    destination_lat DECIMAL(10,8) NOT NULL,
    destination_lng DECIMAL(11,8) NOT NULL,
    destination_address TEXT,
    tier VARCHAR(20) NOT NULL CHECK (tier IN ('economy', 'premium', 'luxury')),
    status VARCHAR(20) NOT NULL DEFAULT 'matching' CHECK (status IN (
        'matching', 'assigned', 'cancelled', 'expired', 'converted_to_trip'
    )),
    fare_estimate DECIMAL(10,2),
    surge_multiplier DECIMAL(4,2) DEFAULT 1.00,
    payment_method_id VARCHAR(100) NOT NULL,
    promo_code VARCHAR(20),
    scheduled_time TIMESTAMP,
    assigned_at TIMESTAMP,
    cancelled_at TIMESTAMP,
    cancellation_reason VARCHAR(100),
    cancelled_by VARCHAR(20) CHECK (cancelled_by IN ('rider', 'driver', 'system')),
    region VARCHAR(50) NOT NULL,
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_rides_rider_id ON rides(rider_id, created_at DESC);
CREATE INDEX idx_rides_driver_id ON rides(driver_id, created_at DESC) WHERE driver_id IS NOT NULL;
CREATE INDEX idx_rides_status ON rides(status, created_at DESC);
CREATE INDEX idx_rides_created_at ON rides(created_at DESC);
CREATE INDEX idx_rides_region ON rides(region, created_at DESC);

-- Partial index for active rides (hot queries)
CREATE INDEX idx_rides_active ON rides(ride_id, status) 
WHERE status IN ('matching', 'assigned');
```

---

### trips

```sql
CREATE TABLE trips (
    trip_id UUID PRIMARY KEY,
    ride_id UUID NOT NULL REFERENCES rides(ride_id),
    rider_id UUID NOT NULL REFERENCES users(user_id),
    driver_id UUID NOT NULL REFERENCES drivers(driver_id),
    vehicle_id UUID NOT NULL REFERENCES vehicles(vehicle_id),
    status VARCHAR(20) NOT NULL DEFAULT 'pending' CHECK (status IN (
        'pending', 'in_progress', 'completed', 'cancelled'
    )),
    start_lat DECIMAL(10,8),
    start_lng DECIMAL(11,8),
    start_odometer INTEGER,
    end_lat DECIMAL(10,8),
    end_lng DECIMAL(11,8),
    end_odometer INTEGER,
    distance_km DECIMAL(10,2),
    duration_sec INTEGER,
    base_fare DECIMAL(10,2),
    distance_fare DECIMAL(10,2),
    time_fare DECIMAL(10,2),
    surge_multiplier DECIMAL(4,2) DEFAULT 1.00,
    subtotal DECIMAL(10,2),
    fees DECIMAL(10,2) DEFAULT 0.00,
    promo_discount DECIMAL(10,2) DEFAULT 0.00,
    total_fare DECIMAL(10,2),
    started_at TIMESTAMP,
    completed_at TIMESTAMP,
    cancelled_at TIMESTAMP,
    cancellation_reason VARCHAR(100),
    region VARCHAR(50) NOT NULL,
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP NOT NULL DEFAULT NOW()
) PARTITION BY RANGE (created_at);

-- Partitions for trips (monthly)
CREATE TABLE trips_2026_05 PARTITION OF trips
    FOR VALUES FROM ('2026-05-01') TO ('2026-06-01');

CREATE TABLE trips_2026_06 PARTITION OF trips
    FOR VALUES FROM ('2026-06-01') TO ('2026-07-01');

-- Create partitions for next 12 months
-- (repeat pattern above)

CREATE INDEX idx_trips_rider_id ON trips(rider_id, created_at DESC);
CREATE INDEX idx_trips_driver_id ON trips(driver_id, created_at DESC);
CREATE INDEX idx_trips_ride_id ON trips(ride_id);
CREATE INDEX idx_trips_status ON trips(status, created_at DESC);
CREATE INDEX idx_trips_completed_at ON trips(completed_at DESC) WHERE completed_at IS NOT NULL;
CREATE INDEX idx_trips_region ON trips(region, created_at DESC);

-- Partial index for active trips
CREATE INDEX idx_trips_active ON trips(trip_id, status) 
WHERE status IN ('pending', 'in_progress');
```

---

### payments

```sql
CREATE TABLE payments (
    payment_id UUID PRIMARY KEY,
    trip_id UUID NOT NULL REFERENCES trips(trip_id),
    rider_id UUID NOT NULL REFERENCES users(user_id),
    driver_id UUID NOT NULL REFERENCES drivers(driver_id),
    amount DECIMAL(10,2) NOT NULL CHECK (amount >= 0),
    currency VARCHAR(3) NOT NULL DEFAULT 'USD',
    payment_method_id VARCHAR(100) NOT NULL,
    status VARCHAR(20) NOT NULL DEFAULT 'pending' CHECK (status IN (
        'pending', 'processing', 'succeeded', 'failed', 'refunded'
    )),
    psp_name VARCHAR(50) NOT NULL,
    psp_transaction_id VARCHAR(255),
    psp_charge_id VARCHAR(255),
    failure_code VARCHAR(50),
    failure_message TEXT,
    retry_count INTEGER DEFAULT 0,
    succeeded_at TIMESTAMP,
    failed_at TIMESTAMP,
    refunded_at TIMESTAMP,
    refund_amount DECIMAL(10,2),
    region VARCHAR(50) NOT NULL,
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP NOT NULL DEFAULT NOW()
) PARTITION BY HASH (payment_id);

-- Hash partitions for payments (64 partitions)
CREATE TABLE payments_p0 PARTITION OF payments FOR VALUES WITH (MODULUS 64, REMAINDER 0);
CREATE TABLE payments_p1 PARTITION OF payments FOR VALUES WITH (MODULUS 64, REMAINDER 1);
-- ... (repeat for p2 through p63)

CREATE INDEX idx_payments_trip_id ON payments(trip_id);
CREATE INDEX idx_payments_rider_id ON payments(rider_id, created_at DESC);
CREATE INDEX idx_payments_status ON payments(status, created_at DESC);
CREATE INDEX idx_payments_psp_txn_id ON payments(psp_transaction_id) WHERE psp_transaction_id IS NOT NULL;
CREATE INDEX idx_payments_created_at ON payments(created_at DESC);

-- Partial index for failed payments (retry queue)
CREATE INDEX idx_payments_failed ON payments(payment_id, retry_count, failed_at)
WHERE status = 'failed' AND retry_count < 5;
```

---

### ratings

```sql
CREATE TABLE ratings (
    rating_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    trip_id UUID NOT NULL UNIQUE REFERENCES trips(trip_id),
    rider_id UUID NOT NULL REFERENCES users(user_id),
    driver_id UUID NOT NULL REFERENCES drivers(driver_id),
    rider_score INTEGER CHECK (rider_score >= 1 AND rider_score <= 5),
    rider_comment TEXT,
    driver_score INTEGER CHECK (driver_score >= 1 AND driver_score <= 5),
    driver_comment TEXT,
    rider_rated_at TIMESTAMP,
    driver_rated_at TIMESTAMP,
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_ratings_trip_id ON ratings(trip_id);
CREATE INDEX idx_ratings_rider_id ON ratings(rider_id, created_at DESC);
CREATE INDEX idx_ratings_driver_id ON ratings(driver_id, created_at DESC);
```

---

### pricing_policies

```sql
CREATE TABLE pricing_policies (
    policy_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    region VARCHAR(50) NOT NULL,
    tier VARCHAR(20) NOT NULL CHECK (tier IN ('economy', 'premium', 'luxury')),
    base_fare DECIMAL(10,2) NOT NULL CHECK (base_fare >= 0),
    per_km_rate DECIMAL(10,2) NOT NULL CHECK (per_km_rate >= 0),
    per_minute_rate DECIMAL(10,2) NOT NULL CHECK (per_minute_rate >= 0),
    minimum_fare DECIMAL(10,2) NOT NULL CHECK (minimum_fare >= 0),
    cancellation_fee DECIMAL(10,2) NOT NULL DEFAULT 0.00,
    booking_fee DECIMAL(10,2) NOT NULL DEFAULT 0.00,
    service_fee_percent DECIMAL(5,2) NOT NULL DEFAULT 0.00 CHECK (service_fee_percent >= 0 AND service_fee_percent <= 100),
    effective_from TIMESTAMP NOT NULL DEFAULT NOW(),
    effective_until TIMESTAMP,
    is_active BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP NOT NULL DEFAULT NOW(),
    UNIQUE(region, tier, effective_from)
);

CREATE INDEX idx_pricing_policies_region_tier ON pricing_policies(region, tier, effective_from DESC)
WHERE is_active = TRUE;
```

---

### surge_multipliers

```sql
CREATE TABLE surge_multipliers (
    surge_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    geo_cell VARCHAR(20) NOT NULL, -- H3 cell ID
    region VARCHAR(50) NOT NULL,
    multiplier DECIMAL(4,2) NOT NULL DEFAULT 1.00 CHECK (multiplier >= 1.00 AND multiplier <= 5.00),
    supply_count INTEGER DEFAULT 0,
    demand_count INTEGER DEFAULT 0,
    valid_from TIMESTAMP NOT NULL DEFAULT NOW(),
    valid_until TIMESTAMP NOT NULL,
    created_at TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_surge_geo_cell ON surge_multipliers(geo_cell, valid_until DESC);
CREATE INDEX idx_surge_region ON surge_multipliers(region, valid_until DESC);
CREATE INDEX idx_surge_valid_until ON surge_multipliers(valid_until) WHERE valid_until > NOW();
```

---

### feature_flags

```sql
CREATE TABLE feature_flags (
    flag_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    flag_name VARCHAR(100) NOT NULL UNIQUE,
    description TEXT,
    is_enabled BOOLEAN DEFAULT FALSE,
    rollout_percentage INTEGER DEFAULT 0 CHECK (rollout_percentage >= 0 AND rollout_percentage <= 100),
    regions TEXT[], -- Array of regions where flag is active
    user_whitelist UUID[], -- Array of user IDs for testing
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_feature_flags_name ON feature_flags(flag_name);
CREATE INDEX idx_feature_flags_enabled ON feature_flags(is_enabled) WHERE is_enabled = TRUE;
```

---

### audit_logs

```sql
CREATE TABLE audit_logs (
    log_id BIGSERIAL PRIMARY KEY,
    entity_type VARCHAR(50) NOT NULL,
    entity_id UUID NOT NULL,
    action VARCHAR(50) NOT NULL,
    actor_id UUID,
    actor_role VARCHAR(20),
    changes JSONB,
    ip_address INET,
    user_agent TEXT,
    region VARCHAR(50),
    created_at TIMESTAMP NOT NULL DEFAULT NOW()
) PARTITION BY RANGE (created_at);

-- Partitions for audit logs (monthly)
CREATE TABLE audit_logs_2026_05 PARTITION OF audit_logs
    FOR VALUES FROM ('2026-05-01') TO ('2026-06-01');

CREATE INDEX idx_audit_logs_entity ON audit_logs(entity_type, entity_id, created_at DESC);
CREATE INDEX idx_audit_logs_actor ON audit_logs(actor_id, created_at DESC);
CREATE INDEX idx_audit_logs_created_at ON audit_logs(created_at DESC);
```

---

## Redis Data Structures

### 1. Driver Locations (GEOSPATIAL)

**Key Pattern:** `geo:drivers:{region}`

**Structure:** Sorted Set (Redis GEOSPATIAL)

```
GEOADD geo:drivers:us-west-1 -122.4194 37.7749 driver:d_660e8400

GEORADIUS geo:drivers:us-west-1 -122.4194 37.7749 5 km WITHDIST WITHCOORD ASC
```

**TTL:** None (updated continuously, removed on driver offline)

**Size Estimate:** 10GB per region (300k drivers × 40 bytes)

---

### 2. Driver Availability (HASH)

**Key Pattern:** `driver:{driver_id}:state`

**Structure:** Hash

```redis
HSET driver:d_660e8400:state
  status "online"
  active_ride_id ""
  tier "economy"
  rating "4.85"
  last_update "1746360645"
```

**TTL:** 5 minutes (refreshed on location update)

---

### 3. Active Rides (HASH)

**Key Pattern:** `ride:{ride_id}`

**Structure:** Hash

```redis
HSET ride:r_550e8400
  rider_id "550e8400-e29b-41d4-a716-446655440000"
  driver_id "d_660e8400-e29b-41d4-a716-446655440002"
  status "assigned"
  pickup_lat "37.7749"
  pickup_lng "-122.4194"
  created_at "1746360645"
```

**TTL:** 2 hours (deleted after trip conversion or expiry)

---

### 4. Active Trips (HASH)

**Key Pattern:** `trip:{trip_id}`

**Structure:** Hash

```redis
HSET trip:t_770e8400
  rider_id "550e8400"
  driver_id "d_660e8400"
  status "in_progress"
  started_at "1746360900"
```

**TTL:** 4 hours (deleted after completion)

---

### 5. Surge Cache (STRING)

**Key Pattern:** `surge:{geo_cell}`

**Structure:** String (JSON)

```redis
SET surge:8928308280fffff '{"multiplier":1.5,"supply":45,"demand":120,"valid_until":1746360945}'
```

**TTL:** 5 minutes (dynamically updated)

---

### 6. Fare Estimate Cache (STRING)

**Key Pattern:** `fare:{ride_id}`

**Structure:** String (JSON)

```redis
SET fare:r_550e8400 '{"estimate":12.50,"surge":1.2,"distance_km":15.2}' EX 300
```

**TTL:** 5 minutes

---

### 7. Idempotency Keys (STRING)

**Key Pattern:** `idempotency:{key}`

**Structure:** String (ride_id or response JSON)

```redis
SET idempotency:550e8400-key "r_550e8400-e29b-41d4-a716-446655440001" EX 86400
```

**TTL:** 24 hours

---

### 8. User Session (HASH)

**Key Pattern:** `session:{session_id}`

**Structure:** Hash

```redis
HSET session:sess_550e8400
  user_id "550e8400"
  device_id "device_123"
  fcm_token "fcm_abc123"
  last_activity "1746360645"
```

**TTL:** 30 days (rolling expiration on activity)

---

### 9. Rate Limiting (STRING)

**Key Pattern:** `ratelimit:{user_id}:{window}`

**Structure:** String (counter)

```redis
INCR ratelimit:550e8400:1746360600
EXPIRE ratelimit:550e8400:1746360600 60
```

**TTL:** 60 seconds (sliding window)

---

### 10. Dispatch Queue (LIST)

**Key Pattern:** `dispatch:queue:{geo_cell}`

**Structure:** List (FIFO)

```redis
LPUSH dispatch:queue:8928308280fffff '{"ride_id":"r_550e8400","priority":1,"created_at":1746360645}'
```

**TTL:** 10 minutes (auto-expire old requests)

---

## Data Flow & Consistency

### Write Path: Ride Request

```
1. API Gateway → Ride Service
   ↓
2. Ride Service writes to Kafka (ride.requested)
   ↓
3. Dispatch Service reads from Kafka
   ↓
4. Dispatch Service queries Redis GEORADIUS
   ↓
5. Dispatch Service updates Redis (ride:{ride_id})
   ↓
6. Driver accepts → Ride Service updates PostgreSQL (rides table)
   ↓
7. Ride Service publishes Kafka (ride.assigned)
```

**Consistency Model:** Eventual consistency (Redis → PostgreSQL)

**Reconciliation:** Hourly job syncs Redis state to PostgreSQL

---

### Write Path: Location Update

```
1. Driver App → API Gateway → Driver Service
   ↓
2. Driver Service publishes Kafka (driver.location.updated)
   ↓
3. Location Processor consumes Kafka
   ↓
4. Location Processor updates Redis GEOADD
```

**Consistency Model:** Best-effort (no durability guarantee)

**Data Loss Tolerance:** 2-second stale locations acceptable

---

### Write Path: Trip Completion

```
1. Driver App → API Gateway → Trip Service
   ↓
2. Trip Service writes to PostgreSQL (trips table)
   ↓
3. Trip Service publishes Kafka (trip.completed)
   ↓
4. Payment Service consumes Kafka → initiates payment
   ↓
5. Payment Service writes to PostgreSQL (payments table)
   ↓
6. Payment Service publishes Kafka (payment.captured)
```

**Consistency Model:** Strong consistency (ACID transactions)

**Idempotency:** All writes use idempotency keys

---

## Partitioning & Sharding Strategy

### PostgreSQL Partitioning

#### trips (RANGE by created_at)
- **Partition Key:** `created_at`
- **Partition Size:** 1 month
- **Retention:** 6 months hot, archive to S3 after

#### payments (HASH by payment_id)
- **Partition Key:** `payment_id`
- **Partition Count:** 64
- **Rationale:** Uniform distribution, no hotspots

#### audit_logs (RANGE by created_at)
- **Partition Key:** `created_at`
- **Partition Size:** 1 month
- **Retention:** 3 months in DB, move to cold storage after

---

### Redis Sharding

**Sharding Strategy:** Redis Cluster (16384 slots)

**Shard Allocation:**
- **Geo-cluster:** 100 shards (GEOSPATIAL data)
- **State-cluster:** 50 shards (driver/ride/trip state)
- **Cache-cluster:** 30 shards (surge, fare, idempotency)

**Key Prefix Routing:**
```
geo:drivers:{region} → Geo-cluster
driver:{driver_id} → State-cluster
surge:{geo_cell} → Cache-cluster
```

**Hot Key Mitigation:**
- Replicate frequently accessed keys (e.g., surge for busy areas)
- Use Redis Cluster client-side caching

---

### Database Connection Pooling

**PgBouncer Configuration:**
```
default_pool_size = 100
max_client_conn = 10000
pool_mode = transaction
```

**Connection Allocation:**
- Ride Service: 500 connections
- Trip Service: 1000 connections
- Payment Service: 500 connections
- Background jobs: 200 connections

---

## Data Retention & Archival

### Hot Data (PostgreSQL)

| Table | Retention | Archive After |
|-------|-----------|---------------|
| rides | 6 months | → S3 |
| trips | 6 months | → S3 |
| payments | 7 years | → Glacier (compliance) |
| ratings | 2 years | → S3 |
| audit_logs | 3 months | → S3 |

### Cold Storage (S3/Glacier)

**Archival Process:**
1. Nightly job exports old partitions to Parquet
2. Upload to S3 (Standard tier)
3. Lifecycle policy moves to Glacier after 90 days
4. DROP partition from PostgreSQL

**Query Access:**
- Cold data queryable via Athena/Presto
- 1-5 minute query latency (vs. milliseconds for hot data)

---

### Redis TTL Strategy

| Key Pattern | TTL | Eviction Policy |
|-------------|-----|-----------------|
| `geo:drivers:*` | None | Manual removal |
| `driver:*:state` | 5 min | TTL-based |
| `ride:*` | 2 hours | TTL-based |
| `trip:*` | 4 hours | TTL-based |
| `surge:*` | 5 min | TTL-based |
| `idempotency:*` | 24 hours | TTL-based |

**Eviction Policy:** `volatile-ttl` (evict keys with shortest TTL first)

---

## Data Migration Strategy

### Schema Changes

**Zero-Downtime Migration:**
1. Add new column (nullable)
2. Backfill data (batched, off-peak)
3. Update application code (read new column)
4. Deploy application
5. Update application code (write new column)
6. Deploy application
7. Mark old column as deprecated

**Example: Adding `tier` to `drivers` table**
```sql
-- Step 1
ALTER TABLE drivers ADD COLUMN tier VARCHAR(20);

-- Step 2 (backfill)
UPDATE drivers SET tier = 'economy' WHERE tier IS NULL;

-- Step 3 (after app deployment)
ALTER TABLE drivers ALTER COLUMN tier SET NOT NULL;
```

---

### Data Backfill

**Backfill Job Pattern:**
```python
def backfill_driver_tiers():
    batch_size = 1000
    offset = 0
    
    while True:
        drivers = db.query(
            "SELECT driver_id FROM drivers WHERE tier IS NULL LIMIT %s OFFSET %s",
            (batch_size, offset)
        )
        
        if not drivers:
            break
        
        for driver in drivers:
            tier = infer_tier(driver.vehicle_id)
            db.execute("UPDATE drivers SET tier = %s WHERE driver_id = %s", (tier, driver.driver_id))
        
        offset += batch_size
        time.sleep(1)  # Rate limiting
```

---

## Backup & Disaster Recovery

### PostgreSQL Backups

**Full Backup:** Daily (2 AM UTC)
**Incremental Backup:** Every 6 hours
**WAL Archival:** Continuous (1-minute lag)

**Retention:**
- Daily backups: 30 days
- Weekly backups: 12 weeks
- Monthly backups: 12 months

**RTO:** 15 minutes (promote replica)
**RPO:** 5 minutes (WAL replay)

---

### Redis Backups

**RDB Snapshots:** Every 6 hours
**AOF:** Enabled (fsync every second)

**Retention:**
- Snapshots: 7 days
- AOF: 24 hours

**RTO:** 10 minutes (restore from snapshot + AOF replay)
**RPO:** 1 second (AOF)

---

## Security

### Data Encryption

**At Rest:**
- PostgreSQL: AES-256 (Transparent Data Encryption)
- Redis: AES-256 (AWS ElastiCache encryption)
- S3: AES-256 (SSE-S3)

**In Transit:**
- PostgreSQL: TLS 1.3
- Redis: TLS 1.3
- Application to DB: Mandatory TLS

---

### PII Protection

**Encrypted Columns:**
- `users.email`: Encrypted
- `users.phone`: Encrypted
- `users.full_name`: Encrypted

**Hashed Columns:**
- `users.password_hash`: bcrypt

**Masked in Logs:**
- Phone numbers: `+1415555****`
- Email: `j***@example.com`

---

### Access Control

**PostgreSQL Roles:**
- `app_read`: SELECT only
- `app_write`: SELECT, INSERT, UPDATE
- `app_admin`: All operations (used by migrations)

**Row-Level Security:**
```sql
CREATE POLICY user_isolation ON rides
  USING (rider_id = current_setting('app.user_id')::uuid);
```

---

### Compliance

**GDPR/DPDP:**
- Right to erasure: Soft delete (set `deleted_at`)
- Right to portability: Export API
- Data residency: Region-locked writes

**PCI DSS:**
- No card data stored (PSP tokens only)
- Annual compliance audit

---

## Monitoring

### Key Metrics

**Database:**
- Connection pool utilization
- Query latency (p50, p95, p99)
- Slow query count (>100ms)
- Replication lag

**Redis:**
- Memory usage
- Eviction count
- Cache hit rate
- Command latency

**Alerts:**
- DB connection pool >80%: P1
- Redis memory >90%: P0
- Replication lag >30s: P0

---

## Performance Optimization

### Query Optimization

**Example: Fetch driver's recent trips**
```sql
-- Before (slow: 2s)
SELECT * FROM trips WHERE driver_id = 'd_660e8400' ORDER BY created_at DESC LIMIT 10;

-- After (fast: 10ms)
SELECT trip_id, status, total_fare, created_at 
FROM trips 
WHERE driver_id = 'd_660e8400' 
ORDER BY created_at DESC 
LIMIT 10;

-- Index used: idx_trips_driver_id
```

---

### Connection Pooling

**Problem:** Each service instance opening 100 connections → 20k total connections

**Solution:** PgBouncer in transaction mode
- 10k app connections → 100 DB connections
- 100x reduction in DB connection overhead

---

### Caching Strategy

**Cache-Aside Pattern:**
```python
def get_driver_rating(driver_id):
    # Check cache first
    cached = redis.get(f"driver:{driver_id}:rating")
    if cached:
        return float(cached)
    
    # Cache miss, fetch from DB
    rating = db.query("SELECT rating FROM drivers WHERE driver_id = %s", (driver_id,))
    
    # Update cache
    redis.setex(f"driver:{driver_id}:rating", 3600, rating)
    
    return rating
```

---

## References

- **HLD:** `HLD_Overview.md`
- **LLD Ride Service:** `LLD_Ride_Service.md`
- **APIs & Events:** `APIs_and_Events.md`
