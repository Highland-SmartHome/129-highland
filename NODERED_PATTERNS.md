# Node-RED Patterns & Conventions

## Overview

Design patterns and conventions for Node-RED flows in the Highland home automation system. These patterns prioritize readability, maintainability, and alignment with the event-driven architecture.

---

## Core Principles

1. **Visibility over abstraction** — Keep logic visible in flows; don't hide complexity in subflows unless truly reusable
2. **Horizontal scrolling is the enemy** — Use link nodes and groups to keep flows compact and readable
3. **Pub/sub for inter-flow communication** — Flows talk via MQTT events, not direct dependencies
4. **Centralized error handling** — Flow-wide catch with targeted overrides
5. **Configurable logging** — Per-flow log levels for flexible debugging

---

## Node-RED Environment Configuration

### Using Node.js Modules in Function Nodes

`require()` is not available directly in function nodes in Node-RED 3.x. Modules must be declared in the function node's **Setup tab** and are injected as named variables.

**To use `fs` and `path` (or any other module):**
1. Open the function node
2. Go to the **Setup** tab
3. Add entries under **Modules**:
   - Name: `fs` / Module: `fs`
   - Name: `path` / Module: `path`
4. In the function body, use `fs` and `path` directly — do **not** call `require()`

```javascript
// WRONG — will throw "require is not defined"
const fs = require('fs');

// CORRECT — declared in Setup tab, available as plain variable
const raw = fs.readFileSync(filepath, 'utf8');
```

This applies to any Node.js built-in or npm module used in function nodes.

### Context Storage (settings.js)

Node-RED context is configured with two named stores:

```javascript
contextStorage: {
    default: {
        module: "localfilesystem"
    },
    initializers: {
        module: "memory"
    }
}
```

**`default` (localfilesystem):** Persists to disk. Used for flow state, config cache, and any value that must survive a Node-RED restart. This is the store used when no store name is specified.

**`initializers` (memory):** In-memory only. Used exclusively for runtime utilities populated by `Utility: Initializers` at startup — functions, helpers, and other values that cannot be JSON-serialized and therefore cannot use `localfilesystem`. These are re-populated on every restart.

**Usage convention:**

```javascript
// Utility: Initializers — storing a helper function
global.set('utils.formatStatus', function(text) { ... }, 'initializers');

// Any function node — retrieving it
const formatStatus = global.get('utils.formatStatus', 'initializers');

// Default store — no store name needed
global.set('config', configObject);
const config = global.get('config');
```

The store name in `global.get` / `global.set` is what makes the naming self-documenting — seeing `'initializers'` as the third argument tells you exactly where the value was defined and where to look if it's missing.

### Home Assistant Integration

**Primary method:** `node-red-contrib-home-assistant-websocket`

Provides:
- HA entity state access
- Service calls (notifications, backups, etc.)
- Event subscription
- WebSocket connection to HA

**Configuration:**
- Base URL: `http://{ha_ip}:8123`
- Access Token: Long-lived access token from HA (stored in Node-RED credentials, not in config files)

**Use cases:**
| Action | Method |
|--------|--------|
| Trigger HA backup | Service call: `backup.create` |
| Send notification via Companion App | Service call: `notify.mobile_app_*` |
| Check HA entity state | HA API node or WebSocket |
| React to HA events | HA events node |

*Note: Most device control goes through MQTT directly (Z2M, Z-Wave JS). HA integration is primarily for HA-specific features (backups, notifications, entity state that only exists in HA).*

### Healthchecks.io Pinging

Node-RED pings Healthchecks.io **directly via outbound HTTP** from the Health Monitor flow. This is the correct model because:

- Node-RED can make outbound HTTP calls independently of MQTT state
- Using MQTT as an intermediary (e.g. a watchdog script listening for a heartbeat topic) conflates two failure modes — if MQTT goes down but Node-RED is up, the watchdog sees silence and falsely reports Node-RED as unhealthy
- Direct HTTP from Node-RED proves Node-RED liveness independent of any other service

Each service check that Node-RED is responsible for pings its corresponding Healthchecks.io URL on success. Ping URLs are stored in `config.secrets.healthchecks_io`.

**Node-RED's Healthchecks.io ping** (proving Node-RED itself is alive) is sent on a fixed interval from the Health Monitor flow — not via any external script.

> **Watchdog script:** The original watchdog design (subscribing to a Node-RED MQTT heartbeat) is superseded by direct HTTP pinging. Whether a watchdog script has a remaining role (e.g. monitoring something Node-RED genuinely cannot monitor itself) will be determined as each service check is designed. The watchdog script in the runbook Post-Build section should be considered a placeholder pending that analysis.

### Essential Services

Certain flows must not depend on infrastructure that might itself be the thing that failed. These are **essential services** — flows that other flows depend on for error reporting and that must remain functional even during partial system failure.

**Current essential services:**
- `Utility: Logging` — must not use the Initializer Latch or any utility from the `initializers` store. Inlines all required helpers (e.g. `formatStatus`) directly. This ensures logging remains available even if Initializers fails.

**Rule:** If a flow is responsible for reporting failures, it cannot itself depend on the thing that might fail. Dependencies on the `initializers` store or MQTT are acceptable for non-essential flows but must be avoided for essential services.

---

## Flow Organization

### Flow Types

| Type | Purpose | Examples |
|------|---------|----------|
| **Area flows** | Own devices and automations for a physical area | `Area: Garage`, `Area: Living Room`, `Area: Front Porch` |
| **Utility flows** | Cross-cutting concerns that transcend areas | `Utility: Scheduler`, `Utility: Security`, `Utility: Logging`, `Utility: Health Checks`, `Utility: Config Loader`, `Utility: Initializers` |

### Tab Naming Convention

Tabs use a prefix to indicate their type:
- `Utility: Config Loader`
- `Utility: Health Checks`
- `Utility: Logging`
- `Utility: Initializers`
- `Area: Garage`
- `Area: Living Room`

### Flow Style Preferences

- **Groups are the primary organizing unit** — every logical section of a flow lives in a named group
- **Flows should be as linear as possible within groups** — left to right, top to bottom
- **Excessive branching signals a need for additional groups** — if a group is getting complex, split it
- **No node should have more than two outputs** — more outputs change node height and break visual linearity
- **Link nodes connect groups** — never run wires across group boundaries
- **Node names describe function, not topic strings** — `Parse Configuration` not `highland/command/config/reload`

---

## Link Nodes & Groups

### Preferred Over Subflows For:
- Keeping logic visible within a flow
- Breaking up long horizontal chains
- Creating logical sections within a flow

### Pattern: Grouped Logic with Link Nodes

```
┌─────────────────────────────────────────────────────────────────┐
│ Group: Handle Motion Event                                      │
│                                                                 │
│  ┌─────────┐    ┌─────────┐    ┌─────────┐    ┌─────────────┐  │
│  │ Link In │───►│ Process │───►│ Decide  │───►│ Link Out    │  │
│  │ motion  │    │ payload │    │ action  │    │ to lights   │  │
│  └─────────┘    └─────────┘    └─────────┘    └─────────────┘  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│ Group: Control Lights                                           │
│                                                                 │
│  ┌─────────────┐    ┌─────────┐    ┌─────────┐                 │
│  │ Link In     │───►│ Set     │───►│ MQTT    │                 │
│  │ from motion │    │ payload │    │ Out     │                 │
│  └─────────────┘    └─────────┘    └─────────┘                 │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## Subflows

### Use Sparingly, For Truly Reusable Components

**Good candidates for subflows:**
- Latches — reusable startup gates (see below)
- Common transformations used identically across many flows

**Not good candidates:**
- Flow-specific logic (keep it visible)
- One-off utilities (just use a function node)
- Anything that hides important business logic

### Latch Pattern

Latches are subflows that gate message flow until some condition is met. The naming convention is `{Condition} Latch` — e.g. `Initializer Latch`, leaving room for future variants like `MQTT Latch` or `Network Latch`.

All latches share the same interface:
- **1 input** — any message from any source (inject, MQTT, HTTP, etc.)
- **Output 1 (OK)** — messages pass through once condition is met; buffered messages drain in order
- **Output 2 (TIMEOUT)** — single signal message when condition is never met within retry window

### Initializer Latch

Gates flow execution until `Utility: Initializers` has populated the `initializers` context store. Drop this into any flow that uses utilities from the `initializers` store.

**Do NOT use in essential services** (e.g. `Utility: Logging`) — flows that report failures must not depend on the thing that might fail.

**Environment variables (configurable per instance):**
- `RETRY_INTERVAL_MS` — delay between retries in milliseconds (default: 250)
- `MAX_RETRIES` — maximum retry attempts before timeout (default: 20)

Total timeout at defaults: 250ms × 20 = 5 seconds.

**Internal behavior:**
- Every incoming message is buffered immediately
- On first message, starts polling `global.get('initializers.ready', 'initializers')`
- If flag is `true` → sets `flow.initialized = true`, clears `flow.degraded`, drains buffer via Output 1
- If max retries exceeded → sets `flow.degraded = true`, discards buffer, emits signal via Output 2
- If already initialized → passes message through Output 1 directly (no buffering)
- If already degraded → drops message silently

**Calling flow responsibilities:**
- Wire Output 1 to normal processing logic — messages arrive as if nothing happened
- Wire Output 2 to error handler — log CRITICAL, set node status, etc.

---

## Flow Registration

### Purpose

Each area flow self-registers its identity and owned devices. This creates a queryable global registry that enables:
- Targeting messages by area
- Looking up devices by capability
- Knowing which area owns which device

### Flow Registry Structure

```json
{
  "foyer": {
    "devices": ["foyer_entry_door", "foyer_environment"]
  },
  "garage": {
    "devices": ["garage_entry_door", "garage_carriage_left", "garage_carriage_right", "garage_motion_sensor", "garage_environment"]
  }
}
```

**Register Flow function node:**

```javascript
const flowIdentity = {
  area: 'foyer',
  devices: ['foyer_entry_door', 'foyer_environment']
};

flow.set('identity', flowIdentity);

const registry = global.get('flowRegistry') || {};
registry[flowIdentity.area] = { devices: flowIdentity.devices };
global.set('flowRegistry', registry);

const formatStatus = global.get('utils.formatStatus', 'initializers');
node.status({ fill: 'green', shape: 'dot', text: formatStatus(`Registered: ${flowIdentity.devices.length} devices`) });

return msg;
```

### Capability Lookup at Runtime

```javascript
function getAreasByCapability(capability) {
  const flowRegistry = global.get('flowRegistry');
  const deviceRegistry = global.get('config').deviceRegistry;
  const result = {};
  for (const [area, areaData] of Object.entries(flowRegistry)) {
    const matching = areaData.devices.filter(id => {
      const d = deviceRegistry[id];
      return d && d.capabilities.includes(capability);
    });
    if (matching.length > 0) result[area] = matching;
  }
  return result;
}
```

---

## Startup Sequencing

### The Problem

On Node-RED startup or deploy, three things can conflict:

1. MQTT subscriptions deliver retained messages immediately
2. Config Loader and flow context restoration may still be in progress
3. `Utility: Initializers` may not have finished populating the `initializers` context store

Node-RED makes no startup ordering guarantees between inject nodes across flows.

### The Solution: Initializer Latch

The `Initializer Latch` subflow handles the full startup sequencing problem:

- Buffers all incoming messages until `initializers.ready` is true
- Polls the flag with configurable retry (default 250ms × 20 = 5 second timeout)
- On success: sets `flow.initialized = true`, drains buffer through Output 1
- On timeout: sets `flow.degraded = true`, discards buffer, signals Output 2

This correctly handles both orderings — whether the calling flow starts before or after Initializers completes — because:
- The `initializers` store is a memory backend that resets on every restart, making stale state structurally impossible
- Polling detects readiness regardless of which inject fired first

### Initializers Startup Sequence

`Utility: Initializers` follows this sequence on every startup:

```javascript
// Step 1: Populate all utility functions
global.set('utils.formatStatus', function(text) {
    const ts = new Date().toLocaleString('en-US', {
        month: 'short', day: 'numeric',
        hour: 'numeric', minute: '2-digit', hour12: true
    });
    return `${text} at: ${ts}`;
}, 'initializers');

// Step 2: Set the ready flag
global.set('initializers.ready', true, 'initializers');

// Step 3: Update node status
const formatStatus = global.get('utils.formatStatus', 'initializers');
node.status({ fill: 'green', shape: 'dot', text: formatStatus(`${global.keys('initializers').length} function(s)`) });
```

The `initializers` store is memory-backed — it resets on every restart. There is no stale state from previous sessions.

### Essential Services Exception

`Utility: Logging` and other essential services do **not** use the Initializer Latch. They inline all required helpers directly so they remain functional even if Initializers fails. See **Essential Services** section.

### Degraded State and Recovery

When the Initializer Latch times out, `flow.degraded = true` is set. The failure handler on Output 2 should:

```javascript
// Inline formatStatus — intentionally not using utils, initializers store is unavailable
const ts = new Date().toLocaleString('en-US', {
    month: 'short', day: 'numeric',
    hour: 'numeric', minute: '2-digit', hour12: true
});
node.status({ fill: 'red', shape: 'ring', text: `Degraded at: ${ts}` });

msg.topic = 'highland/event/log';
msg.payload = {
    timestamp: new Date().toISOString(),
    system: 'node_red',
    source: 'flow_name_here',
    level: 'CRITICAL',
    message: 'Initializer latch timed out — flow will not process messages',
    context: {}
};
return msg;  // wired to MQTT out → highland/event/log
```

**Recovery procedure:**

1. Fix the issue in `Utility: Initializers`
2. Deploy `Utility: Initializers`
3. Redeploy the affected flow(s)

Step 3 is required because `flow.degraded` persists in context storage across restarts. Redeploy resets flow context and re-runs the startup inject.

### Bootstrapping Limitation

There is an inherent bootstrapping limitation in any event-driven system: **you cannot use infrastructure to report infrastructure failures.**

If both Initializers and MQTT are simultaneously unavailable, Node-RED has no self-reporting mechanism. This is a physical reality, not a design flaw.

**Accepted fallbacks when MQTT is unavailable:**
- **Node-RED debug sidebar** — node status (red ring) is visible in the editor regardless of MQTT state
- **Node-RED console log** — `node.error()` writes to Node-RED's own log, visible via `docker compose logs nodered`
- **Healthchecks.io** — the Health Monitor pings via direct HTTP, independently of MQTT

The correct mitigation is to **monitor MQTT health** so you know when this condition exists. Once MQTT health monitoring is in place, a simultaneous MQTT and Initializers failure becomes a known, observable state.

### Probe Topic Convention

```
highland/command/nodered/init_probe/{flow_name}
```

Used for the echo probe pattern (verifying retained messages have been processed). Each flow uses its own probe topic to avoid cross-flow interference.

---

## Error Handling

### Two-Tier Approach

1. **Targeted handlers** — Catch errors in specific groups where you need custom handling
2. **Flow-wide catch-all** — Single Error node per flow catches anything unhandled, dispatches to `highland/event/log`

---

## Logging Framework

### Concept

Logging answers: *"How important is this for troubleshooting/audit?"* Separate from notifications, though CRITICAL logs auto-forward to `highland/event/notify`.

### Log Storage

**Format:** JSONL (JSON Lines) — one JSON object per line
**Location:** `/var/log/highland/highland-YYYY-MM-DD.jsonl`
**Rotation:** Daily, retain 30 days

### Log Entry Structure

| Field | Purpose | Examples |
|-------|---------|----------|
| `timestamp` | When it happened | `2025-02-24T10:00:00Z` |
| `system` | Which system | `node_red`, `ha`, `z2m`, `zwave_js` |
| `source` | Component within system | `garage`, `config_loader` |
| `level` | Severity | `VERBOSE`, `DEBUG`, `INFO`, `WARN`, `ERROR`, `CRITICAL` |
| `message` | Human-readable description | |
| `context` | Structured additional data | |

### Log Levels

| Level | When to Use |
|-------|-------------|
| `VERBOSE` | Granular trace; active debugging only |
| `DEBUG` | Detailed troubleshooting info |
| `INFO` | Normal operational events |
| `WARN` | Unexpected but not broken |
| `ERROR` | Something failed but flow continues |
| `CRITICAL` | Catastrophic failure; intervention needed |

### Node Status Color Coding

Log-emitting function nodes use color-coded status:

| Level | Fill | Shape |
|-------|------|-------|
| VERBOSE / DEBUG | grey | dot |
| INFO | green | dot |
| WARN | yellow | dot |
| ERROR | red | dot |
| CRITICAL | red | ring |

Status text uses `utils.formatStatus(level)` for consistent `"{LEVEL} at: {TIME}"` formatting. **Essential services** (e.g. `Utility: Logging`) inline `formatStatus` rather than reading from the `initializers` store.

### Logging Utility Flow (`Utility: Logging`)

**Essential service — no Initializer Latch.** Subscribes to `highland/event/log` (QoS 2), normalizes malformed entries, appends to daily JSONL file, forwards CRITICAL to `highland/event/notify`.

**Groups:** Sinks, Log Writer, Critical Forwarder, Error Handling, Test Cases

### Querying Logs

```bash
jq 'select(.level == "ERROR")' highland-2025-02-24.jsonl
jq 'select(.system == "node_red")' highland-2025-02-24.jsonl
tail -10 highland-2025-02-24.jsonl | jq '.'
```

### Auto-Notify Behavior

**CRITICAL only** auto-notifies. Escalation beyond that is the responsibility of the flow that owns the logic.

---

## Device Registry

### Structure

```json
{
  "garage_carriage_left": {
    "friendly_name": "Garage Carriage Light (Left)",
    "protocol": "zigbee",
    "topic": "zigbee2mqtt/garage_carriage_left",
    "area": "garage",
    "capabilities": ["on_off", "brightness"],
    "battery": null
  },
  "foyer_entry_door": {
    "friendly_name": "Front Door Lock",
    "protocol": "zwave",
    "topic": "zwave/foyer_entry_door",
    "area": "foyer",
    "capabilities": ["lock", "battery"],
    "battery": { "type": "AA", "quantity": 4 }
  }
}
```

**File:** `/home/nodered/config/device_registry.json`
**Access:** `global.get('config').deviceRegistry`

---

## Configuration Management

### File Structure

```
/home/nodered/config/
├── device_registry.json        ← git: yes
├── flow_registry.json          ← git: yes
├── notifications.json          ← git: yes
├── thresholds.json             ← git: yes
├── healthchecks.json           ← git: yes
├── secrets.json                ← git: NO (.gitignore)
└── README.md                   ← git: yes
```

### secrets.json Structure

```json
{
  "home_assistant": {
    "token": "long-lived-access-token"
  },
  "mqtt": {
    "username": "svc_nodered",
    "password": "..."
  },
  "healthchecks_io": {
    "node_red": "https://hc-ping.com/uuid",
    "home_assistant": "https://hc-ping.com/uuid",
    "mqtt": "https://hc-ping.com/uuid",
    "z2m": "https://hc-ping.com/uuid",
    "zwave": "https://hc-ping.com/uuid"
  },
  "smtp": { "host": "...", "port": 587, "user": "...", "password": "..." },
  "weather_api_key": "...",
  "google_calendar_api_key": "..."
}
```

### Config Loader (`Utility: Config Loader`) ✅

Loads all config files into `global.config` at startup. Uses Initializer Latch to ensure utilities are available before processing.

**Triggers:** startup inject (0.5s delay), manual inject, `highland/command/config/reload`, `highland/command/config/reload/{name}`
**Publishes:** retained status to `highland/status/config/loaded`

---

## Health Monitoring

### Philosophy

Treat this as a line-of-business application. Degradation detection is as important as outage detection.

### Single Point of Failure Problem

Node-RED is currently the sole reporter for all service health checks. If Node-RED goes down, all service checks stop pinging Healthchecks.io simultaneously — making it impossible to distinguish "Node-RED is down" from "everything is down".

**Solution: each service self-reports its own liveness independently of Node-RED.** Node-RED's integration checks prove the *connection* is healthy; each service's self-report proves the *service* is healthy. Two distinct signals, two distinct failure modes.

**Current implementation status:**
- `highland-node-red` — Node-RED self-reports via direct HTTP ping ✅
- `highland-home-assistant` — Node-RED integration check (HTTP to HA API) ✅; HA self-report (native HA automation) ⏳ pending
- `highland-mqtt`, `highland-z2m`, `highland-zwave` — pending

### Status Values

| Status | Meaning |
|--------|---------|
| `healthy` | Responding AND all metrics within acceptable ranges |
| `degraded` | Responding BUT one or more metrics in warning territory |
| `unhealthy` | Not responding OR critical threshold exceeded |

### Check Frequency

| Service | Frequency | Grace Period |
|---------|-----------|-------------|
| Node-RED | 30 sec | 3 min |
| Home Assistant | 30 sec | 3 min |
| MQTT | 1 min | 3 min |
| Z2M | 1 min | 3 min |
| Z-Wave JS | 1 min | 3 min |

> Healthchecks.io only supports whole-minute grace periods. 3 minutes is the practical minimum.

### Watchdog Script

The original watchdog design (cron script subscribing to Node-RED's MQTT heartbeat) has been superseded by direct HTTP pinging from Node-RED. Whether a watchdog script has a remaining role will be evaluated per-service. TBD.

---

## ACK Tracker Utility Flow

### Purpose

Centralized tracking of acknowledgment requests. Keeps ACK bookkeeping out of individual flows.

### Topics

| Topic | Purpose |
|-------|---------|
| `highland/ack/register` | Register expectation |
| `highland/ack` | ACK responses |
| `highland/ack/result` | Outcome after timeout |

### Result Payload

```json
{
  "correlation_id": "abc123",
  "expected": 2,
  "received": 1,
  "sources": ["foyer_entry_door"],
  "missing": ["garage_entry_door"],
  "success": false
}
```

---

## Battery Monitor Utility Flow

| State | Threshold | Notification |
|-------|-----------|--------------|
| `normal` | > 35% | None |
| `low` | 35–15% | Normal priority, once |
| `critical` | < 15% | High priority, repeats 24h |

---

## Notification Framework

### Topic

```
highland/event/notify
```

### Severity Levels

| Severity | DND Override |
|----------|--------------|
| `low` | No |
| `medium` | No |
| `high` | Yes |
| `critical` | Yes |

### Android Notification Channels

| Channel | DND Override |
|---------|--------------|
| `highland_low` | No |
| `highland_default` | No |
| `highland_high` | Yes |
| `highland_critical` | Yes |

---

## Daily Digest

**Trigger:** Midnight + 5 second delay
**Content:** Calendar (next 24–48h), weather, battery status, system health
**Implementation:** Markdown → HTML → SMTP email

---

## Open Questions

- [x] ~~Pub/sub subflow implementation details~~ → **Flow Registration pattern**
- [x] ~~Logging persistence destination~~ → **JSONL files, daily rotation**
- [x] ~~Mobile notification channel selection~~ → **HA Companion App (Android)**
- [x] ~~Should ERROR-level logs also auto-notify?~~ → **CRITICAL only**
- [x] ~~Device Registry storage~~ → **External JSON, global.config.deviceRegistry**
- [x] ~~ACK pattern design~~ → **Centralized ACK Tracker**
- [x] ~~Health monitoring approach~~ → **Each service self-reports + Node-RED integration checks**
- [x] ~~Startup sequencing / race conditions~~ → **Initializer Latch subflow**
- [x] ~~Global utility functions~~ → **`initializers` memory context store**
- [ ] **HA self-reporting** — native HA automation pinging Healthchecks.io independently of Node-RED
- [ ] **MQTT latch** — gate for flows that depend on MQTT being available
- [ ] **MQTT_TOPICS.md reconciliation** — add attention state/event topics, video pipeline, calendar suppression

---

*Last Updated: 2026-03-18*
