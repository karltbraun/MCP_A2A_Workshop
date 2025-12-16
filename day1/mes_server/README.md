# MES MCP Server — Press 103

## Purpose

A domain-specific MCP server designed for AI agents to execute MES (Manufacturing Execution System) objectives on Press 103. Unlike generic data access servers, this server exposes **MES-domain tools** that map directly to manufacturing operations and decisions.

This server demonstrates the "single-asset agent" pattern — purpose-built tooling scoped to one piece of equipment.

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────┐
│                     MES Agent (Claude)                       │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                   MES MCP Server (Press 103)                 │
│                                                              │
│  ┌─────────────────┐              ┌──────────────────────┐  │
│  │  MQTT Client    │              │   MySQL Client       │  │
│  │  (UNS Cache)    │              │   (MES_Lite)         │  │
│  └────────┬────────┘              └──────────┬───────────┘  │
│           │                                  │              │
│           ▼                                  ▼              │
│  Enterprise/Dallas/Press/Press 103/#    mes_lite.* tables   │
└─────────────────────────────────────────────────────────────┘
```

**Key Design Decisions:**
- Standalone server with own MQTT + MySQL connections (not dependent on other MCP servers)
- Scoped to Press 103 only (`LineID = 1` in mes_lite, `Enterprise/Dallas/Press/Press 103/` in UNS)
- MES-domain tools (not generic query/read)
- File-based MQTT cache for instant responses (following established pattern)

---

## Data Sources

### MQTT / UNS (Real-time State)

**Broker:** `balancer.virtualfactory.online:1883`  
**Topic Base:** `Enterprise/Dallas/Press/Press 103/`  
**Subscription:** `Enterprise/Dallas/Press/Press 103/#`

#### Key Topic Areas

| Path | Description | Key Tags |
|------|-------------|----------|
| `/Dashboard/` | High-level KPIs | `Running`, `Line Speed`, `Shift`, `Shift Production`, `Color` |
| `/Line/OEE/` | OEE metrics | `OEE`, `OEE Availability`, `OEE Performance`, `OEE Quality`, `Good Count`, `Bad Count`, `Runtime`, `WorkOrder` |
| `/Line/` | Line status | `State`, `Infeed`, `Outfeed`, `Waste`, `RunTime`, `Rate Setpoint` |
| `/MQTT/` | Machine-level signals | `machine_running`, `actual_speed_mts_min`, `State`, `Produced LF` |
| `/MQTT/Shop Floor/` | Current production context | `Production ID`, `Work Order DS`, `Master Item No`, `Operator ID` |
| `/MQTT/Specifications/` | Job specs | `LF Per HR`, `Standard Rate`, `No Colors`, `Print Width`, `Repeat` |
| `/Roll Info/` | Roll tracking | `Job Jacket`, `Run ID`, `Roll Consume Details`, `Roll Produce Details` |

#### Important Tag Mappings

| MES Concept | UNS Topic |
|-------------|-----------|
| Equipment Running | `Enterprise/Dallas/Press/Press 103/Dashboard/Running` |
| Current State Code | `Enterprise/Dallas/Press/Press 103/Line/State` |
| Current Work Order | `Enterprise/Dallas/Press/Press 103/Line/OEE/WorkOrder` |
| Good Count | `Enterprise/Dallas/Press/Press 103/Line/OEE/Good Count` |
| Target Count | `Enterprise/Dallas/Press/Press 103/Line/OEE/Target Count` |
| OEE | `Enterprise/Dallas/Press/Press 103/Line/OEE/OEE` |
| OEE Availability | `Enterprise/Dallas/Press/Press 103/Line/OEE/OEE Availability` |
| OEE Performance | `Enterprise/Dallas/Press/Press 103/Line/OEE/OEE Performance` |
| OEE Quality | `Enterprise/Dallas/Press/Press 103/Line/OEE/OEE Quality` |
| Current Shift | `Enterprise/Dallas/Press/Press 103/Dashboard/Shift Name` |
| Production Rate | `Enterprise/Dallas/Press/Press 103/Line/OEE/Production Rate` |
| Standard Rate | `Enterprise/Dallas/Press/Press 103/Line/OEE/Standard Rate` |
| Machine Speed | `Enterprise/Dallas/Press/Press 103/MQTT/Dashboard Machine Speed` |
| Run ID | `Enterprise/Dallas/Press/Press 103/Line/OEE/RunID` |

### MySQL / MES_Lite (Historical & Transactional)

**Host:** `proveit.virtualfactory.online:3306`  
**Schema:** `mes_lite`  
**Press 103 LineID:** `1`

#### Key Tables

| Table | Description | Key Columns |
|-------|-------------|-------------|
| `line` | Equipment definitions | `ID`, `Name`, `Disable` |
| `workorder` | Work order master | `ID`, `WorkOrder`, `Quantity`, `Closed`, `ProductCode` |
| `schedule` | Scheduled production | `ID`, `LineID`, `WorkOrderID`, `ScheduleStartDateTime`, `Quantity`, `RunID`, `Status` |
| `run` | Production runs | `ID`, `ScheduleID`, `RunStartDateTime`, `GoodCount`, `WasteCount`, `OEE`, `Availability`, `Performance`, `Quality` |
| `statehistory` | State/downtime events | `ID`, `StateReasonID`, `StartDateTime`, `EndDateTime`, `ReasonName`, `LineID`, `RunID`, `Active` |
| `statereason` | Downtime reason codes | `ID`, `ReasonName`, `ReasonCode`, `PlannedDowntime`, `RecordDowntime` |
| `notes` | Operator notes | `ID`, timestamps, content |

#### Useful Queries

**Current/Active Run for Press 103:**
```sql
SELECT r.*, s.WorkOrderID, w.WorkOrder, w.ProductCode
FROM mes_lite.run r
JOIN mes_lite.schedule s ON r.ScheduleID = s.ID
JOIN mes_lite.workorder w ON s.WorkOrderID = w.ID
WHERE s.LineID = 1 AND r.Closed IS NULL OR r.Closed = 0
ORDER BY r.RunStartDateTime DESC
LIMIT 1
```

**Active Downtime Events:**
```sql
SELECT sh.*, sr.ReasonName, sr.PlannedDowntime
FROM mes_lite.statehistory sh
LEFT JOIN mes_lite.statereason sr ON sh.StateReasonID = sr.ID
WHERE sh.LineID = 1 AND sh.Active = 1
ORDER BY sh.StartDateTime DESC
```

**Shift Production Summary (Last 24 Hours):**
```sql
SELECT 
    DATE(r.RunStartDateTime) as ProductionDate,
    SUM(r.GoodCount) as TotalGood,
    SUM(r.WasteCount) as TotalWaste,
    AVG(r.OEE) as AvgOEE
FROM mes_lite.run r
JOIN mes_lite.schedule s ON r.ScheduleID = s.ID
WHERE s.LineID = 1 
  AND r.RunStartDateTime >= DATE_SUB(NOW(), INTERVAL 24 HOUR)
GROUP BY DATE(r.RunStartDateTime)
```

**Downtime Pareto (Last 7 Days):**
```sql
SELECT 
    sh.ReasonName,
    COUNT(*) as Occurrences,
    SUM(TIMESTAMPDIFF(MINUTE, sh.StartDateTime, COALESCE(sh.EndDateTime, NOW()))) as TotalMinutes
FROM mes_lite.statehistory sh
WHERE sh.LineID = 1 
  AND sh.StartDateTime >= DATE_SUB(NOW(), INTERVAL 7 DAY)
  AND sh.ReasonCode != 0
GROUP BY sh.ReasonName
ORDER BY TotalMinutes DESC
LIMIT 10
```

---

## Tool Specifications

### Tool 1: `get_equipment_status`

**MES Objective:** Answer "What is Press 103 doing right now?"

**Data Sources:** UNS (primary)

**Returns:**
- Running state (boolean)
- Current state code and meaning
- Current speed vs. setpoint
- Current shift
- Active operator (if available)

**Implementation:**
```python
# Read from cached UNS topics:
# - Enterprise/Dallas/Press/Press 103/Dashboard/Running
# - Enterprise/Dallas/Press/Press 103/Line/State
# - Enterprise/Dallas/Press/Press 103/MQTT/Dashboard Machine Speed
# - Enterprise/Dallas/Press/Press 103/Line/Rate Setpoint
# - Enterprise/Dallas/Press/Press 103/Dashboard/Shift Name
```

---

### Tool 2: `get_active_work_order`

**MES Objective:** Answer "What are we making and what's the target?"

**Data Sources:** UNS + MySQL

**Returns:**
- Work order number
- Product code / description
- Target quantity
- Current good count
- Percent complete
- Estimated completion time

**Implementation:**
```python
# 1. Get current WorkOrder from UNS:
#    Enterprise/Dallas/Press/Press 103/Line/OEE/WorkOrder
#
# 2. Query MySQL for work order details:
#    SELECT w.*, s.Quantity as ScheduledQty, s.ScheduleStartDateTime
#    FROM mes_lite.workorder w
#    JOIN mes_lite.schedule s ON s.WorkOrderID = w.ID
#    WHERE w.WorkOrder = '{wo_number}' AND s.LineID = 1
#    ORDER BY s.ScheduleStartDateTime DESC LIMIT 1
#
# 3. Get current counts from UNS:
#    Enterprise/Dallas/Press/Press 103/Line/OEE/Good Count
#    Enterprise/Dallas/Press/Press 103/Line/OEE/Target Count
#    Enterprise/Dallas/Press/Press 103/Line/OEE/Estimate Finish Time
```

---

### Tool 3: `get_oee_summary`

**MES Objective:** Answer "How is Press 103 performing?"

**Data Sources:** UNS (real-time) + MySQL (historical context)

**Returns:**
- Current OEE (and A/P/Q components)
- Comparison to standard/target
- Current run statistics
- Historical context (shift/day average if time permits)

**Implementation:**
```python
# Real-time from UNS:
# - Enterprise/Dallas/Press/Press 103/Line/OEE/OEE
# - Enterprise/Dallas/Press/Press 103/Line/OEE/OEE Availability
# - Enterprise/Dallas/Press/Press 103/Line/OEE/OEE Performance
# - Enterprise/Dallas/Press/Press 103/Line/OEE/OEE Quality
# - Enterprise/Dallas/Press/Press 103/Line/OEE/Good Count
# - Enterprise/Dallas/Press/Press 103/Line/OEE/Bad Count
# - Enterprise/Dallas/Press/Press 103/Line/OEE/Runtime
# - Enterprise/Dallas/Press/Press 103/Line/OEE/Unplanned Downtime
#
# Historical context from MySQL (optional):
# SELECT AVG(OEE) as AvgOEE FROM mes_lite.run r
# JOIN mes_lite.schedule s ON r.ScheduleID = s.ID
# WHERE s.LineID = 1 AND r.RunStartDateTime >= DATE_SUB(NOW(), INTERVAL 7 DAY)
```

---

### Tool 4: `get_downtime_summary`

**MES Objective:** Answer "Why has Press 103 been down?" or "What's causing losses?"

**Data Sources:** MySQL (primary), UNS (current state)

**Parameters:**
- `hours_back` (int, default 24): How far back to look

**Returns:**
- Active downtime event (if any) with duration
- Top downtime reasons (Pareto)
- Total downtime minutes
- Planned vs. unplanned breakdown

**Implementation:**
```python
# Current state from UNS:
# - Enterprise/Dallas/Press/Press 103/Line/State (0 = running, other = down)
# - Enterprise/Dallas/Press/Press 103/Line/Dispatch/StateReason (current reason details)
#
# Historical from MySQL:
# SELECT sh.ReasonName, sr.PlannedDowntime,
#        COUNT(*) as Events,
#        SUM(TIMESTAMPDIFF(MINUTE, sh.StartDateTime, 
#            COALESCE(sh.EndDateTime, NOW()))) as Minutes
# FROM mes_lite.statehistory sh
# LEFT JOIN mes_lite.statereason sr ON sh.StateReasonID = sr.ID
# WHERE sh.LineID = 1 
#   AND sh.StartDateTime >= DATE_SUB(NOW(), INTERVAL {hours_back} HOUR)
# GROUP BY sh.ReasonName, sr.PlannedDowntime
# ORDER BY Minutes DESC
```

---

### Tool 5: `log_observation`

**MES Objective:** Allow agent to record an observation or note (illustrates write pattern)

**Data Sources:** UNS (write)

**Parameters:**
- `message` (str): The observation to log
- `category` (str, optional): Category tag (e.g., "quality", "maintenance", "safety")

**Returns:**
- Confirmation with timestamp
- Topic written to

**Implementation:**
```python
# Publish to a dedicated agent notes topic:
# Enterprise/Dallas/Press/Press 103/Agent/Observations
#
# Payload format (JSON):
# {
#   "timestamp": "2024-12-16T10:30:00Z",
#   "source": "mes-agent",
#   "category": "quality",
#   "message": "Operator reported color variation on last 3 rolls"
# }
#
# Use retain=False, qos=1
```

---

## Implementation Requirements

### File Structure
```
day1/mes_server/
├── README.md           # This file
├── requirements.txt    # Dependencies
└── src/
    └── mes_mcp_server.py   # Main server implementation
```

### Dependencies (requirements.txt)
```
mcp>=1.0.0
paho-mqtt>=2.0.0
mysql-connector-python>=8.0.0
python-dotenv>=1.0.0
```

### Environment Variables

Load from `../../.env` (root of MCP_A2A_Workshop):

```
MQTT_BROKER=balancer.virtualfactory.online
MQTT_PORT=1883
MQTT_USERNAME=<from .env>
MQTT_PASSWORD=<from .env>

MYSQL_HOST=proveit.virtualfactory.online
MYSQL_PORT=3306
MYSQL_USERNAME=<from .env>
MYSQL_PASSWORD=<from .env>
```

### Code Patterns to Follow

**Reference:** `day1/mqtt_server/src/mqtt_mcp_server.py`

1. **Logging:** Use `stderr` (stdout reserved for MCP protocol)
2. **Env loading:** `Path(__file__).parent.parent.parent.parent / ".env"`
3. **MQTT Client ID:** Include UUID suffix to prevent collisions
4. **MQTT Cache:** File-based JSON cache with thread-safe atomic writes
5. **MCP Server:** Use `Server`, `@server.list_tools()`, `@server.call_tool()` pattern
6. **Tool Returns:** Return `list[TextContent]`
7. **Subscription:** Subscribe only to `Enterprise/Dallas/Press/Press 103/#`

### Configuration Constants

```python
# Press 103 identification
PRESS_103_LINE_ID = 1
PRESS_103_UNS_BASE = "Enterprise/Dallas/Press/Press 103"

# MQTT subscription (scoped to Press 103 only)
MQTT_SUBSCRIBE_TOPIC = f"{PRESS_103_UNS_BASE}/#"

# Cache file
CACHE_FILE = Path(__file__).parent / "mes_cache.json"
```

### MySQL Connection Pattern

```python
import mysql.connector
from mysql.connector import pooling

# Create connection pool for efficiency
db_pool = pooling.MySQLConnectionPool(
    pool_name="mes_pool",
    pool_size=3,
    host=MYSQL_HOST,
    port=MYSQL_PORT,
    user=MYSQL_USERNAME,
    password=MYSQL_PASSWORD,
    database="mes_lite"
)

def execute_query(query: str, params: tuple = None) -> list[dict]:
    """Execute a read-only query and return results as list of dicts."""
    conn = db_pool.get_connection()
    try:
        cursor = conn.cursor(dictionary=True)
        cursor.execute(query, params)
        return cursor.fetchall()
    finally:
        cursor.close()
        conn.close()
```

---

## Claude Desktop Configuration

Add to `claude_desktop_config.json`:

```json
{
  "mcpServers": {
    "mes-press103": {
      "command": "python",
      "args": ["/path/to/MCP_A2A_Workshop/day1/mes_server/src/mes_mcp_server.py"],
      "env": {}
    }
  }
}
```

---

## Testing Scenarios

After implementation, verify with these prompts in Claude Desktop:

1. **Status Check:** "What is Press 103 doing right now?"
2. **Work Order:** "What work order is running on Press 103 and how close are we to completion?"
3. **Performance:** "How is Press 103 performing? Give me the OEE breakdown."
4. **Downtime:** "What have been the main causes of downtime on Press 103 in the last 24 hours?"
5. **Combined:** "Give me a full status report on Press 103 including current production, OEE, and any downtime issues."

---

## Notes for Cursor

1. **Start with the MQTT MCP server as your template** — the caching pattern, connection handling, and MCP server structure are already proven
2. **Add MySQL as a second data source** — use connection pooling for efficiency
3. **Tools should combine data sources** — the value is in the aggregation, not raw access
4. **Keep error handling robust** — tools should return useful error messages, not crash
5. **Log generously to stderr** — helps debugging without breaking MCP protocol
6. **Test each tool individually** before testing combinations
