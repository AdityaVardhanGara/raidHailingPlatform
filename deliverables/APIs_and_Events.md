# APIs & Event Schemas

## Table of Contents
1. [API Gateway Endpoints](#api-gateway-endpoints)
2. [Service-to-Service APIs](#service-to-service-apis)
3. [Event Topics & Schemas](#event-topics--schemas)
4. [Common Patterns](#common-patterns)

---

## API Gateway Endpoints

### Authentication
All endpoints require JWT authentication unless marked as `[PUBLIC]`.

**Header:**
```
Authorization: Bearer <jwt_token>
```

---

### 1. Ride Management APIs

#### POST /v1/rides
Create a new ride request.

**Request:**
```json
{
  "rider_id": "550e8400-e29b-41d4-a716-446655440000",
  "pickup": {
    "lat": 37.7749,
    "lng": -122.4194,
    "address": "123 Market St, San Francisco, CA 94103"
  },
  "destination": {
    "lat": 37.8044,
    "lng": -122.2712,
    "address": "456 Broadway, Oakland, CA 94612"
  },
  "tier": "economy",
  "payment_method_id": "pm_1234567890",
  "promo_code": "RIDE50",
  "scheduled_time": "2026-05-04T15:30:00Z",
  "notes": "Please call when you arrive"
}
```

**Response (202 Accepted):**
```json
{
  "ride_id": "r_550e8400-e29b-41d4-a716-446655440001",
  "status": "MATCHING",
  "fare_estimate": 12.50,
  "surge_multiplier": 1.2,
  "estimated_wait_sec": 90,
  "created_at": "2026-05-04T10:30:45Z"
}
```

**Error Responses:**
- `400 Bad Request`: Invalid input
- `403 Forbidden`: User not eligible
- `429 Too Many Requests`: Rate limit exceeded
- `503 Service Unavailable`: System overload

---

#### GET /v1/rides/{ride_id}
Get ride details.

**Response (200 OK):**
```json
{
  "ride_id": "r_550e8400-e29b-41d4-a716-446655440001",
  "status": "ASSIGNED",
  "rider_id": "550e8400-e29b-41d4-a716-446655440000",
  "driver_id": "d_660e8400-e29b-41d4-a716-446655440002",
  "pickup": {
    "lat": 37.7749,
    "lng": -122.4194,
    "address": "123 Market St, San Francisco, CA 94103"
  },
  "destination": {
    "lat": 37.8044,
    "lng": -122.2712,
    "address": "456 Broadway, Oakland, CA 94612"
  },
  "tier": "economy",
  "fare_estimate": 12.50,
  "driver_details": {
    "name": "John Doe",
    "phone": "+14155551234",
    "vehicle": {
      "make": "Toyota",
      "model": "Camry",
      "year": 2022,
      "color": "Silver",
      "plate": "ABC1234"
    },
    "rating": 4.8,
    "eta_sec": 300
  },
  "created_at": "2026-05-04T10:30:45Z",
  "updated_at": "2026-05-04T10:31:20Z"
}
```

---

#### POST /v1/rides/{ride_id}/cancel
Cancel a ride.

**Request:**
```json
{
  "reason": "changed_plans",
  "notes": "Found another ride"
}
```

**Response (200 OK):**
```json
{
  "ride_id": "r_550e8400-e29b-41d4-a716-446655440001",
  "status": "CANCELLED",
  "cancellation_fee": 0.00,
  "cancelled_at": "2026-05-04T10:32:00Z"
}
```

---

#### GET /v1/rides
List user's rides (paginated).

**Query Parameters:**
- `status`: Filter by status (optional)
- `limit`: Results per page (default: 20, max: 100)
- `cursor`: Pagination cursor

**Response (200 OK):**
```json
{
  "rides": [
    {
      "ride_id": "r_550e8400-e29b-41d4-a716-446655440001",
      "status": "COMPLETED",
      "pickup": {...},
      "destination": {...},
      "fare": 13.20,
      "created_at": "2026-05-04T10:30:45Z"
    }
  ],
  "next_cursor": "eyJvZmZzZXQiOjIwfQ==",
  "has_more": true
}
```

---

### 2. Driver Management APIs

#### POST /v1/drivers/location
Update driver location (sent every 1-2 seconds).

**Request:**
```json
{
  "driver_id": "d_660e8400-e29b-41d4-a716-446655440002",
  "lat": 37.7750,
  "lng": -122.4195,
  "heading": 45,
  "speed": 25,
  "timestamp": "2026-05-04T10:30:45.123Z"
}
```

**Response (200 OK):**
```json
{
  "status": "ok"
}
```

---

#### POST /v1/drivers/status
Update driver online/offline status.

**Request:**
```json
{
  "driver_id": "d_660e8400-e29b-41d4-a716-446655440002",
  "status": "online",
  "location": {
    "lat": 37.7750,
    "lng": -122.4195
  }
}
```

**Response (200 OK):**
```json
{
  "driver_id": "d_660e8400-e29b-41d4-a716-446655440002",
  "status": "online",
  "updated_at": "2026-05-04T10:30:45Z"
}
```

---

#### GET /v1/drivers/{driver_id}/rides
Get driver's ride requests (pending acceptance).

**Response (200 OK):**
```json
{
  "requests": [
    {
      "ride_id": "r_550e8400-e29b-41d4-a716-446655440001",
      "rider_name": "Jane Smith",
      "rider_rating": 4.9,
      "pickup": {
        "lat": 37.7749,
        "lng": -122.4194,
        "address": "123 Market St"
      },
      "destination": {
        "lat": 37.8044,
        "lng": -122.2712,
        "address": "456 Broadway"
      },
      "tier": "economy",
      "fare_estimate": 12.50,
      "distance_to_pickup_m": 500,
      "expires_at": "2026-05-04T10:31:15Z"
    }
  ]
}
```

---

#### POST /v1/drivers/{driver_id}/rides/{ride_id}/accept
Accept a ride request.

**Response (200 OK):**
```json
{
  "ride_id": "r_550e8400-e29b-41d4-a716-446655440001",
  "status": "ASSIGNED",
  "rider_details": {
    "name": "Jane Smith",
    "phone": "+14155559876",
    "rating": 4.9
  },
  "pickup": {...},
  "destination": {...}
}
```

---

#### POST /v1/drivers/{driver_id}/rides/{ride_id}/decline
Decline a ride request.

**Request:**
```json
{
  "reason": "too_far"
}
```

**Response (200 OK):**
```json
{
  "status": "ok"
}
```

---

### 3. Trip Management APIs

#### POST /v1/trips/{trip_id}/start
Start a trip.

**Request:**
```json
{
  "driver_id": "d_660e8400-e29b-41d4-a716-446655440002",
  "location": {
    "lat": 37.7749,
    "lng": -122.4194
  },
  "odometer_start": 12345
}
```

**Response (200 OK):**
```json
{
  "trip_id": "t_770e8400-e29b-41d4-a716-446655440003",
  "status": "IN_PROGRESS",
  "started_at": "2026-05-04T10:35:00Z"
}
```

---

#### POST /v1/trips/{trip_id}/end
End a trip.

**Request:**
```json
{
  "driver_id": "d_660e8400-e29b-41d4-a716-446655440002",
  "location": {
    "lat": 37.8044,
    "lng": -122.2712
  },
  "odometer_end": 12360,
  "distance_km": 15.2,
  "duration_sec": 1200
}
```

**Response (200 OK):**
```json
{
  "trip_id": "t_770e8400-e29b-41d4-a716-446655440003",
  "status": "COMPLETED",
  "fare": {
    "base_fare": 5.00,
    "distance_fare": 7.60,
    "time_fare": 2.00,
    "surge_multiplier": 1.2,
    "subtotal": 17.52,
    "fees": 1.48,
    "total": 19.00
  },
  "ended_at": "2026-05-04T10:55:00Z"
}
```

---

### 4. Payment APIs

#### GET /v1/payment-methods
List user's payment methods.

**Response (200 OK):**
```json
{
  "payment_methods": [
    {
      "id": "pm_1234567890",
      "type": "card",
      "card": {
        "brand": "visa",
        "last4": "4242",
        "exp_month": 12,
        "exp_year": 2027
      },
      "is_default": true
    }
  ]
}
```

---

#### POST /v1/payment-methods
Add a payment method.

**Request:**
```json
{
  "type": "card",
  "token": "tok_visa_4242424242424242",
  "billing_address": {
    "line1": "123 Main St",
    "city": "San Francisco",
    "state": "CA",
    "postal_code": "94102",
    "country": "US"
  }
}
```

**Response (201 Created):**
```json
{
  "id": "pm_1234567890",
  "type": "card",
  "card": {
    "brand": "visa",
    "last4": "4242",
    "exp_month": 12,
    "exp_year": 2027
  },
  "created_at": "2026-05-04T10:30:45Z"
}
```

---

### 5. Pricing APIs

#### GET /v1/pricing/estimate
Get fare estimate.

**Query Parameters:**
- `pickup_lat`: Pickup latitude
- `pickup_lng`: Pickup longitude
- `dest_lat`: Destination latitude
- `dest_lng`: Destination longitude
- `tier`: Ride tier (economy, premium, luxury)

**Response (200 OK):**
```json
{
  "fare_estimate": 12.50,
  "surge_multiplier": 1.2,
  "base_fare": 5.00,
  "distance_fare": 7.60,
  "time_estimate_sec": 1200,
  "distance_km": 15.2,
  "breakdown": {
    "base": 5.00,
    "per_km": 0.50,
    "per_minute": 0.10,
    "surge": 1.2
  }
}
```

---

#### GET /v1/pricing/surge
Get current surge multiplier for a location.

**Query Parameters:**
- `lat`: Latitude
- `lng`: Longitude

**Response (200 OK):**
```json
{
  "surge_multiplier": 1.5,
  "geo_cell": "8928308280fffff",
  "valid_until": "2026-05-04T10:35:00Z"
}
```

---

### 6. Admin APIs

#### POST /v1/admin/feature-flags
Update feature flag.

**Request:**
```json
{
  "flag_name": "enable_surge_pricing_v2",
  "enabled": true,
  "rollout_percentage": 50,
  "regions": ["us-west-1", "us-east-1"]
}
```

**Response (200 OK):**
```json
{
  "flag_name": "enable_surge_pricing_v2",
  "enabled": true,
  "rollout_percentage": 50,
  "updated_at": "2026-05-04T10:30:45Z"
}
```

---

#### POST /v1/admin/kill-switch
Activate emergency kill switch.

**Request:**
```json
{
  "action": "disable_new_rides",
  "reason": "payment_processor_down",
  "duration_sec": 3600
}
```

**Response (200 OK):**
```json
{
  "status": "activated",
  "expires_at": "2026-05-04T11:30:45Z"
}
```

---

## Service-to-Service APIs

### Dispatch Service

#### POST /internal/v1/dispatch/match
Internal API to request driver matching.

**Request:**
```json
{
  "ride_id": "r_550e8400-e29b-41d4-a716-446655440001",
  "rider_id": "550e8400-e29b-41d4-a716-446655440000",
  "pickup": {
    "lat": 37.7749,
    "lng": -122.4194
  },
  "tier": "economy",
  "max_wait_time_sec": 120,
  "max_search_radius_m": 5000
}
```

**Response (200 OK):**
```json
{
  "status": "MATCHING",
  "estimated_wait_sec": 90,
  "candidates_count": 12
}
```

---

#### POST /internal/v1/dispatch/cancel
Cancel dispatch request.

**Request:**
```json
{
  "ride_id": "r_550e8400-e29b-41d4-a716-446655440001",
  "reason": "user_cancelled"
}
```

**Response (200 OK):**
```json
{
  "status": "CANCELLED"
}
```

---

### Pricing Service

#### GET /internal/v1/pricing/surge
Get surge multiplier for geo-cell.

**Query Parameters:**
- `cell`: H3 geo-cell ID

**Response (200 OK):**
```json
{
  "cell": "8928308280fffff",
  "surge_multiplier": 1.5,
  "supply": 45,
  "demand": 120,
  "valid_until": "2026-05-04T10:35:00Z"
}
```

---

### UUID Service

#### GET /internal/v1/uuid/generate
Generate globally unique ID.

**Query Parameters:**
- `type`: Entity type (ride, trip, payment)

**Response (200 OK):**
```json
{
  "uuid": "550e8400-e29b-41d4-a716-446655440000",
  "type": "ride",
  "timestamp": "2026-05-04T10:30:45.123Z"
}
```

---

## Event Topics & Schemas

### Topic Naming Convention
```
{domain}.{entity}.{action}
```

Examples:
- `ride.requested`
- `driver.location.updated`
- `trip.completed`
- `payment.captured`

---

### 1. Ride Events

#### ride.requested

**Partition Key:** `ride_id`
**Retention:** 7 days

```json
{
  "event_id": "e_550e8400-e29b-41d4-a716-446655440000",
  "event_type": "ride.requested",
  "event_version": "1.0",
  "timestamp": "2026-05-04T10:30:45.123Z",
  "trace_id": "trace_550e8400",
  "data": {
    "ride_id": "r_550e8400-e29b-41d4-a716-446655440001",
    "rider_id": "550e8400-e29b-41d4-a716-446655440000",
    "pickup": {
      "lat": 37.7749,
      "lng": -122.4194,
      "address": "123 Market St, San Francisco, CA 94103",
      "geo_cell": "8928308280fffff"
    },
    "destination": {
      "lat": 37.8044,
      "lng": -122.2712,
      "address": "456 Broadway, Oakland, CA 94612",
      "geo_cell": "8928308281fffff"
    },
    "tier": "economy",
    "payment_method_id": "pm_1234567890",
    "fare_estimate": 12.50,
    "surge_multiplier": 1.2,
    "promo_code": "RIDE50",
    "scheduled_time": null,
    "region": "us-west-1"
  }
}
```

---

#### ride.assigned

**Partition Key:** `ride_id`

```json
{
  "event_id": "e_550e8400-e29b-41d4-a716-446655440001",
  "event_type": "ride.assigned",
  "event_version": "1.0",
  "timestamp": "2026-05-04T10:31:20.456Z",
  "trace_id": "trace_550e8400",
  "data": {
    "ride_id": "r_550e8400-e29b-41d4-a716-446655440001",
    "rider_id": "550e8400-e29b-41d4-a716-446655440000",
    "driver_id": "d_660e8400-e29b-41d4-a716-446655440002",
    "assigned_at": "2026-05-04T10:31:20Z",
    "eta_sec": 300,
    "dispatch_duration_ms": 350
  }
}
```

---

#### ride.cancelled

**Partition Key:** `ride_id`

```json
{
  "event_id": "e_550e8400-e29b-41d4-a716-446655440002",
  "event_type": "ride.cancelled",
  "event_version": "1.0",
  "timestamp": "2026-05-04T10:32:00.789Z",
  "trace_id": "trace_550e8400",
  "data": {
    "ride_id": "r_550e8400-e29b-41d4-a716-446655440001",
    "rider_id": "550e8400-e29b-41d4-a716-446655440000",
    "driver_id": "d_660e8400-e29b-41d4-a716-446655440002",
    "cancelled_by": "rider",
    "reason": "changed_plans",
    "cancellation_fee": 0.00,
    "cancelled_at": "2026-05-04T10:32:00Z"
  }
}
```

---

### 2. Driver Events

#### driver.location.updated

**Partition Key:** `driver_id`
**Retention:** 1 day

```json
{
  "event_id": "e_660e8400-e29b-41d4-a716-446655440000",
  "event_type": "driver.location.updated",
  "event_version": "1.0",
  "timestamp": "2026-05-04T10:30:45.123Z",
  "data": {
    "driver_id": "d_660e8400-e29b-41d4-a716-446655440002",
    "lat": 37.7750,
    "lng": -122.4195,
    "heading": 45,
    "speed": 25,
    "accuracy": 10,
    "geo_cell": "8928308280fffff",
    "is_online": true,
    "active_ride_id": null
  }
}
```

---

#### driver.status.changed

**Partition Key:** `driver_id`

```json
{
  "event_id": "e_660e8400-e29b-41d4-a716-446655440001",
  "event_type": "driver.status.changed",
  "event_version": "1.0",
  "timestamp": "2026-05-04T10:30:45.123Z",
  "data": {
    "driver_id": "d_660e8400-e29b-41d4-a716-446655440002",
    "old_status": "offline",
    "new_status": "online",
    "location": {
      "lat": 37.7750,
      "lng": -122.4195
    },
    "changed_at": "2026-05-04T10:30:45Z"
  }
}
```

---

### 3. Trip Events

#### trip.started

**Partition Key:** `trip_id`

```json
{
  "event_id": "e_770e8400-e29b-41d4-a716-446655440000",
  "event_type": "trip.started",
  "event_version": "1.0",
  "timestamp": "2026-05-04T10:35:00.123Z",
  "trace_id": "trace_550e8400",
  "data": {
    "trip_id": "t_770e8400-e29b-41d4-a716-446655440003",
    "ride_id": "r_550e8400-e29b-41d4-a716-446655440001",
    "rider_id": "550e8400-e29b-41d4-a716-446655440000",
    "driver_id": "d_660e8400-e29b-41d4-a716-446655440002",
    "start_location": {
      "lat": 37.7749,
      "lng": -122.4194
    },
    "odometer_start": 12345,
    "started_at": "2026-05-04T10:35:00Z"
  }
}
```

---

#### trip.completed

**Partition Key:** `trip_id`

```json
{
  "event_id": "e_770e8400-e29b-41d4-a716-446655440001",
  "event_type": "trip.completed",
  "event_version": "1.0",
  "timestamp": "2026-05-04T10:55:00.456Z",
  "trace_id": "trace_550e8400",
  "data": {
    "trip_id": "t_770e8400-e29b-41d4-a716-446655440003",
    "ride_id": "r_550e8400-e29b-41d4-a716-446655440001",
    "rider_id": "550e8400-e29b-41d4-a716-446655440000",
    "driver_id": "d_660e8400-e29b-41d4-a716-446655440002",
    "start_location": {
      "lat": 37.7749,
      "lng": -122.4194
    },
    "end_location": {
      "lat": 37.8044,
      "lng": -122.2712
    },
    "distance_km": 15.2,
    "duration_sec": 1200,
    "fare": {
      "base_fare": 5.00,
      "distance_fare": 7.60,
      "time_fare": 2.00,
      "surge_multiplier": 1.2,
      "subtotal": 17.52,
      "fees": 1.48,
      "promo_discount": 0.00,
      "total": 19.00
    },
    "started_at": "2026-05-04T10:35:00Z",
    "completed_at": "2026-05-04T10:55:00Z"
  }
}
```

---

### 4. Payment Events

#### payment.initiated

**Partition Key:** `payment_id`

```json
{
  "event_id": "e_880e8400-e29b-41d4-a716-446655440000",
  "event_type": "payment.initiated",
  "event_version": "1.0",
  "timestamp": "2026-05-04T10:55:01.123Z",
  "trace_id": "trace_550e8400",
  "data": {
    "payment_id": "pay_880e8400-e29b-41d4-a716-446655440004",
    "trip_id": "t_770e8400-e29b-41d4-a716-446655440003",
    "rider_id": "550e8400-e29b-41d4-a716-446655440000",
    "amount": 19.00,
    "currency": "USD",
    "payment_method_id": "pm_1234567890",
    "initiated_at": "2026-05-04T10:55:01Z"
  }
}
```

---

#### payment.captured

**Partition Key:** `payment_id`

```json
{
  "event_id": "e_880e8400-e29b-41d4-a716-446655440001",
  "event_type": "payment.captured",
  "event_version": "1.0",
  "timestamp": "2026-05-04T10:55:03.456Z",
  "trace_id": "trace_550e8400",
  "data": {
    "payment_id": "pay_880e8400-e29b-41d4-a716-446655440004",
    "trip_id": "t_770e8400-e29b-41d4-a716-446655440003",
    "rider_id": "550e8400-e29b-41d4-a716-446655440000",
    "amount": 19.00,
    "currency": "USD",
    "psp_transaction_id": "txn_stripe_1234567890",
    "captured_at": "2026-05-04T10:55:03Z"
  }
}
```

---

#### payment.failed

**Partition Key:** `payment_id`

```json
{
  "event_id": "e_880e8400-e29b-41d4-a716-446655440002",
  "event_type": "payment.failed",
  "event_version": "1.0",
  "timestamp": "2026-05-04T10:55:05.789Z",
  "trace_id": "trace_550e8400",
  "data": {
    "payment_id": "pay_880e8400-e29b-41d4-a716-446655440004",
    "trip_id": "t_770e8400-e29b-41d4-a716-446655440003",
    "rider_id": "550e8400-e29b-41d4-a716-446655440000",
    "amount": 19.00,
    "currency": "USD",
    "error_code": "card_declined",
    "error_message": "Insufficient funds",
    "retry_allowed": true,
    "failed_at": "2026-05-04T10:55:05Z"
  }
}
```

---

### 5. Notification Events

#### notification.push.requested

**Partition Key:** `user_id`

```json
{
  "event_id": "e_990e8400-e29b-41d4-a716-446655440000",
  "event_type": "notification.push.requested",
  "event_version": "1.0",
  "timestamp": "2026-05-04T10:31:20.123Z",
  "data": {
    "user_id": "550e8400-e29b-41d4-a716-446655440000",
    "notification_type": "ride_assigned",
    "title": "Driver is on the way!",
    "body": "John will arrive in 5 minutes. Toyota Camry - ABC1234",
    "data": {
      "ride_id": "r_550e8400-e29b-41d4-a716-446655440001",
      "driver_id": "d_660e8400-e29b-41d4-a716-446655440002"
    },
    "priority": "high"
  }
}
```

---

## Common Patterns

### Idempotency

All state-changing requests must include an idempotency key:

**Header:**
```
X-Idempotency-Key: {uuid}
```

**Behavior:**
- Duplicate requests (same key) return cached response
- Keys expire after 24 hours
- 409 Conflict if key used for different operation

---

### Pagination

**Cursor-based pagination:**

```
GET /v1/rides?limit=20&cursor=eyJvZmZzZXQiOjIwfQ==
```

**Response:**
```json
{
  "data": [...],
  "next_cursor": "eyJvZmZzZXQiOjQwfQ==",
  "has_more": true
}
```

---

### Error Response Format

```json
{
  "error": "error_code",
  "message": "Human-readable error message",
  "field": "field_name (for validation errors)",
  "retry_after_sec": 60,
  "trace_id": "trace_550e8400"
}
```

**Common Error Codes:**
- `validation_error`: Invalid input
- `not_found`: Resource not found
- `forbidden`: Permission denied
- `rate_limit_exceeded`: Too many requests
- `service_unavailable`: System overload
- `payment_declined`: Payment method declined

---

### Rate Limiting

**Headers:**
```
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 45
X-RateLimit-Reset: 1746360000
```

**Response (429 Too Many Requests):**
```json
{
  "error": "rate_limit_exceeded",
  "message": "You have exceeded the rate limit. Try again in 60 seconds.",
  "retry_after_sec": 60
}
```

---

### Versioning

**URL-based versioning:**
```
/v1/rides
/v2/rides (future)
```

**Version deprecation:**
- Announce 6 months in advance
- Support both versions for 12 months
- Add deprecation header:
  ```
  X-API-Deprecated: true
  X-API-Deprecation-Date: 2027-01-01
  X-API-Sunset-Date: 2027-06-01
  ```

---

### Event Schema Versioning

**Backward-compatible changes (minor version):**
- Add optional fields
- Add new event types

**Breaking changes (major version):**
- Remove fields
- Change field types
- Change field meanings

**Version field:**
```json
{
  "event_version": "1.0"
}
```

**Consumers must handle unknown fields gracefully.**

---

## Event Consumers

### Driver Location Updater
**Consumes:** `driver.location.updated`
**Updates:** Redis GEOSPATIAL index
**Throughput:** 500k events/sec

### Trip Fare Calculator
**Consumes:** `trip.completed`
**Updates:** PostgreSQL `trips` table
**Throughput:** 1k events/sec

### Payment Processor
**Consumes:** `trip.completed`
**Produces:** `payment.initiated`, `payment.captured`, `payment.failed`
**Throughput:** 1k events/sec

### Notification Sender
**Consumes:** `ride.assigned`, `trip.started`, `trip.completed`
**Produces:** `notification.push.requested`
**Throughput:** 3k events/sec

### Analytics Pipeline
**Consumes:** All events
**Writes to:** Data warehouse (Snowflake)
**Throughput:** 100k events/sec (batched)

---

## Security

### API Authentication
- **Users:** JWT tokens (RS256, 15min expiry)
- **Services:** mTLS + API keys
- **Admin:** SSO (Okta) + MFA

### Event Security
- **Kafka ACLs:** Per-topic read/write permissions
- **Encryption in transit:** TLS 1.3
- **Encryption at rest:** AES-256

### PII Handling
- **Masked in logs:** Phone numbers, addresses
- **Tokenized:** Payment card numbers (never stored)
- **Encrypted:** User data in database

---

## Testing

### API Contract Tests
- OpenAPI/Swagger spec validation
- Postman collections for manual testing
- Automated contract tests (Pact)

### Event Schema Tests
- Avro/JSON schema validation
- Schema registry (Confluent Schema Registry)
- Consumer compatibility tests

---

## References

- **HLD:** `HLD_Overview.md`
- **LLD Ride Service:** `LLD_Ride_Service.md`
- **Data Models:** `Data_Models.md`
