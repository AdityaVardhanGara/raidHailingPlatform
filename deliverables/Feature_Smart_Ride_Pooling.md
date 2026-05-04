# Feature Design: Smart Ride Pooling with Dynamic Routing

## Executive Summary

**Feature Name:** Smart Ride Pooling with Dynamic Routing

**Description:** An intelligent ride-sharing feature that dynamically matches multiple riders with similar routes, optimizes pickup/drop-off sequences in real-time, and provides cost savings while maintaining acceptable detour limits.

**Value Proposition:**
- **For Riders:** 30-50% fare reduction, eco-friendly option
- **For Drivers:** Higher earnings per hour (more riders per trip)
- **For Platform:** Increased utilization, reduced congestion, environmental impact

**Complexity:** High (requires sophisticated routing algorithms, real-time optimization, multi-party coordination)

---

## Problem Statement

### Current State
- One rider per trip (inefficient vehicle utilization)
- High fares during surge periods
- Increased traffic congestion in dense urban areas
- Lost revenue opportunity for drivers during low-demand periods

### Opportunity
- 40% of rides have overlapping routes within 2km radius
- Riders willing to accept 10-15 minute detours for 40% fare discount
- Drivers can earn 60% more per hour with pooled rides

---

## Feature Requirements

### Functional Requirements

1. **Ride Matching**
   - Match 2-4 riders with compatible routes
   - Calculate optimal pickup/drop-off sequence
   - Maximum detour: 15 minutes or 5km (whichever is less)
   - Support mixing of tiers (economy + premium not allowed)

2. **Dynamic Re-routing**
   - Adjust route when new rider added mid-trip
   - Recalculate ETAs for all passengers
   - Cancel pool if constraints violated (detour too large)

3. **Pricing**
   - Base fare discount: 30% for first match, 40% for second+
   - Dynamic pricing based on demand
   - Split cancellation fees proportionally

4. **User Experience**
   - Opt-in per ride (not default)
   - Real-time ETA updates for all passengers
   - In-app chat for coordination
   - Clear fare breakdown (individual share)

5. **Driver Experience**
   - Accept/decline pool requests
   - Clear pickup/drop-off sequence
   - Navigation guidance for multi-stop route
   - Bonus incentives for high pool completion rate

### Non-Functional Requirements

1. **Latency:** Matching decision within 2s
2. **Accuracy:** Route optimization within 5% of optimal
3. **Availability:** 99.9% (same as regular rides)
4. **Scale:** Support 20% of total rides pooled
5. **Privacy:** No PII shared between riders

---

## System Design

### Architecture

```
┌──────────────────────────────────────────────────────────────┐
│                     New Components                            │
│                                                                │
│  ┌─────────────────┐   ┌─────────────────┐   ┌────────────┐ │
│  │  Pool Matcher   │   │  Route Optimizer│   │  Pool State│ │
│  │                 │──→│                 │──→│  Manager   │ │
│  └─────────────────┘   └─────────────────┘   └────────────┘ │
│          ↓                     ↓                     ↓        │
│    ┌──────────┐         ┌──────────┐        ┌──────────┐   │
│    │  Redis   │         │  Graph   │        │PostgreSQL│   │
│    │ (Active  │         │ Database │        │ (Pool    │   │
│    │  Pools)  │         │ (Routes) │        │  Trips)  │   │
│    └──────────┘         └──────────┘        └──────────┘   │
│                                                                │
└──────────────────────────────────────────────────────────────┘
              ↓                    ↓                    ↓
    ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐
    │   Ride Service  │  │ Dispatch Service│  │  Trip Service   │
    │   (Modified)    │  │   (Modified)    │  │   (Modified)    │
    └─────────────────┘  └─────────────────┘  └─────────────────┘
```

### New Components

#### 1. Pool Matcher Service

**Purpose:** Match riders with compatible routes.

**Algorithm:**
1. Receive pool ride request
2. Query active pools within 2km radius (Redis GEORADIUS)
3. For each candidate pool:
   - Calculate route with new passenger
   - Check detour constraint (<15 min, <5 km)
   - Score match (minimize total detour, maximize fare)
4. Select best match or create new pool
5. Timeout: 5 seconds (fallback to regular ride)

**Scoring Function:**
```python
def score_pool_match(pool, new_rider):
    # Minimize detour for existing riders
    existing_detour_penalty = sum([rider.detour_increase for rider in pool.riders])
    
    # Maximize fare efficiency
    fare_efficiency = (pool.total_fare + new_rider.fare) / pool.estimated_duration
    
    # Prefer fuller pools (up to max capacity)
    capacity_bonus = len(pool.riders) * 10
    
    # Penalize if pickup far from current route
    pickup_detour_penalty = haversine(pool.current_location, new_rider.pickup)
    
    score = (fare_efficiency * 100 + capacity_bonus) - (existing_detour_penalty + pickup_detour_penalty)
    return score
```

**Data Flow:**
```
Ride Service → Pool Matcher → Query Redis (active pools) → 
Calculate route permutations → Score matches → 
Select best match → Update Redis → Return match_id
```

**Scaling:**
- **Horizontal:** 30-150 pods
- **Vertical:** 8 vCPU, 16GB RAM (CPU-intensive routing calculations)
- **Capacity:** 50 matches/sec per instance

---

#### 2. Route Optimizer Service

**Purpose:** Calculate optimal pickup/drop-off sequence for pooled rides.

**Algorithm:** 
- **Heuristic:** Nearest neighbor with 2-opt improvement
- **Constraint:** Total detour ≤ 15 minutes per rider
- **Optimization Goal:** Minimize total trip duration

**Implementation:**
```python
from ortools.constraint_solver import routing_enums_pb2, pywrapcp

def optimize_route(pool):
    """
    Use Google OR-Tools for vehicle routing problem (VRP)
    """
    # Create routing model
    manager = pywrapcp.RoutingIndexManager(
        len(pool.locations),  # pickups + dropoffs
        1,  # single vehicle
        0   # depot (driver's current location)
    )
    routing = pywrapcp.RoutingModel(manager)
    
    # Distance callback
    def distance_callback(from_index, to_index):
        from_node = manager.IndexToNode(from_index)
        to_node = manager.IndexToNode(to_index)
        return distance_matrix[from_node][to_node]
    
    transit_callback_index = routing.RegisterTransitCallback(distance_callback)
    routing.SetArcCostEvaluatorOfAllVehicles(transit_callback_index)
    
    # Add pickup/dropoff constraints
    for rider in pool.riders:
        pickup_index = manager.NodeToIndex(rider.pickup_node)
        dropoff_index = manager.NodeToIndex(rider.dropoff_node)
        routing.AddPickupAndDelivery(pickup_index, dropoff_index)
    
    # Solve
    search_parameters = pywrapcp.DefaultRoutingSearchParameters()
    search_parameters.first_solution_strategy = (
        routing_enums_pb2.FirstSolutionStrategy.PATH_CHEAPEST_ARC
    )
    search_parameters.time_limit.seconds = 2  # 2s timeout
    
    solution = routing.SolveWithParameters(search_parameters)
    
    return extract_route(manager, routing, solution)
```

**Data Flow:**
```
Pool Matcher → Route Optimizer → Load route segments from Graph DB → 
Run VRP solver → Validate constraints → Return optimized sequence
```

**Scaling:**
- **Horizontal:** 20-100 pods
- **Vertical:** 16 vCPU, 32GB RAM (computation-heavy)
- **Capacity:** 20 optimizations/sec per instance
- **Caching:** Cache common route segments in Redis

---

#### 3. Pool State Manager

**Purpose:** Manage pool lifecycle, track riders, handle cancellations.

**State Machine:**
```
MATCHING → COLLECTING → IN_PROGRESS → COMPLETED
    ↓           ↓             ↓
CANCELLED  CANCELLED    CANCELLED
```

**State Transitions:**
- **MATCHING:** First rider added, looking for more
- **COLLECTING:** Pool full or timeout (30s), driver en route to first pickup
- **IN_PROGRESS:** First pickup completed, trip started
- **COMPLETED:** All riders dropped off
- **CANCELLED:** Pool disbanded (rider cancellation, constraints violated)

**Data Model (Redis):**
```redis
HSET pool:{pool_id}
  status "COLLECTING"
  driver_id "d_123"
  capacity 4
  current_riders 2
  rider_ids "r_456,r_789"
  route_sequence "pickup_r456,pickup_r789,dropoff_r456,dropoff_r789"
  created_at 1746360645
  expires_at 1746360675
```

**Data Flow:**
```
Ride Request → Pool Matcher → Pool State Manager (create/update pool) → 
Dispatch Service (assign driver) → Pool State Manager (track progress) → 
Trip Service (complete individual rider trips)
```

---

### Modified Components

#### Ride Service (Modified)

**New Endpoint:** `POST /v1/rides/pool`

**Request:**
```json
{
  "rider_id": "u_123",
  "pickup": {"lat": 37.7749, "lng": -122.4194},
  "destination": {"lat": 37.8044, "lng": -122.2712},
  "tier": "economy",
  "max_detour_min": 15,
  "max_wait_time_min": 10
}
```

**Response:**
```json
{
  "ride_id": "r_456",
  "pool_id": "pool_789",
  "status": "MATCHING",
  "estimated_fare": 8.50,
  "discount": 40,
  "original_fare": 14.00,
  "estimated_wait_sec": 300,
  "current_pool_size": 2,
  "max_detour_min": 15
}
```

---

#### Dispatch Service (Modified)

**Changes:**
- Query Pool Matcher before driver assignment
- If pool match found, assign shared driver
- If no match, create new pool and assign driver
- Handle multi-pickup routing

---

#### Trip Service (Modified)

**Changes:**
- Support multi-rider trips (1 trip_id, multiple rider_ids)
- Calculate per-rider fare (proportional to distance + base share)
- Track individual rider start/end times
- Handle partial cancellations (one rider cancels mid-trip)

**Data Model:**
```sql
CREATE TABLE pooled_trips (
    pooled_trip_id UUID PRIMARY KEY,
    driver_id UUID NOT NULL,
    vehicle_id UUID NOT NULL,
    status VARCHAR(20) NOT NULL,
    rider_ids UUID[] NOT NULL,
    route_sequence JSONB NOT NULL,
    total_fare DECIMAL(10,2),
    started_at TIMESTAMP,
    completed_at TIMESTAMP,
    created_at TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE TABLE pooled_trip_riders (
    id UUID PRIMARY KEY,
    pooled_trip_id UUID NOT NULL REFERENCES pooled_trips(pooled_trip_id),
    rider_id UUID NOT NULL,
    pickup_lat DECIMAL(10,8),
    pickup_lng DECIMAL(11,8),
    dropoff_lat DECIMAL(10,8),
    dropoff_lng DECIMAL(11,8),
    fare DECIMAL(10,2),
    discount_percent INTEGER,
    picked_up_at TIMESTAMP,
    dropped_off_at TIMESTAMP,
    cancelled_at TIMESTAMP
);
```

---

## Data Flow: Complete Pooled Ride

### Step 1: Rider A Requests Pool Ride

```
1. Rider A → POST /v1/rides/pool
2. Ride Service → Pool Matcher Service
3. Pool Matcher → Query Redis (no active pools nearby)
4. Pool Matcher → Create new pool (pool_789)
5. Pool Matcher → Response: status=MATCHING, pool_id=pool_789
6. Pool State Manager → Set Redis (pool:pool_789, expires_at=+30s)
7. Ride Service → Dispatch Service (assign driver, driver accepts)
8. Pool State Manager → Update status=COLLECTING, driver_id=d_123
```

### Step 2: Rider B Requests Pool Ride (Overlapping Route)

```
1. Rider B → POST /v1/rides/pool
2. Ride Service → Pool Matcher Service
3. Pool Matcher → Query Redis (finds pool_789)
4. Pool Matcher → Route Optimizer (calculate route with B)
5. Route Optimizer → Optimal sequence: pickup_A, pickup_B, dropoff_A, dropoff_B
6. Route Optimizer → Detour check: A detour +5 min (acceptable)
7. Pool Matcher → Add Rider B to pool_789
8. Pool State Manager → Update pool_789 (rider_ids=[A, B], route_sequence=...)
9. Pool State Manager → Notify Rider A: "Another rider added, ETA updated"
10. Ride Service → Response to Rider B: status=MATCHED, pool_id=pool_789
```

### Step 3: Driver Picks Up Riders

```
1. Driver → Arrive at Rider A pickup → Trip Service (pickup_A)
2. Pool State Manager → Update pool status=IN_PROGRESS
3. Notification Service → Notify Rider B: "Driver will arrive in 3 minutes"
4. Driver → Arrive at Rider B pickup → Trip Service (pickup_B)
5. Notification Service → Notify Rider A: "One more pickup, ETA updated"
```

### Step 4: Driver Drops Off Riders

```
1. Driver → Arrive at Rider A dropoff → Trip Service (dropoff_A)
2. Pool State Manager → Mark Rider A complete, calculate fare
3. Payment Service → Charge Rider A ($8.50)
4. Notification Service → Send receipt to Rider A
5. Driver → Continue to Rider B dropoff → Trip Service (dropoff_B)
6. Pool State Manager → Mark Rider B complete, calculate fare
7. Payment Service → Charge Rider B ($9.00)
8. Pool State Manager → Update pool status=COMPLETED
9. Trip Service → Record pooled trip (pooled_trip_id, rider_ids=[A, B])
```

---

## Pricing Algorithm

### Fare Calculation

**Base Formula:**
```
original_fare = base_fare + (distance_km * per_km_rate) + (time_min * per_min_rate) + surge
```

**Pooled Fare:**
```
# Distance share (proportional to rider's leg)
distance_share = (rider_distance_km / total_trip_distance_km)

# Time share (flat split + distance-weighted)
time_share = 0.5 / num_riders + 0.5 * distance_share

# Base fare split (equally among all riders)
base_share = base_fare / num_riders

# Pooled fare
pooled_fare = base_share + (distance_share * distance_component) + (time_share * time_component)

# Apply discount (30-40% based on pool size)
discount_percent = min(30 + (num_riders - 1) * 10, 40)
final_fare = pooled_fare * (1 - discount_percent / 100)
```

**Example:**
- **Rider A:** 10 km solo leg, 5 km shared leg → 60% distance share → $8.50 (40% discount)
- **Rider B:** 5 km shared leg, 8 km solo leg → 40% distance share → $9.00 (40% discount)
- **Driver:** Earns $17.50 total (vs. $14 for single ride) → 25% more

---

## Trade-offs

### Design Decisions

| Decision | Rationale | Trade-off |
|----------|-----------|-----------|
| Max 4 riders per pool | Balance efficiency vs complexity | Larger pools = more savings but harder to optimize |
| 15-minute detour limit | User research shows acceptable threshold | May miss some valid matches |
| Opt-in (not default) | Avoid surprise shared rides | Lower adoption rate initially |
| Sequential pickup (not parallel) | Simpler routing, single vehicle | Longer wait for last rider |
| Cache route segments | Reduce latency | Stale data if traffic changes |

---

## Resilience

### Failure Modes

#### Pool Matcher Down
- **Fallback:** Create regular (non-pooled) ride
- **User Impact:** No discount, but ride completes

#### Route Optimizer Timeout
- **Fallback:** Use simple nearest-neighbor heuristic
- **User Impact:** Suboptimal route, slightly longer detour

#### Mid-Trip Cancellation
- **Behavior:** 
  - Rider A cancels → Remove from route, recalculate for Rider B
  - Driver continues with remaining riders
  - Charge Rider A cancellation fee (reduced if after pickup)

#### Constraint Violation Detected
- **Example:** Traffic jam causes 20-minute detour (exceeds 15-min limit)
- **Behavior:** 
  - Pool State Manager monitors ETA updates
  - If constraint violated, offer compensation (credit, upgrade)
  - Option to convert to private ride (no additional charge)

---

## Monitoring

### Key Metrics

```
# Pool matching
pool_match_rate (target: >60%)
pool_match_latency_ms (target: <2s)

# Route optimization
route_optimization_latency_ms (target: <1s)
average_detour_per_rider_min (target: <10 min)

# User experience
pool_cancellation_rate (target: <5%)
rider_satisfaction_score (target: >4.0/5.0)

# Business
pooled_ride_percentage (target: 20% of total rides)
driver_earnings_lift (target: +25%)
```

### Alerts

- **P0:** Pool Matcher down (fallback to regular rides)
- **P1:** Route optimizer latency >3s (degraded experience)
- **P2:** Detour violations >10% (tune constraints)

---

## Rollout Plan

### Phase 1: Limited Beta (Weeks 1-4)
- Enable in 1 region (US-West-1, San Francisco)
- Whitelist 5% of users (high-frequency riders)
- Max 2 riders per pool
- Manual monitoring and feedback collection

### Phase 2: Expanded Beta (Weeks 5-8)
- Expand to 3 regions (US-East, EU-West)
- 20% of users
- Increase to 3 riders per pool
- A/B test pricing (30% vs 40% discount)

### Phase 3: General Availability (Weeks 9+)
- All regions
- All users (opt-in)
- Max 4 riders per pool
- Feature flags for kill switch

---

## Success Metrics

### User Metrics
- **Adoption Rate:** 15% of rides opted into pool within 3 months
- **Satisfaction:** 4.2/5.0 rating for pooled rides
- **Repeat Usage:** 60% of users who try pool use it again

### Business Metrics
- **Revenue:** 10% increase in GMV (more rides, higher driver earnings)
- **Driver Earnings:** 20% increase per hour for drivers accepting pools
- **Efficiency:** 30% reduction in empty vehicle km

### Platform Metrics
- **Matching Rate:** 65% of pool requests successfully matched
- **Completion Rate:** 95% of pooled rides complete without cancellation
- **Detour Accuracy:** Actual detour within 20% of estimated

---

## Future Enhancements

1. **Predictive Pooling:** ML model predicts likely routes, pre-matches riders
2. **Fixed Routes:** Recurring routes (commute paths) with scheduled pickups
3. **Multi-Vehicle Coordination:** Transfer riders between vehicles mid-trip
4. **Gamification:** Badges for eco-friendly riders, streak bonuses
5. **Corporate Pools:** Company-sponsored pooled rides for employees

---

## Technical Challenges

### Challenge 1: Real-Time Route Optimization
- **Problem:** VRP is NP-hard, exact solution too slow
- **Solution:** Use heuristics (nearest neighbor + 2-opt), 2s timeout

### Challenge 2: Handling Cancellations
- **Problem:** Mid-trip cancellation disrupts entire pool
- **Solution:** Graceful degradation, recalculate route for remaining riders

### Challenge 3: Privacy Concerns
- **Problem:** Riders see each other's pickup/dropoff locations
- **Solution:** Show only approximate locations (within 100m), no addresses

### Challenge 4: Fair Pricing
- **Problem:** Complex fare calculation with shared segments
- **Solution:** Transparent breakdown in app, dispute resolution process

---

## References

- **HLD Overview:** `HLD_Overview.md`
- **Service-Specific HLD:** `Service_Specific_HLD.md`
- **APIs & Events:** `APIs_and_Events.md`
- **Academic Paper:** "On-Demand High-Capacity Ride-Sharing via Dynamic Trip-Vehicle Assignment" (MIT, 2017)
