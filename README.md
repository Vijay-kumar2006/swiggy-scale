# swiggy-scale

Swiggy-scale is a first-principles incident analysis of a Swiggy-like food delivery platform under World Cup final traffic, with a redesigned AWS architecture, cost model, and on-call runbook.

## Scenario

At 8:05 PM during India vs Pakistan World Cup final night, a 50% promo push reaches a 10M-user spike against a single Node.js server and a single PostgreSQL database. The repository explains what fails first, why it fails, and what to build instead.

## Documents

| Document | What it contains |
| --- | --- |
| [docs/FAILURE-CASCADE.md](docs/FAILURE-CASCADE.md) | Traffic math, component limits, cascade order, and the full incident timeline. |
| [docs/ARCHITECTURE.md](docs/ARCHITECTURE.md) | The current monolith, the redesigned multi-tier architecture, and a component justification table. |
| [docs/COST-ESTIMATE.md](docs/COST-ESTIMATE.md) | Baseline and peak AWS cost calculations using concrete instance types and formulas. |
| [docs/RUNBOOK.md](docs/RUNBOOK.md) | A 5-step incident runbook for detecting, triaging, responding, rolling back, and postmortems. |

## Key Findings

- The PostgreSQL pool is the first hard limit: with 30% payment traffic and 800 ms hold time, the 100-connection pool exhausts at about 394 RPS.
- The incoming spike is roughly 500,000 RPS, which is more than 40x the single Node.js instance's effective capacity.
- Synchronous payment calls multiply connection hold time and create a self-reinforcing queue behind the database.
- Serving images from Node.js without a CDN can saturate the NIC before the API layer stabilizes.
- Redis, PgBouncer, read replicas, SQS, and CloudFront each map directly to one failure in the cascade rather than adding complexity for its own sake.

## Architecture Overview

The redesigned system pushes static traffic to CloudFront, places an ALB in front of stateless Node.js instances, and moves menu reads and promo locking into Redis. PostgreSQL is kept for writes, with PgBouncer and read replicas isolating it from read-heavy traffic, while payments become asynchronous through SQS and worker services.

## Tech Stack Context

This analysis covers Node.js, PostgreSQL, Redis, and AWS services including CloudFront, ALB, EC2, RDS, ElastiCache, PgBouncer, and SQS.