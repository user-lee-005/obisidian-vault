# AWS Monitoring

> **Summary**: Monitoring means knowing when things break BEFORE your users do. AWS provides a suite of monitoring tools — CloudWatch for metrics/logs/alarms, CloudTrail for audit trails, X-Ray for distributed tracing, and SNS for notifications. This note covers all of them with hands-on commands.

---

## Table of Contents

- [[#Why Monitor?]]
- [[#Amazon CloudWatch]]
- [[#CloudWatch Metrics]]
- [[#CloudWatch Alarms]]
- [[#CloudWatch Logs]]
- [[#CloudWatch Dashboards]]
- [[#AWS CloudTrail]]
- [[#AWS X-Ray]]
- [[#Amazon SNS]]
- [[#Monitoring Best Practices]]

---

## Why Monitor?

> **Analogy**: Monitoring is like the dashboard in your car. You don't stare at it every second, but when the engine temperature spikes or the fuel runs low, you NEED to know immediately — before the engine blows up.

### Without Monitoring

```
User: "Your website is down."
You:  "Oh... since when?"
User: "3 hours ago."
You:  "😱"
```

### With Monitoring

```
CloudWatch Alarm → SNS → Your phone: "CPU at 95%! Auto-scaling triggered."
You:  "Cool, the system handled it. Let me check the dashboard."
```

### What Should You Monitor?

| Category | What to Watch | Why |
|----------|--------------|-----|
| **Compute** | CPU, memory, disk | Know when instances are overloaded |
| **Network** | Latency, packet loss, throughput | Detect connectivity issues |
| **Application** | Error rates, response times | Know when users are affected |
| **Database** | Connections, query time, storage | Prevent database bottlenecks |
| **Cost** | Daily spend, budget alerts | Avoid surprise bills |
| **Security** | Failed logins, unauthorized access | Detect breaches early |

---

## Amazon CloudWatch

> **What is CloudWatch?** THE central monitoring service for all of AWS. Almost every AWS service sends data to CloudWatch automatically.

CloudWatch has **four main features**:

```
┌─────────────────────────────────────────────┐
│               Amazon CloudWatch              │
│                                              │
│  📊 Metrics    — Numbers over time           │
│  🔔 Alarms     — Alert when thresholds hit   │
│  📝 Logs       — Centralized log storage      │
│  📈 Dashboards — Visual graphs and widgets    │
│                                              │
└─────────────────────────────────────────────┘
```

---

## CloudWatch Metrics

> **What**: A metric is a numerical data point over time. Like "CPU utilization was 45% at 2:00 PM".

### Default Metrics (Free, Automatic)

AWS services automatically send basic metrics to CloudWatch:

| Service | Default Metrics | Frequency |
|---------|----------------|-----------|
| **EC2** | CPU, Network In/Out, Disk Read/Write, Status Checks | Every 5 minutes |
| **RDS** | CPU, Connections, Read/Write IOPS, Free Storage | Every 1 minute |
| **ELB** | Request Count, Latency, HTTP errors (4xx, 5xx) | Every 1 minute |
| **Lambda** | Invocations, Duration, Errors, Throttles | Every 1 minute |
| **S3** | Bucket Size, Number of Objects | Daily |

> ⚠️ **Important**: EC2 does NOT send **memory** or **disk space** metrics by default! You need the CloudWatch Agent for those.

### Detailed Monitoring

- **Basic** (default): Metrics every **5 minutes** — FREE
- **Detailed**: Metrics every **1 minute** — costs extra (~$3.50/month per instance)

```bash
# Enable detailed monitoring on an EC2 instance
aws ec2 monitor-instances --instance-ids i-0123456789abcdef0

# Disable detailed monitoring
aws ec2 unmonitor-instances --instance-ids i-0123456789abcdef0
```

### Viewing Metrics via CLI

```bash
# View EC2 CPU utilization for a specific instance
aws cloudwatch get-metric-statistics \
  --namespace AWS/EC2 \
  --metric-name CPUUtilization \
  --dimensions Name=InstanceId,Value=i-0123456789abcdef0 \
  --start-time 2024-01-01T00:00:00Z \
  --end-time 2024-01-02T00:00:00Z \
  --period 3600 \
  --statistics Average

# List all available metrics for EC2
aws cloudwatch list-metrics --namespace AWS/EC2

# List metrics for a specific instance
aws cloudwatch list-metrics \
  --namespace AWS/EC2 \
  --dimensions Name=InstanceId,Value=i-0123456789abcdef0
```

**Understanding the parameters**:
- `--namespace`: Service name (e.g., `AWS/EC2`, `AWS/RDS`, `AWS/ELB`)
- `--metric-name`: The specific metric (e.g., `CPUUtilization`)
- `--dimensions`: Filters (e.g., by instance ID)
- `--period`: Time interval in seconds (300 = 5 min, 3600 = 1 hour)
- `--statistics`: `Average`, `Sum`, `Minimum`, `Maximum`, `SampleCount`

### Custom Metrics

> **What**: Send your OWN application metrics to CloudWatch. Like "number of active users" or "orders processed per minute".

```bash
# Publish a custom metric
aws cloudwatch put-metric-data \
  --namespace MyApp \
  --metric-name ActiveUsers \
  --value 42 \
  --unit Count

# Publish with dimensions (e.g., per environment)
aws cloudwatch put-metric-data \
  --namespace MyApp \
  --metric-name ResponseTime \
  --value 230 \
  --unit Milliseconds \
  --dimensions Name=Environment,Value=Production Name=Endpoint,Value=/api/orders

# Publish multiple data points at once
aws cloudwatch put-metric-data \
  --namespace MyApp \
  --metric-data '[
    {"MetricName": "ActiveUsers", "Value": 42, "Unit": "Count"},
    {"MetricName": "ErrorCount", "Value": 3, "Unit": "Count"},
    {"MetricName": "ResponseTime", "Value": 230, "Unit": "Milliseconds"}
  ]'
```

**Common custom metrics to track**:
- Active users / sessions
- Request latency (p50, p95, p99)
- Error count by type
- Queue depth / processing rate
- Business metrics (orders/minute, signups/day)

### CloudWatch Agent (for Memory & Disk Metrics)

The CloudWatch Agent runs on your EC2 instance and sends **memory**, **disk**, and **custom log** data.

```bash
# Install CloudWatch Agent on Amazon Linux 2
sudo yum install amazon-cloudwatch-agent -y

# Configure it using the wizard
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-config-wizard

# Start the agent
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl \
  -a fetch-config \
  -m ec2 \
  -c file:/opt/aws/amazon-cloudwatch-agent/bin/config.json \
  -s

# Check agent status
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl -a status
```

---

## CloudWatch Alarms

> **What**: Automatically take action when a metric crosses a threshold. Like "if CPU goes above 80% for 10 minutes, send me an email and add more servers."

### Alarm States

| State | Meaning |
|-------|---------|
| **OK** | Metric is within the acceptable range |
| **ALARM** | Metric has crossed the threshold |
| **INSUFFICIENT_DATA** | Not enough data to determine state (common when newly created) |

### Creating an Alarm

```bash
# Create a CPU alarm that triggers when average CPU > 80% for 10 minutes
aws cloudwatch put-metric-alarm \
  --alarm-name HighCPU-WebServer \
  --alarm-description "Alarm when CPU exceeds 80 percent" \
  --metric-name CPUUtilization \
  --namespace AWS/EC2 \
  --statistic Average \
  --period 300 \
  --threshold 80 \
  --comparison-operator GreaterThanThreshold \
  --evaluation-periods 2 \
  --alarm-actions arn:aws:sns:ap-south-1:123456789012:my-alerts \
  --ok-actions arn:aws:sns:ap-south-1:123456789012:my-alerts \
  --dimensions Name=InstanceId,Value=i-0123456789abcdef0
```

**Breaking this down**:
- `--period 300`: Check every 5 minutes (300 seconds)
- `--evaluation-periods 2`: Must exceed threshold for 2 consecutive periods
- So: CPU must be > 80% for **10 minutes** (2 × 5 min) before alarm fires
- `--alarm-actions`: What to do when ALARM state (send SNS notification)
- `--ok-actions`: What to do when back to OK (send recovery notification)

### Common Alarm Actions

| Action | What It Does |
|--------|-------------|
| **SNS Notification** | Send email/SMS/push notification |
| **Auto Scaling** | Add or remove EC2 instances |
| **EC2 Action** | Stop, terminate, reboot, or recover an instance |
| **Lambda** | Trigger a Lambda function |

### More Alarm Examples

```bash
# Alarm: Low disk space on RDS
aws cloudwatch put-metric-alarm \
  --alarm-name LowDiskSpace-MyDB \
  --metric-name FreeStorageSpace \
  --namespace AWS/RDS \
  --statistic Average \
  --period 300 \
  --threshold 5000000000 \
  --comparison-operator LessThanThreshold \
  --evaluation-periods 1 \
  --alarm-actions arn:aws:sns:ap-south-1:123456789012:my-alerts \
  --dimensions Name=DBInstanceIdentifier,Value=my-database

# Alarm: Too many 5xx errors on ALB
aws cloudwatch put-metric-alarm \
  --alarm-name High5xxErrors-ALB \
  --metric-name HTTPCode_Target_5XX_Count \
  --namespace AWS/ApplicationELB \
  --statistic Sum \
  --period 60 \
  --threshold 10 \
  --comparison-operator GreaterThanThreshold \
  --evaluation-periods 3 \
  --alarm-actions arn:aws:sns:ap-south-1:123456789012:my-alerts \
  --dimensions Name=LoadBalancer,Value=app/my-alb/1234567890

# List all alarms
aws cloudwatch describe-alarms

# View a specific alarm
aws cloudwatch describe-alarms --alarm-names HighCPU-WebServer

# Delete an alarm
aws cloudwatch delete-alarms --alarm-names HighCPU-WebServer
```

### Composite Alarms

> Combine multiple alarms with AND/OR logic. For example: "Alert me only if BOTH CPU is high AND memory is high."

```bash
aws cloudwatch put-composite-alarm \
  --alarm-name Critical-WebServer \
  --alarm-rule "ALARM(HighCPU-WebServer) AND ALARM(HighMemory-WebServer)" \
  --alarm-actions arn:aws:sns:ap-south-1:123456789012:critical-alerts
```

---

## CloudWatch Logs

> **What**: Centralized log management. Instead of SSH-ing into each server to read logs, send ALL logs to CloudWatch Logs and search them from one place.

### Log Hierarchy

```
CloudWatch Logs
  └── Log Group       (e.g., /my-app/production)
       └── Log Stream  (e.g., instance-i-abc123)
            └── Log Events  (individual log lines)
```

- **Log Group** = Container for related logs (usually one per application or service)
- **Log Stream** = One source of logs (usually one per instance or container)
- **Log Event** = A single log entry with a timestamp and message

### Setting Up Log Collection

```bash
# Create a log group
aws logs create-log-group --log-group-name /my-app/production

# Set retention period (default is FOREVER — costs money!)
aws logs put-retention-policy \
  --log-group-name /my-app/production \
  --retention-in-days 30

# View log groups
aws logs describe-log-groups

# View log streams in a group
aws logs describe-log-streams \
  --log-group-name /my-app/production \
  --order-by LastEventTime \
  --descending
```

### Viewing Logs

```bash
# Get log events from a specific stream
aws logs get-log-events \
  --log-group-name /my-app/production \
  --log-stream-name instance-i-abc123 \
  --limit 50

# Tail logs in real-time (like tail -f)
aws logs tail /my-app/production --follow

# Tail logs from the last hour
aws logs tail /my-app/production --since 1h
```

### CloudWatch Logs Insights (Querying Logs)

> **What**: A powerful query language to search and analyze logs — like SQL for your logs.

**Query syntax examples** (run these in the CloudWatch Console → Logs → Insights):

```sql
# Find all ERROR messages in the last hour
fields @timestamp, @message
| filter @message like /ERROR/
| sort @timestamp desc
| limit 20
```

```sql
# Count errors per hour
fields @timestamp, @message
| filter @message like /ERROR/
| stats count(*) as error_count by bin(1h)
| sort error_count desc
```

```sql
# Find slowest API requests
fields @timestamp, @message
| parse @message "Request took * ms" as response_time
| filter response_time > 1000
| sort response_time desc
| limit 10
```

```sql
# Count requests by HTTP status code
fields @timestamp, @message
| parse @message "status=*" as status_code
| stats count(*) by status_code
| sort count(*) desc
```

```bash
# Run a Logs Insights query via CLI
aws logs start-query \
  --log-group-name /my-app/production \
  --start-time $(date -d '1 hour ago' +%s) \
  --end-time $(date +%s) \
  --query-string 'fields @timestamp, @message | filter @message like /ERROR/ | limit 20'

# Get query results (use the queryId from the previous command)
aws logs get-query-results --query-id <query-id>
```

### Metric Filters

> **What**: Automatically create CloudWatch metrics from log patterns. For example: every time "ERROR" appears in logs, increment an error counter metric.

```bash
# Create a metric filter that counts ERROR occurrences
aws logs put-metric-filter \
  --log-group-name /my-app/production \
  --filter-name ErrorCount \
  --filter-pattern "ERROR" \
  --metric-transformations \
    metricName=ApplicationErrors,metricNamespace=MyApp,metricValue=1

# Now you can create an ALARM on this metric!
aws cloudwatch put-metric-alarm \
  --alarm-name TooManyErrors \
  --metric-name ApplicationErrors \
  --namespace MyApp \
  --statistic Sum \
  --period 300 \
  --threshold 10 \
  --comparison-operator GreaterThanThreshold \
  --evaluation-periods 1 \
  --alarm-actions arn:aws:sns:ap-south-1:123456789012:my-alerts
```

---

## CloudWatch Dashboards

> **What**: Custom visual dashboards that display your metrics as graphs, numbers, and text widgets. Create one per application or environment.

### Console Walkthrough

1. **Open CloudWatch** → Dashboards → "Create dashboard"
2. **Name it**: `Production-Overview`
3. **Add widgets**:
   - **Line chart**: EC2 CPU utilization over time
   - **Number**: Current active ALB connections
   - **Stacked area**: Request count by status code (2xx, 4xx, 5xx)
   - **Text**: Runbook links and team contacts
   - **Log table**: Recent error logs
4. **Arrange widgets** by dragging and resizing
5. **Set auto-refresh** to 1 minute
6. **Save**

### Dashboard via CLI

```bash
# Create a dashboard with a CPU widget
aws cloudwatch put-dashboard \
  --dashboard-name Production-Overview \
  --dashboard-body '{
    "widgets": [
      {
        "type": "metric",
        "x": 0, "y": 0, "width": 12, "height": 6,
        "properties": {
          "metrics": [
            ["AWS/EC2", "CPUUtilization", "InstanceId", "i-abc123"]
          ],
          "period": 300,
          "stat": "Average",
          "region": "ap-south-1",
          "title": "EC2 CPU Utilization"
        }
      }
    ]
  }'

# List dashboards
aws cloudwatch list-dashboards

# Delete a dashboard
aws cloudwatch delete-dashboards --dashboard-names Production-Overview
```

### Recommended Dashboard Layout

```
┌─────────────────────────────────────────────────────┐
│                  PRODUCTION OVERVIEW                  │
├──────────────────────┬──────────────────────────────┤
│  EC2 CPU (line)      │  ALB Request Count (bars)    │
│  ████████████████    │  ▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓         │
├──────────────────────┼──────────────────────────────┤
│  RDS Connections     │  Error Rate (5xx)             │
│  ▓▓▓▓▓▓▓▓            │  ████                         │
├──────────────────────┴──────────────────────────────┤
│  Active Alarms: 0 OK │ Recent Errors (log widget)   │
└─────────────────────────────────────────────────────┘
```

---

## AWS CloudTrail

> **What**: An audit trail that records ALL API calls made to your AWS account. Who did what, when, and from where. Think of it as the security camera for your AWS account.

### What CloudTrail Records

Every API call becomes an **event**:
- **Who**: Which IAM user or role made the call
- **What**: Which API action (e.g., `RunInstances`, `DeleteBucket`)
- **When**: Timestamp of the call
- **Where**: Source IP address, region
- **How**: Console, CLI, SDK, or another service

### CloudTrail vs CloudWatch

| Feature | CloudTrail | CloudWatch |
|---------|-----------|------------|
| **Purpose** | WHO did WHAT (audit) | HOW is it performing (metrics) |
| **Data** | API call logs | Metrics, logs, alarms |
| **Question** | "Who deleted the database?" | "Is the database running slow?" |

### Hands-On

```bash
# View your trails
aws cloudtrail describe-trails

# Look up recent events — who launched EC2 instances?
aws cloudtrail lookup-events \
  --lookup-attributes AttributeKey=EventName,AttributeValue=RunInstances \
  --max-results 5

# Look up events by user
aws cloudtrail lookup-events \
  --lookup-attributes AttributeKey=Username,AttributeValue=john.doe \
  --max-results 10

# Create a trail that logs to S3
aws cloudtrail create-trail \
  --name my-audit-trail \
  --s3-bucket-name my-cloudtrail-bucket \
  --is-multi-region-trail

# Start logging
aws cloudtrail start-logging --name my-audit-trail

# Get trail status
aws cloudtrail get-trail-status --name my-audit-trail
```

### CloudTrail Best Practices

- ✅ Enable CloudTrail in **ALL regions** (catches activity everywhere)
- ✅ Enable **log file validation** (detect tampering)
- ✅ Store logs in a **separate, restricted S3 bucket**
- ✅ Enable **CloudTrail Insights** for anomaly detection
- ✅ Set up **SNS notifications** for critical events

> See [[AWS Security]] for how CloudTrail fits into your overall security strategy.

---

## AWS X-Ray

> **What**: Distributed tracing service. When a request flows through multiple microservices, X-Ray traces the ENTIRE journey — showing you exactly where time is spent and where errors occur.

### The Problem X-Ray Solves

```
User Request → API Gateway → Lambda → DynamoDB
                                    → SQS → Another Lambda → RDS

"The request is slow. But WHERE is it slow?"
```

Without X-Ray: You'd check each service's logs individually. Painful.
With X-Ray: One visual trace showing time spent at each hop.

### Key Concepts

| Concept | What It Is |
|---------|-----------|
| **Trace** | The full journey of one request through your system |
| **Segment** | One service's part of the trace |
| **Subsegment** | Detailed breakdown within a segment (e.g., a DB query) |
| **Service Map** | Visual diagram of all services and their connections |
| **Annotations** | Key-value pairs for filtering traces |

### How to Enable X-Ray

**For Lambda**: Enable "Active Tracing" in the function configuration.

**For EC2/ECS**: Install the X-Ray Daemon:
```bash
# Install X-Ray Daemon on Amazon Linux 2
curl https://s3.us-east-2.amazonaws.com/aws-xray-assets.us-east-2/xray-daemon/aws-xray-daemon-3.x.rpm -o xray.rpm
sudo yum install -y xray.rpm
sudo systemctl start xray
```

**For API Gateway**: Enable tracing in stage settings.

### Viewing Traces

```bash
# Get a summary of traces from the last 10 minutes
aws xray get-trace-summaries \
  --start-time $(date -d '10 minutes ago' +%s) \
  --end-time $(date +%s)

# Get full trace details
aws xray batch-get-traces --trace-ids <trace-id>

# Get the service graph (service map data)
aws xray get-service-graph \
  --start-time $(date -d '1 hour ago' +%s) \
  --end-time $(date +%s)
```

### Service Map (Console)

The X-Ray **Service Map** in the Console is incredibly powerful:
- Visual graph showing all your services as nodes
- Edges show request flow between services
- Color-coded: Green (healthy), Yellow (errors), Red (faults)
- Click any node to see latency distribution and error rates

---

## Amazon SNS (Simple Notification Service)

> **What**: A pub/sub messaging service for sending notifications. CloudWatch Alarms use SNS to alert you via email, SMS, or other channels.

### How SNS Works

```
CloudWatch Alarm ──▶ SNS Topic ──▶ Email subscriber
                                 ──▶ SMS subscriber
                                 ──▶ Lambda function
                                 ──▶ SQS queue
                                 ──▶ HTTP webhook
```

- **Topic** = A channel that subscribers listen to
- **Subscriber** = Something that receives messages from a topic
- **Publisher** = Something that sends messages to a topic

### Hands-On

```bash
# Create an SNS topic
aws sns create-topic --name my-alerts
# Returns: arn:aws:sns:ap-south-1:123456789012:my-alerts

# Subscribe your email to the topic
aws sns subscribe \
  --topic-arn arn:aws:sns:ap-south-1:123456789012:my-alerts \
  --protocol email \
  --notification-endpoint you@example.com

# ⚠️ IMPORTANT: Check your email and click "Confirm subscription"!

# Subscribe an SMS number
aws sns subscribe \
  --topic-arn arn:aws:sns:ap-south-1:123456789012:my-alerts \
  --protocol sms \
  --notification-endpoint +91XXXXXXXXXX

# List subscriptions
aws sns list-subscriptions-by-topic \
  --topic-arn arn:aws:sns:ap-south-1:123456789012:my-alerts

# Publish a test message
aws sns publish \
  --topic-arn arn:aws:sns:ap-south-1:123456789012:my-alerts \
  --message "Test alert: Everything is working!" \
  --subject "Test Notification"

# Delete a topic
aws sns delete-topic --topic-arn arn:aws:sns:ap-south-1:123456789012:my-alerts
```

### SNS for Alarm Notifications

This is how it all connects:

1. **Create an SNS topic** for alerts
2. **Subscribe** your email/phone to the topic
3. **Create CloudWatch Alarms** with `--alarm-actions` pointing to the topic ARN
4. When an alarm fires → SNS sends you a notification 🔔

---

## Monitoring Best Practices

### 1. Set Alarms for All Critical Metrics

**Minimum alarms every production system should have**:

| Alarm | Threshold | Why |
|-------|-----------|-----|
| CPU Utilization | > 80% for 10 min | Instance overloaded |
| Memory Utilization | > 85% | OOM risk (needs CW Agent) |
| Disk Space | < 20% free | Application will crash |
| HTTP 5xx Errors | > 10 per minute | Users seeing errors |
| Response Latency | p95 > 2 seconds | User experience degrading |
| Database Connections | > 80% of max | DB connection exhaustion |
| Health Check Failures | Any failure | Instance is unhealthy |

### 2. Use CloudTrail for Audit Trail

- Enable in all regions
- Store logs in a locked-down S3 bucket
- Regularly review for unusual activity
- Set up alerts for sensitive API calls (e.g., `DeleteBucket`, `CreateUser`)

### 3. Centralize Logs in CloudWatch Logs

- Don't SSH into instances to read logs
- Send all application logs to CloudWatch
- Set **retention policies** (don't keep logs forever — it's expensive)
- Use **Logs Insights** for searching and analysis

### 4. Create Dashboards for Each Environment

- **Production Dashboard**: Real-time view of all critical metrics
- **Development Dashboard**: Build/test results, deployment pipeline status
- Pin dashboards to your browser — check them daily

### 5. Set Up SNS Notifications

- Create separate topics for different severity levels:
  - `critical-alerts` → SMS + email (wake people up)
  - `warning-alerts` → Email only (review next business day)
  - `info-alerts` → Slack/Teams integration
- Test your notification pipeline regularly!

### 6. Review Metrics Proactively

```
❌ Bad:  Only look at dashboards when something breaks
✅ Good: Review dashboards daily, spot trends before they become problems
```

Look for:
- **Gradual increases** in CPU/memory (growing load, memory leaks)
- **Increasing error rates** (even small ones)
- **Latency creep** (gradual slowdown over days/weeks)
- **Cost anomalies** (unexpected spend spikes)

### 7. Implement the Monitoring Pyramid

```
         ▲ Business Metrics
        ▲▲▲ (revenue, signups, orders)
       ▲▲▲▲▲
      ▲▲▲▲▲▲▲ Application Metrics
     ▲▲▲▲▲▲▲▲▲ (error rates, latency, throughput)
    ▲▲▲▲▲▲▲▲▲▲▲
   ▲▲▲▲▲▲▲▲▲▲▲▲▲ Infrastructure Metrics
  ▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲ (CPU, memory, disk, network)
```

Monitor at ALL levels, not just infrastructure.

---

## Quick Reference: Monitoring Commands Cheat Sheet

```bash
# === CloudWatch Metrics ===
aws cloudwatch list-metrics --namespace AWS/EC2
aws cloudwatch get-metric-statistics --namespace AWS/EC2 --metric-name CPUUtilization --dimensions Name=InstanceId,Value=i-xxx --start-time ... --end-time ... --period 3600 --statistics Average
aws cloudwatch put-metric-data --namespace MyApp --metric-name ActiveUsers --value 42 --unit Count

# === CloudWatch Alarms ===
aws cloudwatch put-metric-alarm --alarm-name HighCPU --metric-name CPUUtilization --namespace AWS/EC2 --statistic Average --period 300 --threshold 80 --comparison-operator GreaterThanThreshold --evaluation-periods 2 --alarm-actions <sns-arn> --dimensions Name=InstanceId,Value=i-xxx
aws cloudwatch describe-alarms
aws cloudwatch delete-alarms --alarm-names HighCPU

# === CloudWatch Logs ===
aws logs create-log-group --log-group-name /my-app/production
aws logs put-retention-policy --log-group-name /my-app/production --retention-in-days 30
aws logs tail /my-app/production --follow

# === CloudTrail ===
aws cloudtrail describe-trails
aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=RunInstances --max-results 5

# === SNS ===
aws sns create-topic --name my-alerts
aws sns subscribe --topic-arn <arn> --protocol email --notification-endpoint you@email.com
aws sns publish --topic-arn <arn> --message "Test alert!"
```

---

## What to Learn Next

- [[Monitoring and Observability]] — General monitoring concepts across clouds
- [[AWS Getting Started]] — Set up your AWS account and CLI
- [[AWS Security]] — CloudTrail, GuardDuty, and security monitoring
- [[AWS Deployment]] — Monitor your CI/CD pipelines
- [[Azure Monitoring]] — Compare with Azure Monitor and Log Analytics

---

> 📝 **Created for**: AWS Cloud Learning Path
> 🏷️ Tags: #aws #monitoring #cloudwatch #cloudtrail #xray #sns #observability #alarms
