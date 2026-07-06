# Incident Runbook

This runbook is written for a junior engineer waking up at 2 AM. Follow it in order.

## Step 1 - Detect

Create and maintain these CloudWatch alarms:

| Metric | Threshold | Alert Type |
| --- | --- | --- |
| ALB 5xx error rate | > 5% for 2 minutes | Critical |
| ALB target response time p99 | > 2 seconds for 5 minutes | Critical |
| RDS DatabaseConnections | > 80% of max_connections | Warning |
| RDS CPUUtilization | > 85% for 3 minutes | Warning |
| Node.js instance CPUUtilization | > 80% for 3 minutes | Critical |
| Node.js memory usage | > 75% for 3 minutes | Warning |
| Redis memory usage | > 75% | Warning |
| Redis cache miss rate | > 50% | Critical |
| SQS queue depth | > 10,000 messages | Warning |
| Promo budget remaining | < 10% | Warning |

## Step 2 - Triage

Use this order. Stop at the first red signal.

1. Check CloudWatch -> RDS -> DatabaseConnections.
If red, the root cause is DB pool pressure. Go to Step 3a.

2. Check CloudWatch -> EC2 -> CPUUtilization for all Node.js instances.
If all are red, the root cause is compute saturation. Go to Step 3b.

3. Check Redis -> CacheMisses and MemoryUsage.
If cache misses are above 50% or memory is near the limit, go to Step 3c.

4. Check SQS -> ApproximateNumberOfMessagesVisible.
If the queue is above 50,000, the payment worker path is backed up. Go to Step 3d.

5. Check ALB -> 5xx and TargetResponseTime.
If only the load balancer is red, confirm whether the app targets are unhealthy or whether one upstream system is failing first.

The goal is to identify the first failing component in 30 seconds, not to prove every theory.

## Step 3 - Respond

### 3a) DB pool exhausted

- What to do:
	- Verify PgBouncer pool saturation and RDS connection count.
	- Temporarily lower read traffic by increasing cache TTLs for menus if needed.
	- Scale the app only after the database queue is stabilizing.
- Success looks like:
	- RDS connections fall below 70% of max.
	- p99 response time drops below 2 seconds within 5 minutes.
- Responsible team:
	- Database on-call
	- Slack channel: #oncall-db

### 3b) Compute saturation

- What to do:
	- Increase the ECS or EC2 auto-scaling target.
	- Remove any non-essential background jobs from the API process.
	- Check whether one instance is hot due to bad deployment or uneven traffic.
- Success looks like:
	- CPU falls below 70% on all instances.
	- ALB 5xx rate falls below 1% within 10 minutes.
- Responsible team:
	- Platform on-call
	- Slack channel: #oncall-platform

### 3c) Redis cache miss spike

- What to do:
	- Verify cache TTLs and key eviction.
	- Rewarm top menu keys from the database if necessary.
	- Check for bad deploys that changed the cache key format.
- Success looks like:
	- Cache hit rate rises above 80%.
	- DB read traffic drops back to expected baseline.
- Responsible team:
	- Infra on-call
	- Slack channel: #oncall-infra

### 3d) Payment queue backed up

- What to do:
	- Scale out payment workers.
	- Check external payment gateway latency and error rate.
	- Pause promo traffic if the queue continues to grow.
- Success looks like:
	- SQS visible messages trend down to under 10,000.
	- Payment completion latency returns to normal within 15 minutes.
- Responsible team:
	- Payments on-call
	- Slack channel: #oncall-payments

## Step 4 - Rollback

Roll back application code only if all of these are true for more than 5 minutes:

- ALB 5xx rate is above 20%.
- The error rate is not improving.
- There was a deploy in the last 2 hours.
- The root cause is not an external dependency.

Exact rollback command:

```bash
aws ecs update-service \
	--cluster swiggy-prod \
	--service api \
	--task-definition swiggy-api:PREVIOUS_STABLE_VERSION
```

Warning: never roll back the database schema during the incident. Roll back application code only.

## Step 5 - Postmortem Template

Use this template within 24 hours.

### Timeline

- Write the incident from first alert to recovery in timestamp order.
- Include CloudWatch, deploy, and operator action timestamps.

### Root Cause

- State the single deepest technical cause.
- Do not write "the server crashed". Name the mechanism, such as PgBouncer saturation or a promo race condition.

### Impact

- Record outage duration.
- Record affected user count.
- Record estimated revenue loss.

### What Worked

- List the mitigations that reduced blast radius.
- Include alerts, dashboards, or manual actions that helped.

### Action Items

- Number each item.
- Include one owner.
- Include one due date.
- Make each action item specific and testable.
