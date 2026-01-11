# System Design: Ticket Booking Service (Ticketmaster)

## Overview

This document provides a comprehensive breakdown of designing a ticket booking service similar to Ticketmaster. This is a common system design interview question at companies like Meta, focusing on handling high-traffic scenarios, transaction consistency, and real-time updates.

## Interview Approach Framework

The recommended approach for tackling this system design interview follows a structured roadmap:

1. Define functional requirements
2. Establish non-functional requirements
3. Identify core entities and data models
4. Design API endpoints
5. Create high-level system architecture
6. Deep dive into challenging aspects

## Functional Requirements

The system must support three core user flows:

### 1. Event Discovery
Users need to search for events based on various criteria such as artist name, event type, location, and date range. This is the entry point for most user journeys.

### 2. Event Details Viewing
Once users find an event, they need to view comprehensive details including venue information, performer details, available seats with pricing, and a visual seat map representation.

### 3. Ticket Booking
Users must be able to select specific seats, reserve them temporarily, and complete the purchase through a payment gateway. This is the most critical flow requiring strong consistency guarantees.

## Non-Functional Requirements

### Consistency vs Availability Trade-offs

**For Ticket Booking (Write Operations):**
- Prioritize consistency over availability
- Prevent double booking at all costs
- Brief unavailability is acceptable over selling the same seat twice
- Implements the "CP" side of the CAP theorem

**For Search and Event Viewing (Read Operations):**
- Prioritize high availability
- Eventual consistency is acceptable
- Stale data for a few seconds won't significantly impact user experience
- Implements the "AP" side of the CAP theorem

### Workload Characteristics

**Read-Heavy System:**
- Read-to-write ratio approximately 100:1
- Millions of users browse events but only a fraction complete purchases
- Design should optimize for read performance

**Surge Traffic Handling:**
- Popular events (Taylor Swift concerts, Super Bowl) create massive traffic spikes
- System must handle sudden surges without crashing
- Need mechanisms to throttle and queue users gracefully

**Out of Scope (but worth mentioning):**
- GDPR compliance and data privacy
- Disaster recovery and fault tolerance details
- Analytics and reporting systems

## Core Entities

### Event Entity
```
Event {
  eventId: UUID
  name: String
  description: Text
  eventType: Enum (concert, sports, theater, etc.)
  startDateTime: DateTime
  endDateTime: DateTime
  venueId: UUID (foreign key)
  performerId: UUID (foreign key)
  status: Enum (upcoming, ongoing, completed, cancelled)
}
```

### Venue Entity
```
Venue {
  venueId: UUID
  name: String
  address: String
  city: String
  state: String
  country: String
  zipCode: String
  coordinates: GeoPoint
  totalCapacity: Integer
  seatingMap: JSON (describes sections, rows, seats)
}
```

### Performer Entity
```
Performer {
  performerId: UUID
  name: String
  type: Enum (musician, band, sports_team, etc.)
  description: Text
  imageUrl: String
}
```

### Ticket Entity
```
Ticket {
  ticketId: UUID
  eventId: UUID (foreign key)
  venueId: UUID (foreign key)
  section: String
  row: String
  seatNumber: String
  price: Decimal
  status: Enum (available, reserved, booked)
  reservedBy: UUID (nullable, user ID)
  reservedAt: DateTime (nullable)
  bookedBy: UUID (nullable, user ID)
  bookedAt: DateTime (nullable)
}
```

## API Design

### 1. Get Event Details
```
GET /event/{eventId}

Response:
{
  event: Event,
  venue: Venue,
  performer: Performer,
  tickets: [
    {
      ticketId: UUID,
      section: String,
      row: String,
      seatNumber: String,
      price: Decimal,
      status: Enum
    }
  ]
}
```

**Purpose:** Returns all information needed to render the event details page including the interactive seat map.

### 2. Search Events
```
GET /search?term={searchTerm}&location={location}&type={eventType}&date={dateRange}

Response:
{
  events: [
    {
      eventId: UUID,
      name: String,
      startDateTime: DateTime,
      venueName: String,
      city: String,
      minPrice: Decimal
    }
  ],
  totalResults: Integer,
  page: Integer
}
```

**Purpose:** Returns a filtered list of events matching search criteria. Returns partial event data for performance.

### 3. Reserve Ticket
```
POST /booking/reserve

Headers:
  Authorization: Bearer {userToken}

Body:
{
  ticketId: UUID
}

Response:
{
  success: Boolean,
  reservationId: UUID,
  expiresAt: DateTime,
  message: String
}
```

**Purpose:** Temporarily locks a ticket for a user (typically 10 minutes) to allow time for payment processing.

### 4. Confirm Booking
```
POST /booking/confirm

Headers:
  Authorization: Bearer {userToken}

Body:
{
  reservationId: UUID,
  paymentDetails: {
    paymentMethodId: String,
    billingAddress: Object
  }
}

Response:
{
  success: Boolean,
  bookingId: UUID,
  ticketId: UUID,
  confirmationEmail: String,
  message: String
}
```

**Purpose:** Finalizes the ticket purchase by processing payment through a third-party service like Stripe and converting the reservation to a confirmed booking.

## High-Level Architecture

### Microservices Approach

The system uses a microservices architecture with the following components:

**API Gateway:**
- Single entry point for all client requests
- Handles authentication and authorization
- Routes requests to appropriate microservices
- Implements rate limiting and throttling

**Event Service:**
- Manages CRUD operations for events, venues, and performers
- Serves event details and metadata
- Handles admin operations for creating/updating events

**Search Service:**
- Processes search queries with filters
- Integrates with Elasticsearch for optimized search
- Implements caching for popular queries

**Booking Service:**
- Manages ticket reservations and confirmations
- Handles transaction logic with strong consistency
- Integrates with payment gateways
- Implements distributed locking mechanisms

### Database Selection

**Primary Database: PostgreSQL**

Reasons for choosing SQL over NoSQL:
- Strong ACID guarantees for transactional booking operations
- Complex relational queries between events, venues, performers, and tickets
- Built-in support for row-level locking
- Well-established patterns for handling concurrent transactions

**Schema Design:**
- Events table with foreign keys to venues and performers
- Tickets table with foreign key to events
- Proper indexing on eventId, status, and reservation timestamps
- Composite indexes for common query patterns

**NoSQL Consideration:**
While PostgreSQL is preferred, NoSQL databases like MongoDB or DynamoDB could work with careful transaction handling and application-level consistency checks.

## Deep Dive: Booking Flow and Consistency

### The Two-Phase Booking Process

**Phase 1: Reservation**
1. User selects a seat and clicks "Reserve"
2. System checks ticket availability and status
3. If available, updates ticket status to "reserved"
4. Sets reservedBy to user ID and reservedAt to current timestamp
5. Returns success with expiration time (current time + 10 minutes)

**Phase 2: Confirmation**
1. User completes payment form
2. System validates reservation hasn't expired
3. Processes payment through third-party gateway
4. On success, updates ticket status to "booked"
5. Sends confirmation email and updates bookedBy and bookedAt fields

### Challenge: Handling Reservation Expiration

**Problem:** If users abandon their checkout session, tickets remain reserved indefinitely, blocking other users from purchasing them.

**Solution 1: Query-Time Filtering (Basic)**
```sql
SELECT * FROM tickets 
WHERE eventId = ? 
  AND (status = 'available' 
       OR (status = 'reserved' AND reservedAt < NOW() - INTERVAL '10 minutes'))
```

Pros: Simple to implement
Cons: Stale reservations remain in database, queries become complex

**Solution 2: Cron Job Cleanup (Intermediate)**
- Run a scheduled job every minute to clear expired reservations
- Update expired tickets back to "available" status

Pros: Keeps database clean, simple logic
Cons: Cleanup is delayed, brief window where expired tickets still show as reserved

**Solution 3: Distributed Lock with Redis (Optimal)**

This is the recommended approach for production systems:

**Implementation:**
```
When reserving:
1. Attempt to acquire lock in Redis: 
   SET ticket:{ticketId}:lock {userId} EX 600 NX
   
2. If lock acquired successfully:
   - Update PostgreSQL ticket status to "reserved"
   - Return success to user
   
3. If lock not acquired:
   - Return failure (ticket already reserved)

When confirming:
1. Verify lock exists and belongs to user
2. Process payment
3. Update PostgreSQL to "booked"
4. Delete lock from Redis: DEL ticket:{ticketId}:lock

Automatic expiration:
- Redis TTL automatically expires lock after 10 minutes
- No manual cleanup needed
```

**Why Redis Distributed Lock?**
- Atomic operations prevent race conditions
- TTL (Time To Live) provides automatic expiration
- Works across horizontally scaled booking service instances
- Fast in-memory operations (microsecond latency)
- Single source of truth for reservation state

**Handling Redis Failure:**
- If Redis becomes unavailable, fall back to database-level locking
- Use PostgreSQL SELECT FOR UPDATE to acquire row-level locks
- Brief degradation in performance but consistency maintained
- Consider Redis replication and clustering for high availability

### Preventing Double Booking in Distributed Systems

When multiple booking service instances run simultaneously:

**Without Distributed Lock:**
- Instance A checks ticket availability: available
- Instance B checks ticket availability: available
- Instance A reserves ticket
- Instance B reserves ticket (double booking!)

**With Distributed Lock:**
- Instance A attempts Redis lock: success
- Instance B attempts Redis lock: failure (already locked)
- Only Instance A proceeds with reservation
- Instance B returns "ticket unavailable" to user

## Deep Dive: Search Optimization

### Problem with SQL-Based Search

Initial implementation using SQL queries:
```sql
SELECT * FROM events 
WHERE name LIKE '%concert%' 
  AND city = 'New York' 
  AND startDateTime BETWEEN ? AND ?
ORDER BY startDateTime
```

Issues:
- Full table scans on text fields
- LIKE operations are slow
- Geospatial queries are inefficient
- Doesn't support fuzzy matching or relevance ranking

### Solution: Elasticsearch Integration

**Why Elasticsearch?**
- Built specifically for search use cases
- Creates inverted indexes for fast text search
- Supports geospatial queries efficiently
- Provides relevance scoring and ranking
- Handles fuzzy matching and typo tolerance

**Architecture:**
```
PostgreSQL (source of truth)
    ↓
Change Data Capture (CDC) / Dual Writes
    ↓
Elasticsearch (search index)
    ↓
Search Service queries Elasticsearch
```

**Synchronization Strategies:**

**Option 1: Application-Level Dual Writes**
- When creating/updating events, write to both PostgreSQL and Elasticsearch
- Pros: Simple, immediate consistency
- Cons: Risk of sync failures, increased latency

**Option 2: Change Data Capture (CDC)**
- Use tools like Debezium to stream PostgreSQL changes
- CDC captures transaction log and publishes to Kafka
- Elasticsearch consumes from Kafka and updates indexes
- Pros: Decoupled, reliable, eventual consistency acceptable
- Cons: Slight delay in search index updates

**Sample Elasticsearch Query:**
```json
{
  "query": {
    "bool": {
      "must": [
        {
          "match": {
            "name": "taylor swift concert"
          }
        },
        {
          "geo_distance": {
            "distance": "50km",
            "location": {
              "lat": 40.7128,
              "lon": -74.0060
            }
          }
        },
        {
          "range": {
            "startDateTime": {
              "gte": "2024-01-01",
              "lte": "2024-12-31"
            }
          }
        }
      ]
    }
  }
}
```

### Caching Strategy for Search

**Layer 1: Elasticsearch Query Cache**
- Elasticsearch caches frequent query results automatically
- Segment-level caching improves repeated queries
- Limited control over eviction policies

**Layer 2: Redis/Memcached Cache**
- Cache API responses for popular search queries
- Key: hash of search parameters
- TTL: 30-60 seconds (balance freshness vs performance)
- Reduces Elasticsearch load for identical queries

**Layer 3: CDN Caching**
- Cache GET /search responses at edge locations
- Short TTL (30 seconds) to prevent stale results
- Geo-distributed for low latency
- Best for high-traffic, repeated searches

**Challenges:**
- High variability in search queries reduces cache hit rate
- Personalized results may prevent effective caching
- Balance between freshness and performance

## Deep Dive: Real-Time Seat Availability

### Problem Statement

Users viewing the same event simultaneously need to see seat availability updates in real-time. Without this:
- User A reserves seat 10A
- User B still sees seat 10A as available
- User B attempts to reserve, gets error
- Poor user experience and potential frustration

### Solution Options

**Option 1: Client-Side Polling**
- Client requests seat availability every 5-10 seconds
- Pros: Simple to implement
- Cons: High server load, delayed updates, inefficient

**Option 2: Long Polling**
- Client sends request, server holds connection open
- Server responds only when seat availability changes or timeout occurs
- Client immediately sends new request after receiving response
- Pros: More efficient than polling, simple HTTP
- Cons: Holds connections open, server resources consumed

**Option 3: Server-Sent Events (Recommended)**
- Persistent unidirectional connection from server to client
- Server pushes updates when seat status changes
- Built on HTTP, simpler than WebSockets
- Automatic reconnection handling

**Implementation Flow:**
```
1. Client opens SSE connection: 
   GET /events/{eventId}/seats/stream
   
2. Server maintains connection, sends updates:
   data: {"ticketId": "123", "status": "reserved"}
   
3. When any seat status changes:
   - Booking service publishes to Redis Pub/Sub
   - Event service subscribes and forwards to connected clients
   
4. Client receives update and refreshes seat map UI
```

**Why Not WebSockets?**
- WebSockets provide bidirectional communication
- Seat updates are unidirectional (server → client)
- SSE is simpler and sufficient for this use case
- WebSockets add unnecessary complexity

**Scalability Consideration:**
- Popular events may have thousands of concurrent SSE connections
- Use Redis Pub/Sub to fan out updates across service instances
- Consider connection limits and load balancing

## Deep Dive: Handling Surge Traffic

### Challenge: Popular Event Releases

When tickets for highly anticipated events go on sale:
- Millions of users hit the system simultaneously
- Database and backend services overwhelmed
- System may crash, creating poor experience for everyone
- "Thundering herd" problem

### Solution: Virtual Waiting Queue

**Concept:**
Instead of allowing all users direct access to booking system, implement a queue that admits users in controlled batches.

**Implementation with Redis:**

**Data Structure: Sorted Set**
```
ZADD waiting_queue:{eventId} {timestamp} {userId}
```

Users are ranked by join timestamp (first-come-first-served) or can use random scoring for fairness.

**Queue Flow:**
```
1. User attempts to access booking page
   ↓
2. Check if user has active session token
   ↓
3. If not, add to waiting queue with score = current_timestamp
   ↓
4. Return queue position and estimated wait time
   ↓
5. Background job periodically admits users:
   - ZPOPMIN waiting_queue:{eventId} {batch_size}
   - Generate session tokens for admitted users
   - Tokens valid for limited time (e.g., 10 minutes)
   ↓
6. Client polls for admission or uses SSE for real-time updates
   ↓
7. Once admitted, user can access booking page
   ↓
8. If user doesn't complete booking within time limit, 
   recycle their slot
```

**Benefits:**
- Prevents system overload
- Fair user experience with predictable wait
- Maintains backend stability
- Can scale admission rate based on system capacity
- Better than random access denials or crashes

**User Experience:**
- Show queue position: "You are number 15,234 in line"
- Estimated wait time: "Approximately 8 minutes"
- Progress updates as position advances
- Better than spinning loaders or error pages

## Additional Scalability Considerations

### Database Sharding

**When to Shard:**
- Single PostgreSQL instance cannot handle read/write load
- Data size exceeds single server capacity

**Sharding Strategy:**

**Option 1: Shard by Event ID**
```
shard = hash(eventId) % num_shards
```
Pros: Even distribution, simple
Cons: Cross-event queries difficult

**Option 2: Shard by Venue/Region**
```
shard = region_mapping[venueCity]
```
Pros: Geographically aligned, better for regional queries
Cons: Uneven distribution (NYC vs small cities)

**Read Replicas:**
- Create read replicas for Event Service queries
- Route search and view operations to replicas
- Write operations go to primary
- Reduces primary database load

### Caching Static Data

**What to Cache:**
- Event details (rarely change after creation)
- Venue information (static)
- Performer data (static)
- Seating maps (static per venue)

**Implementation:**
```
Cache Key Pattern:
- event:{eventId} → full event object
- venue:{venueId} → venue details
- performer:{performerId} → performer info

TTL Strategy:
- Static data: long TTL (1 hour to 1 day)
- Event dates/status: shorter TTL (5-10 minutes)
- Ticket availability: no cache (always fresh)
```

**Cache Eviction:**
- Use LRU (Least Recently Used) for memory-constrained caches
- For event data that fits in memory, avoid eviction
- Proactively invalidate on updates

### Load Balancing

**API Gateway Load Balancing:**
- Use managed services (AWS ALB, Nginx)
- Round-robin or least-connections algorithm
- Health checks to remove unhealthy instances

**Service Mesh:**
- For microservices communication
- Client-side load balancing
- Circuit breakers for fault tolerance

## Key Takeaways and Best Practices

### Design Principles

1. **Separate Read and Write Concerns**
   - Use different consistency models for different operations
   - Optimize reads separately from writes

2. **Choose the Right Tool for Each Job**
   - PostgreSQL for transactional consistency
   - Redis for distributed locks and caching
   - Elasticsearch for search
   - SSE for real-time updates

3. **Plan for Failure**
   - What happens if Redis goes down?
   - How to handle payment gateway failures?
   - Graceful degradation over complete failure

4. **Optimize for Common Cases**
   - 100:1 read-write ratio means optimize reads aggressively
   - Cache what users access most frequently
   - Index database queries based on actual patterns

### Interview Strategy

1. **Start Broad, Then Dive Deep**
   - Establish requirements before jumping to solutions
   - Get agreement on scope and priorities
   - Choose 2-3 areas for detailed discussion

2. **Communicate Trade-offs**
   - "We could use WebSockets, but SSE is simpler for our use case"
   - "NoSQL could work, but SQL gives us stronger consistency guarantees"
   - Show you understand multiple approaches

3. **Use Calculations Strategically**
   - Don't do math unless it changes the design
   - Back-of-envelope useful for: storage needs, cache sizing, traffic estimation
   - Skip if it doesn't inform decisions

4. **Show Iteration**
   - Start with simple solution (SQL search)
   - Identify problems (slow, doesn't scale)
   - Propose improvement (Elasticsearch)
   - Discuss next-level optimizations (caching)

5. **Ask Clarifying Questions**
   - "Should we prioritize consistency or availability for booking?"
   - "What's the expected read-write ratio?"
   - "How popular can events get?"

### Common Pitfalls to Avoid

- Over-engineering early (don't start with Kafka if simple queue works)
- Under-specifying consistency requirements
- Ignoring the distributed systems challenges
- Forgetting about operational aspects (monitoring, alerting)
- Not discussing failure scenarios
- Focusing too much on one area, neglecting others

## Conclusion

Designing a ticket booking service requires balancing multiple competing concerns: consistency for bookings, availability for browsing, performance under surge traffic, and real-time user experience. The key is understanding which requirements are critical (no double booking) versus nice-to-have (instant search results), and architecting each component appropriately.

The combination of microservices, distributed locking, search optimization, caching strategies, and traffic management creates a robust system capable of handling millions of users purchasing tickets for popular events while maintaining data integrity and providing excellent user experience.