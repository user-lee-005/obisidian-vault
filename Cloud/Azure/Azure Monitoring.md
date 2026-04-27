# Azure Monitoring

> **Beginner-friendly guide** to monitoring your Azure resources — from understanding metrics and logs to setting up alerts, dashboards, and Application Insights.

---

## Table of Contents

- [[#Why Monitor in Azure?]]
- [[#Azure Monitor — The Central Platform]]
- [[#Log Analytics Workspace]]
- [[#Application Insights]]
- [[#Azure Activity Log]]
- [[#Diagnostic Settings]]
- [[#Azure Monitor vs AWS CloudWatch]]
- [[#Monitoring Best Practices]]

---

## Why Monitor in Azure?

Imagine you deploy an app and walk away. A week later, users complain: "The site is slow!" or "I keep getting errors!" Without monitoring, you'd have **no idea** what's happening.

### Monitoring answers critical questions:

| Question | What Helps |
|----------|------------|
| "Is my app running?" | **Availability monitoring**, health checks |
| "Is it fast enough?" | **Performance metrics** (response time, throughput) |
| "Are there errors?" | **Log analysis**, exception tracking |
| "Why is it slow?" | **Distributed tracing**, dependency tracking |
| "Who changed what?" | **Activity logs**, audit trails |
| "Will it break soon?" | **Alerts**, anomaly detection |

### The Three Pillars of Observability

1. **Metrics** — Numbers measured over time (CPU %, memory usage, request count)
2. **Logs** — Text records with detailed context (error messages, audit trails)
3. **Traces** — End-to-end journey of a request through your system

> 💡 **Key Insight:** Azure collects basic metrics **automatically** for all resources. You don't have to do anything — just look at them!

---

## Azure Monitor — The Central Platform

**Azure Monitor** is the **single monitoring service** for everything in Azure. All monitoring features feed into or are part of Azure Monitor.

```
                     ┌─────────────────────────┐
                     │      Azure Monitor       │
                     │                          │
  Sources:           │  ┌───────┐  ┌────────┐  │   Actions:
  ─ Applications     │  │Metrics│  │  Logs  │  │   ─ Alerts
  ─ VMs              │  │       │  │        │  │   ─ Dashboards
  ─ Databases  ────► │  │Numbers│  │  Text  │  │ ─► Autoscale
  ─ Networks         │  │ over  │  │records │  │   ─ Workbooks
  ─ Custom apps      │  │ time  │  │        │  │   ─ Export
                     │  └───────┘  └────────┘  │
                     └─────────────────────────┘
```

### Two Types of Data

| | Metrics | Logs |
|---|---------|------|
| **What** | Numbers (e.g., CPU = 75%) | Text records (e.g., "Error: connection refused") |
| **Storage** | Time-series database | Log Analytics Workspace |
| **Query with** | Metrics Explorer (charts) | KQL (Kusto Query Language) |
| **Retention** | 93 days (automatic) | Configurable (30 days–2 years) |
| **Best for** | Real-time dashboards, alerts | Troubleshooting, auditing |

### Metrics Explorer — Visualize Metrics in Real-Time

In the Azure Portal: **Monitor → Metrics** — then pick a resource, pick a metric, and see a chart.

From the CLI:

```bash
# List all available metrics for a VM
az monitor metrics list-definitions \
  --resource /subscriptions/<sub>/resourceGroups/<rg>/providers/Microsoft.Compute/virtualMachines/<vm> \
  --output table

# Get CPU percentage for the last hour
az monitor metrics list \
  --resource /subscriptions/<sub>/resourceGroups/<rg>/providers/Microsoft.Compute/virtualMachines/<vm> \
  --metric "Percentage CPU" \
  --interval PT1H

# Get average CPU over the last 24 hours, sampled every hour
az monitor metrics list \
  --resource /subscriptions/<sub>/resourceGroups/<rg>/providers/Microsoft.Compute/virtualMachines/<vm> \
  --metric "Percentage CPU" \
  --interval PT1H \
  --start-time 2024-01-01T00:00:00Z \
  --end-time 2024-01-02T00:00:00Z \
  --aggregation Average
```

**Common metrics by resource type:**

| Resource | Key Metrics |
|----------|------------|
| **VM** | Percentage CPU, Available Memory, Disk Read/Write |
| **App Service** | HTTP response time, Requests, HTTP 5xx errors |
| **SQL Database** | DTU percentage, Connection failed, Deadlocks |
| **Storage** | Transactions, Ingress/Egress, Availability |

### Alerts — Get Notified When Something Goes Wrong

Alerts monitor metrics or logs and **trigger actions** when a condition is met.

```
Metric crosses threshold → Alert fires → Action Group executes
                                          ├── Send email
                                          ├── Send SMS
                                          ├── Call webhook
                                          └── Trigger Azure Function
```

```bash
# Create a metric alert: trigger when CPU > 80% for 5 minutes
az monitor metrics alert create \
  --name HighCPUAlert \
  --resource-group myResourceGroup \
  --scopes /subscriptions/<sub>/resourceGroups/<rg>/providers/Microsoft.Compute/virtualMachines/<vm> \
  --condition "avg Percentage CPU > 80" \
  --action /subscriptions/<sub>/resourceGroups/<rg>/providers/Microsoft.Insights/actionGroups/<ag> \
  --description "Alert when CPU exceeds 80%" \
  --window-size 5m \
  --evaluation-frequency 1m

# List all alerts in a resource group
az monitor metrics alert list --resource-group myResourceGroup --output table

# Delete an alert
az monitor metrics alert delete --name HighCPUAlert --resource-group myResourceGroup
```

**Alert severity levels:**

| Severity | Meaning | Example |
|----------|---------|---------|
| 0 - Critical | Needs immediate action | Production app down |
| 1 - Error | Something is broken | High error rate |
| 2 - Warning | Potential issue | CPU consistently > 80% |
| 3 - Informational | FYI | Deployment completed |
| 4 - Verbose | Debugging | Detailed diagnostics |

### Action Groups — What Happens When Alerts Fire

Action Groups define WHO gets notified and HOW:

```bash
# Create an Action Group that sends email
az monitor action-group create \
  --resource-group myResourceGroup \
  --name myActionGroup \
  --short-name myAG \
  --action email admin you@email.com

# Add SMS notification to the action group
az monitor action-group update \
  --resource-group myResourceGroup \
  --name myActionGroup \
  --add-action sms opsphone 91 9876543210

# Add a webhook (e.g., Slack, PagerDuty, Teams)
az monitor action-group update \
  --resource-group myResourceGroup \
  --name myActionGroup \
  --add-action webhook mywebhook https://hooks.slack.com/services/xxx/yyy/zzz
```

**Action types:**
- **Email** — Send to any email address
- **SMS** — Text messages
- **Voice** — Phone calls
- **Webhook** — HTTP POST to any URL (Slack, PagerDuty, Teams, custom)
- **Azure Function** — Run serverless code
- **Logic App** — Run a workflow
- **ITSM** — Create tickets in ServiceNow, etc.

---

## Log Analytics Workspace

A Log Analytics Workspace is a **centralized repository** where all your log data is collected and stored. Think of it as a massive searchable database of events.

### Creating a Workspace

```bash
# Create a Log Analytics workspace
az monitor log-analytics workspace create \
  --resource-group myResourceGroup \
  --workspace-name myWorkspace \
  --location centralindia

# Get the workspace ID (needed for connecting resources)
az monitor log-analytics workspace show \
  --resource-group myResourceGroup \
  --workspace-name myWorkspace \
  --query customerId \
  --output tsv
```

### Kusto Query Language (KQL) — Query Your Logs

KQL is the query language for Log Analytics. It's **incredibly powerful** and reads almost like English.

> 💡 **Where to run KQL:** Azure Portal → Monitor → Logs (or Log Analytics workspace → Logs)

#### Basic KQL Syntax

```kusto
// KQL reads left-to-right with pipe (|) operators — like Unix commands!
// TableName | filter | transform | sort | display

// Example: Find all errors in the last 24 hours
AzureDiagnostics
| where TimeGenerated > ago(24h)
| where Level == "Error"
| project TimeGenerated, Resource, Message
| order by TimeGenerated desc
```

#### Common KQL Queries

```kusto
// Count requests by HTTP status code
AppRequests
| summarize Count=count() by ResultCode
| order by Count desc

// Average response time over time (renders as a chart!)
AppRequests
| summarize AvgDuration=avg(DurationMs) by bin(TimeGenerated, 1h)
| render timechart

// Find the slowest requests
AppRequests
| where DurationMs > 5000
| project TimeGenerated, Name, DurationMs, ResultCode
| order by DurationMs desc
| take 20

// Count exceptions by type
AppExceptions
| summarize Count=count() by ExceptionType
| order by Count desc

// Find specific error messages
AppTraces
| where Message contains "connection refused"
| project TimeGenerated, Message, SeverityLevel
| order by TimeGenerated desc

// Resource utilization over time
Perf
| where CounterName == "% Processor Time"
| summarize AvgCPU=avg(CounterValue) by bin(TimeGenerated, 5m), Computer
| render timechart
```

#### KQL Cheat Sheet

| Operator | Purpose | Example |
|----------|---------|---------|
| `where` | Filter rows | `where Level == "Error"` |
| `project` | Select columns | `project TimeGenerated, Message` |
| `summarize` | Aggregate | `summarize count() by Status` |
| `order by` | Sort | `order by TimeGenerated desc` |
| `take` | Limit results | `take 10` |
| `extend` | Add calculated column | `extend DurationSec = DurationMs / 1000` |
| `ago()` | Time relative to now | `where TimeGenerated > ago(1h)` |
| `bin()` | Time bucketing | `bin(TimeGenerated, 5m)` |
| `render` | Visualize | `render timechart` |
| `contains` | String search | `where Message contains "error"` |
| `join` | Join tables | `join kind=inner (Table2) on Key` |

---

## Application Insights

Application Insights is Azure's **Application Performance Management (APM)** tool. While Azure Monitor tracks infrastructure metrics, Application Insights tracks **your application's behavior**.

### What Application Insights Tracks

| Feature | What It Does |
|---------|-------------|
| **Requests** | Every HTTP request to your app (URL, duration, status code) |
| **Dependencies** | Calls your app makes to databases, APIs, external services |
| **Exceptions** | Unhandled errors with full stack traces |
| **Page Views** | Browser-side performance (for web apps) |
| **Custom Events** | Anything you want to track (e.g., "user clicked buy button") |
| **Live Metrics** | Real-time view of requests, failures, and performance |
| **Application Map** | Visual diagram of your app and its dependencies |
| **Smart Detection** | AI-powered anomaly detection |

### Setting Up Application Insights

```bash
# Create Application Insights resource
az monitor app-insights component create \
  --app myAppInsights \
  --location centralindia \
  --resource-group myResourceGroup \
  --application-type web

# Get the instrumentation key (needed for SDK integration)
az monitor app-insights component show \
  --app myAppInsights \
  --resource-group myResourceGroup \
  --query instrumentationKey \
  --output tsv

# Get the connection string (newer, preferred method)
az monitor app-insights component show \
  --app myAppInsights \
  --resource-group myResourceGroup \
  --query connectionString \
  --output tsv
```

### Connecting Application Insights to Your App

**For Azure App Service (easiest — no code changes!):**

```bash
# Enable Application Insights on an existing App Service
az webapp config appsettings set \
  --name myuniqueapp123 \
  --resource-group myResourceGroup \
  --settings APPLICATIONINSIGHTS_CONNECTION_STRING="<your-connection-string>"
```

**For a Python app (with SDK):**

```bash
pip install opencensus-ext-azure
```

```python
# In your app
from opencensus.ext.azure.log_exporter import AzureLogHandler
import logging

logger = logging.getLogger(__name__)
logger.addHandler(AzureLogHandler(
    connection_string='InstrumentationKey=<your-key>'
))

logger.warning("This log goes to Application Insights!")
```

**For a Java/Spring Boot app:**

```xml
<!-- Add to pom.xml -->
<dependency>
    <groupId>com.microsoft.azure</groupId>
    <artifactId>applicationinsights-spring-boot-starter</artifactId>
    <version>2.6.4</version>
</dependency>
```

```yaml
# application.yml
azure:
  application-insights:
    connection-string: InstrumentationKey=<your-key>
```

### Application Insights Key Features

**Application Map** — Visual view of your app and all its dependencies:

```
┌──────┐     ┌──────────┐     ┌──────────┐
│ User │────►│ Web App  │────►│ SQL DB   │
│      │     │ 200ms    │     │ 50ms     │
└──────┘     │ 2% fail  │     │ 0.1% fail│
             └────┬─────┘     └──────────┘
                  │
                  ▼
             ┌──────────┐
             │ Redis    │
             │ Cache    │
             │ 5ms      │
             └──────────┘
```

**Live Metrics** — Real-time dashboard showing requests per second, response times, and failures as they happen. Great for monitoring during deployments.

**Smart Detection** — Azure AI automatically detects:
- Sudden spike in failure rates
- Abnormally slow response times
- Memory leaks
- Unusual exception patterns

### Querying Application Insights Data

```bash
# Query Application Insights using CLI
az monitor app-insights query \
  --app myAppInsights \
  --resource-group myResourceGroup \
  --analytics-query "requests | where resultCode == '500' | take 10"

# Get metrics summary
az monitor app-insights metrics show \
  --app myAppInsights \
  --resource-group myResourceGroup \
  --metrics requests/count
```

---

## Azure Activity Log

The Activity Log is Azure's **audit trail** — it records every management operation performed on your Azure resources. Think of it as a security camera for your Azure environment.

### What It Records

- **Who** did something (which user or service principal)
- **What** they did (create, delete, modify, start, stop)
- **When** it happened (timestamp)
- **What resource** was affected

> 💡 **Important:** Activity Log records **control plane** operations (management actions like creating a VM), NOT **data plane** operations (like reading data from a database). For data plane auditing, you need diagnostic logs.

### Viewing the Activity Log

```bash
# View activity log for a resource group
az monitor activity-log list \
  --resource-group myResourceGroup \
  --output table

# Filter by caller (who did it)
az monitor activity-log list \
  --resource-group myResourceGroup \
  --caller user@domain.com \
  --output table

# Filter by time range
az monitor activity-log list \
  --resource-group myResourceGroup \
  --start-time 2024-01-01T00:00:00Z \
  --end-time 2024-01-02T00:00:00Z \
  --output table

# Filter by status (succeeded, failed)
az monitor activity-log list \
  --resource-group myResourceGroup \
  --status Failed \
  --output table
```

### Activity Log Retention

- **Default retention: 90 days** — after that, logs are gone
- For longer retention, send to Log Analytics workspace or Storage Account

```bash
# Send Activity Log to Log Analytics for long-term storage
az monitor diagnostic-settings create \
  --name activity-log-to-workspace \
  --resource /subscriptions/<sub-id> \
  --workspace <workspace-id> \
  --logs '[{"category":"Administrative","enabled":true},{"category":"Security","enabled":true}]'
```

---

## Diagnostic Settings

Diagnostic Settings tell Azure WHERE to send metrics and logs for a specific resource. Without them, detailed logs may not be collected.

### Why You Need Diagnostic Settings

By default, Azure collects **basic metrics** automatically. But detailed logs (query logs, audit logs, connection logs) need to be explicitly enabled and routed.

```
Resource (e.g., SQL Database)
  │
  ├── Metrics → automatically available in Metrics Explorer
  │
  └── Detailed Logs → need Diagnostic Settings to route them
        ├──► Log Analytics Workspace (for KQL queries)
        ├──► Storage Account (for long-term archival)
        └──► Event Hub (for streaming to external tools like Splunk)
```

### Creating Diagnostic Settings

```bash
# Enable diagnostics for a resource and send to Log Analytics
az monitor diagnostic-settings create \
  --name myDiagSettings \
  --resource <resource-id> \
  --workspace <workspace-id> \
  --metrics '[{"category":"AllMetrics","enabled":true}]' \
  --logs '[{"category":"AuditEvent","enabled":true}]'

# List existing diagnostic settings for a resource
az monitor diagnostic-settings list --resource <resource-id> --output table

# Delete diagnostic settings
az monitor diagnostic-settings delete --name myDiagSettings --resource <resource-id>
```

### Example: Enable Diagnostics for a SQL Database

```bash
# Get the SQL Database resource ID
SQL_ID=$(az sql db show --resource-group myResourceGroup --server myserver --name mydb --query id --output tsv)

# Get the Log Analytics workspace ID
WORKSPACE_ID=$(az monitor log-analytics workspace show --resource-group myResourceGroup --workspace-name myWorkspace --query id --output tsv)

# Enable all diagnostics
az monitor diagnostic-settings create \
  --name sql-diagnostics \
  --resource $SQL_ID \
  --workspace $WORKSPACE_ID \
  --metrics '[{"category":"AllMetrics","enabled":true}]' \
  --logs '[{"category":"SQLSecurityAuditEvents","enabled":true},{"category":"QueryStoreRuntimeStatistics","enabled":true}]'
```

---

## Azure Monitor vs AWS CloudWatch

If you're coming from AWS, here's how Azure monitoring maps to what you already know:

| Azure | AWS | Purpose |
|-------|-----|---------|
| **Azure Monitor** | CloudWatch | Central monitoring platform |
| **Metrics Explorer** | CloudWatch Metrics | Visualize metrics |
| **Log Analytics** | CloudWatch Logs | Centralized log storage |
| **KQL** | CloudWatch Logs Insights | Query log data |
| **Application Insights** | X-Ray + CloudWatch custom | APM and tracing |
| **Activity Log** | CloudTrail | Audit trail of management actions |
| **Alerts** | CloudWatch Alarms | Threshold-based notifications |
| **Action Groups** | SNS Topics | Notification routing |
| **Diagnostic Settings** | CloudWatch Agent config | Route logs to destinations |
| **Azure Dashboards** | CloudWatch Dashboards | Custom visualizations |
| **Workbooks** | CloudWatch Insights | Interactive reports |

### Key Differences

1. **Azure uses KQL, AWS uses CloudWatch Logs Insights** — KQL is generally considered more powerful
2. **Application Insights is deeper** than X-Ray — it's a full APM solution
3. **Azure Monitor is unified** — one service for metrics, logs, alerts. AWS splits these more
4. **Log Analytics Workspace** is a separate concept — AWS doesn't have an equivalent (logs go directly to CloudWatch Logs)

See [[AWS Monitoring]] for the AWS side of this comparison.

---

## Monitoring Best Practices

### 1. Enable Diagnostics on All Critical Resources

Don't wait until something breaks. Enable diagnostics when you create a resource:

```bash
# Make it a habit: create resource → enable diagnostics
az monitor diagnostic-settings create \
  --name diagnostics \
  --resource <resource-id> \
  --workspace <workspace-id> \
  --metrics '[{"category":"AllMetrics","enabled":true}]' \
  --logs '[{"category":"AuditEvent","enabled":true}]'
```

### 2. Set Alerts for Key Metrics

At minimum, set alerts for:

| Resource | Alert On | Threshold |
|----------|----------|-----------|
| **VMs** | CPU % | > 80% for 5 min |
| **VMs** | Available Memory | < 500 MB |
| **App Service** | HTTP 5xx errors | > 10 in 5 min |
| **App Service** | Response time | > 3 seconds avg |
| **SQL Database** | DTU % | > 80% for 10 min |
| **Storage** | Availability | < 99.9% |

### 3. Use Application Insights for Web Applications

If you're running a web app on Azure, **always** enable Application Insights. It's one of the best APM tools available and integrates seamlessly.

### 4. Create Dashboards for Each Environment

```
Dashboard: Production Overview
├── App Service: Response Time (chart)
├── App Service: HTTP 5xx (chart)
├── SQL Database: DTU % (chart)
├── VM: CPU % (chart)
└── Active Alerts (list)
```

Create dashboards in: **Azure Portal → Dashboard → New Dashboard**

### 5. Send Logs to Log Analytics for Long-Term Retention

Default retention periods are short. For compliance and troubleshooting:

```bash
# Set retention to 365 days
az monitor log-analytics workspace update \
  --resource-group myResourceGroup \
  --workspace-name myWorkspace \
  --retention-time 365
```

### 6. Learn KQL — It's Worth It

KQL is one of the most useful skills for Azure. Start with these patterns:

```kusto
// Pattern 1: Find errors
<Table> | where Level == "Error" | take 20

// Pattern 2: Count by category
<Table> | summarize count() by <Column>

// Pattern 3: Time series chart
<Table> | summarize avg(<Metric>) by bin(TimeGenerated, 1h) | render timechart

// Pattern 4: Search across all tables
search "error message text" | take 20
```

### 7. Set Up Action Groups for Notifications

Don't just create alerts — make sure someone gets notified:

```bash
# At minimum: email + Slack/Teams webhook
az monitor action-group create \
  --resource-group myResourceGroup \
  --name prod-alerts \
  --short-name prodAG \
  --action email oncall oncall@company.com \
  --action webhook slack https://hooks.slack.com/services/xxx
```

---

## Quick Reference — Monitoring Commands Cheat Sheet

```bash
# === Azure Monitor ===
az monitor metrics list --resource <id> --metric "Percentage CPU"
az monitor metrics alert create --name <alert> -g <rg> --condition "avg CPU > 80"
az monitor action-group create -g <rg> --name <ag> --action email admin you@email.com

# === Log Analytics ===
az monitor log-analytics workspace create -g <rg> --workspace-name <ws>
az monitor log-analytics workspace show -g <rg> --workspace-name <ws>

# === Application Insights ===
az monitor app-insights component create --app <name> -g <rg> --location centralindia
az monitor app-insights component show --app <name> -g <rg>
az monitor app-insights query --app <name> -g <rg> --analytics-query "<KQL>"

# === Activity Log ===
az monitor activity-log list -g <rg> --output table

# === Diagnostic Settings ===
az monitor diagnostic-settings create --name <name> --resource <id> --workspace <ws-id>
az monitor diagnostic-settings list --resource <id>
```

---

## Related Notes

- [[Monitoring and Observability]] — General monitoring concepts across all platforms
- [[Azure Getting Started]] — Setting up Azure CLI and subscriptions
- [[Azure Deployment]] — Deploy apps, then monitor them with these tools
- [[Azure Security]] — Security monitoring with Defender for Cloud
- [[AWS Monitoring]] — Compare Azure Monitor with AWS CloudWatch

---

> **Next Steps:** Now that you can monitor your apps, learn to secure them → [[Azure Security]]
