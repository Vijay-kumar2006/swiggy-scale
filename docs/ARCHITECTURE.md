# Architecture Redesign

## 1) Current Architecture

```text
10M users
	|
	v
Single Node.js Express server
1 process, 1 CPU, 4 GB RAM
|-- menu API
|-- order placement
|-- synchronous payment processing    <-- Failure 3: connection hold amplification
|-- delivery tracking
|-- promo validation                  <-- Failure 4: race condition
|-- static images and assets          <-- Failure 5: NIC saturation
	|
	v
Single PostgreSQL database
max_connections = 100                 <-- Failure 1: pool exhaustion at about 394 RPS
no replicas
no index on orders.user_id
no cache layer                        <-- Failure 1: every read hits the DB
no load balancer                      <-- single point of failure
```

Weaknesses are intentional in this diagram: every hot path shares the same process and the same database.

## 2) New Architecture

```text
10M users
	|
	v
CloudFront CDN
cache: images, JS, CSS, static menus
TTL: 5 minutes for menus, long TTL for versioned assets
	|
	v
Application Load Balancer + AWS WAF
SSL termination, health checks, rate limits
	|
	+-------------------+-------------------+-------------------+
	v                   v                   v
Node.js API-1       Node.js API-2       Node.js API-3      Node.js API-4
stateless           stateless           stateless          stateless
autoscale from 4 to 20 instances on CPU > 70% or p99 > 2s
	|
	+-------------------+-------------------+
	v                   v
Redis Cluster        PgBouncer
menus cache          connection pooler
promo lock           keeps small physical DB pool
atomic SETNX or Lua
	|                   |
	|                   v
	|             PostgreSQL Primary
	|             writes only
	|             read replicas for browse/history
	|                   |
	|                   v
	|             Read Replica 1
	|             browse queries
	|                   v
	|             Read Replica 2
	|             order history
	|
	v
SQS payment queue
order is written, payment is queued, DB connection is released
	|
	v
Payment worker service
consumes SQS, calls payment gateway, writes final status
```

## 3) Component Justification Table

| Component | Failure It Prevents | How It Prevents It |
| --- | --- | --- |
| CloudFront CDN | Failure 5: NIC saturation | Serves images, CSS, JS, and cached menus from edge locations so the Node process no longer pushes large static payloads. |
| ALB + WAF | Single point of failure and abusive traffic | Spreads traffic across healthy instances and rate-limits per IP before the app layer gets overloaded. |
| Stateless Node.js instances | Failure 2: event loop saturation on one process | Horizontal scaling adds more event loops and more memory headroom instead of putting all load on one server. |
| Redis cache | Failure 1: DB pool exhaustion from read traffic | Menu reads and promo state are served from Redis in sub-millisecond time, so most requests never reach PostgreSQL. |
| Redis atomic promo lock | Failure 4: promo race condition | SETNX or an atomic Lua script makes promo usage and decrement a single operation. |
| PgBouncer | Failure 1: too many direct DB connections | Keeps a small physical PostgreSQL pool while allowing many application-level clients to queue safely. |
| PostgreSQL read replicas | Read/write contention on the primary | Browse and history traffic goes to replicas, leaving the primary for writes. |
| SQS payment queue | Failure 3: synchronous payment amplification | The API writes the order and releases the DB connection before payment processing completes. |
| Payment worker service | Failure 3: user-facing payment latency stalls | Workers drain SQS asynchronously and isolate payment gateway latency from request latency. |

## 4) Why This Works

Each new component maps to one specific failure in the cascade. Nothing is added as decoration. The design survives because the hottest traffic is moved to cache and CDN, the database is protected by pooling and replicas, and the slowest external dependency, payments, is moved off the synchronous request path.
