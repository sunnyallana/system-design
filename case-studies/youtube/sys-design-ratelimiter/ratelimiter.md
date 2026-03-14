# Design a Distributed Rate Limiter w/ an Ex-Meta Staff Engineer: System Design Breakdown

---

## What is a Rate Limiter?

A rate limiter controls how many requests a client can make within a specific time frame. Its primary purpose is to protect backend services from abuse, traffic spikes, and overload. A classic example: limiting users of a social media app to **100 requests per minute per user**.

---

## Framework for Approaching the Design

The design follows a structured delivery framework broken into five stages:

1. **Requirements** — Functional and non-functional
2. **Core Entities** — Key objects involved (requests, clients, rules)
3. **System Interface** — How components interact internally, e.g. `isRequestAllowed(clientID, rule)`
4. **High-Level Design** — Architecture and data flow
5. **Deep Dives** — Scalability, latency, fault tolerance

---

## Requirements

### Functional Requirements
- Identify clients uniquely via **user ID**, **IP address**, or **API key**
- Enforce configurable rules (e.g., 100 requests/minute per user)
- Return **HTTP 429 (Too Many Requests)** with headers indicating rate limit thresholds and reset times

### Non-Functional Requirements

| Property | Target |
|---|---|
| **Scalability** | 1M requests/second (for ~100M daily active users) |
| **Latency** | Rate limit check must complete in **< 10ms** |
| **Availability** | Prefer availability over strict consistency (CAP theorem) |
| **Fault Tolerance** | Graceful degradation via fail-open or fail-close strategies |

---

## System Placement

Where you place the rate limiter matters significantly:

| Placement | Description | Tradeoff |
|---|---|---|
| Inside each microservice | Fast, no network hop | No global coordination across services |
| Centralized service | Global coordination | Adds latency to every request |
| **API Gateway (edge)** ✅ | Acts as a "bouncer" before routing | Best of both — blocks abuse early, global view |

**The API Gateway is the preferred placement.** It intercepts all incoming traffic before it reaches any backend service, acting as a first line of defense.

---

## Client Identification

Clients are identified using a combination of:
- **User ID** — for authenticated users
- **IP Address** — for anonymous or unauthenticated traffic
- **API Key** — for developer/service clients

Different client types can have different limits — for example, premium users may get higher thresholds than anonymous IPs. Identification data is typically extracted from **request headers** (e.g., decoded from a JWT token).

---

## Rate Limiting Algorithms

### 1. Fixed Window Counter
Counts requests within a fixed time window (e.g., per minute).
- ✅ Simple to implement
- ❌ **Boundary effect**: a user can double their effective rate by sending bursts right at the window edges

### 2. Sliding Window Log
Tracks the exact timestamp of every request within a rolling window.
- ✅ Accurate, no boundary effects
- ❌ **High memory usage** — storing timestamps for every request at scale is expensive

### 3. Sliding Window Counter
Uses two counters — one for the current window, one for the previous — weighted by how far into the current window you are.
- ✅ Memory efficient, better accuracy than fixed window
- ❌ Approximate — assumes even request distribution across windows

### 4. Token Bucket ✅ *(Chosen Algorithm)*
Each client gets a "bucket" of tokens that refills at a steady rate. Each request consumes one token. Requests are denied when the bucket is empty.
- ✅ Handles **both bursts and steady-state** traffic naturally
- ✅ Simple to implement and reason about
- ❌ Requires parameter tuning (bucket size, refill rate)
- ❌ Race conditions must be handled carefully in distributed environments

---

## High-Level Architecture

```
Client Request
      │
      ▼
 API Gateway
      │
      ├──► Redis (Token Bucket State per Client)
      │         └── clientID → { tokens, lastRefillTime }
      │
      ├── Allowed → Route to Backend Service
      └── Denied  → Return HTTP 429
```

---

## Implementation Deep Dives

### State Storage — Redis
All token bucket state (token count + last refill timestamp per client) is stored in a **distributed in-memory cache (Redis)**. Redis is ideal because:
- Sub-millisecond read/write latency
- Native support for atomic operations
- Horizontally scalable via sharding

### Atomic Updates — Lua Scripting
In a distributed system, a read-then-write on token state creates a race condition:

```
Thread A reads: 5 tokens available
Thread B reads: 5 tokens available
Both write: 4 tokens   ← double-spend bug
```

The fix is to use **Redis Lua scripts**, which execute atomically — the read and write happen as a single, uninterruptible operation on the Redis instance.

### Sharding for Scale
At 1M requests/second, a single Redis instance is a bottleneck. The solution is to **shard Redis by client ID**:

- If a single Redis instance handles ~50K req/s, you need roughly **~20 Redis shards**
- A consistent hashing function maps each `clientID` to a specific shard
- Each shard operates independently, eliminating cross-shard coordination

### Replication for Fault Tolerance
Each Redis shard has **replica nodes**. If a primary fails, a replica can take over. Replicas can also serve reads to distribute load.

### Failover Strategy — Fail-Close ✅
Two approaches when the rate limiter itself goes down:

| Strategy | Behavior | Use Case |
|---|---|---|
| **Fail-Open** | Allow all traffic through | Prioritizes availability |
| **Fail-Close** ✅ | Deny traffic when uncertain | Protects backend services |

**Fail-close is preferred here** — it's safer to reject requests than to allow a flood of uncontrolled traffic to overwhelm backend services.

### Latency Optimization
- Use **connection pooling** to Redis — avoids TCP handshake overhead on every request
- **Colocate** API Gateways and Redis nodes in the same data center / availability zone
- For global systems, distribute gateways and Redis nodes **geographically close to users**

---

## Dynamic Rule Configuration

Hardcoding rules into gateway logic doesn't scale. You need to update limits without redeploying.

### ❌ Polling Approach
Gateways periodically poll a config database or Redis. This introduces either latency (long poll intervals) or resource waste (high-frequency polling).

### ✅ Push-Based Configuration with ZooKeeper / etcd
- A config store (ZooKeeper or etcd) holds all rate limiting rules
- Gateways **subscribe** to the config store and receive **instant push notifications** on rule changes
- Rules are updated in-memory immediately, with zero polling overhead
- This is the same pattern used by large-scale distributed systems for config management

---

## Interview Expectations by Seniority

| Level | What's Expected |
|---|---|
| **Mid-level** | Understand the algorithms, propose a global state store, basic sharding, basic fault tolerance |
| **Senior** | Proactively identify bottlenecks, calculate required shard count, discuss fail-open vs fail-close tradeoffs, deeper fault tolerance |
| **Staff** | Deep knowledge of Redis cluster internals, connection pool tuning, atomic operations, and advanced race condition handling |

---

## Key Takeaways

- Place the rate limiter at the **API Gateway (edge)** to intercept traffic as early as possible
- Use the **Token Bucket** algorithm for its balance of burst handling and steady-rate enforcement
- Use **Redis Lua scripts** for atomic read-modify-write operations to prevent race conditions
- **Shard Redis** by client ID to scale to millions of requests per second
- Prefer **fail-close** to protect backend services from uncontrolled traffic
- Use **push-based config management** (ZooKeeper/etcd) for dynamic, zero-latency rule updates
