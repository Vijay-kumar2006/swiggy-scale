# AWS Cost Estimate

Prices are approximate on-demand AWS prices and are used here for architecture planning, not as a billing quote.

## 1) Baseline Monthly Cost

Assume 720 hours per month.

| Service | Line Item | Formula | Monthly Cost |
| --- | --- | --- | --- |
| EC2 | t3.medium x 4 | $0.0416/hr x 4 x 720h | $119.81 |
| RDS PostgreSQL primary | db.r6g.large x 1 | $0.182/hr x 720h | $131.04 |
| RDS read replicas | db.r6g.large x 2 | $0.182/hr x 2 x 720h | $262.08 |
| ElastiCache Redis | cache.r6g.large x 3 | $0.166/hr x 3 x 720h | $358.56 |
| ALB | fixed + LCU estimate | $16.20 base + $40 LCU | $56.20 |
| CloudFront | 10 TB transfer + request fees | $0.0085/GB x 10,000 GB + about $15 requests | $100.00 |
| SQS | 1M messages/day x 30 days | $0.40/million x 30 million | $12.00 |

### Baseline Total

```text
$119.81 + $131.04 + $262.08 + $358.56 + $56.20 + $100.00 + $12.00 = $1,039.69 / month
```

## 2) Peak Event Extra Cost

Peak window: 4 hours during the World Cup final.

| Service | Extra Line Item | Formula | Extra Cost |
| --- | --- | --- | --- |
| EC2 autoscaling | 16 additional t3.2xlarge instances | $0.3328/hr x 16 x 4h | $21.30 |
| CloudFront surge | 50 TB extra transfer | $0.0085/GB x 50,000 GB | $425.00 |

### Peak Extra Total

```text
$21.30 + $425.00 = $446.30 for the 4-hour peak window
```

## 3) Business Justification

A 45-minute outage at a loss rate of ₹4.2 crore per minute costs about ₹189 crore in lost orders. Against that, a baseline infrastructure spend of about $1,040 per month and a peak-night increment of about $446 is negligible. The ROI is not subtle: the cost of resilience is tiny compared to the cost of a single prime-time failure.
