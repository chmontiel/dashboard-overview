## Enhancing Your Grafana Azure Monitor Dashboard for Alert Context

Here's a comprehensive breakdown covering all three areas.

---

### 1. Improving the Alert Table — Adding "Why" Context

The current dashboard tracks VM performance metrics well, but alert panels typically only show counts. The key fields you're missing from `AlertsManagementResources` are:

| Field | Path in ARG | Value |
|---|---|---|
| Alert reason | `properties.description` | Human-readable trigger reason |
| Monitor condition | `properties.monitorCondition` | `Fired` or `Resolved` |
| Signal type | `properties.signalType` | `Metric`, `Log`, `Activity Log` |
| Monitor service | `properties.monitorService` | `Platform`, `Log Analytics`, etc. |
| Affected resource | `properties.targetResourceName` | VM or resource name |
| Severity label | `properties.severity` | `Sev0`–`Sev4` |
| Essential criteria | `properties.essentials` | Contains threshold + condition details |
| Alert rule description | `properties.context.context.condition.allOf[].metricName` | The specific metric that triggered |

---

### 2. Azure Resource Graph Queries

**A. Enriched Active Alert Details Table** — the core panel replacement:

```kusto
AlertsManagementResources
| where type == "microsoft.alertsmanagement/alerts"
| where properties.essentials.monitorCondition == "Fired"
| extend
    AlertName       = tostring(properties.essentials.alertRule),
    Severity        = tostring(properties.essentials.severity),
    MonitorService  = tostring(properties.essentials.monitorService),
    SignalType      = tostring(properties.essentials.signalType),
    State           = tostring(properties.essentials.alertState),
    Condition       = tostring(properties.essentials.monitorCondition),
    FiredAt         = todatetime(properties.essentials.startDateTime),
    AffectedResource = tostring(properties.essentials.targetResourceName),
    ResourceType    = tostring(properties.essentials.targetResourceType),
    ResourceGroup   = tostring(properties.essentials.targetResourceGroup),
    Description     = tostring(properties.essentials.description)
| extend
    SeverityLabel = case(
        Severity == "Sev0", "🔴 Critical",
        Severity == "Sev1", "🟠 Error",
        Severity == "Sev2", "🟡 Warning",
        Severity == "Sev3", "🔵 Informational",
        Severity == "Sev4", "⚪ Verbose",
        Severity
    ),
    AgeMins = datetime_diff('minute', now(), FiredAt)
| project
    SeverityLabel,
    AlertName,
    AffectedResource,
    ResourceType,
    MonitorService,
    SignalType,
    Description,
    AgeMins,
    FiredAt,
    State,
    ResourceGroup
| order by FiredAt desc
```

**B. Warning-Only Panel (Sev2–Sev4) with Threshold Context:**

```kusto
AlertsManagementResources
| where type == "microsoft.alertsmanagement/alerts"
| where properties.essentials.monitorCondition == "Fired"
| where properties.essentials.severity in ("Sev2", "Sev3", "Sev4")
| extend
    AlertName       = tostring(properties.essentials.alertRule),
    Severity        = tostring(properties.essentials.severity),
    AffectedResource = tostring(properties.essentials.targetResourceName),
    FiredAt         = todatetime(properties.essentials.startDateTime),
    Description     = tostring(properties.essentials.description),
    SignalType      = tostring(properties.essentials.signalType),
    MonitorService  = tostring(properties.essentials.monitorService)
| extend
    WhyFired = case(
        isnotempty(Description), Description,
        strcat("Alert rule '", AlertName, "' fired via ", MonitorService, " (", SignalType, " signal)")
    )
| project
    Severity,
    AlertName,
    AffectedResource,
    WhyFired,
    MonitorService,
    FiredAt
| order by Severity asc, FiredAt desc
```

**C. Alert Count by Severity Over Time (for a time series panel using Log Analytics):**

```kusto
AlertsManagementResources
| where type == "microsoft.alertsmanagement/alerts"
| where properties.essentials.monitorCondition == "Fired"
| extend
    Severity = tostring(properties.essentials.severity),
    FiredAt  = todatetime(properties.essentials.startDateTime)
| where FiredAt >= ago(24h)
| summarize Count = count() by Severity, bin(FiredAt, 1h)
| order by FiredAt asc
```

**D. Cross-correlating Alerts with Your VM Perf Data** — join alert context to your existing CPU/memory signals in Log Analytics:

```kusto
let alertedVMs = 
    AlertsManagementResources
    | where type == "microsoft.alertsmanagement/alerts"
    | where properties.essentials.monitorCondition == "Fired"
    | extend ResourceName = tolower(tostring(properties.essentials.targetResourceName))
    | project ResourceName, AlertName = tostring(properties.essentials.alertRule), Severity = tostring(properties.essentials.severity);
Perf
| where CounterName == "% Processor Time" and InstanceName == "_Total"
| extend ResourceName = tolower(Computer)
| summarize AvgCPU = round(avg(CounterValue), 1) by ResourceName
| join kind=inner alertedVMs on ResourceName
| project ResourceName, AvgCPU, AlertName, Severity
| order by AvgCPU desc
```

> **Note:** The cross-correlation query runs in Log Analytics, not Resource Graph. Use your existing `${ds}` datasource for it.

---

### 3. Best Practices for Actionable Dashboards

**Panel design improvements:**

- **Replace plain count stats with color-coded background stats.** You're already doing this for CPU/memory VMs — apply the same `colorMode: background` pattern to alert severity counts so the panel itself turns red when critical alerts are active.

- **Add a `Description` column with text wrap enabled.** In your table panel's field override, set `custom.cellOptions.type: auto` and enable `wrapText: true` for the `Description`/`WhyFired` column so the reason is fully visible without truncation.

- **Use data links to bridge Grafana → Azure Portal.** Add a data link on the `AffectedResource` or `AlertName` column:
  ```
  https://portal.azure.com/#resource/subscriptions/${sub}/resourceGroups/${__data.fields.ResourceGroup}/overview
  ```
  This lets users click directly into the affected resource in one step.

- **Add an "Age" column with threshold coloring.** Alerts older than 2 hours that are still firing indicate something wasn't acted on. Set a threshold override on `AgeMins`: green below 30, yellow at 60, red at 120.

**Layout recommendations:**

- Place a **"Why are alerts firing?"** row directly above your existing CPU/memory rows, not buried at the bottom. Operators need context first, then raw metrics.
- Use a **single-row summary bar** at the top: `Critical Alerts | Warning Alerts | Healthy VMs | Monitored VMs` — similar to your existing hotspot panels but scoped to the alert management data rather than derived from Perf queries.
- Add a **"Last alert fired"** stat panel using `max(FiredAt)` from the ARG query. If this is recent, something is actively degrading.

**Reducing noise:**

- Filter to `monitorCondition == "Fired"` only (exclude `Resolved`) — you're likely already doing this but worth confirming.
- Add a `$severity` template variable populated from `AlertsManagementResources | distinct properties.essentials.severity` so operators can focus on just Critical or just Warning without editing queries.
- Suppress `Sev4` (Verbose) from your main table by default; show it only when the variable is explicitly selected.

**Making "degraded" meaningful:**

Rather than just showing "Sev2 - Warning", add a **Threshold Explanation column** using `case()` logic in your ARG query:

```kusto
| extend ThresholdContext = case(
    AlertName contains "CPU",    "CPU sustained above threshold",
    AlertName contains "Memory", "Memory pressure detected",
    AlertName contains "Disk",   "Disk space running low",
    AlertName contains "Heartbeat", "VM may be unreachable",
    "See alert description for details"
)
```

This gives operators an immediate plain-English signal even when the Description field is empty or generic.
