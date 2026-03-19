Here are the two additional rows with full panel specs and queries.

---

## Row 5 — VM Health

This row answers "are all my machines online and healthy" using Heartbeat telemetry and agent status.

---

### Panel 5.1: VMs Online (Stat)

**Panel Type:** Stat
**Grid Position:** 4w × 5h
**Color Mode:** Background
**Query Type:** Azure Log Analytics

**KQL Query:**
```kql
Heartbeat
| where TimeGenerated >= ago(15m)
| summarize dcount(Computer)
```

**Thresholds:** Blue (#6e9fff) at 0

---

### Panel 5.2: VMs Offline (Stat)

**Panel Type:** Stat
**Grid Position:** 4w × 5h

**KQL Query:**
```kql
let allVMs = Heartbeat
| where TimeGenerated >= ago(24h)
| distinct Computer;
let recentVMs = Heartbeat
| where TimeGenerated >= ago(15m)
| distinct Computer;
allVMs
| where Computer !in (recentVMs)
| summarize OfflineVMs = count()
```

This compares machines seen in the last 24 hours against machines seen in the last 15 minutes. The difference is considered offline.

**Thresholds:**
| Value | Color |
|---|---|
| 0 | Green |
| 1 | Dark red |

---

### Panel 5.3: Avg Heartbeat Age (Stat)

**Panel Type:** Stat
**Grid Position:** 4w × 5h
**Unit:** Minutes (m)

**KQL Query:**
```kql
Heartbeat
| summarize LastBeat = max(TimeGenerated) by Computer
| extend AgeMinutes = datetime_diff('minute', now(), LastBeat)
| summarize round(avg(AgeMinutes), 1)
```

**Thresholds:**
| Value | Color |
|---|---|
| 0 | Green |
| 5 | Yellow (#EAB839) |
| 15 | Dark red |

---

### Panel 5.4: Max Heartbeat Age (Stat)

**Panel Type:** Stat
**Grid Position:** 4w × 5h
**Unit:** Minutes (m)

**KQL Query:**
```kql
Heartbeat
| summarize LastBeat = max(TimeGenerated) by Computer
| extend AgeMinutes = datetime_diff('minute', now(), LastBeat)
| summarize round(max(AgeMinutes), 1)
```

**Thresholds:**
| Value | Color |
|---|---|
| 0 | Green |
| 10 | Yellow (#EAB839) |
| 30 | Semi-dark orange |
| 60 | Dark red |

---

### Panel 5.5: Agent Types (Stat)

**Panel Type:** Stat
**Grid Position:** 4w × 5h
**Text Mode:** Value and name

**KQL Query:**
```kql
Heartbeat
| where TimeGenerated >= ago(1h)
| summarize dcount(Computer) by Category
```

Returns something like `Direct Agent: 5`, `Azure Monitor Agent: 3`. Shows you the mix of agent types across the fleet.

**Thresholds:** Blue (#6e9fff) at 0

---

### Panel 5.6: OS Distribution (Stat)

**Panel Type:** Stat
**Grid Position:** 4w × 5h
**Text Mode:** Value and name

**KQL Query:**
```kql
Heartbeat
| where TimeGenerated >= ago(1h)
| summarize VMs = dcount(Computer) by OSType
```

**Thresholds:** Blue (#6e9fff) at 0

---

### Panel 5.7: Heartbeat Timeline — Per VM (Time Series)

**Panel Type:** Time Series
**Grid Position:** 12w × 10h, left side

**KQL Query:**
```kql
Heartbeat
| summarize Heartbeats = count() by Computer, bin(TimeGenerated, 5m)
| order by TimeGenerated asc
```

Each VM gets its own line. Gaps in a VM's line indicate periods where no heartbeat was received.

**Settings:**
- Draw style: Line
- Line interpolation: Smooth
- Line width: 2
- Fill opacity: 10
- Gradient mode: Opacity
- Color: Fixed green (#73bf69)
- Show points: Never
- Threshold style: Off
- Legend: Table, placement Right, calcs = Mean, Last
- Tooltip: All series, sort descending

---

### Panel 5.8: Heartbeat Freshness — Per VM (Time Series)

**Panel Type:** Time Series
**Grid Position:** 12w × 10h, right side

**KQL Query:**
```kql
Heartbeat
| summarize LastBeat = max(TimeGenerated) by Computer, bin(TimeGenerated, 5m)
| extend AgeMins = datetime_diff('minute', bin(now(), 5m), LastBeat)
| project Computer, TimeGenerated, AgeMins
| order by TimeGenerated asc
```

Shows how stale each VM's heartbeat is over time. A flat line at 0-1 minutes is healthy. Rising lines mean the VM stopped reporting.

**Settings:**
- Unit: Minutes (m)
- Min: 0
- Threshold style: Area
- Color mode: Thresholds

**Thresholds:**
| Value | Color |
|---|---|
| 0 | Green |
| 5 | Yellow (#EAB839) |
| 15 | Dark red |

---

### Panel 5.9: VM Health Detail (Table)

**Panel Type:** Table
**Grid Position:** 24w × 8h, full width

**KQL Query:**
```kql
let startDateTime = $__timeFrom;
let endDateTime = $__timeTo;
let allHeartbeats = Heartbeat
| where TimeGenerated between (startDateTime .. endDateTime);
allHeartbeats
| summarize
    LastHeartbeat = max(TimeGenerated),
    HeartbeatCount = count(),
    OSType = take_any(OSType),
    OSName = take_any(OSMajorVersion),
    AgentVersion = take_any(Version),
    AgentCategory = take_any(Category),
    ComputerIP = take_any(ComputerIP)
    by Computer
| extend
    AgeMins = datetime_diff('minute', now(), LastHeartbeat),
    Status = iff(datetime_diff('minute', now(), LastHeartbeat) <= 15, "Online", "Offline")
| project
    Computer,
    Status,
    AgeMins,
    LastHeartbeat,
    HeartbeatCount,
    OSType,
    AgentCategory,
    AgentVersion,
    ComputerIP
| order by AgeMins desc
```

**Format:** Table (not Logs), dashboardTime off

**Column Overrides:**

**Status column:**
- Cell display: Color background
- Value mappings: "Online" → Green, "Offline" → Dark red

**AgeMins column:**
- Display name: "Age (min)"
- Thresholds: Green 0 / Yellow 5 / Red 15
- Cell display: Color background

**How to recreate in UI:**
1. Add Panel → Table
2. Paste the query, format = Table, dashboard time OFF
3. Click the Status column header → Override field → Cell display mode = Color background
4. Add value mappings: Online = Green, Offline = Dark red
5. Click AgeMins column → Override → Rename to "Age (min)", set thresholds, cell display = Color background

---

## Row 6 — Hotspots

This row answers "which specific VMs need attention right now" by ranking machines across all four metrics and flagging anything breaching thresholds.

---

### Panel 6.1: Critical Hotspots (Stat)

**Panel Type:** Stat
**Grid Position:** 6w × 5h
**Purpose:** Total count of VMs exceeding critical thresholds on any metric

**KQL Query:**
```kql
let cpuHot = Perf
| where CounterName == "% Processor Time" and InstanceName == "_Total"
| summarize AvgCPU = avg(CounterValue) by Computer
| where AvgCPU > 90
| project Computer;
let memHot = Perf
| where CounterName == "% Committed Bytes In Use"
| summarize AvgMem = avg(CounterValue) by Computer
| where AvgMem > 90
| project Computer;
let diskHot = Perf
| where ObjectName == "LogicalDisk" and CounterName == "% Free Space" and InstanceName != "_Total"
| extend UsedPct = 100.0 - CounterValue
| summarize AvgDisk = avg(UsedPct) by Computer
| where AvgDisk > 90
| project Computer;
union cpuHot, memHot, diskHot
| distinct Computer
| summarize CriticalVMs = count()
```

**Thresholds:**
| Value | Color |
|---|---|
| 0 | Green |
| 1 | Dark red |

---

### Panel 6.2: Warning Hotspots (Stat)

**Panel Type:** Stat
**Grid Position:** 6w × 5h

**KQL Query:**
```kql
let cpuWarn = Perf
| where CounterName == "% Processor Time" and InstanceName == "_Total"
| summarize AvgCPU = avg(CounterValue) by Computer
| where AvgCPU between (70.0 .. 90.0)
| project Computer;
let memWarn = Perf
| where CounterName == "% Committed Bytes In Use"
| summarize AvgMem = avg(CounterValue) by Computer
| where AvgMem between (70.0 .. 90.0)
| project Computer;
let diskWarn = Perf
| where ObjectName == "LogicalDisk" and CounterName == "% Free Space" and InstanceName != "_Total"
| extend UsedPct = 100.0 - CounterValue
| summarize AvgDisk = avg(UsedPct) by Computer
| where AvgDisk between (75.0 .. 90.0)
| project Computer;
union cpuWarn, memWarn, diskWarn
| distinct Computer
| summarize WarningVMs = count()
```

**Thresholds:**
| Value | Color |
|---|---|
| 0 | Green |
| 1 | Yellow (#EAB839) |
| 3 | Semi-dark orange |

---

### Panel 6.3: Healthy VMs (Stat)

**Panel Type:** Stat
**Grid Position:** 6w × 5h

**KQL Query:**
```kql
let allVMs = Perf
| where CounterName == "% Processor Time" and InstanceName == "_Total"
| distinct Computer;
let hotVMs = Perf
| where CounterName == "% Processor Time" and InstanceName == "_Total"
| summarize Avg = avg(CounterValue) by Computer
| where Avg > 70
| project Computer;
let memHot = Perf
| where CounterName == "% Committed Bytes In Use"
| summarize Avg = avg(CounterValue) by Computer
| where Avg > 70
| project Computer;
let diskHot = Perf
| where ObjectName == "LogicalDisk" and CounterName == "% Free Space" and InstanceName != "_Total"
| extend Used = 100.0 - CounterValue
| summarize Avg = avg(Used) by Computer
| where Avg > 75
| project Computer;
let unhealthy = union hotVMs, memHot, diskHot | distinct Computer;
allVMs
| where Computer !in (unhealthy)
| summarize HealthyVMs = count()
```

**Thresholds:** Green at 0

---

### Panel 6.4: Most Impacted VM (Stat)

**Panel Type:** Stat
**Grid Position:** 6w × 5h
**Text Mode:** Value and name

**KQL Query:**
```kql
let cpu = Perf
| where CounterName == "% Processor Time" and InstanceName == "_Total"
| summarize CPU = avg(CounterValue) by Computer;
let mem = Perf
| where CounterName == "% Committed Bytes In Use"
| summarize Mem = avg(CounterValue) by Computer;
let disk = Perf
| where ObjectName == "LogicalDisk" and CounterName == "% Free Space" and InstanceName != "_Total"
| extend Used = 100.0 - CounterValue
| summarize Disk = avg(Used) by Computer;
cpu
| join kind=leftouter mem on Computer
| join kind=leftouter disk on Computer
| extend Score = round((coalesce(CPU, 0) + coalesce(Mem, 0) + coalesce(Disk, 0)) / 3.0, 1)
| top 1 by Score desc
| project Computer, Score
```

Returns the single VM with the highest combined pressure score. The value displayed is the composite score (average of CPU + Memory + Disk utilization).

**Unit:** Percent
**Thresholds:**
| Value | Color |
|---|---|
| 0 | Green |
| 70 | Yellow (#EAB839) |
| 85 | Dark red |

---

### Panel 6.5: Hotspot Heatmap Table (Table)

**Panel Type:** Table
**Grid Position:** 24w × 10h, full width
**Purpose:** The main panel of this row — ranks every VM across all metrics side by side

**KQL Query:**
```kql
let startDateTime = $__timeFrom;
let endDateTime = $__timeTo;
let cpu = Perf
| where TimeGenerated between (startDateTime .. endDateTime)
| where CounterName == "% Processor Time" and InstanceName == "_Total"
| summarize CPU = round(avg(CounterValue), 1) by Computer;
let mem = Perf
| where TimeGenerated between (startDateTime .. endDateTime)
| where CounterName == "% Committed Bytes In Use"
| summarize Memory = round(avg(CounterValue), 1) by Computer;
let disk = Perf
| where TimeGenerated between (startDateTime .. endDateTime)
| where ObjectName == "LogicalDisk" and CounterName == "% Free Space" and InstanceName != "_Total"
| extend Used = 100.0 - CounterValue
| summarize Disk = round(avg(Used), 1) by Computer;
let net = Perf
| where TimeGenerated between (startDateTime .. endDateTime)
| where CounterName == "Bytes Total/sec"
| summarize NetMBps = round(avg(CounterValue) / 1024 / 1024, 2) by Computer;
cpu
| join kind=leftouter mem on Computer
| join kind=leftouter disk on Computer
| join kind=leftouter net on Computer
| extend Score = round((coalesce(CPU, 0) + coalesce(Memory, 0) + coalesce(Disk, 0)) / 3.0, 1)
| project Computer, Score, CPU, Memory, Disk, NetMBps
| order by Score desc
```

**Format:** Table, dashboardTime off

**Column Overrides:**

**CPU column:**
- Cell display: Gauge (basic mode)
- Min: 0, Max: 100
- Thresholds: Green 0 / Yellow 70 / Red 90

**Memory column:**
- Cell display: Gauge (basic mode)
- Min: 0, Max: 100
- Thresholds: Blue 0 / Yellow 70 / Red 90

**Disk column:**
- Cell display: Gauge (basic mode)
- Min: 0, Max: 100
- Thresholds: Purple 0 / Yellow 75 / Red 90

**Score column:**
- Cell display: Color background
- Thresholds: Green 0 / Yellow 60 / Orange 75 / Red 85

**NetMBps column:**
- Display name: "Network (MB/s)"
- No gauge — just right-aligned numeric

**How to recreate in UI:**
1. Add Panel → Table
2. Paste the query, format = Table, dashboard time OFF
3. For each metric column (CPU, Memory, Disk): click column header → Override → Cell display mode = Gauge (basic), set min 0 max 100, add thresholds matching the metric's color scheme
4. For Score: Override → Cell display = Color background, add thresholds
5. Sort by Score descending

---

### Panel 6.6: Hotspot Trend — Top 5 by Score (Time Series)

**Panel Type:** Time Series
**Grid Position:** 24w × 10h, full width below table

**KQL Query:**
```kql
let topVMs = Perf
| where CounterName == "% Processor Time" and InstanceName == "_Total"
| summarize CPU = avg(CounterValue) by Computer
| top 5 by CPU desc
| project Computer;
Perf
| where CounterName == "% Processor Time" and InstanceName == "_Total"
| where Computer in (topVMs)
| summarize CPU = round(avg(CounterValue), 1) by Computer, bin(TimeGenerated, 5m)
| order by TimeGenerated asc
```

Shows the CPU trend over time for only the top 5 busiest VMs. This lets you see if they're spiking or consistently saturated.

**Settings:**
- Unit: Percent (0-100)
- Min: 0, Max: 100
- Line interpolation: Smooth
- Fill opacity: 15
- Gradient mode: Scheme
- Threshold style: Area
- Thresholds: Green 0 / Yellow 70 / Red 90
- Legend: Table, Right, calcs = Mean, Max, Last

**How to recreate:**
1. Add Panel → Time Series
2. Paste the query — the inner `let` statement identifies the top 5, then the outer query filters to just those VMs
3. Apply the standard time series styling from the styling reference

---

These two rows slot in after your existing Row 4 (Network). So the final performance dashboard structure becomes: CPU → Memory → Disk → Network → VM Health → Hotspots.
