# Failure Cascade Analysis

## 1) Traffic Simulation Math

We use the conservative spike described in the scenario: 10,000,000 users hit the app within roughly 60 seconds after the push notification.

Assume each user generates 3 API calls in the first minute:

- GET /restaurants
- GET /restaurant/:id
- POST /orders

Peak RPS:

```text
10,000,000 users x 3 calls / 60 seconds = 500,000 RPS
```

That is the demand hitting the backend at the same time.

## 2) Component Capacity Numbers

### Monolith limits

- PostgreSQL `max_connections = 100`
- Single Node.js process on 1 CPU and 4 GB RAM
- Node.js event loop saturation on a single instance: about 12,000 to 15,000 RPS
- Synchronous payment call hold time: 200 ms to 2,000 ms per request
- No cache layer
- No CDN
- No load balancer

### When does the DB pool exhaust?

Use the assignment formula:

```text
Connections held at any moment
= (non-payment RPS x query_time_s) + (payment RPS x payment_hold_time_s)
```

With 30% payment traffic, 70% non-payment traffic, 20 ms query time, and 800 ms payment hold time:

```text
Connections held = (0.70 x RPS x 0.02) + (0.30 x RPS x 0.8)
				 = 0.014RPS + 0.24RPS
				 = 0.254RPS
```

Set that equal to the pool size:

```text
0.254RPS = 100
RPS = 100 / 0.254
RPS = 393.7
```

So the pool exhausts at about 394 RPS.

## 3) The Cascade

### Failure 1: PostgreSQL connection pool exhaustion

- Severity: CRITICAL
- Trigger: about 394 RPS
- User symptom: menu loads hang, order submission times out, and errors spike from the API layer
- Next failure: Node requests queue while waiting for a connection, which grows latency and memory pressure

Why it fails first: payment requests hold connections for 800 ms, so a 100-slot pool is consumed almost immediately.

### Failure 2: Node.js event loop saturation

- Severity: CRITICAL
- Trigger: roughly 12,000 to 15,000 RPS on one instance
- User symptom: p99 latency climbs from tens of milliseconds to seconds, then 5xx errors appear
- Next failure: request backlog increases memory usage until the process is unstable or OOMs

Why it follows: once the DB pool is blocked, Node keeps accepting work faster than it can complete it.

### Failure 3: Synchronous payment call amplification

- Severity: HIGH
- Trigger: concurrent with pool exhaustion, because every payment call holds a connection for 200 to 2,000 ms
- User symptom: order placement hangs after the payment step or returns timeout errors
- Next failure: the effective DB hold time expands by 20x to 200x relative to a simple read query

Why it matters: payment latency is not just a payment problem, it becomes a database capacity problem.

### Failure 4: Promo code race condition

- Severity: HIGH
- Trigger: within seconds of simultaneous promo checks
- User symptom: some users see successful promo application even after the budget is already gone
- Next failure: financial loss and user trust loss, not just a technical error

Why it fails: a read-then-write promo flow is not atomic under concurrency.

### Failure 5: Static asset NIC saturation

- Severity: HIGH
- Trigger: within about 15 seconds if images are served directly by Node.js
- User symptom: images stall, app feels blank, and even successful API responses are delayed by network saturation
- Next failure: API traffic is starved by static content transfer

Why it fails: the same process is trying to serve application traffic and large image payloads.

### Failure 6: Monitoring blindness

- Severity: MEDIUM
- Trigger: immediately, because there is no structured tracing or correlation ID
- User symptom: on-call sees generic 500s without a clear root cause
- Next failure: mean time to detect and repair balloons from minutes to tens of minutes

Why it matters: the outage lasts longer because the team cannot isolate the first broken component quickly.

## 4) Timeline

| Time | Event |
| --- | --- |
| T+0s | Push notification lands on 180 million devices and the first wave of 10 million users opens the app. |
| T+3s | PostgreSQL connection pool reaches 100/100 and new requests begin queueing. |
| T+5s | Node.js latency climbs sharply as the event loop waits on blocked DB connections. |
| T+8s | New DB connection attempts are rejected, so more API calls return errors instead of waiting. |
| T+10s | Payment gateway calls begin timing out, holding connections for even longer. |
| T+12s | Promo budget is oversold because validation and deduction are not atomic. |
| T+15s | Static assets saturate the NIC, making the API feel unreachable even for healthy requests. |
| T+18s | The Node.js process becomes unstable and exits after memory pressure rises. |
| T+20s | Load balancer health checks fail and all instances are marked unhealthy. |
| T+45m | On-call finally isolates the root cause after manual investigation across logs and dashboards. |
| T+2h | Service is restored after a restart, connection flush, and traffic reduction. |

## 5) Cascade Summary

The root technical cause is not a single bug. It is a system design that allows slow payment work, read traffic, promo validation, and static content to compete for the same small pool of Node.js and PostgreSQL capacity.
