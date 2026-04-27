# Monitoring and Observability

> **Goal**: Understand how to keep your cloud applications healthy, detect problems before users do, and respond to incidents quickly.

---

## Why Monitoring Matters

Imagine you deploy your app to the cloud. It works! Users are happy. Then one day:
- The app becomes **slow** — but you don't know why
- A database runs out of storage — but you find out from an angry customer
- An API starts returning errors — but nobody notices for 3 hours

**Monitoring** = continuously watching your systems so you know what's happening at all times.

### Without Monitoring

```
User: "Your app is broken!"
You:  "What? Since when?"
User: "I don't know, maybe hours?"
You:  "Let me check... I have no idea what's wrong."
      *starts guessing*
```

### With Monitoring

```
Alert: "⚠️ Error rate on /api/orders exceeded 5% at 14:32"
You:   "I see the spike in the dashboard. The database connection pool
        is maxed out. Let me increase the pool size."
       *fixes in 10 minutes*
```

> **Key Principle**: You can't fix what you can't see. Monitoring gives you visibility into your systems.

### Real-World Impact

- **SLA (Service Level Agreement)** — A promise to customers: "Our service will be available 99.9% of the time." That's only ~8.7 hours of downtime allowed per YEAR.
- **MTTR (Mean Time To Recovery)** — How fast you fix problems. Good monitoring = faster MTTR.
- **Revenue** — Every minute of downtime for a large e-commerce site can mean thousands in lost sales.

| Availability | Downtime/Year | Downtime/Month | Downtime/Week |
|-------------|---------------|----------------|---------------|
| 99% ("two nines") | 3.65 days | 7.3 hours | 1.68 hours |
| 99.9% ("three nines") | 8.77 hours | 43.8 minutes | 10.1 minutes |
| 99.99% ("four nines") | 52.6 minutes | 4.38 minutes | 1.01 minutes |
| 99.999% ("five nines") | 5.26 minutes | 26.3 seconds | 6.05 seconds |

---

## The Three Pillars of Observability

**Observability** is the ability to understand what's happening inside your system by looking at its outputs. It's built on three pillars:

```
┌─────────────────────────────────────────────────────┐
│                  OBSERVABILITY                       │
│                                                      │
│   ┌───────────┐  ┌───────────┐  ┌───────────┐      │
│   │   LOGS    │  │  METRICS  │  │  TRACES   │      │
│   │           │  │           │  │           │      │
│   │ What      │  │ How much  │  │ Where in  │      │
│   │ happened? │  │ / how     │  │ the path? │      │
│   │           │  │ fast?     │  │           │      │
│   └───────────┘  └───────────┘  └───────────┘      │
└─────────────────────────────────────────────────────┘
```

---

### Pillar 1: Logs

**Logs** are timestamped records of events that happened in your application.

#### Unstructured Logs (Hard to Search)

```
[2024-01-15 14:32:01] ERROR - Failed to connect to database
[2024-01-15 14:32:02] WARN - Retrying database connection (attempt 2)
[2024-01-15 14:32:05] INFO - Database connection restored
```

#### Structured Logs (Easy to Search and Filter)

```json
{
  "timestamp": "2024-01-15T14:32:01Z",
  "level": "ERROR",
  "service": "order-service",
  "message": "Failed to connect to database",
  "error_code": "DB_CONN_TIMEOUT",
  "host": "prod-web-03",
  "request_id": "abc-123-def"
}
```

> **Always prefer structured logs.** They can be parsed, filtered, and analyzed by machines. Use JSON format.

#### Log Levels

| Level | When to Use | Example |
|-------|-------------|---------|
| **DEBUG** | Detailed diagnostic info (dev only) | "Variable x = 42" |
| **INFO** | Normal operations | "User login successful" |
| **WARN** | Something unexpected, but not broken | "Disk usage at 85%" |
| **ERROR** | Something failed, needs attention | "Payment processing failed" |
| **FATAL/CRITICAL** | System is unusable | "Database is completely unreachable" |

```
Verbosity:  DEBUG > INFO > WARN > ERROR > FATAL

Production: Usually set to INFO or WARN (DEBUG is too noisy)
Development: Usually set to DEBUG (you want all the details)
```

#### Centralized Logging

In the cloud, you might have dozens of servers. You need all logs in ONE place.

```
Server 1 ──┐
Server 2 ──┤
Server 3 ──┼──→  [Central Log System]  ──→  Search & Analyze
Server 4 ──┤          (e.g., ELK Stack,
Server 5 ──┘           CloudWatch Logs)
```

**Popular centralized logging tools**:
- **ELK Stack** — Elasticsearch + Logstash + Kibana (open source)
- **AWS CloudWatch Logs** — Built-in AWS logging
- **Azure Log Analytics** — Built-in Azure logging
- **Datadog** — Third-party SaaS monitoring platform
- **Splunk** — Enterprise log analysis

---

### Pillar 2: Metrics

**Metrics** are numerical measurements collected over time. They tell you "how much" or "how fast."

#### Types of Metrics

| Metric Type | What It Measures | Example |
|-------------|-----------------|---------|
| **Counter** | Cumulative total (only goes up) | Total HTTP requests: 1,000,000 |
| **Gauge** | Current value (goes up and down) | Current CPU usage: 73% |
| **Histogram** | Distribution of values | Response time: p50=120ms, p99=800ms |

#### Common Metrics to Track

**Infrastructure Metrics:**
- CPU utilization (%)
- Memory usage (%)
- Disk I/O (reads/writes per second)
- Network throughput (bytes in/out)

**Application Metrics:**
- Request count (requests per second)
- Error rate (% of failed requests)
- Response time / latency (milliseconds)
- Active connections
- Queue depth

**Business Metrics:**
- Orders per minute
- Revenue per hour
- User signups per day
- Cart abandonment rate

#### Time-Series Data

Metrics are stored as **time-series data** — values recorded at regular intervals:

```
Timestamp            | CPU Usage | Memory Usage | Requests/sec
---------------------|-----------|--------------|-------------
2024-01-15 14:00:00  | 45%       | 62%          | 150
2024-01-15 14:01:00  | 48%       | 63%          | 165
2024-01-15 14:02:00  | 92%       | 78%          | 430    ← spike!
2024-01-15 14:03:00  | 95%       | 85%          | 450    ← still high
2024-01-15 14:04:00  | 50%       | 64%          | 160    ← back to normal
```

---

### Pillar 3: Traces (Distributed Tracing)

In a microservices architecture, a single user request might travel through **many services**:

```
User clicks "Place Order"
    ↓
[API Gateway] → [Order Service] → [Payment Service] → [Inventory Service]
                                                     → [Notification Service]
```

If the order is slow, **which service is the bottleneck?** Traces answer this.

#### How Tracing Works

1. A unique **Trace ID** is created when the request enters the system
2. Each service creates a **Span** (a unit of work with start/end times)
3. The Trace ID is passed along to every service
4. You can reconstruct the full journey of the request

```
Trace ID: abc-123

├── [API Gateway]         0ms ─────── 250ms  (total: 250ms)
│   ├── [Order Service]   10ms ────── 200ms  (total: 190ms)
│   │   ├── [Payment]     20ms ──── 150ms    (total: 130ms)  ← SLOW!
│   │   └── [Inventory]   155ms ── 180ms     (total: 25ms)
│   └── [Notification]    205ms ─ 240ms      (total: 35ms)
```

Now you can see: **Payment Service** is taking 130ms — that's the bottleneck!

**Popular distributed tracing tools**:
- **AWS X-Ray** — Built-in AWS tracing
- **Azure Application Insights** — Built-in Azure tracing
- **Jaeger** — Open source (CNCF project)
- **Zipkin** — Open source
- **OpenTelemetry** — Vendor-neutral standard for traces, metrics, and logs

---

## Key Metrics Frameworks

### The Four Golden Signals (Google SRE)

From Google's Site Reliability Engineering book — the four most important signals to monitor for any service:

| Signal | What It Measures | Example Alert |
|--------|-----------------|---------------|
| **Latency** | Time to serve a request | "p99 latency > 500ms for 5 minutes" |
| **Traffic** | How much demand on the system | "Requests/sec dropped 50% suddenly" |
| **Errors** | Rate of failed requests | "Error rate > 1% for 10 minutes" |
| **Saturation** | How full the system is | "CPU at 90% for 15 minutes" |

### The USE Method (for Resources)

Best for monitoring **infrastructure resources** (CPU, memory, disk, network):

| Metric | Question | Example |
|--------|----------|---------|
| **Utilization** | How busy is the resource? | CPU at 85% |
| **Saturation** | Is work waiting/queuing? | 50 requests queued |
| **Errors** | Are there error events? | 3 disk I/O errors |

### The RED Method (for Services)

Best for monitoring **microservices and APIs**:

| Metric | Question | Example |
|--------|----------|---------|
| **Rate** | How many requests per second? | 200 req/s |
| **Errors** | How many of those are failing? | 5 errors/s (2.5%) |
| **Duration** | How long do requests take? | p50=50ms, p99=200ms |

> **Tip**: Use **Golden Signals** for overall service health, **USE** for infrastructure, and **RED** for individual microservices.

---

## Alerting

### What is Alerting?

**Alerting** = Automatically notifying someone when a metric crosses a threshold.

```
Rule: IF error_rate > 5% FOR 5 minutes THEN alert on-call engineer
```

### Alert Severity Levels

| Severity | Meaning | Response | Example |
|----------|---------|----------|---------|
| **Critical / P1** | Service is down | Wake someone up (page) | Website completely unreachable |
| **High / P2** | Major feature broken | Respond within 30 min | Checkout failing for all users |
| **Medium / P3** | Degraded experience | Respond within 4 hours | Search is slow but working |
| **Low / P4** | Minor issue | Respond next business day | Typo in error message |

### Alert Fatigue

> **The biggest monitoring anti-pattern**: Too many alerts that nobody cares about.

```
❌ Bad:  500 alerts/day → Team ignores ALL alerts → Real incidents missed
✅ Good: 2-3 alerts/day → Every alert is actionable → Team responds quickly
```

**Tips to avoid alert fatigue:**
- Only alert on things that require **human action**
- Use different channels: critical → pager, warning → Slack, info → dashboard
- Tune thresholds over time (remove noisy alerts)
- Add context to alerts: "CPU high because of scheduled batch job" ≠ "CPU high because of DDoS"

### Alerting Tools

- **PagerDuty** — On-call management and incident response
- **Opsgenie** — Alert management (now part of Atlassian)
- **AWS SNS** — Simple Notification Service for alert delivery
- **Azure Alerts** — Built-in alerting in Azure Monitor

---

## Dashboards

**Dashboards** are visual displays of your key metrics — a "control panel" for your system.

### What Makes a Good Dashboard?

```
┌────────────────────────────────────────────────────────┐
│  Production Dashboard                                    │
│                                                          │
│  Requests/sec    Error Rate     p99 Latency    CPU      │
│  ┌──────────┐   ┌──────────┐  ┌──────────┐  ┌────────┐│
│  │  1,250   │   │   0.3%   │  │  180ms   │  │  45%   ││
│  │  ↑ 12%   │   │  ✅ OK    │  │  ✅ OK    │  │ ✅ OK  ││
│  └──────────┘   └──────────┘  └──────────┘  └────────┘│
│                                                          │
│  [Request Rate Graph over last 24h]                      │
│  [Error Rate Graph over last 24h]                        │
│  [Latency Percentiles Graph over last 24h]               │
└────────────────────────────────────────────────────────┘
```

**Dashboard best practices:**
- Put the most critical metrics at the top
- Use RED/GREEN indicators for quick status checks
- Show trends (last 24h, last 7 days) not just current values
- Create separate dashboards for different audiences (ops team vs business)

### Dashboard Tools

| Tool | Type | Notes |
|------|------|-------|
| **Grafana** | Open source | The most popular dashboard tool. Works with many data sources. |
| **AWS CloudWatch Dashboards** | AWS | Built into AWS. Easy for AWS metrics. |
| **Azure Dashboards** | Azure | Built into Azure portal. |
| **Datadog** | SaaS | Full observability platform with built-in dashboards. |
| **New Relic** | SaaS | Application performance monitoring with dashboards. |

---

## Health Checks

**Health checks** are endpoints or mechanisms that report whether your application is running correctly.

### Types of Health Checks

#### Liveness Probe — "Is the app alive?"

Checks if the application process is running and not deadlocked.

```
GET /health/live → 200 OK     (app is running)
GET /health/live → 503 Error  (app is stuck — restart it!)
```

**If liveness fails** → The platform restarts the container/instance.

#### Readiness Probe — "Is the app ready to accept traffic?"

Checks if the app can handle requests (database connected, cache loaded, etc.).

```
GET /health/ready → 200 OK     (send traffic to me)
GET /health/ready → 503 Error  (don't send traffic yet — still warming up)
```

**If readiness fails** → The load balancer stops sending traffic to this instance (but doesn't restart it).

### Example Health Check Endpoint

```json
// GET /health
{
  "status": "healthy",
  "checks": {
    "database": { "status": "healthy", "latency_ms": 5 },
    "cache": { "status": "healthy", "latency_ms": 2 },
    "external_api": { "status": "degraded", "latency_ms": 1200 }
  },
  "uptime_seconds": 86400,
  "version": "2.3.1"
}
```

### Uptime Monitoring

External services that check your application from the outside:
- **Pingdom** — Checks if your website is reachable from multiple locations
- **UptimeRobot** — Free tier with 5-minute checks
- **AWS Route 53 Health Checks** — DNS-level health checks

---

## Incident Response Basics

When something goes wrong in production, you need a structured process:

### The Incident Lifecycle

```
┌──────────┐   ┌──────────┐   ┌──────────┐   ┌──────────┐   ┌──────────────┐
│  Detect  │ → │  Triage  │ → │ Mitigate │ → │ Resolve  │ → │ Post-Mortem  │
│          │   │          │   │          │   │          │   │              │
│ Alert    │   │ Assess   │   │ Stop the │   │ Fix the  │   │ What went    │
│ fires    │   │ severity │   │ bleeding │   │ root     │   │ wrong? How   │
│          │   │ & impact │   │          │   │ cause    │   │ do we prevent│
│          │   │          │   │          │   │          │   │ it again?    │
└──────────┘   └──────────┘   └──────────┘   └──────────┘   └──────────────┘
```

### Step by Step

1. **Detect** — An alert fires or a user reports an issue
2. **Triage** — How bad is it? Who's affected? What's the severity?
3. **Mitigate** — Stop the immediate damage (rollback, scale up, redirect traffic)
4. **Resolve** — Find and fix the root cause properly
5. **Post-Mortem** — Blameless review: What happened? Why? How do we prevent it?

### Post-Mortem Template

```markdown
## Incident Post-Mortem: [Title]

**Date**: 2024-01-15
**Duration**: 45 minutes (14:30 - 15:15 UTC)
**Severity**: P2
**Impact**: 30% of users unable to complete checkout

### Timeline
- 14:30 — Alert: Error rate > 5% on payment-service
- 14:35 — On-call engineer acknowledged
- 14:40 — Identified: database connection pool exhausted
- 14:50 — Mitigation: increased pool size from 10 to 50
- 15:15 — Error rate back to normal

### Root Cause
A new batch job was running during peak hours, consuming all DB connections.

### Action Items
- [ ] Schedule batch jobs outside peak hours
- [ ] Add monitoring for DB connection pool usage
- [ ] Set connection limits per service
```

> **Key Rule**: Post-mortems are **blameless**. Focus on systems and processes, not individuals.

---

## Cloud Monitoring Services

### AWS Monitoring Stack

| Service | Purpose | Learn More |
|---------|---------|------------|
| **CloudWatch** | Metrics, logs, alarms, dashboards | [[AWS Monitoring]] |
| **CloudTrail** | Audit log of ALL API calls (who did what, when) | [[AWS Monitoring]] |
| **X-Ray** | Distributed tracing for microservices | [[AWS Monitoring]] |
| **CloudWatch Logs Insights** | Query and analyze logs with SQL-like syntax | [[AWS Monitoring]] |

### Azure Monitoring Stack

| Service | Purpose | Learn More |
|---------|---------|------------|
| **Azure Monitor** | Central monitoring hub for metrics and logs | [[Azure Monitoring]] |
| **Log Analytics** | Query and analyze logs with KQL (Kusto Query Language) | [[Azure Monitoring]] |
| **Application Insights** | Application performance monitoring + tracing | [[Azure Monitoring]] |
| **Azure Alerts** | Alerting based on metrics and logs | [[Azure Monitoring]] |

---

## Best Practices

### 1. Monitor Everything in Production

```
✅ Monitor:                        ❌ Don't Skip:
- Application metrics              - Database performance
- Infrastructure metrics           - Third-party API health
- Business metrics                 - Background job status
- Security events                  - Queue depths
```

### 2. Set Meaningful Alerts

```
❌ Bad Alert:  "CPU > 50%"         → Fires constantly, nobody cares
✅ Good Alert: "CPU > 90% for 10 minutes AND error rate > 2%"
               → Fires rarely, always actionable
```

### 3. Use Structured Logging (JSON)

```
❌ Bad:  "Error processing order 12345 for user john"
✅ Good: {"level":"ERROR","order_id":"12345","user":"john","action":"process_order","error":"payment_declined"}
```

Why? You can query structured logs: "Show me all ERROR logs where order_id = 12345"

### 4. Correlate Logs, Metrics, and Traces

Use a **request ID** that flows through all three:

```
Log:    {"request_id": "abc-123", "message": "Payment failed"}
Metric: error_count{request_id="abc-123"} = 1
Trace:  Trace abc-123: API Gateway → Order → Payment (FAILED at 130ms)
```

Now you can jump from a log entry → to the trace → see the full request path.

### 5. Automate Responses Where Possible

```
IF    disk_usage > 90%
THEN  automatically delete old log files older than 30 days

IF    instance_unhealthy for 5 minutes
THEN  automatically replace with a new instance

IF    error_rate > 10%
THEN  automatically rollback to previous deployment
```

This is sometimes called **self-healing infrastructure**.

---

## Key Takeaways

1. **Monitoring** = watching your systems; **Observability** = understanding them
2. The three pillars: **Logs** (events), **Metrics** (numbers), **Traces** (request paths)
3. Use the **Four Golden Signals** for overall service health
4. Avoid **alert fatigue** — every alert should be actionable
5. **Health checks** tell the platform if your app is alive and ready
6. **Incident response** is a process, not a panic
7. **Post-mortems** are blameless and lead to real improvements

---

## Related Notes

- [[AWS Monitoring]] — AWS CloudWatch, CloudTrail, X-Ray in detail
- [[Azure Monitoring]] — Azure Monitor, Application Insights in detail
- [[Cloud Deployment Strategies]] — Deploy strategies that affect monitoring needs
- [[Cost Management]] — Monitoring itself has costs — budget for it
- [[Cloud Fundamentals]] — Core cloud concepts
