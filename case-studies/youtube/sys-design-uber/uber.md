# System Design: Uber Ride-Hailing Service

## Overview

This document provides a detailed breakdown of designing a ride-hailing service similar to Uber. This is a popular system design interview question at top tech companies, focusing on real-time location tracking, low-latency matching, and handling high-throughput scenarios. The design emphasizes practical system challenges encountered in production ride-hailing platforms.

## Interview Approach Framework

The structured roadmap for this system design interview:

1. Define functional requirements
2. Establish non-functional requirements with quantifiable metrics
3. Identify core entities and data models
4. Design API endpoints
5. Create high-level system architecture
6. Deep dive into critical challenges (latency, consistency, throughput)

## Functional Requirements

The system must support three core user flows:

### 1. Fare Estimation
Users need to input their start location and destination to receive an estimated fare and ETA before committing to a ride request. This provides transparency and helps users make informed decisions.

### 2. Ride Request and Matching
Once users decide to proceed, they can request a ride, which triggers real-time matching with nearby available drivers. The system must find and notify the closest available drivers efficiently.

### 3. Driver Operations
Drivers can accept or deny ride requests and receive navigation instructions for pickup and dropoff locations. The system tracks the ride lifecycle from request through completion.

### Scope Limitations

To keep the design focused, explicitly exclude:
- Multiple car types (focus on single tier like UberX only)
- Driver and passenger rating systems
- Scheduled/future rides
- Detailed turn-by-turn navigation (use third-party APIs)
- Payment processing details
- Surge pricing algorithms

## Non-Functional Requirements

### 1. Low Latency Matching
- Matching should complete within approximately 1 minute
- If no driver accepts within this timeframe, fail gracefully
- Users should receive immediate feedback on matching status
- Critical for user experience and retention

### 2. Strong Consistency in Matching
- Ensure one-to-one relationship between rides and drivers
- No double booking: a ride cannot be assigned to multiple drivers
- No concurrent requests: a driver cannot receive multiple simultaneous ride requests
- This is the most critical consistency requirement in the system

### 3. High Availability
- System must operate 24/7 with minimal downtime
- All operations except the matching process should prioritize availability
- Critical for a service-oriented business where downtime equals lost revenue
- Geographic redundancy to handle regional failures

### 4. High Throughput
- Handle massive request surges during peak times
- Examples: concerts, sporting events, New Year's Eve
- System must scale to hundreds of thousands of requests in a single region
- Must handle millions of location updates per minute globally

### Quantifying the Scale

**Location Updates:**
- 3 million active drivers globally
- Each driver updates location every 5 seconds
- Total: 3M / 5 = 600,000 location updates per second
- This is write-heavy workload requiring specialized infrastructure

**Ride Requests:**
- During surge events, a single city might see 100,000+ simultaneous requests
- Normal operations: tens of thousands of concurrent rides per major city
- Need to buffer and process requests without overwhelming the system

### Out of Scope
While important, these are explicitly marked as out of scope:
- GDPR compliance and data privacy details
- Disaster recovery and fault tolerance specifics
- Monitoring and alerting infrastructure
- CI/CD and deployment pipelines
- Analytics and business intelligence

## Core Entities

### Ride Entity
```
Ride {
  rideId: UUID
  riderId: UUID (foreign key)
  driverId: UUID (foreign key, nullable until matched)
  status: Enum (requested, matched, en_route, in_progress, completed, cancelled)
  sourceLocation: GeoPoint {latitude, longitude}
  sourceAddress: String
  destinationLocation: GeoPoint {latitude, longitude}
  destinationAddress: String
  estimatedFare: Decimal
  actualFare: Decimal (nullable)
  estimatedDuration: Integer (minutes)
  estimatedDistance: Decimal (kilometers)
  requestedAt: DateTime
  matchedAt: DateTime (nullable)
  pickupAt: DateTime (nullable)
  completedAt: DateTime (nullable)
  carType: Enum (UberX only for this design)
}
```

### Driver Entity
```
Driver {
  driverId: UUID
  name: String
  phoneNumber: String
  email: String
  licenseNumber: String
  carModel: String
  carPlate: String
  carYear: Integer
  status: Enum (offline, available, en_route_to_pickup, on_trip)
  currentRideId: UUID (nullable)
  rating: Decimal
  totalRides: Integer
  registeredAt: DateTime
}
```

### Rider Entity
```
Rider {
  riderId: UUID
  name: String
  phoneNumber: String
  email: String
  paymentMethodId: String
  rating: Decimal
  totalRides: Integer
  registeredAt: DateTime
}
```

### Location Entity
```
Location {
  driverId: UUID (primary key)
  latitude: Decimal
  longitude: Decimal
  geohash: String (for efficient proximity queries)
  heading: Integer (direction in degrees)
  speed: Decimal (km/h)
  accuracy: Integer (meters)
  lastUpdated: DateTime
}
```

## API Design

### 1. Get Fare Estimate
```
POST /ride/estimate

Headers:
  Authorization: Bearer {riderToken}

Body:
{
  sourceLocation: {
    latitude: 37.7749,
    longitude: -122.4194
  },
  destinationLocation: {
    latitude: 37.8044,
    longitude: -122.2712
  }
}

Response:
{
  rideId: UUID,
  estimatedFare: 25.50,
  estimatedDuration: 22,
  estimatedDistance: 15.3,
  surgeMultiplier: 1.0
}
```

**Purpose:** Creates a ride record with estimate. The rider ID is extracted from the authentication token, not passed in the body. This prevents users from impersonating others.

**Why Create Ride Here?**
- Ride entity is created at estimation time, not request time
- Allows tracking user behavior (how many estimates vs actual rides)
- Simplifies the request flow (just need to update existing ride)
- Prevents race conditions on ride creation

### 2. Request a Ride
```
PATCH /ride/request

Headers:
  Authorization: Bearer {riderToken}

Body:
{
  rideId: UUID
}

Response:
{
  success: Boolean,
  message: "Searching for nearby drivers",
  rideId: UUID
}
```

**Purpose:** Updates the ride status to "requested" and triggers asynchronous driver matching. Returns immediately without waiting for driver match.

**Why PATCH not POST?**
- We're updating an existing ride created during estimation
- More semantically correct than creating a new resource
- Prevents duplicate ride creation

### 3. Update Driver Location
```
POST /location/update

Headers:
  Authorization: Bearer {driverToken}

Body:
{
  latitude: 37.7749,
  longitude: -122.4194,
  heading: 180,
  speed: 45.5,
  accuracy: 10
}

Response:
{
  success: Boolean
}
```

**Purpose:** Drivers continuously send location updates to keep the system aware of their current position. Driver ID is extracted from token.

**Frequency Considerations:**
- Every 5 seconds when available/on trip
- Every 10-15 seconds when offline/parked
- Dynamic frequency based on speed (faster = more frequent)

### 4. Driver Accept/Deny Ride
```
PATCH /ride/driver/accept

Headers:
  Authorization: Bearer {driverToken}

Body:
{
  rideId: UUID,
  accepted: Boolean
}

Response:
{
  success: Boolean,
  ride: {
    rideId: UUID,
    pickupLocation: GeoPoint,
    pickupAddress: String,
    destinationLocation: GeoPoint,
    destinationAddress: String,
    riderName: String,
    riderPhone: String,
    estimatedFare: Decimal
  }
}
```

**Purpose:** Allows driver to accept or decline a ride request. If accepted, returns full ride details. Driver ID extracted from authentication token.

### 5. Driver Update Ride Status
```
PATCH /ride/driver/update

Headers:
  Authorization: Bearer {driverToken}

Body:
{
  rideId: UUID,
  status: Enum (arrived_at_pickup, started_trip, completed_trip)
}

Response:
{
  success: Boolean,
  nextDestination: GeoPoint (nullable),
  message: String
}
```

**Purpose:** Driver updates ride status as they progress through pickup and dropoff. System returns next destination based on current status.

### Security Note: Authentication Token Pattern

**Critical Security Practice:**
Never pass user IDs or driver IDs in the request body. Always derive identity from authentication tokens in headers.

**Why?**
- Prevents impersonation: malicious user can't send requests as another user
- Simplifies client logic: clients don't need to track and send their own ID
- Standard security practice: identity should come from cryptographically signed tokens (JWT, session tokens)

**Token Contents:**
```
JWT Payload:
{
  userId: UUID,
  userType: "rider" | "driver",
  exp: timestamp,
  iat: timestamp
}
```

## High-Level System Architecture

### Client Applications

**Rider Mobile App:**
- iOS and Android native apps
- No web interface (mobile-first experience)
- Features: location input, fare estimation, ride requests, real-time tracking

**Driver Mobile App:**
- Separate app with different UI/UX
- Continuous background location tracking
- Push notification handling for ride requests
- Navigation integration

**Why Mobile Only?**
- GPS and location services work better on mobile
- Push notifications are critical for driver experience
- Users expect native app performance for real-time tracking

### API Gateway

**Managed Service (AWS API Gateway / Kong):**
- Single entry point for all client requests
- Load balancing across service instances
- SSL/TLS termination
- Authentication and authorization validation
- Rate limiting and throttling
- Request/response transformation
- API versioning

**Why Managed Gateway?**
- Handles infrastructure concerns
- Built-in scaling and high availability
- Security features out of the box
- Reduces operational burden

### Microservices Architecture

**Ride Service:**
- Handles fare estimation
- Manages ride CRUD operations
- Integrates with third-party mapping APIs (Google Maps, Mapbox)
- Persists ride data to primary database
- Stateless and horizontally scalable

**Ride Matching Service:**
- Core matching logic
- Queries location service for nearby available drivers
- Implements driver selection algorithm (closest first, rating-based, etc.)
- Handles consistency with distributed locking
- Sends ride requests to drivers via notification service
- Processes driver responses (accept/deny)

**Location Service:**
- Receives and stores driver location updates
- Provides proximity queries (find drivers within radius)
- Optimized for high write throughput
- Uses in-memory data store for fast reads

**Notification Service:**
- Sends push notifications to drivers
- Integrates with Apple Push Notification Service (APNs)
- Integrates with Firebase Cloud Messaging (FCM) for Android
- Handles notification retry and delivery confirmation
- Can be treated as a black box for this design

### Database Layer

**Primary Database (DynamoDB / PostgreSQL):**
- Stores ride data
- Stores user profiles (riders and drivers)
- Supports transactions for critical operations
- Horizontally scalable (DynamoDB preferred for scale)

**Location Database (Redis with Geospatial):**
- Optimized for high-frequency location updates
- Supports radius queries efficiently
- In-memory for low-latency access
- TTL for automatic cleanup of stale locations

### Data Flow Example

**Fare Estimation Flow:**
```
1. Rider opens app, inputs source and destination
   ↓
2. API Gateway → Ride Service
   ↓
3. Ride Service calls Google Maps API for distance/duration
   ↓
4. Calculate fare based on distance, time, base fare
   ↓
5. Create ride record in primary database
   ↓
6. Return estimate to rider
```

**Ride Request and Matching Flow:**
```
1. Rider requests ride
   ↓
2. API Gateway → Ride Service
   ↓
3. Ride Service updates ride status to "requested"
   ↓
4. Ride Service publishes to Ride Request Queue
   ↓
5. Ride Matching Service consumes from queue
   ↓
6. Query Location Service for drivers within radius
   ↓
7. Sort drivers by distance (closest first)
   ↓
8. Acquire distributed lock for first driver
   ↓
9. Send push notification via Notification Service
   ↓
10. Wait for driver response (with timeout)
   ↓
11. If accepted → update ride with driverId, release lock
12. If denied/timeout → release lock, try next driver
   ↓
13. Notify rider of successful match
```

**Location Update Flow:**
```
1. Driver app sends location every 5 seconds
   ↓
2. API Gateway → Location Service
   ↓
3. Calculate geohash for location
   ↓
4. Update Redis with driver location and geohash
   ↓
5. Set TTL of 30 seconds (auto-expire if no updates)
   ↓
6. Return success
```

## Deep Dive: Low Latency Matching & Location Service

### Challenge: High-Throughput Location Updates

**Scale Requirements:**
- 3 million active drivers globally
- Location updates every 5 seconds
- 3,000,000 / 5 = 600,000 writes per second
- Need to query this data for proximity searches in real-time

### Why Traditional SQL Fails

**PostgreSQL with B-Tree Indexes:**
```sql
-- Naive approach: range query
SELECT * FROM driver_locations
WHERE latitude BETWEEN 37.7 AND 37.8
  AND longitude BETWEEN -122.5 AND -122.4
  AND driver_status = 'available'
```

**Problems:**
- B-tree indexes don't efficiently handle 2D range queries
- Requires full index scan or table scan
- Write throughput limited to 2,000-4,000 TPS per instance
- Would need 150+ PostgreSQL instances for 600k writes/sec
- Expensive and complex to maintain

**PostGIS with Spatial Indexes:**
- Adds R-tree indexes for geospatial queries
- Better than plain B-tree but still struggles with write volume
- Good for read-heavy geospatial data, not write-heavy

### Solution 1: QuadTree

**Concept:**
Recursively divide the map into four quadrants. Each node represents a geographic region and splits into four children when density exceeds a threshold.

**Structure:**
```
Root (entire world)
├── NW Quadrant
│   ├── NW sub-quadrant
│   ├── NE sub-quadrant
│   ├── SW sub-quadrant
│   └── SE sub-quadrant
├── NE Quadrant
├── SW Quadrant
└── SE Quadrant
```

**How It Works:**
- Each leaf node contains drivers in that region
- When a node has too many drivers (e.g., >100), split it
- To find nearby drivers, traverse tree to relevant region
- Check boundary nodes for drivers near edges

**Advantages:**
- Adapts to density: more granular in cities, coarser in rural areas
- Efficient for uneven distribution of drivers
- Supported by PostGIS extension

**Disadvantages:**
- Expensive to rebalance on frequent writes
- Tree structure changes require locks or complex synchronization
- 600k updates/sec would cause constant rebalancing
- High CPU overhead for tree maintenance

**When to Use:**
- Relatively static data with occasional updates
- Uneven density is important to handle
- Read-heavy workloads

### Solution 2: Geohashing (Recommended)

**Concept:**
Divide the entire world into a grid at multiple precision levels. Encode each location as a string (geohash) that represents its grid cell.

**How Geohashing Works:**

**Step 1: Binary Encoding**
```
Latitude: 37.7749
Longitude: -122.4194

Encode as interleaved binary:
lon, lat, lon, lat, lon, lat...
01011 11000 00101 11001 10111 11101...
```

**Step 2: Base32 Encoding**
```
Convert to base32 string:
"9q8yy"
```

**Step 3: Precision Levels**
```
Precision  Grid Size       Example
1          ±2500 km        "9" (covers Western US)
2          ±630 km         "9q" (covers Bay Area)
3          ±78 km          "9q8" (San Francisco)
4          ±20 km          "9q8y" (downtown SF)
5          ±2.4 km         "9q8yy" (specific neighborhood)
6          ±610 m          "9q8yy9" (few blocks)
7          ±76 m           "9q8yy9j" (single building)
```

**Proximity Search:**
```
1. Calculate geohash for rider location: "9q8yy"
2. Query all drivers with same geohash prefix
3. If too few results, reduce precision: "9q8y"
4. Calculate actual distance for all results
5. Return closest drivers
```

**Implementation with Redis:**

**Storing Locations:**
```
Redis Geospatial Commands:

GEOADD drivers:locations 
  -122.4194 37.7749 driver_123
  -122.4212 37.7751 driver_456
  -122.4180 37.7740 driver_789

# Internal: Redis calculates geohash automatically
# Stores as sorted set with geohash as score
```

**Querying Nearby Drivers:**
```
GEORADIUS drivers:locations 
  -122.4194 37.7749 
  5 km 
  WITHDIST 
  ASC 
  COUNT 10

Returns:
[
  ["driver_123", "0.05"],
  ["driver_456", "0.12"],
  ["driver_789", "0.18"]
]
```

**Advantages:**
- In-memory: extremely fast reads and writes
- Native Redis support: simple API
- Handles 600k writes/sec easily with clustering
- Constant-time write operations (no rebalancing)
- Simple prefix matching for proximity

**Disadvantages:**
- Fixed grid: doesn't adapt to density
- Edge cases: drivers on grid boundaries might be missed
- Need to query multiple precisions if few results

**Why Redis Over PostgreSQL:**
- 10-100x faster writes (in-memory vs disk)
- Simpler horizontal scaling (Redis Cluster)
- Built-in geospatial commands
- TTL for automatic cleanup
- Pub/Sub for real-time updates if needed

### Optimization: Dynamic Location Update Frequency

Instead of fixed 5-second intervals, adjust frequency dynamically:

**Based on Driver Status:**
- Available/searching: 5 seconds
- On trip: 3 seconds (more frequent for rider tracking)
- Offline/parked: 30 seconds or stopped

**Based on Speed:**
- Stationary (0 km/h): 30 seconds
- Slow (0-20 km/h): 10 seconds
- Medium (20-60 km/h): 5 seconds
- Fast (60+ km/h): 3 seconds

**Based on Location Density:**
- High density (urban): 5 seconds (competition for rides)
- Low density (suburban): 10 seconds
- Very low density (rural): 15 seconds

**Benefits:**
- Reduces total write volume by 30-50%
- Saves battery on driver devices
- Maintains accuracy where it matters most

## Deep Dive: Consistency of Matching

### Goal: One-to-One Matching

**Critical Requirements:**
- Each ride assigned to exactly one driver
- Each driver receives at most one ride request at a time
- No race conditions in distributed environment

### Single Instance Logic (Simplified)

If only one Ride Matching Service instance:

```
function matchRide(rideId):
  drivers = getAvailableDrivers(rideLocation, radius=5km)
  
  for driver in drivers:
    if driver.status == "available":
      sendNotification(driver, rideId)
      
      response = waitForResponse(timeout=10s)
      
      if response == "accepted":
        assignRide(rideId, driver.driverId)
        return success
      
      if response == "denied" or timeout:
        continue to next driver
  
  return no_driver_found
```

**Why This Works:**
- Sequential processing ensures only one driver gets notification at a time
- Simple state management
- No concurrency issues

### Distributed System Challenge

With multiple Ride Matching Service instances:

**Race Condition Example:**
```
Time  Instance A                 Instance B
----  -------------------------  -------------------------
t0    Process ride_1             Process ride_2
t1    Query available drivers    Query available drivers
t2    Both get [driver_123, driver_456, ...]
t3    Send ride_1 to driver_123  Send ride_2 to driver_123
t4    Driver_123 receives BOTH requests simultaneously
```

**Problem:**
- Driver sees two ride requests at once (confusing UX)
- Driver might accept both (impossible to fulfill)
- Need atomicity in driver selection

### Solution 1: Database Status Column with Cron Job

**Implementation:**

**Driver Table Addition:**
```
Driver {
  ...
  lastRequestSentAt: DateTime (nullable)
  requestExpiresAt: DateTime (nullable)
}
```

**Matching Logic:**
```
function matchRide(rideId):
  drivers = getAvailableDrivers(rideLocation, radius=5km)
  
  for driver in drivers:
    # Check if driver is truly available
    if driver.status == "available" 
       AND (driver.requestExpiresAt IS NULL 
            OR driver.requestExpiresAt < NOW()):
      
      # Mark driver as "pending request"
      UPDATE drivers 
      SET lastRequestSentAt = NOW(),
          requestExpiresAt = NOW() + INTERVAL '10 seconds'
      WHERE driverId = driver.driverId
        AND (requestExpiresAt IS NULL 
             OR requestExpiresAt < NOW())
      
      if updateSuccessful:
        sendNotification(driver, rideId)
        return waiting_for_response
      else:
        # Another instance grabbed this driver
        continue to next driver
```

**Cron Job for Cleanup:**
```
Every 1 minute:
  UPDATE drivers
  SET requestExpiresAt = NULL
  WHERE requestExpiresAt < NOW()
    AND status = "available"
```

**Advantages:**
- Simple to understand and implement
- Works with existing relational database
- Acceptable for mid-level engineer interviews

**Disadvantages:**
- Cron job runs periodically (not real-time)
- Drivers locked for up to 1 extra minute after timeout
- Database write on every driver notification attempt
- Doesn't scale well under extreme load

### Solution 2: Distributed Lock with Redis (Recommended)

**Implementation with Redis:**

**Lock Acquisition:**
```
function tryLockDriver(driverId, rideId, timeout=10s):
  key = f"driver_lock:{driverId}"
  
  # SET with NX (only if not exists) and EX (expiry)
  success = REDIS.SET(
    key, 
    rideId, 
    NX,  # Only set if key doesn't exist
    EX=timeout  # Expire after timeout seconds
  )
  
  return success
```

**Matching Logic with Distributed Lock:**
```
function matchRide(rideId):
  drivers = getAvailableDrivers(rideLocation, radius=5km)
  
  for driver in drivers:
    # Try to acquire exclusive lock on driver
    if tryLockDriver(driver.driverId, rideId, timeout=10):
      
      sendNotification(driver, rideId)
      
      response = waitForResponse(timeout=10s)
      
      if response == "accepted":
        assignRide(rideId, driver.driverId)
        # Keep lock until ride starts to prevent double requests
        return success
      
      if response == "denied":
        releaseLock(driver.driverId)
        continue to next driver
      
      if timeout:
        # Lock automatically expires after 10s
        # No manual cleanup needed
        continue to next driver
```

**Why This Works:**

**Atomicity:**
- Redis SET with NX is atomic
- Only one instance can acquire lock
- Other instances immediately see lock exists

**Automatic Expiration:**
- Lock expires after timeout without manual intervention
- Driver becomes available again automatically
- No stale locks if service crashes

**Low Latency:**
- In-memory operation (microseconds)
- No disk I/O or complex transactions
- Horizontal scaling with Redis Cluster

**Example Timeline:**
```
Time  Instance A                      Instance B
----  ------------------------------  ------------------------------
t0    tryLock(driver_123, ride_1)     
t1    SUCCESS: lock acquired          
t2    Send notification to driver_123 
t3                                    tryLock(driver_123, ride_2)
t4                                    FAILURE: lock exists
t5                                    Try next driver (driver_456)
t6    Driver accepts ride_1           
t7    Keep lock, assign ride          
```

**Handling Lock Expiration:**
```
# Driver doesn't respond within 10 seconds
t0    Instance A acquires lock
t1    Send notification to driver_123
t2    Wait for response...
...
t10   Lock automatically expires
t11   Instance B can now lock driver_123
t12   Send notification for different ride
```

### Handling Redis Failure

**What if Redis goes down?**

**Fallback Strategy:**
1. Detect Redis unavailability
2. Fall back to database-level locking
3. Use PostgreSQL SELECT FOR UPDATE:

```sql
BEGIN TRANSACTION;

SELECT * FROM drivers
WHERE driverId = ?
  AND status = 'available'
FOR UPDATE SKIP LOCKED;

-- If row returned, driver is available
-- Update driver status
-- Send notification

COMMIT;
```

**Characteristics:**
- Slower than Redis (milliseconds vs microseconds)
- Still prevents race conditions
- Maintains consistency at the cost of performance
- Brief degradation better than complete failure

**Alternative: Queue-Based Approach**
- Buffer ride requests in queue
- Single-threaded consumer processes sequentially
- Eliminates need for distributed locking
- Trade-off: increased latency, single point of failure

## Deep Dive: Handling High Throughput Surges

### Challenge: Traffic Spikes

**Scenario: Major Event Ends**
- Concert with 50,000 attendees ends at 11 PM
- Everyone opens Uber app simultaneously
- 50,000+ ride requests within 5 minutes
- Matching service overwhelmed
- Database connections exhausted
- System crashes or becomes unresponsive

**Without Surge Handling:**
```
Result: Complete system failure
- Matching service OOM (Out of Memory)
- Database connection pool exhausted
- All users see errors
- No rides matched
- Reputation damage
```

### Solution: Ride Request Queue with Geographic Partitioning

**Architecture:**

**Message Queue (Kafka/SQS/RabbitMQ):**
```
Ride Service → Ride Request Queue → Ride Matching Service
```

**Queue Structure:**
```
Topic: ride_requests
Partitions:
  - partition_0: SF_downtown
  - partition_1: SF_mission
  - partition_2: SF_sunset
  - partition_3: LA_downtown
  - partition_4: LA_westside
  ...
```

**Flow with Queue:**
```
1. Ride Service receives ride request
   ↓
2. Calculate partition key from pickup location
   partition = geohash(pickupLocation, precision=3)
   ↓
3. Publish to queue with partition key
   queue.publish(rideData, partition=partition)
   ↓
4. Multiple Matching Service instances consume from partitions
   ↓
5. Each instance processes rides from assigned partitions
   ↓
6. Process at sustainable rate (e.g., 100 rides/sec per instance)
   ↓
7. Backpressure: queue buffers excess requests
   ↓
8. Scale matching service instances up during surge
```

**Benefits:**

**Load Smoothing:**
- Queue absorbs sudden spikes
- Matching service processes at steady rate
- No service overload

**Horizontal Scaling:**
- Add more matching service instances
- Each instance consumes from different partitions
- Linear scalability

**Fault Tolerance:**
- If matching service crashes, messages remain in queue
- Another instance picks up unacknowledged messages
- No lost ride requests

**Partitioning Benefits:**
- Geographic partitioning prevents head-of-line blocking
- Slow-to-match rides in one area don't block others
- Better cache locality (each instance handles specific region)

**Retry Logic:**
- If matching fails (no drivers), retry after delay
- Exponential backoff: 10s, 30s, 60s, 180s
- Eventually notify rider if no driver found

**User Experience:**
```
Without queue:
"Error: Service unavailable" (immediate failure)

With queue:
"Searching for a driver..." (transparent waiting)
If delayed: "High demand, you're in queue position #245"
Better UX even under stress
```

### Monitoring and Auto-Scaling

**Queue Depth Monitoring:**
```
if queueDepth > 1000:
  alert("High ride request backlog")
  autoScaleUp(matchingServiceInstances)

if queueDepth < 100 AND instances > minInstances:
  autoScaleDown(matchingServiceInstances)
```

**Metrics to Track:**
- Queue depth per partition
- Processing rate per instance
- Average matching time
- Driver response rate
- Timeout rate

## Deep Dive: High Availability Outside Matching

### Horizontal Scaling of Microservices

**Stateless Services:**
- All services designed to be stateless
- No local session storage
- All state in databases or caches
- Can scale instances up/down freely

**Load Balancing:**
```
API Gateway
    ↓
Application Load Balancer
    ↓
[Service Instance 1] [Service Instance 2] ... [Service Instance N]
```

**Auto-Scaling Policies:**
- CPU utilization > 70% → scale up
- Request rate > threshold → scale up
- Queue depth > threshold → scale up
- Scale down during low traffic hours

### Database High Availability

**Primary Database (DynamoDB):**
- Fully managed, auto-scaling
- Multi-AZ replication by default
- Handles millions of requests per second
- No single point of failure

**Alternative: PostgreSQL with Replication:**
```
Primary (writes)
    ↓ replication
[Read Replica 1] [Read Replica 2] [Read Replica 3]
```

- Write traffic to primary
- Read traffic distributed across replicas
- Automatic failover if primary fails

### Regional Data Center Deployment

**Geographic Distribution:**
```
US-West (San Francisco)
- Serves West Coast users
- Reduced latency for CA, WA, OR

US-East (Virginia)
- Serves East Coast users
- Reduced latency for NY, MA, FL

EU (Ireland)
- Serves European users

Asia-Pacific (Singapore)
- Serves Asian users
```

**Benefits:**
- Lower latency for users
- Compliance with data residency laws
- Fault tolerance (one region fails, others continue)

**Cross-Region User Travel:**
- User profiles replicated across regions
- User opens app in new region → data available
- Read replicas in each region
- Eventual consistency acceptable for user data

### Caching Strategy

**Redis Cache Layer:**
```
Application → Redis Cache → Database

Cached Data:
- User profiles (1 hour TTL)
- Driver profiles (1 hour TTL)
- Ride history (1 hour TTL)
- Frequently accessed static data
```

**Cache Invalidation:**
- On user/driver update → invalidate cache
- Time-based expiration for safety
- Cache-aside pattern: lazy loading

## Additional Considerations

### Driver Selection Algorithm

Beyond "closest driver," consider:

**Weighted Scoring:**
```
score = w1 * (1/distance) 
      + w2 * driverRating 
      + w3 *