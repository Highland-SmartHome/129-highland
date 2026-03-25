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
    },
    volatile: {
        module: "memory"
    }
}
```

**`default` (localfilesystem):** Persists to disk. Used for flow state, config cache, and any value that must survive a Node-RED restart. This is the store used when no store name is specified.

**`initializers` (memory):** In-memory only. Used exclusively for runtime utilities populated by `Utility: Initializers` at startup — functions, helpers, and other values that cannot be JSON-serialized and therefore cannot use `localfilesystem`. These are re-populated on every restart.

**`volatile` (memory):** In-memory only. Used for transient, non-serializable runtime values that must not be persisted to disk — timer handles, open connection references, or anything that would cause a circular reference error if Node-RED attempted to serialize it. Values here are intentionally lost on restart. Seeing `'volatile'` as the third argument signals that the value is transient by design.

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

---

## External HTTP Calls

Any flow that makes outbound HTTP requests to external APIs must identify itself via a `User-Agent` header. This is standard practice and explicitly required by some APIs (notably NWS, which uses the User-Agent to contact clients that are misbehaving).

### Standard User-Agent

The User-Agent string is stored in `system.json` and accessed via `global.config.system`:

```javascript
const userAgent = global.get('config')?.system?.http?.user_agent;
msg.headers = {
    'Accept': 'application/json',
    'User-Agent': userAgent
};
```

Do not hardcode the User-Agent string in individual flows. All external HTTP calls read it from config.

### NWS Format

The NWS API specifically requests User-Agent in the format `(app_name, contact_email)`. The value in `system.json` follows this convention and is suitable for all APIs:

```json
{
  "http": {
    "user_agent": "(Highland-SmartHome, highland@ferris.network)"
  }
}
```

### Applies To

All external HTTP request nodes — NWS forecast, NWS alerts, Google Calendar, Pirate Weather, Noonlight, and any future external API. No exceptions.

---

## Flow Organization

### Flow Types

| Type | Purpose | Examples |
|------|---------|----------|
| **Area flows** | Own devices and automations for a physical area | `Garage`, `Living Room`, `Front Porch` |
| **Utility flows** | Cross-cutting concerns that transcend areas | `Scheduler`, `Security`, `Notifications`, `Logging`, `Backup`, `ACK Tracker`, `Battery Monitor`, `Health Monitor`, `Config Loader`, `Calendaring`, `Weather Forecasts`, `Weather Alerts`, `Daily Digest` |

### Naming Convention

Flows are named by their area or utility function:
- `Garage`
- `Living Room`
- `Scheduler`
- `Notifications`

*No prefixes or suffixes needed — the flow list in Node-RED is the organizing structure.*

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

**Benefits:**
- Each group is a logical unit with a clear purpose
- Link nodes connect groups without spaghetti wires
- Flow reads top-to-bottom or left-to-right in sections
- Minimizes horizontal scrolling

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

Latches are subflows that gate message flow until some condition is met. The naming convention is `{Condition} Latch` — e.g. `Initializer Latch`, leaving room for future variants like `Network Latch` or `Availability Latch`.

All latches share the same interface:
- **1 input** — any message from any source (inject, MQTT, HTTP, etc.)
- **Output 1 (OK)** — messages pass through once condition is met; buffered messages drain in order
- **Output 2 (TIMEOUT)** — single signal message when condition is never met within retry window

### Initializer Latch

Gates flow execution until `Utility: Initializers` has populated the `initializers` context store. Drop this into any flow's startup sequencing group.

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
- No gate check function node needed in Sinks — the latch handles everything

---

## Flow Registration

### Purpose

Each area flow self-registers its identity and owned devices. This creates a queryable global registry that enables:
- Targeting messages by area
- Looking up devices by capability
- Knowing which area owns which device

### Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                          On Startup / Deploy                        │
│                                                                     │
│  ┌─────────────────┐                                                │
│  │   Foyer Flow    │──► global.flowRegistry['foyer'] = {...}        │
│  └─────────────────┘                                                │
│                                                                     │
│  ┌─────────────────┐                                                │
│  │   Garage Flow   │──► global.flowRegistry['garage'] = {...}       │
│  └─────────────────┘                                                │
│                                                                     │
│  ┌─────────────────┐                                                │
│  │  Living Room    │──► global.flowRegistry['living_room'] = {...}  │
│  └─────────────────┘                                                │
│                                                                     │
│  Each flow overwrites its own key. No global purge, no timing issues│
└─────────────────────────────────────────────────────────────────────┘
```

### Storage

| Storage | Persistence | Purpose |
|---------|-------------|---------|
| `flow.identity` | Disk | This flow's identity and devices |
| `global.flowRegistry` | Disk | All flows' registrations |
| `global.config.deviceRegistry` | Disk | Device details (single source of truth for capabilities) |

**Note:** Node-RED context storage is configured for disk persistence. Survives restarts.

### Flow Registry Structure

```json
{
  "foyer": {
    "devices": ["foyer_entry_door", "foyer_environment"]
  },
  "garage": {
    "devices": ["garage_entry_door", "garage_carriage_left", "garage_carriage_right", "garage_motion_sensor", "garage_environment"]
  },
  "living_room": {
    "devices": ["living_room_overhead", "living_room_environment"]
  }
}
```

### Registration Boilerplate

Every area flow includes this pattern:

```
┌─────────────────────────────────────────────────────────────────────┐
│  Flow: Foyer                                                        │
│                                                                     │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │  Group: Flow Registration                                   │   │
│  │                                                             │   │
│  │  ┌──────────────────────┐    ┌─────────────────────────┐   │   │
│  │  │ Inject               │───►│ Register Flow           │   │   │
│  │  │ • On startup         │    │                         │   │   │
│  │  │ • On deploy          │    │ • Set flow.identity     │   │   │
│  │  │ • Manual trigger     │    │ • Update flowRegistry   │   │   │
│  │  └──────────────────────┘    └─────────────────────────┘   │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
│  ... rest of flow logic ...                                         │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

**Register Flow function node:**

```javascript
const flowIdentity = {
  area: 'foyer',
  devices: ['foyer_entry_door', 'foyer_environment']
};

// Set flow-level identity
flow.set('identity', flowIdentity);

// Update global registry (overwrite this flow's section only)
const registry = global.get('flowRegistry') || {};
registry[flowIdentity.area] = {
  devices: flowIdentity.devices
};
global.set('flowRegistry', registry);

node.status({ fill: 'green', shape: 'dot', text: `Registered: ${flowIdentity.devices.length} devices` });

return msg;
```

### Capability Lookup at Runtime

Device capabilities are NOT stored in the flow registry. They live in the Device Registry (single source of truth). Query at runtime:

```javascript
// "Find all areas with locks"
function getAreasByCapability(capability) {
  const flowRegistry = global.get('flowRegistry');
  const deviceRegistry = global.get('config.deviceRegistry');
  const result = {};
  
  for (const [area, areaData] of Object.entries(flowRegistry)) {
    const matchingDevices = areaData.devices.filter(deviceId => {
      const device = deviceRegistry[deviceId];
      return device && device.capabilities.includes(capability);
    });
    
    if (matchingDevices.length > 0) {
      result[area] = matchingDevices;
    }
  }
  
  return result;
}

// Usage:
getAreasByCapability('lock');
// → { "foyer": ["foyer_entry_door"], "garage": ["garage_entry_door"] }
```

### Message Targeting Pattern

**Security flow wants to lock all locks:**

```javascript
// 1. Find all areas with lock capability
const locksByArea = getAreasByCapability('lock');
// → { "foyer": ["foyer_entry_door"], "garage": ["garage_entry_door"] }

// 2. Target areas in message
const targetAreas = Object.keys(locksByArea); // ["foyer", "garage"]

// 3. Register expected ACKs at device level
const expectedAcks = Object.values(locksByArea).flat(); 
// → ["foyer_entry_door", "garage_entry_door"]

// 4. Publish lockdown with area targets
msg.payload = {
  message_id: 'lock_123',
  source: 'security',
  recipients: targetAreas,
  request_ack: true
};
// Publish to: highland/event/security/lockdown

// 5. Register with ACK Tracker
msg.ackRegistration = {
  correlation_id: 'lock_123',
  expected_sources: expectedAcks,
  timeout_seconds: 30
};
// Publish to: highland/ack/register
```

**Area flow receives and responds:**

```javascript
// Foyer flow receives lockdown message
// recipients: ["foyer", "garage"]
// flow.identity.area: "foyer" → matches, process message

// Command the lock
// ... (via Command Dispatcher)

// Send ACK at device level
msg.payload = {
  ack_correlation_id: 'lock_123',
  source: 'foyer_entry_door',  // Device, not area
  timestamp: new Date().toISOString()
};
// Publish to: highland/ack
```

### Staleness Handling

**On device removal:**
1. Update the flow's registration boilerplate to remove device
2. Deploy flow
3. Flow overwrites its registry entry, device is gone

**On flow removal:**
1. Flow's registry entry persists (stale)
2. Acceptable: stale entry causes no harm (messages to deleted flow just aren't received)
3. Optional: Manual cleanup or periodic audit

*Details TBD during implementation.*

---

## Startup Sequencing

### The Problem

On Node-RED startup or deploy, three things can conflict:

1. MQTT subscriptions are established and the broker **immediately** delivers all retained messages for those topics
2. The flow's own initialization (Config Loader, flow registration, context restoration from disk) is still in progress
3. `Utility: Initializers` may not have finished populating the `initializers` context store yet

A retained message can arrive and trigger handler logic before `global.config` is loaded, before flow context is restored, and before utility functions like `utils.formatStatus` are available. There is no guaranteed ordering between inject nodes across flows — Node-RED makes no startup ordering guarantees.

### The Two-Condition Gate

The correct solution combines the echo probe pattern with an Initializers readiness check. A flow's gate opens only when **both** conditions are true:

1. **Echo probe returned** — guarantees all retained MQTT messages for this session have been processed
2. **Initializers ready** — guarantees utility functions are available in the `initializers` context store

```
On startup:
  1. Set flow.initialized = false  (gate closed)
  2. Subscribe to all topics (including retained state topics)
  3. Subscribe to highland/status/initializers/ready (non-retained)
  4. Publish probe to highland/command/nodered/init_probe/{flow_name} (non-retained)
  5. Retained messages begin arriving → buffer or discard (gate is closed)
  6. Own probe returns → retained messages are done
     → Check global.get('initializers.ready', 'initializers')
     → If true: both conditions met → open gate immediately
     → If undefined: wait for highland/status/initializers/ready message
  7. highland/status/initializers/ready arrives (non-retained)
     → If probe already returned: both conditions met → open gate
  8. Gate opens → process buffered state → normal operation begins
```

### Why Non-Retained for the Ready Signal

Using a retained message for `highland/status/initializers/ready` introduces a stale session problem — a flow restarting at 09:00:02a would see the retained message from the previous session's 08:00:05a initialization and incorrectly open its gate before the current session's utilities are ready.

Using a **non-retained** message combined with a **global flag** in the `initializers` store solves this cleanly:

- If Initializers runs first → sets `global('initializers.ready', true, 'initializers')` → dependent flows check the flag when their probe returns and open immediately
- If a dependent flow starts first → probe returns, flag is `undefined` → flow waits for the non-retained ready message
- On restart → the `initializers` store clears (memory backend resets) → flag is gone → stale ready state is structurally impossible

### Initializers Startup Sequence

```javascript
// Step 1: Mark not ready
global.set('initializers.ready', false, 'initializers');

// Step 2: Populate all utility functions
global.set('utils.formatStatus', function(text) { ... }, 'initializers');
// ... other utilities ...

// Step 3: Mark ready and signal dependent flows (non-retained)
global.set('initializers.ready', true, 'initializers');
msg.topic   = 'highland/status/initializers/ready';
msg.payload = { timestamp: new Date().toISOString() };
return msg;
// → publish non-retained to highland/status/initializers/ready
```

### Gate Pattern in Sinks Groups

The gate check belongs in the Sinks group — at the point of ingress — before messages reach any processing logic:

```javascript
if (!flow.get('initialized')) {
    const buffer = flow.get('state_buffer') || [];
    buffer.push(msg);
    flow.set('state_buffer', buffer);
    return null;
}
return msg;
```

### State vs Event Handling During Init

**Retained state messages** (from `highland/state/#`) — buffer these. They represent current truth and will be needed once the gate opens.

**Point-in-time events** (from `highland/event/#`) — discard these. If a real-time event fires during the brief init window it is genuinely gone and cannot be recovered. This is acceptable — the window is very short and events are by definition momentary.

### Processing the Buffer

Once the gate opens, process buffered state in order:

```javascript
const buffer = flow.get('state_buffer') || [];
flow.set('state_buffer', []);
for (const bufferedMsg of buffer) {
    node.send(bufferedMsg);
}
```

### Reacting to State vs Reacting to Events

**Two entry points, one handler:**

```
highland/state/scheduler/period  ──┐  (retained — arrives during init, buffered,
  (startup recovery path)          │   processed after gate opens)
                                   ├──► mutate flow.current_period ──► period logic
highland/event/scheduler/evening ──┘  (real-time — arrives during normal operation,
  (real-time transition path)         gate already open)
```

### Probe Topic Convention

```
highland/command/nodered/init_probe/{flow_name}
```

Each flow uses its own probe topic to avoid cross-flow interference.

### Notes

- The init window is typically well under one second. The gate is a safety net, not a performance concern.
- This pattern applies to every flow that subscribes to retained state topics OR uses utilities from the `initializers` store.
- Config Loader and Initializers do not use the two-condition gate — they are the things being waited for, not the things waiting.
- If the MQTT broker is unavailable on startup, the probe never returns. Flows should have a startup timeout (e.g., 10 seconds) after which they log an error and enter a degraded state.

### Bootstrapping Limitation

There is an inherent bootstrapping limitation in any event-driven system: **you cannot use infrastructure to report infrastructure failures.**

If both Initializers and MQTT are simultaneously unavailable, Node-RED has no self-reporting mechanism. This is not a design flaw — it is a physical reality. You cannot publish to a broker that isn't there.

**Accepted fallbacks when MQTT is unavailable:**
- **Node-RED debug sidebar** — node status (red ring, "Degraded") is visible in the editor regardless of MQTT state
- **Node-RED console log** — `node.error()` and `node.warn()` write to Node-RED's own log, visible via `docker compose logs nodered`
- **Healthchecks.io** — the Health Monitor pings Healthchecks.io via direct HTTP, independently of MQTT. If Node-RED is alive but MQTT is down, Healthchecks.io still receives pings and you know Node-RED itself is running

The correct mitigation is not to engineer around this limitation but to **monitor MQTT health** so you know when this condition exists. Once MQTT health monitoring is in place, a simultaneous MQTT outage and Initializers failure becomes a known, observable state rather than a silent one.

### Degraded State and Recovery

When the `Initializer Latch` subflow times out, the calling flow sets `flow.set('degraded', true)`. The gate check in Sinks handles three states:

```javascript
// Check degraded first — permanent failure, drop message
if (flow.get('degraded')) {
    node.warn('Flow is degraded — dropping message');
    return null;
}

// Check initialized — temporary, waiting
if (!flow.get('initialized')) {
    const buffer = flow.get('state_buffer') || [];
    buffer.push(msg);
    flow.set('state_buffer', buffer);
    return null;
}

// Gate open — proceed
return msg;
```

Degraded is checked before initialized because a degraded flow also has `initialized = false` — without this ordering, messages would buffer forever with no way out.

When entering the degraded state, the failure handler should also clear the buffer since those messages will never be processed:

```javascript
flow.set('degraded', true);
flow.set('state_buffer', []);  // No point buffering — gate will never open
node.status({ fill: 'red', shape: 'ring', text: 'Degraded: init timeout' });
// Publish CRITICAL log entry to highland/event/log
```

**Recovery procedure:**

The root cause of a degraded flow is always in `Utility: Initializers` — a bug preventing one or more utilities from being registered, or preventing `initializers.ready` from being set to `true`. The degraded state in dependent flows is a symptom, not the cause.

1. Identify and fix the issue in `Utility: Initializers`
2. Deploy `Utility: Initializers`
3. Redeploy the affected flow(s)

Step 3 is required because `flow.get('degraded')` persists in context storage across restarts. Redeploying resets flow context and re-runs the startup inject, giving the two-condition gate a fresh start against the now-healthy Initializers.

### Two-Tier Approach

1. **Targeted handlers** — Catch errors in specific groups where you need custom handling
2. **Flow-wide catch-all** — Single Error node per flow catches anything unhandled

```
┌─────────────────────────────────────────────────────────────────┐
│ Flow: Garage                                                    │
│                                                                 │
│  ┌─────────────────────────────────────┐                       │
│  │ Group: Critical Operation           │                       │
│  │                                     │                       │
│  │  [nodes] ───► [targeted error] ─────┼──► (custom handling)  │
│  │                                     │                       │
│  └─────────────────────────────────────┘                       │
│                                                                 │
│  ┌─────────────────────────────────────┐                       │
│  │ Group: Normal Operation             │                       │
│  │                                     │                       │
│  │  [nodes] ──────────────────────────►│ (errors bubble up)    │
│  │                                     │                       │
│  └─────────────────────────────────────┘                       │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │ Flow-wide Error Node                                    │   │
│  │ Catches all unhandled errors → dispatches to logging    │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## Logging Framework

### Concept

Logging answers: *"How important is this for troubleshooting/audit?"*

Logging is separate from notifications. They intersect (a CRITICAL log may auto-generate a notification), but serve different purposes.

### Log Storage

**Format:** JSONL (JSON Lines) — one JSON object per line

**Location:** `/var/log/highland/` (or equivalent on Node-RED host)

**Rotation:** Daily files

```
/var/log/highland/
├── highland-2025-02-22.jsonl
├── highland-2025-02-23.jsonl
└── highland-2025-02-24.jsonl  (current)
```

**Retention:** Keep N days, delete older (scheduled cleanup task)

### Unified Log

A single daily log file contains entries from ALL systems — Node-RED, Z2M, Z-Wave JS, HA, watchdog, etc. This provides a unified view similar to Windows Event Viewer.

**Log entry structure:**

| Field | Purpose | Examples |
|-------|---------|----------|
| `timestamp` | When it happened | `2025-02-24T10:00:00Z` |
| `system` | Which system generated the log | `node_red`, `ha`, `z2m`, `zwave_js`, `watchdog` |
| `source` | Component within that system | `garage`, `scheduler`, `coordinator` |
| `level` | Severity | `VERBOSE`, `DEBUG`, `INFO`, `WARN`, `ERROR`, `CRITICAL` |
| `message` | Human-readable description | `Failed to turn on carriage lights` |
| `context` | Structured additional data | `{"device": "...", "error": "..."}` |

**Example entries:**

```json
{"timestamp":"2025-02-24T10:00:00Z","system":"node_red","source":"garage","level":"ERROR","message":"Failed to turn on carriage lights","context":{"device":"light.garage_carriage","error":"MQTT timeout"}}
{"timestamp":"2025-02-24T10:00:05Z","system":"z2m","source":"coordinator","level":"WARN","message":"Device interview failed","context":{"device":"garage_motion_sensor"}}
{"timestamp":"2025-02-24T10:00:10Z","system":"ha","source":"recorder","level":"INFO","message":"Database purge completed","context":{"rows_deleted":15000}}
{"timestamp":"2025-02-24T10:00:15Z","system":"watchdog","source":"node_red_monitor","level":"INFO","message":"Heartbeat received","context":{}}
```

### Log Levels

| Level | Value | When to Use |
|-------|-------|-------------|
| `VERBOSE` | 0 | Granular trace; active debugging only |
| `DEBUG` | 1 | Detailed info useful for troubleshooting |
| `INFO` | 2 | Normal operational events worth recording |
| `WARN` | 3 | Something unexpected but not broken |
| `ERROR` | 4 | Something failed but flow continues |
| `CRITICAL` | 5 | Catastrophic failure; intervention needed |

### Per-Flow Log Level Threshold

Each flow has a configured minimum log level (stored in flow context):

```javascript
// Flow context
flow.set('logLevel', 'WARN');  // This flow only emits WARN and above
```

When a flow emits a log message:
- If message level >= flow threshold → emit to logging utility
- If message level < flow threshold → suppress

**Use case:** Set a flow to `DEBUG` while developing, `WARN` in steady state.

### Log Event Topic

Single topic — all systems publish here:

```
highland/event/log
```

### Log Event Payload (MQTT)

```json
{
  "timestamp": "2025-02-24T14:30:00Z",
  "system": "node_red",
  "source": "garage",
  "level": "ERROR",
  "message": "Failed to turn on carriage lights",
  "context": {
    "device": "light.garage_carriage",
    "error": "MQTT timeout"
  }
}
```

### How Systems Log

| System | Mechanism |
|--------|-----------|
| **Node-RED** | Flows publish to `highland/event/log`; Logging utility writes to file |
| **Z2M / Z-Wave JS** | Publish to `highland/event/log` (if configurable), or sidecar script |
| **Home Assistant** | Publish to `highland/event/log` via automation, or sidecar script |
| **Watchdog** | Publish to `highland/event/log` |

Node-RED's Logging utility flow subscribes to `highland/event/log` and writes ALL entries to the unified JSONL file, regardless of `system`.

### Logging Utility Flow

Centralized flow that:
1. Subscribes to `highland/event/log`
2. Appends to today's JSONL file
3. If level = `CRITICAL` → auto-dispatch to Notification Utility

```
highland/event/log ──► Logging Utility ──► Append to JSONL
                              │
                              │ (if CRITICAL)
                              ▼
                       highland/event/notify
```

### Querying Logs

JSONL + `jq` provides powerful ad-hoc querying:

```bash
# All errors from any system
jq 'select(.level == "ERROR")' highland-2025-02-24.jsonl

# All Node-RED entries
jq 'select(.system == "node_red")' highland-2025-02-24.jsonl

# All entries from garage (regardless of system)
jq 'select(.source == "garage")' highland-2025-02-24.jsonl

# Z2M warnings and above
jq 'select(.system == "z2m" and (.level == "WARN" or .level == "ERROR" or .level == "CRITICAL"))' highland-2025-02-24.jsonl

# Last 10 entries
tail -10 highland-2025-02-24.jsonl | jq '.'
```

### Future: Log Shipping (Deferred)

When NAS is available or if cloud aggregation is desired:
- Ship JSONL files to central location
- JSONL is compatible with most aggregators (Loki, Elastic, Datadog)
- Could also stream via MQTT to external subscriber

*Details TBD when infrastructure supports it.*

### Auto-Notify Behavior

**Only CRITICAL logs auto-notify.** ERROR and below do not.

| Level | Auto-Notify | Rationale |
|-------|-------------|-----------|
| CRITICAL | Yes | System health, potential data loss, immediate intervention |
| ERROR | No | Something failed but system continues; log and move on |
| WARN and below | No | Informational |

**CRITICAL examples:**
- Database size threshold exceeded
- Disk usage critical
- Sustained abnormal CPU spikes
- Core service unresponsive

**ERROR examples (no auto-notify):**
- API timeout (data stale but system functional)
- Device command failed (retry later)
- Automation couldn't complete non-critical path

### Escalation is Flow Responsibility

If a flow wants to notify after repeated ERRORs, *that flow* decides:

```
┌─────────────────────────────────────────────────────────────┐
│  Example: Weather Flow                                      │
│                                                             │
│  API call fails → log ERROR                                 │
│       │                                                     │
│       ▼                                                     │
│  Increment failure counter (flow context)                   │
│       │                                                     │
│       ▼                                                     │
│  Counter > threshold? ──► YES ──► Publish to notify         │
│       │                           (deliberate choice)       │
│       ▼                                                     │
│      NO → continue, try again next cycle                    │
└─────────────────────────────────────────────────────────────┘
```

The logging framework doesn't escalate. Flows own their escalation logic.

---

## Device Registry

### Purpose

Centralized knowledge about devices — protocol, topic structure, capabilities, and metadata. Abstracts the differences between Z2M and Z-Wave JS UI so flows don't need to know protocol details.

### Registry Structure

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
  "garage_motion_sensor": {
    "friendly_name": "Garage Motion Sensor",
    "protocol": "zigbee",
    "topic": "zigbee2mqtt/garage_motion_sensor",
    "area": "garage",
    "capabilities": ["motion", "battery"],
    "battery": {
      "type": "CR2032",
      "quantity": 1
    }
  },
  "foyer_entry_door": {
    "friendly_name": "Front Door Lock",
    "protocol": "zwave",
    "topic": "zwave/foyer_entry_door",
    "area": "foyer",
    "capabilities": ["lock", "battery"],
    "battery": {
      "type": "AA",
      "quantity": 4
    }
  }
}
```

**Fields:**

| Field | Purpose |
|-------|---------|
| `friendly_name` | User-facing name for notifications, dashboards |
| `protocol` | `zigbee` or `zwave` — determines command formatting |
| `topic` | Base MQTT topic for this device |
| `area` | Physical area (for grouping, context) |
| `capabilities` | What actions this device supports |
| `battery` | Battery metadata (null if mains-powered) |

**Note:** The registry key (e.g., `foyer_entry_door`) is used for internal references and ACK correlation. `friendly_name` is used for user-facing output.

### Storage

Device registry is stored as an external JSON file, loaded into global context at startup. See **Configuration Management** section for full details.

**File:** `/home/nodered/config/device_registry.json`

**Access:** `global.get('config.deviceRegistry')`

### Population

**Manual with validation:**
- Maintain the JSON file directly (IDE, version control)
- Validation flow checks actual devices against registry
- Reports discrepancies (log/notify), does not block commands

---

## Configuration Management

### Overview

Centralized configuration using external JSON files. Separation of version-controllable config from secrets.

### File Structure

```
/home/nodered/config/
├── device_catalog.json         ← git: yes (model battery specs, friendly name overrides)
├── device_registry.json        ← git: yes
├── flow_registry.json          ← git: yes (area→device mappings, if persisted)
├── location.json               ← git: yes (lat/lon, timezone, elevation)
├── notifications.json          ← git: yes (recipient mappings, channels)
├── system.json                 ← git: yes (system identity, HTTP defaults)
├── thresholds.json             ← git: yes (battery, health, etc.)
├── healthchecks.json           ← git: yes (service config)
├── secrets.json                ← git: NO (.gitignore)
└── README.md                   ← git: yes (documents config structure)
```

*Note: Scheduler configuration (periods, sunrise/sunset) lives in schedex nodes within the Scheduler flow, not external config. `location.json` is the authoritative source for coordinates — schedex nodes reference it as documentation but require the values to be entered manually in the UI.*

### Config Categories

| Category | Examples | Version Control |
|----------|----------|-----------------|
| **Structural** | Device registry, flow registry, notification recipients | Yes |
| **Tunable** | Thresholds, scheduler times, timeouts | Yes |
| **Secrets** | API keys, credentials, tokens, passwords | **No** |

### Example: system.json

```json
{
  "http": {
    "user_agent": "(Highland-SmartHome, highland@ferris.network)"
  }
}
```

### Example: secrets.json

```json
{
  "mqtt": {
    "username": "highland",
    "password": "..."
  },
  "smtp": {
    "host": "smtp.example.com",
    "port": 587,
    "secure": false,
    "user": "...",
    "password": "..."
  },
  "weather_api_key": "abc123...",
  "google_calendar_api_key": "...",
  "healthchecks_io": {
    "mqtt": "https://hc-ping.com/uuid-1",
    "z2m": "https://hc-ping.com/uuid-2",
    "zwave": "https://hc-ping.com/uuid-3",
    "ha": "https://hc-ping.com/uuid-4",
    "node_red": "https://hc-ping.com/uuid-5"
  },
  "ai_providers": {
    "openai_api_key": "sk-...",
    "anthropic_api_key": "sk-ant-..."
  }
}
```

### Example: thresholds.json

```json
{
  "battery": {
    "warning": 35,
    "critical": 15
  },
  "health": {
    "disk_warning": 70,
    "disk_critical": 90,
    "cpu_warning": 80,
    "cpu_critical": 95,
    "memory_warning": 80,
    "memory_critical": 95,
    "devices_offline_critical_percent": 20
  },
  "ack": {
    "default_timeout_seconds": 30
  }
}
```

### Example: notifications.json

```json
{
  "people": {
    "joseph": {
      "admin": true,
      "channels": {
        "ha_companion": "notify.mobile_app_joseph_phone"
      }
    },
    "spouse": {
      "admin": false,
      "channels": {
        "ha_companion": "notify.mobile_app_spouse_phone"
      }
    }
  },
  "areas": {
    "living_room": {
      "channels": {
        "tv": {
          "media_player": "media_player.living_room_tv",
          "sources": [
            { "name": "FIOS TV", "type": "android_tv" },
            { "name": "Xbox", "type": "webos" },
            { "name": "Playstation", "type": "webos" }
          ],
          "endpoints": {
            "android_tv": "notify.living_room_stb",
            "webos": "notify.living_room_lg_tv"
          }
        }
      }
    }
  },
  "daily_digest": {
    "enabled": true
  },
  "defaults": {
    "admin_only": ["joseph"],
    "all": ["joseph", "spouse"]
  }
}
```

**Target addressing:** Notifications use a `targets` array of namespaced strings (`namespace.key.channel`). The `*` wildcard expands all keys in a namespace section.

| Example target | Resolves to |
|----------------|-------------|
| `people.joseph.ha_companion` | Joseph's phone via HA Companion |
| `people.*.ha_companion` | All people's HA Companion |
| `areas.living_room.tv` | Living room TV |
| `areas.*.tv` | All area TVs |

**`people`** — Each person has an `admin` flag and a `channels` map of channel name → HA notify service address. `admin` determines whether a person receives administrative notifications (system health, infrastructure alerts).

**`areas`** — Each area has a `channels` map. The `tv` channel includes a `media_player` entity for HA state lookup, a `sources` array mapping input names to delivery types, and an `endpoints` map of type → HA notify service. Unknown sources (not in `sources` array) log WARN and drop — adding a new input device to a TV is a two-step process: physical setup + config update.

**`defaults`** — Named target lists for convenience. Callers expand these into `targets` entries explicitly — no implicit defaulting.

### Config Loader Utility Flow

Loads all config files into global context at startup.

```
┌─────────────────────────────────────────────────────────────────────┐
│  Config Loader (Utility Flow)                                       │
│                                                                     │
│  Triggers:                                                          │
│    • Node-RED startup                                               │
│    • Node-RED deploy                                                │
│    • Manual inject                                                  │
│    • MQTT: highland/command/config/reload                           │
│    • MQTT: highland/command/config/reload/{config_name}             │
│                                                                     │
│  Actions:                                                           │
│    1. Read each JSON file from /home/nodered/config/                │
│    2. Validate JSON structure                                       │
│    3. Store in global.config namespace:                             │
│         global.config.deviceCatalog                                 │
│         global.config.deviceRegistry                                │
│         global.config.flowRegistry                                  │
│         global.config.location                                      │
│         global.config.notifications                                 │
│         global.config.system                                        │
│         global.config.thresholds                                    │
│         global.config.healthchecks                                  │
│         global.config.secrets                                       │
│    4. Log: "Config loaded: {list}"                                  │
└─────────────────────────────────────────────────────────────────────┘
```

### Accessing Config in Flows

```javascript
// Device catalog — battery specs and friendly name overrides
const catalog = global.get('config.deviceCatalog');
const batterySpec = catalog?.models?.['PS-S04D']?.battery;
const override = catalog?.overrides?.['office_desk_presence'];

// Device info
const device = global.get('config.deviceRegistry.foyer_entry_door');
const friendlyName = device.friendly_name;

// Location (lat/lon for solar calculations, weather APIs, etc.)
const location = global.get('config.location');
const { latitude, longitude, timezone, elevation_ft } = location;

// System identity (User-Agent for external HTTP calls)
const userAgent = global.get('config')?.system?.http?.user_agent;

// Thresholds
const batteryWarn = global.get('config.thresholds.battery.warning');
const batteryCrit = global.get('config.thresholds.battery.critical');

// Secrets
const apiKey = global.get('config.secrets.weather_api_key');
const mqttUser = global.get('config.secrets.mqtt.username');

// Notification recipients
const josephDevice = global.get('config.notifications.recipients.mobile_joseph');
const adminRecipients = global.get('config.notifications.defaults.admin_only');
```

### Structural Validation

On load, validate each config file:
- JSON parses correctly
- Required fields present for each entry type
- Log errors, don't crash Node-RED

### Discovery Validation

Periodic or on-demand check comparing device registry against actual Z2M/Z-Wave device lists:
- Devices in Z2M/Z-Wave but not in registry → log/notify (unregistered)
- Devices in registry but not in Z2M/Z-Wave → log/notify (stale or offline)
- Does **not** block commands to unregistered devices

---

## Command Dispatcher

### Purpose

Translate high-level commands ("turn on garage_carriage_left") into protocol-specific MQTT messages. Flows say *what* they want; the dispatcher knows *how*.

### Common Actions (v1)

| Action | Applies To | Notes |
|--------|------------|-------|
| `on` | lights, switches | Turn on |
| `off` | lights, switches | Turn off |
| `toggle` | lights, switches | Toggle state |
| `brightness` | dimmable lights | Set brightness (0-255 or 0-100, normalized) |
| `lock` | locks | Engage lock |
| `unlock` | locks | Disengage lock |
| `raw` | any | Passthrough for unsupported actions |

### Subflow Interface

**Input:**
```json
{
  "entity": "garage_carriage_left",
  "action": "on"
}
```

```json
{
  "entity": "garage_carriage_left",
  "action": "brightness",
  "value": 50
}
```

```json
{
  "entity": "some_device",
  "action": "raw",
  "payload": { "custom": "data" }
}
```

**Behavior:**
1. Lookup entity in Device Registry
2. Validate action against capabilities (optional, could warn/error)
3. Format payload based on protocol + action
4. Publish to appropriate topic

### Protocol Translation

**Zigbee (Z2M):**
```
Topic: zigbee2mqtt/{device}/set
Payload: {"state": "ON"} / {"brightness": 255}
```

**Z-Wave (Z-Wave JS UI MQTT gateway):**
```
Topic: zwave/{node}/set (configurable)
Payload: Protocol-specific, may differ
```

The dispatcher handles this translation internally.

### Extending Actions

| Scenario | Approach |
|----------|----------|
| New device, existing capability | Add to registry only |
| One-off command | Use `raw` passthrough |
| Repeated new capability | Add to common actions |

### Usage in Flows

```
┌──────────────────┐    ┌────────────────────┐
│ Evening period   │───►│ Command Dispatcher │
│ event arrives    │    │                    │
│                  │    │ entity: garage_    │
│                  │    │   carriage_left    │
│                  │    │ action: on         │
└──────────────────┘    └────────────────────┘
```

Flow doesn't know or care about Zigbee topics or payload formats.

---

## ACK Tracker Utility Flow

### Purpose

Centralized tracking of acknowledgment requests. Flows that need confirmation of actions register their expectations, the tracker collects ACKs, and reports results on timeout. Keeps ACK bookkeeping out of individual flows.

### Topics

| Topic | Purpose | Publisher |
|-------|---------|-----------|
| `highland/ack/register` | Register expectation for ACKs | Requesting flow |
| `highland/ack` | ACK responses | Responding flows |
| `highland/ack/result` | Outcome after timeout | ACK Tracker |

### Flow Diagram

```
┌─────────────────────────────────────────────────────────────────────┐
│ Flow A (Request Originator)                                         │
│                                                                     │
│  1. Generate message_id                                             │
│  2. Register with ACK Tracker via highland/ack/register             │
│  3. Publish event with message_id, request_ack: true                │
│  4. Subscribe to highland/ack/result, filter by correlation_id      │
│  5. Handle success/failure                                          │
└─────────────────────────────────────────────────────────────────────┘
         │                                          │
         │ highland/event/...                       │ highland/ack/register
         ▼                                          ▼
┌──────────────────────┐                 ┌─────────────────────────────┐
│ Flow B (Subscriber)  │                 │ ACK Tracker Utility Flow    │
│                      │                 │                             │
│  • Receives event    │                 │  • Receives registrations   │
│  • Does work         │                 │  • Starts timeout timer     │
│  • Publishes ACK     │────────────────►│  • Collects ACKs by         │
│                      │ highland/ack    │    correlation_id           │
└──────────────────────┘                 │  • On timeout: publish      │
                                         │    result                   │
                                         └──────────────┬──────────────┘
                                                        │
                                                        │ highland/ack/result
                                                        ▼
                                         ┌─────────────────────────────┐
                                         │ Flow A (handles result)     │
                                         │                             │
                                         │  • Lookup missing sources   │
                                         │    in Device Registry       │
                                         │  • Resolve friendly names   │
                                         │  • Notify / escalate        │
                                         └─────────────────────────────┘
```

### Payloads

**Registration (Flow A → Tracker):**
```json
{
  "correlation_id": "abc123",
  "expected_sources": ["foyer_entry_door", "garage_entry_door"],
  "timeout_seconds": 30,
  "source": "security"
}
```

**ACK (Flow B → Tracker):**
```json
{
  "ack_correlation_id": "abc123",
  "source": "foyer_entry_door",
  "timestamp": "2025-02-24T22:00:05Z"
}
```

**Result (Tracker → Flow A):**
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

### Standard Event Properties

All events that may request ACKs include these optional properties:

```json
{
  "message_id": "uuid-or-similar",
  "timestamp": "...",
  "source": "...",
  "request_ack": true
}
```

When `request_ack: false` (or absent), no ACK infrastructure is engaged.

### Timeout-Only Failure Detection

ACK responses are positive only. If a responder fails to complete its work, it simply doesn't send an ACK. The tracker detects this as a missing response on timeout.

No explicit failure ACKs — keeps the pattern simple. If future use cases require explicit failure reporting, we can extend.

### Separation of Concerns

| Component | Responsibility |
|-----------|----------------|
| **ACK Tracker** | Count ACKs, track by correlation_id, report results (raw keys only) |
| **Requesting Flow** | Register expectations, handle success/failure, decide escalation |
| **Notification Flow** | Resolve friendly names from Device Registry, format user-facing messages |

The tracker doesn't know what a "lock" is or what "foyer_entry_door" means. It just counts and reports.

### Friendly Name Resolution

The `source` in ACK payloads and `missing`/`sources` in results are keys into the Device Registry. Consuming flows (or the Notification flow) perform lookups to get `friendly_name` for user-facing messages.

```
missing: ["garage_entry_door"]
    │
    ▼
Device Registry lookup
    │
    ▼
friendly_name: "Garage Door Lock"
    │
    ▼
Notification: "Lockdown failed: Garage Door Lock did not respond"
```

### Example: Lockdown Flow

```
Security Flow (lockdown):
  │
  ├─► Publish: highland/ack/register
  │   { correlation_id: "lock_123", 
  │     expected_sources: ["foyer_entry_door", "garage_entry_door"], 
  │     timeout_seconds: 30 }
  │
  ├─► Publish: highland/event/security/lockdown
  │   { message_id: "lock_123", request_ack: true }
  │
  └─► Subscribe: highland/ack/result (filter: correlation_id == "lock_123")

Front Door Flow:
  │
  ├─► Receives: highland/event/security/lockdown
  ├─► Commands lock via Command Dispatcher
  └─► Publishes: highland/ack
      { ack_correlation_id: "lock_123", source: "foyer_entry_door" }

Garage Door Flow:
  │
  ├─► Receives: highland/event/security/lockdown
  ├─► Commands lock... (but lock jams, command times out)
  └─► (No ACK sent)

ACK Tracker:
  │
  └─► After 30s, publishes: highland/ack/result
      { correlation_id: "lock_123", expected: 2, received: 1, 
        sources: ["foyer_entry_door"], missing: ["garage_entry_door"], 
        success: false }

Security Flow (receives result):
  │
  ├─► Looks up "garage_entry_door" in Device Registry → "Garage Door Lock"
  └─► Publishes: highland/event/notify
      { severity: "high", title: "Lockdown Failed", 
        message: "Garage Door Lock did not respond", ... }
```

---

## Battery Monitor Utility Flow

### Purpose

Track battery-powered device levels, flag devices needing replacement, notify appropriately.

### Battery States

| State | Threshold | Notification | Notes |
|-------|-----------|--------------|-------|
| `normal` | > 35% | None | Healthy |
| `low` | 35% - 15% | Normal priority | Doesn't break DND; sent once when threshold crossed |
| `critical` | < 15% | High priority | Breaks DND; repeats every 24 hours until recovered |

**Threshold rationale:** Some devices (e.g., Sonoff) report in 10% increments. These thresholds provide 7 levels of normal, 2 levels of low, 1 level of critical for coarse reporters.

### Device Registry Extension

Battery-powered devices include battery metadata:

```json
{
  "garage_motion_sensor": {
    "friendly_name": "Garage Motion Sensor",
    "protocol": "zigbee",
    "topic": "zigbee2mqtt/garage_motion_sensor",
    "area": "garage",
    "capabilities": ["motion", "battery"],
    "battery": {
      "type": "CR2032",
      "quantity": 1
    }
  },
  "foyer_entry_door": {
    "friendly_name": "Front Door Lock",
    "protocol": "zwave",
    "topic": "zwave/foyer_entry_door",
    "area": "foyer",
    "capabilities": ["lock", "battery"],
    "battery": {
      "type": "AA",
      "quantity": 4
    }
  }
}
```

### Flow Behavior

```
Raw MQTT (battery level reports)
         │
         ▼
┌─────────────────────────────────────────────────────────────┐
│  Battery Monitor Utility Flow                               │
│                                                             │
│  1. Receive battery level from device                       │
│  2. Lookup device in registry                               │
│  3. Determine state (normal/low/critical)                   │
│  4. Compare to previous state                               │
│  5. If state changed → publish event, notify if appropriate │
│  6. If critical → schedule 24hr repeat reminder             │
│  7. If recovered from critical → cancel repeat reminder     │
└─────────────────────────────────────────────────────────────┘
         │
         ├──► highland/event/battery/low
         ├──► highland/event/battery/critical
         └──► highland/event/battery/recovered
```

### Events

**State transition events:**
```
highland/event/battery/low
highland/event/battery/critical
highland/event/battery/recovered
```

**Payload:**
```json
{
  "timestamp": "2025-02-23T14:30:00Z",
  "source": "battery_monitor",
  "entity": "garage_motion_sensor",
  "level": 32,
  "previous_state": "normal",
  "new_state": "low",
  "battery": {
    "type": "CR2032",
    "quantity": 1
  }
}
```

### Notification Behavior

| Transition | Action |
|------------|--------|
| normal → low | Notify once, normal priority |
| low → critical | Notify immediately, high priority; start 24hr repeat |
| normal → critical | Notify immediately, high priority; start 24hr repeat |
| critical → low | Cancel repeat; notify recovery (normal priority) |
| critical → normal | Cancel repeat; notify recovery (normal priority) |
| low → normal | No notification (silent recovery) |

### Hysteresis

**Re-evaluate, no manual clearing.** If a battery level bounces back above a threshold, the device automatically recovers to the appropriate state. No manual intervention required.

### Open Questions

- [ ] Where to surface "devices needing batteries" data (dashboard widget? periodic summary?)
- [ ] Shopping list aggregation by battery type (deferred)

---

## Notification Framework

### Concept

Notifications answer: *"How urgently does a human need to know about this?"*

Notifications are separate from logging. A CRITICAL log may auto-generate a notification, but many notifications have nothing to do with errors (weather alerts, reminders, security events).

**Example:** NWS issues a Special Weather Statement for patchy fog at 3am.
- Log level: `INFO` at most (not an error, just an event)
- Notification: LOW severity, does NOT break DND (fog lifting by 6am isn't urgent)

### Notification Topic

Single topic — all details in payload:

```
highland/event/notify
```

### Notification Payload (Internal)

```json
{
  "timestamp": "2025-02-24T14:30:00Z",
  "source": "security",
  "channels": ["ha_companion"],
  "recipients": ["joseph", "spouse"],
  "severity": "high",
  "title": "Lock Failed to Engage",
  "message": "Front Door Lock did not respond within 30 seconds",
  "dnd_override": true,
  "media": {
    "image": "http://camera.local/snapshot.jpg"
  },
  "actionable": true,
  "actions": [
    { "id": "retry", "label": "Retry Lock" },
    { "id": "dismiss", "label": "Dismiss" }
  ],
  "sticky": true,
  "group": "security_alerts",
  "correlation_id": "lockdown_20250224_2200"
}
```

### Severity Levels

| Severity | DND Override | Use Case |
|----------|--------------|----------|
| `low` | No | Informational; can wait (fog advisory, routine status) |
| `medium` | No | Worth knowing soon, but not urgent |
| `high` | Yes | Needs attention now (lock failure, unexpected motion) |
| `critical` | Yes | Emergency (fire, flood, intrusion) |

### Fields

| Field | Required | Description |
|-------|----------|-------------|
| `channels` | **Yes** | Which delivery channels to use: `["ha_companion"]`, `["ha_companion", "pushover"]`, etc. Explicit — no defaulting |
| `recipients` | **Yes** | Named people from `notifications.json`: `["joseph"]`, `["joseph", "spouse"]`. Use `defaults.admin_only` or `defaults.all` values explicitly — no implicit defaulting |
| `severity` | Yes | `low`, `medium`, `high`, `critical` |
| `title` | Yes | Short summary |
| `message` | Yes | Full detail |
| `dnd_override` | No | Derived from severity if not specified |
| `media` | No | Image and/or video URLs |
| `actionable` | No | Can recipient respond?; default = false |
| `actions` | No | Available response actions |
| `sticky` | No | Notification persists until dismissed; default = false |
| `group` | No | Group related notifications together |
| `correlation_id` | No | For linking response back to originating event; also used as notification tag |

### Channel Selection Philosophy

**Both `channels` and `recipients` are required** — every notification represents a deliberate design-time decision about who receives it and how. No implicit defaulting avoids the trap of sending all notifications to everyone.

**Multi-channel is intent, not failover.** Specifying `["ha_companion", "pushover"]` means deliver via both regardless of availability. Specifying `["ha_companion"]` means deliver via HA Companion only — the Connection Gate handles availability for that specific channel path.

**Resiliency is the caller's responsibility.** If a notification requires guaranteed delivery, the caller specifies multiple channels. The Notification Utility delivers what it can via the channels specified — it does not retry or compensate for a channel being unavailable. A caller that needs resilience against HA being down should include a secondary channel; a caller that specifies only `ha_companion` has made a conscious decision that delivery is best-effort.

**Graceful degradation within a channel.** The MQTT payload is designed for maximum richness. Each channel adapter extracts what it supports and silently ignores the rest. No payload changes needed when new channels are added.

**Missing channel address → log WARN, skip, continue.** If a recipient has no address for a specified channel, log a warning and continue delivering to other recipients/channels. Deliver as much as possible.

### Mobile Implementation: HA Companion App (Android)

Initial implementation uses Home Assistant Companion App. Future channels added as needed.

#### Device Targeting

Recipients are named people from `notifications.json`. The Notification Utility looks up each person's `channels.ha_companion` address:

| Person | `channels.ha_companion` |
|--------|------------------------|
| `joseph` | `notify.mobile_app_joseph_phone` |
| `spouse` | `notify.mobile_app_spouse_phone` |

**Use case:** Administrative notifications (system health, backups) specify `recipients: ["joseph"]`. Security alerts specify `recipients: ["joseph", "spouse"]`.

#### Android Notification Channels

Pre-configure channels in HA Companion App for user control over sound/vibration/DND:

| Channel ID | Purpose | DND Override |
|------------|---------|--------------|
| `highland_low` | Informational | No |
| `highland_default` | Standard alerts | No |
| `highland_high` | Urgent alerts | Yes |
| `highland_critical` | Emergency | Yes |

#### Severity → HA Companion Mapping

| Our Severity | HA Priority | Channel | Persistent |
|--------------|-------------|---------|------------|
| `low` | `low` | `highland_low` | No |
| `medium` | `default` | `highland_default` | No |
| `high` | `high` | `highland_high` | No (unless `sticky: true`) |
| `critical` | `high` | `highland_critical` | Yes |

#### HA Companion Service Call

Our payload translated to HA service call:

```yaml
service: notify.mobile_app_joseph_phone
data:
  title: "Lock Failed to Engage"
  message: "Front Door Lock did not respond within 30 seconds"
  data:
    channel: "highland_high"
    importance: "high"
    persistent: true
    image: "http://camera.local/snapshot.jpg"
    tag: "lockdown_20250224_2200"
    group: "security_alerts"
    actions:
      - action: "RETRY_lockdown_20250224_2200"
        title: "Retry Lock"
      - action: "DISMISS_lockdown_20250224_2200"
        title: "Dismiss"
```

#### Clearing Notifications

To programmatically dismiss a notification (e.g., lock succeeded on retry):

```yaml
service: notify.mobile_app_joseph_phone
data:
  message: "clear_notification"
  data:
    tag: "lockdown_20250224_2200"
```

Same `tag`, message `"clear_notification"` — notification disappears from device.

**Use case:** Battery critical notification auto-clears when battery recovers.

#### Notification Grouping

Use `group` field to stack related notifications:

```yaml
data:
  group: "battery_alerts"
```

Multiple battery warnings appear grouped rather than as separate notifications.

### Action Responses

When user taps a notification action, HA fires an event. The Notification Utility flow captures this and publishes to MQTT for consistent handling:

**HA Event:**
```
Event: mobile_app_notification_action
Data: { action: "RETRY_lockdown_20250224_2200" }
```

**Published to MQTT:**
```
Topic: highland/event/notify/action_response
Payload: {
  "timestamp": "2025-02-24T14:32:00Z",
  "source": "notification",
  "action": "retry",
  "correlation_id": "lockdown_20250224_2200",
  "device": "mobile_joseph"
}
```

Originating flow (e.g., Security) subscribes to `highland/event/notify/action_response`, filters by `correlation_id`, and handles accordingly.

### Notification Utility Flow

Centralized flow that:
1. Subscribes to `highland/event/notify`
2. Translates internal payload → HA Companion format
3. Calls appropriate `notify.mobile_app_*` service(s) based on `recipients`
4. Subscribes to HA `mobile_app_notification_action` events
5. Translates action responses → `highland/event/notify/action_response`

```
┌─────────────────────────────────────────────────────────────────────┐
│  Notification Utility Flow                                          │
│                                                                     │
│  highland/event/notify ───► Translate ───► notify.mobile_app_*     │
│                                                                     │
│  HA event: mobile_app_notification_action                           │
│       │                                                             │
│       ▼                                                             │
│  highland/event/notify/action_response                              │
└─────────────────────────────────────────────────────────────────────┘
```

### Future Channels (Deferred)

| Channel | Implementation | Notes |
|---------|----------------|-------|
| `telegram` | Telegram Bot API | Two-way interaction possible |
| `signal` | Signal CLI or API | Privacy-focused |
| `tv` | HA notification to TV entity | WebOS, Android TV, etc. |
| `tts` | Text-to-speech on smart speakers | |
| `email` | SMTP integration | |

Channels can be added to the Notification Utility flow as needed. The internal payload structure remains the same; only the translation layer changes.

---

## Utility: Connections

### Purpose

Tracks the live state of external service connections and exposes that state via global context for any flow that needs to make runtime decisions based on it. Distinct from `Utility: Health Checks` — Health Checks *reports* infrastructure health outward (to Healthchecks.io, to logs); Connections *exposes* connection state inward to other flows.

### Detection Mechanism

Uses Node-RED's built-in `status` node scoped to a connection-bearing node. The `status` node fires immediately when the monitored node's connection state changes — no polling, no second connection, no additional palette dependencies.

**Signal mapping (via `msg.status.fill`):**
- `'red'` or `'yellow'` → connection is down → `connections.{key} = 'down'`
- anything else → connection is up → `connections.{key} = 'up'`

This pattern works with any node that reports connection status — MQTT in/out nodes, HA palette nodes, etc. The monitored node exists for its primary purpose; the status node is a free side-channel into its connection state.

### Startup Settling

On restart, connections briefly drop before re-establishing. Without mitigation, every restart generates spurious "connection lost" log entries. The flow uses a **startup settling window** to absorb this noise:

- `Startup Tasks` group fires on startup with a 0.1s delay
- `Establish Cadence` function sets `flow.timer_cadence` (regular flow context — persists as a config value) and starts a `setTimeout` that sets `flow.settled = true` in the `volatile` store after the cadence elapses
- `Evaluate State` nodes read `flow.get('settled', 'volatile')` to determine whether they are in the startup window

**During the startup window (not settled):**
- `'down'` transitions start a debounce timer — if still down when the timer fires, log it
- `'up'` transitions cancel any pending timer silently — startup noise absorbed, no log

**After the window (settled):**
- All transitions logged immediately as real-time events

**Timer handles** are stored in the `volatile` context store — they are non-serializable and would cause a circular reference error if stored in the default (localfilesystem) store.

**Single cadence value** — `flow.timer_cadence` drives both the settling window and the debounce timers. Changing the cadence requires editing exactly one place in `Establish Cadence`.

### Startup vs Runtime Log Behavior

| Scenario | Behavior |
|----------|----------|
| Restart — connections drop and recover within window | Silent — startup noise absorbed |
| Restart — connection genuinely down after window | Logs WARN after debounce timer fires |
| Runtime — connection drops | Logs WARN immediately (settled = true) |
| Runtime — connection recovers | Logs INFO immediately (settled = true) |
| Restart — Node-RED itself | Console errors via `node.error()` — MQTT unavailable for logging |

### MQTT Availability — The Catch-22

When MQTT goes down, the normal log path (`highland/event/log` via MQTT out) is unavailable. The `State Change Logging` group handles this explicitly:

```
Log Event link in → MQTT Available? switch (global.connections.mqtt == 'up')
    ↓ up                                        ↓ else
Format Log Message → MQTT out             Log to Console (node.error/warn)
```

`Log to Console` uses `node.error()` / `node.warn()` which write to Node-RED's internal log regardless of MQTT state — visible via `docker compose logs nodered`.

### Groups

**Home Assistant Connection Monitor** — `status` node scoped to `server-state-changed` (Time Sensor) → Initializer Latch → `Evaluate State` → Log Event link out

**MQTT Connection Monitor** — `status` node scoped to `mqtt in` (Health Probe, `highland/status/mqtt/probe`) → Initializer Latch → `Evaluate State` → Log Event link out

**State Change Logging** — Log Event link in → MQTT Available? switch → Format Log Message → MQTT out / Log to Console

**Startup Tasks** — On Startup inject → Establish Cadence function

### Global Context Keys

| Key | Store | Values | Set by |
|-----|-------|--------|--------|
| `connections.home_assistant` | default | `'up'` / `'down'` | HA Evaluate State |
| `connections.mqtt` | default | `'up'` / `'down'` | MQTT Evaluate State |

### Flow Context Keys

| Key | Store | Value | Set by |
|-----|-------|-------|--------|
| `timer_cadence` | default | ms integer (e.g. 5000) | Establish Cadence |
| `settled` | volatile | `true` / undefined | Establish Cadence (setTimeout) |
| `home_assistant_timer` | volatile | timer handle or null | HA Evaluate State |
| `mqtt_timer` | volatile | timer handle or null | MQTT Evaluate State |

### Usage in Other Flows

```javascript
// Check HA availability before routing a notification
const haAvailable = global.get('connections.home_assistant') !== 'down';

// Check MQTT availability before publishing
const mqttAvailable = global.get('connections.mqtt') !== 'down';
```

The `!== 'down'` guard handles the startup case where the flag hasn't been set yet — `undefined !== 'down'` is `true`, defaulting to assuming the connection is available rather than silently dropping messages during the brief init window.

### Future Additions

Additional connection state flags follow the same pattern — a `status` node scoped to any connection-bearing node, the same `Evaluate State` structure with `NAME` and `KEY` changed, wired into the shared `State Change Logging` group.

- `connections.telegram` — if Telegram is added as a notification channel

---

## Utility: Battery Monitor

### Purpose

Tracks battery levels across all Zigbee devices, detects state transitions, notifies appropriately, and persists state across restarts.

### Topics

| Topic | Retained | Purpose |
|-------|----------|---------|
| `highland/event/battery/low` | No | Device crossed into low state |
| `highland/event/battery/critical` | No | Device crossed into critical state |
| `highland/event/battery/normal` | No | Device recovered to normal |
| `highland/state/battery/states` | Yes | Full battery state map — published on state change and on startup |

### Battery States

| State | Threshold | Notification |
|-------|-----------|-------------|
| `normal` | > 35% | None (or silent recovery from critical) |
| `low` | 15–35% | Once, medium priority |
| `critical` | < 15% | Immediately high priority; repeats every 24hrs until recovered |

**Silent recovery:** `low` → `normal` produces no notification. `critical` → `normal` or `critical` → `low` cancels the repeat timer and notifies recovery at low priority.

**Rechargeable devices:** `rechargeable: true` in the model catalog entry. `Build Notification` branches on this flag — message says "plug in to charge" rather than "replace N× type".

### Groups

**Sinks** — `zigbee2mqtt/#` MQTT in → Initializer Latch → `Extract Battery` → Link Out

**Battery State Pipeline** — Link In → `Evaluate State` (Output 1: state changed, Output 2: no change/drop) → `Build Event` → MQTT out (battery event) + `Build Notification` → MQTT out (notify) + `Build Battery States` → MQTT out (retained state snapshot)

**Device Discovery** — `zigbee2mqtt/bridge/devices` MQTT in → `Build Device Map` (populates `flow.device_models`)

**Device Recovery** — On Startup inject → `Recover Critical State` → MQTT out (notify + battery states snapshot)

**Error Handling** — flow-wide catch → debug

### Device Catalog Pattern

Battery type/quantity is model-level knowledge, not device-instance knowledge. Stored in `device_catalog.json`:

```json
{
  "models": {
    "PS-S04D": {
      "battery": { "type": "CR2450", "quantity": 2 }
    },
    "some-rechargeable-model": {
      "rechargeable": true
    }
  },
  "overrides": {
    "office_desk_presence": {
      "friendly_name": "Joseph's Desk Presence"
    }
  }
}
```

**Maintenance burden:** Add one entry per model the first time you pair a device of that model. All subsequent devices of the same model require no config changes.

### Friendly Name Derivation

Registry absence is never a processing gate. Friendly names are derived automatically from the device key:

```javascript
const friendlyName = deviceKey
    .replace(/_/g, ' ')
    .replace(/\b\w/g, c => c.toUpperCase());
// "office_desk_presence" → "Office Desk Presence"
```

`overrides` in `device_catalog.json` provide explicit friendly name overrides when the derived name isn't sufficient.

### `Build Device Map`

Subscribes to `zigbee2mqtt/bridge/devices` (retained). Fires on startup and on every Z2M restart. Builds `flow.device_models` map: `{ device_key → model_id }`. No latch needed — pure data population, no `global.config` dependency.

### Graceful Degradation

| Condition | Behavior |
|-----------|----------|
| Device not in `device_models` | Process with no model info; no battery spec |
| Model not in catalog | WARN log, process without battery spec detail |
| No battery spec | Notification omits type/quantity detail |
| `battery` field absent | Message filtered in `Extract Battery`, no processing |

### Startup Recovery

On startup, `Recover Critical State` reads `flow.battery_states` (disk-backed) and resumes timers for any device still in `critical` state:
- If `last_notified_critical` > 24hrs ago → notify immediately, start fresh 24hr timer
- If `last_notified_critical` < 24hrs ago → start timer for the remaining window

Also unconditionally publishes the current `flow.battery_states` as a retained snapshot on startup, so `highland/state/battery/states` is always current immediately after any restart — even when no state transitions occur.

### Context Storage

| Key | Store | Content |
|-----|-------|---------|
| `flow.battery_states` | default (disk) | `{ device_key: { state, level, last_notified_critical } }` |
| `flow.device_models` | default (disk) | `{ device_key: model_id }` — rebuilt from Z2M on startup |
| `flow.battery_timers` | volatile (memory) | `{ device_key: timer_handle }` — lost on restart, recovered by Recover Critical State |

---

## Health Monitoring

### Overview

**Philosophy:** Treat this as a line-of-business application. Degradation detection is as important as outage detection.

### Single Point of Failure Problem

Node-RED alone as the health reporter creates ambiguity. If Node-RED goes down, all service checks stop pinging simultaneously — making it impossible to distinguish "Node-RED is down" from "everything is down."

**Solution: each service self-reports its own liveness independently of Node-RED.** Node-RED's edge checks prove the *connection* is healthy; each service's own ping proves the *service* is healthy.

**Healthchecks.io naming convention:**
- `{Service}` — the service's own self-report, independent of Node-RED
- `Node-RED / {Service} Edge` — Node-RED's check proving the connection to that service is healthy

**Failure signature matrix:**

| Failure | Service check | Edge check | Node-RED check |
|---------|---------------|------------|----------------|
| Node-RED down | ✅ pinging | ❌ silent | ❌ silent |
| Service down | ❌ silent | ❌ silent | ✅ pinging |
| Network path broken (both up) | ✅ pinging | ❌ silent | ✅ pinging |

Three distinct signatures — completely unambiguous diagnosis.

**Current implementation status:**
- `Node-RED` — Node-RED self-reports via direct HTTP ping ✅
- `Home Assistant` — native HA automation pinging Healthchecks.io directly ✅
- `Node-RED / Home Assistant Edge` — Node-RED's HTTP check to HA API ✅
- MQTT, Z2M, Z-Wave JS self-reports and edge checks — pending

### Status Values

| Status | Meaning |
|--------|---------|
| `healthy` | Responding AND all metrics within acceptable ranges |
| `degraded` | Responding BUT one or more metrics in warning territory |
| `unhealthy` | Not responding OR critical threshold exceeded |

### Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│  Node-RED (Health Monitor Flow)                                     │
│                                                                     │
│  Monitors: MQTT, Z2M, Z-Wave JS, HA, host resources                 │
│  Publishes: highland/status/{service}/health                        │
│  Notifies: highland/event/notify (on status change)                 │
│  Pings: Healthchecks.io for each monitored service                  │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│  Watchdog Script (Communication Hub)                            │
│                                                                     │
│  Monitors: Node-RED (via heartbeat on MQTT)                         │
│  Publishes: highland/status/node_red/health                         │
│  Pings: Healthchecks.io for Node-RED                                │
│  Note: Cannot notify locally if Node-RED is down — that's the point │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│  Healthchecks.io (Free Tier)                                        │
│                                                                     │
│  Receives pings from Node-RED + Watchdog                            │
│  Only notifies if pings stop (local notification failed/impossible) │
└─────────────────────────────────────────────────────────────────────┘
```

### Threshold Definitions

| Metric | Warning | Critical | Notes |
|--------|---------|----------|-------|
| Disk usage | > 70% | > 90% | Applies to all hosts |
| CPU usage (sustained) | > 80% for 5 min | > 95% for 5 min | Transient spikes ignored |
| Memory usage | > 80% | > 95% | |
| Devices offline (Z2M) | Any (1+) | > 20% of total | Single device offline = degraded |
| Devices offline (Z-Wave) | Any (1+) | > 20% of total | Single device offline = degraded |

### Per-Service Checks

| Service | Responsiveness Check | Threshold Metrics |
|---------|---------------------|-------------------|
| **MQTT broker** | Publish/subscribe test | Host disk, CPU, memory |
| **Zigbee2MQTT** | HTTP API or MQTT bridge topic | Devices online/offline, host resources |
| **Z-Wave JS UI** | WebSocket or HTTP API | Nodes online/dead, host resources |
| **Home Assistant** | HTTP API (`/api/`) | DB size, host resources |
| **Node-RED** | HTTP admin API or heartbeat | Host resources |
| **Communication Hub (host)** | Implicit via services | Disk, CPU, memory |
| **Node-RED host** | Implicit via Node-RED | Disk, CPU, memory |

### Topics

```
highland/status/{service}/heartbeat   → Simple "I'm alive" ping
highland/status/{service}/health      → Detailed health + metrics
```

**Services:**

| Service | Monitored By | Topic |
|---------|--------------|-------|
| MQTT broker | Node-RED | `highland/status/mqtt/health` |
| Zigbee2MQTT | Node-RED | `highland/status/z2m/health` |
| Z-Wave JS UI | Node-RED | `highland/status/zwave/health` |
| Home Assistant | Node-RED | `highland/status/ha/health` |
| Node-RED | Watchdog script | `highland/status/node_red/health` |

### Heartbeat Payload (Simple)

Published by Node-RED to indicate it's alive (consumed by watchdog):

```json
{
  "timestamp": "2025-02-24T10:00:00Z",
  "source": "node_red"
}
```

Topic: `highland/status/node_red/heartbeat`

### Health Payload (Detailed)

```json
{
  "timestamp": "2025-02-24T10:00:00Z",
  "service": "z2m",
  "status": "degraded",
  "checks": {
    "responsive": true,
    "thresholds": {
      "disk_percent": { "value": 45, "status": "ok" },
      "cpu_percent": { "value": 22, "status": "ok" },
      "memory_percent": { "value": 61, "status": "ok" },
      "devices_offline": { "value": 2, "status": "warning" }
    }
  },
  "summary": "2 devices offline"
}
```

### Alarm Evaluation Logic

```
For each threshold metric:
  if value > critical_threshold → metric status = "critical"
  else if value > warning_threshold → metric status = "warning"
  else → metric status = "ok"

Overall status:
  if not responsive → "unhealthy"
  else if any metric = "critical" → "unhealthy"
  else if any metric = "warning" → "degraded"
  else → "healthy"
```

### Check Frequency

Check frequency is per-service based on criticality:

| Service | Frequency | Rationale |
|---------|-----------|-----------|
| MQTT broker | 1 min | Critical — everything depends on it |
| Z2M | 1 min | Critical — Zigbee devices depend on it |
| Z-Wave JS | 1 min | Critical — Z-Wave devices depend on it |
| Home Assistant | 30 sec | Same convention as Node-RED ping |
| Node-RED | 30 sec | Critical — runs all automations |

### Healthchecks.io Configuration

**Grace period:** 3 minutes for all checks — whole-minute minimum, enough buffer for transient latency without being so long that alerts are meaningless.

| Check | Poll Frequency | Grace Period | Notes |
|-------|----------------|--------------|-------|
| Node-RED | 30 sec | 3 min | Pings every 30s; 3 min grace = ~6 missed pings |
| Home Assistant | 30 sec | 3 min | Same convention as Node-RED |
| MQTT | 1 min | 3 min | 3 missed checks before alert |
| Z2M | 1 min | 3 min | 3 missed checks before alert |
| Z-Wave JS | 1 min | 3 min | 3 missed checks before alert |

> **Note:** Healthchecks.io only supports whole-minute grace periods. 3 minutes is the practical minimum — 2 minutes is too aggressive and risks false alerts from transient latency.

### Failure Threshold

**Fail fast:** One missed check = unhealthy. No transient failure allowance initially.

Rationale: Internal checks should not fail transiently. If something fails once, it's genuinely down (or restarting). If this proves too noisy, we can add tolerance later.

### Notification Behavior

**Status change notifications:**

| Status Change | Action |
|---------------|--------|
| healthy → degraded | Notify (normal priority) |
| degraded → unhealthy | Notify (high priority) |
| healthy → unhealthy | Notify (high priority) |
| unhealthy → degraded | Notify recovery (normal priority) |
| unhealthy → healthy | Notify recovery (normal priority) |
| degraded → healthy | No notification (silent recovery) |

**External alerting:**

| Scenario | Who Notifies |
|----------|--------------|
| Z2M goes down | Node-RED (via `highland/event/notify`) |
| HA goes down | Node-RED (via `highland/event/notify`) |
| Node-RED goes down | Healthchecks.io (Node-RED's own ping stops) |
| MQTT goes down | Healthchecks.io (Node-RED's ping is independent of MQTT) |

### Watchdog Script

The original watchdog design (a cron script that subscribes to `highland/status/node_red/heartbeat` via MQTT and pings Healthchecks.io on receipt) has been superseded. Node-RED pings Healthchecks.io directly via HTTP, which correctly separates Node-RED liveness from MQTT liveness.

Whether a watchdog script has a remaining role will be evaluated per-service as the Health Monitor is built. A script may still be appropriate for services that Node-RED cannot reliably check itself (e.g. if Node-RED itself is the thing that's down). This is TBD.

### Future Enhancement

Lightweight MQTT health listener (Python/Go) on Communication Hub:
- Subscribes to `highland/status/#`
- Tracks all service health centrally
- Handles notifications independently of Node-RED
- Services become responsible only for self-reporting heartbeats

This decouples notification logic from monitoring logic, but is not required for initial implementation.

---

## Daily Digest ("State of the Smart Home")

### Purpose

Nightly email summarizing the state of the home. Provides awareness without requiring you to go look.

### Timing

**Trigger:** `highland/event/scheduler/midnight` — the Scheduler's midnight task event.

**Delay:** 5 seconds after midnight — gives the Calendaring flow time to complete its midnight poll before the Digest reads the snapshot, and ensures the date has correctly rolled over.

### Data Sources

The Digest is a pure reader. All data sources are retained MQTT topics — no API calls, no cross-flow context access.

| Section | Source | Topic |
|---------|--------|-------|
| **Appointments** | Utility: Calendaring | `highland/state/calendar/snapshot` |
| **Reminders** | Utility: Calendaring | `highland/state/calendar/snapshot` |
| **Trash & Recycling** | Utility: Calendaring | `highland/state/calendar/snapshot` |
| **Weather** | Utility: Weather Forecasts / Alerts | `highland/state/weather/forecast`, `highland/state/weather/alerts` |
| **Battery Status** | Utility: Battery Monitor | `highland/state/battery/states` |
| **System Health** | Utility: Connections / Health Monitor | `global.connections` + health topics |

### Calendar Filtering

The calendar snapshot is a rolling 7-day window. The Digest filters it for today's date:

```javascript
const today = new Date().toLocaleDateString('en-CA', { timeZone: 'America/New_York' });
const tomorrow = new Date(Date.now() + 86400000).toLocaleDateString('en-CA', { timeZone: 'America/New_York' });

const appointments = snapshot.events.filter(e => e.calendar === 'appointments' && e.date === today);
const reminders = snapshot.events.filter(e => e.calendar === 'reminders' && e.date === today);

// Trash: look-ahead — show if today OR tomorrow
const trash = snapshot.events.filter(e => e.calendar === 'trash' && (e.date === today || e.date === tomorrow));
```

Trash events are surfaced as "Today" or "Tomorrow" based on the date comparison, giving next-day notice for Thursday night's digest.

### Email Design

HTML email generated via template literal in a function node. Inline styles only (email client compatibility). SVG weather icons inline — no external resources.

**Sections:**
1. Appointments — timed events with hour badge and title
2. Reminders — all-day nudges as a simple list
3. Trash & Recycling — today/tomorrow pickup status, two-column layout
4. Weather — optional alert banner (severity-coded), day/night panels with SVG icons + precip/wind, 5-day forecast with small SVG icons
5. Battery Status — devices in `low` or `critical` state only; color-coded left border
6. System Health — green grid when all operational; individual service status badges

**Weather is conditionally omitted** when `highland/state/weather/forecast` has not been published yet (weather flow not yet built). The template handles absent data gracefully.

### Recipients

Configurable in `notifications.json`:

```json
{
  "daily_digest": {
    "enabled": true
  }
}
```

Recipient email addresses stored in `secrets.json`.

---

## Connection Gate Subflow

### Purpose

Guards message flow based on the live state of a connection. Used wherever a flow needs to deliver a message via a connection-dependent path — routing to a fallback or holding briefly for recovery rather than blindly attempting delivery when a connection is down.

Distinct from the `Initializer Latch` — the latch is a one-time startup concern; the gate handles repeated up/down transitions during normal operation.

### Interface

- **1 input** — any message
- **Output 1 (Pass)** — connection is up; message delivered here, either immediately or after recovery
- **Output 2 (Fallback)** — connection is down and not recovered; caller handles alternative delivery or discard

### Environment Variables

| Variable | UI Label | Purpose | Default |
|----------|----------|---------|---------|
| `CONNECTION_TYPE` | Connection | Which connection to check: `home_assistant`, `mqtt` | — |
| `RETENTION_MS` | Retention (ms) | How long to poll for recovery before routing to Output 2. 0 = no retention, route to Output 2 immediately | `0` |
| `CONTEXT_PREFIX` | Scope | Prefix for flow context keys — required when multiple instances on the same flow tab | `''` |

### Behavior

| Scenario | Result |
|----------|--------|
| Connection up | Output 1 immediately |
| Connection down, `RETENTION_MS` = 0 | Output 2 immediately |
| Connection down, `RETENTION_MS` > 0, recovers within window | Output 1 when recovery detected |
| Connection down, `RETENTION_MS` > 0, window expires | Output 2 after window |

**Output 1 always means connected state** — either the connection was up at arrival, or it recovered within the retention window. **Output 2 always means unrecovered down state** — the message is guaranteed to exit one output or the other, no silent drops.

**Latest-only retention** — if a new message arrives while a retention poll is in progress, the existing poll is cancelled and a new one starts for the latest message. Earlier messages are discarded.

**Poll interval** is internalized at 500ms — not configurable per instance. Callers tune behavior via `RETENTION_MS` only.

### Internal Structure

**Evaluate Gate** — single function node with two outputs. Reads `global.connections.{CONNECTION_TYPE}`, applies retention policy, sets `node.status()` for each decision path.

**Status Monitor** — `status` node scoped to `Evaluate Gate`, wired to `Set Status` function node, wired to subflow status output. Surfaces `Evaluate Gate` node status onto the subflow instance in the parent flow without requiring a third output on `Evaluate Gate`.

**Set Status** — translates status event to subflow status payload:
```javascript
msg.payload = {
    fill: msg.status.fill,
    shape: msg.status.shape,
    text: msg.status.text
};
return msg;
```

### Node Status Values

| Status | Meaning |
|--------|---------|
| Green dot — Passed | Gate open, Output 1 immediately |
| Red dot — Fallback | Gate closed, no retention, Output 2 immediately |
| Yellow ring — Waiting... | Gate closed, polling for recovery |
| Green ring — Recovered | Recovered within window, Output 1 |
| Red ring — Expired | Window elapsed without recovery, Output 2 |

Dot = immediate decision, ring = delayed decision.

### Flow Context Keys

| Key | Store | Value | Set by |
|-----|-------|-------|--------|
| `{PREFIX}retention_poll` | volatile | interval handle or null | Evaluate Gate |

### Usage Example

```
Notification msg → Connection Gate (CONNECTION_TYPE=home_assistant, RETENTION_MS=120000)
                        ↓ Output 1                    ↓ Output 2
                   HA Companion delivery          Pushover delivery
```

### Notes

- `CONTEXT_PREFIX` env var is labelled **Scope** in the Node-RED UI — consistent with `Initializer Latch` convention
- Output 2 being unwired is valid — messages that would route to Output 2 are silently discarded
- Timer handles stored in `volatile` context store — non-serializable, intentionally lost on restart

---

## Utility: Notifications

### Purpose

Centralized delivery of notifications to people via configured channels. All notification traffic enters via MQTT and is dispatched by this flow — no other flow calls HA notification services directly.

### Topics

| Topic | Direction | Purpose |
|-------|-----------|---------|
| `highland/event/notify` | Inbound | Deliver a notification |
| `highland/command/notify/clear` | Inbound | Dismiss a previously delivered notification |
| `highland/event/log` | Outbound | Log delivery outcomes |

### Groups

**Receive Notification** — MQTT in (`highland/event/notify`) → Initializer Latch → Validate Payload → Build Targets → `link call` (Deliver, dynamic) → Log Event link out

**HA Companion Delivery** — Link In (`Home Assistant Companion`) → Connection Gate → Build Service Call → HA service call node → `link out` (return mode)

**Clear Notification** — MQTT in (`highland/command/notify/clear`) → Initializer Latch → Build Clear Call → `link call` (Deliver, dynamic) → Log Event link out

**State Change Logging** — Log Event link in → MQTT Available? switch → Format Log Message → MQTT out / Log to Console

**Test Cases** — Persistent sanity tests; intentionally not removed.

### Initializer Latch Scopes

| Group | `CONTEXT_PREFIX` |
|-------|-----------------|
| Receive Notification | `notify_in-` |
| Clear Notification | `notify_clear-` |

### Validate Payload

Required: `targets`, `severity`, `title`, `message`. `targets` must be a non-empty array. Invalid → WARN log, drop.

### Build Targets (Fan Out)

Resolves a `targets` array of namespaced strings into individual delivery messages. Each target is `namespace.key.channel` — e.g. `people.joseph.ha_companion`, `areas.living_room.tv`, `areas.*.tv`. The `*` wildcard expands all keys in a namespace section.

Resolution logic:
1. Split target into `[namespace, key, channel]`
2. Look up `notifications[namespace]` — WARN and skip if unknown
3. Expand `*` to all keys in the namespace section
4. Look up `entry.channels[channel]` for each key — WARN and skip if missing
5. Emit one message per resolved address with `msg.payload._delivery` and `msg.target` set

```javascript
node.send({
    payload: {
        ...msg.payload,
        _delivery: { namespace, key, channel, address }
    },
    target: resolveLinkTarget(channel)
});
```

`resolveLinkTarget()` maps channel names to their `Link In` node names:

```javascript
function resolveLinkTarget(channel) {
    switch (channel) {
        case 'ha_companion': return 'Home Assistant Companion';
        case 'tv':           return 'Television Delivery';
        default: throw new Error(`Unable to resolve channel: ${channel}`);
    }
}
```

Adding a new channel: add a case here and a new delivery group with a matching `Link In` name.

### `link call` Node (Deliver)

Reads `msg.target` dynamically and routes to the matching `Link In` node name. Set to **dynamic** link type, 30 second timeout. Output wires to Log Event link out — logging happens once on the return path after delivery completes. Timeouts handled by a catch node scoped to the `link call` — logs WARN and moves on.

### Connection Gate (HA Companion Delivery)

`CONNECTION_TYPE = home_assistant`, `CONTEXT_PREFIX = ha-`, `RETENTION_MS = 0`. Output 2 unwired — if HA is down the message drops. Resiliency is the caller's responsibility via channel selection.

### Build Service Call

Handles both delivery and clear paths, branched on `_delivery.type`:

```javascript
// Clear path
if (_delivery.type === 'clear') {
    msg.payload = {
        action: _delivery.address,
        data: { message: 'clear_notification', data: { tag: msg.payload.correlation_id } }
    };
    msg.log_message = `Notification cleared for ${_delivery.recipient} via ${_delivery.channel}`;
    return msg;
}

// Delivery path
msg.payload = { action: _delivery.address, data: { title, message, data } };
msg.log_message = `Notification delivered to ${_delivery.recipient} via ${_delivery.channel}`;
return msg;
```

**Severity → HA Companion mapping:**

| Severity | Channel | Importance | Persistent |
|----------|---------|------------|------------|
| `low` | `highland_low` | `low` | No |
| `medium` | `highland_default` | `default` | No |
| `high` | `highland_high` | `high` | No (unless `sticky: true`) |
| `critical` | `highland_critical` | `high` | Yes |

**`api-call-service` node:** Action field blank (reads `msg.payload.action` implicitly); Data field = JSONata `payload.data`.

### Build Clear Call

Structurally identical to `Build Targets` — resolves the `targets` array using the same namespace resolver and wildcard expansion, sets `_delivery.type: 'clear'`, and dispatches via `link call`. Each delivery group decides what to do with a clear — `HA Companion Delivery` sends `clear_notification`, channels that auto-dismiss (e.g. TV) receive it and return immediately via their return `link out`.

`correlation_id` must match the original delivery. `tag` ≠ `correlation_id`.

**Clear payload (MQTT):**
```json
{
  "correlation_id": "lockdown_20250224_2200",
  "targets": ["people.joseph.ha_companion"]
}
```

### HA Companion Delivery — Return Path

The last node in the group is a `link out` set to **return** mode. This returns the message to whichever `link call` dispatched it (Receive Notification or Clear Notification), completing the call/return cycle and triggering downstream logging.

### State Change Logging

Same MQTT/console fallback pattern as `Utility: Connections`. `Build Service Call` sets `msg.log_message` on both delivery and clear paths before returning to the caller. `Format Log Message` reads `msg.log_level` (default `INFO`), `msg.log_message`, and `msg.log_context`.

### Notes

- `Utility: Notifications` is the only flow that calls HA notify services
- All delivery and clear traffic uses the same `Build Targets` / `Build Clear Call` resolver pattern — no channel-specific logic at the dispatcher level
- Each delivery group decides how to handle `_delivery.type === 'clear'` — auto-dismiss channels receive and return immediately
- Adding a new channel: add a case to `resolveLinkTarget()`, build a new delivery group with a matching `Link In` name, wire the return `link out`
- Adding a new device to a TV requires updating `notifications.json` to include the new source — the system logs WARN and drops on unknown sources, acting as a natural reminder that config step two is pending
- Action responses deferred until actionable notifications are implemented
- Test Cases group preserved for sanity testing

---

## Utility: Scheduling

### Purpose

Publishes period transitions and fixed task events to the MQTT bus. All time-based triggers in Highland originate here — no other flow contains scheduling logic.

### Topics

| Topic | Retained | Purpose |
|-------|----------|---------|
| `highland/state/scheduler/period` | Yes | Current period — ground truth for all period-aware flows |
| `highland/event/scheduler/daytime` | No | Fired on transition to daytime |
| `highland/event/scheduler/evening` | No | Fired on transition to evening |
| `highland/event/scheduler/overnight` | No | Fired on transition to overnight |
| `highland/event/scheduler/midnight` | No | Fired daily at 00:00:00 |

### Periods

Three periods driven by `node-red-contrib-schedex` using solar events and fixed times:

| Period | On time | On offset | Off time | Off offset |
|--------|---------|-----------|----------|------------|
| `daytime` | `sunrise` | 0 | `sunset` | -30 min |
| `evening` | `sunset` | -30 min | `22:00` | 0 |
| `overnight` | `22:00` | 0 | `sunrise` | 0 |

Schedex coordinates: lat `41.5204`, lon `-74.0606` (matches `location.json`). All 7 days enabled.

### Groups

**Dynamic Periods** — Three schedex nodes (Daytime, Evening, Overnight), each wiring through a `link call` return pattern into a shared `Publish Dynamic Period` link in → `Is Active?` switch → `Prepare Dynamic` function → two MQTT out nodes (event + state)

**Fixed Events** — Midnight inject (cron `00 00 * * *`) → `Prepare Fixed` function → MQTT out

**Sinks** — On Startup inject → `Recover Last State` function (sets `startup_recovering` flag, sends `send_state` to all three schedex nodes via dynamic `link call`) → Dynamic Period `link call`

**Test Cases** — Manual injects for daytime, evening, overnight (wired to `Publish Dynamic Period` link in), and midnight (wired to `Prepare Fixed`)

### Startup Recovery

On startup, `Recover Last State` sends `send_state` to all three schedex nodes via dynamic `link call`. Each schedex node emits its current state if it is within its active window — exactly one of the three responds with a non-empty payload. The `Is Active?` switch drops the empty off-window responses. `Prepare Dynamic` detects the `startup_recovering` flag and publishes state only (no event) during the recovery window.

**`startup_recovering` flag:** Set to `true` for 2 seconds in the `volatile` store on startup. During this window, `Prepare Dynamic` suppresses events and publishes state only. After the window, all transitions publish both event and state.

### Period-Aware Flow Pattern

Flows that respond to period changes use **two entry points, one handler**:

```
highland/state/scheduler/period  ──┐  (retained — delivered on subscription,
  (startup recovery path)          │   covers restart/init)
                                   ├──► period logic
highland/event/scheduler/evening ──┘  (non-retained — real-time transition)
```

This is a push model, not polling. The retained state delivers once on subscription; events drive everything thereafter.

**State-following flows** (lights, ambiance): read retained period on startup, act immediately. No reconciliation needed — just apply the current period's intent.

**Safety-critical flows** (locks, security): read retained period on startup, query actual device state, reconcile if misaligned or prompt for confirmation. The scheduler publishes truth; consuming flows own reconciliation.

### Midnight Event Payload

```json
{
  "timestamp": "2026-03-23T00:00:00.000Z",
  "source": "scheduler",
  "task": "midnight"
}
```

### Period Event/State Payload

```json
{
  "period": "evening",
  "timestamp": "2026-03-23T19:47:12.000Z",
  "source": "scheduler"
}
```

### Notes

- `send_state` to schedex nodes dispatched via dynamic `link call` — each schedex node is a named `Link In` target (`Daytime`, `Evening`, `Overnight`)
- Spreading a string payload with `{...msg.payload}` produces a character-indexed object — always pass string payloads directly as `msg.payload`
- `Prepare Fixed` sets `node.status()` on every midnight fire for "last fired" visibility in the editor
- Midnight cron uses Node-RED's 5-field format: `"00 00 * * *"` (minute hour day month weekday)

---

## Open Questions

- [x] ~~Pub/sub subflow implementation details~~ → **Flow Registration pattern; area-level targeting, device-level ACKs**
- [x] ~~Logging persistence destination~~ → **JSONL files, daily rotation, unified log with `system` field**
- [x] ~~Mobile notification channel selection~~ → **HA Companion primary; explicit multi-channel via `channels` field; no implicit failover**
- [x] ~~Should ERROR-level logs also auto-notify, or only CRITICAL?~~ → **CRITICAL only; escalation is flow responsibility**
- [x] ~~Device Registry storage location~~ → **External JSON file, loaded to global.config.deviceRegistry**
- [x] ~~Device Registry population~~ → **Manual with discovery validation (log/notify discrepancies, don't block)**
- [x] ~~Where to surface "devices needing batteries" data~~ → **Daily Digest email + immediate notifications for critical; dashboard deferred**
- [x] ~~ACK pattern design~~ → **Centralized ACK Tracker utility flow**
- [x] ~~Health monitoring approach~~ → **Each service self-reports + Node-RED edge checks + HA edge checks + Healthchecks.io**
- [x] ~~Startup sequencing / race conditions~~ → **Initializer Latch subflow**
- [x] ~~HA connection state detection~~ → **`status` node pattern; `connections.home_assistant` and `connections.mqtt` global flags; startup settling window; `Utility: Connections` flow**
- [x] ~~Notification routing when HA is down~~ → **No implicit failover; sender specifies channels explicitly; Connection Gate handles per-channel availability**
- [x] ~~Connection-aware message routing~~ → **`Connection Gate` subflow; OUTPUT_1 = connected, OUTPUT_2 = unrecovered; RETENTION_MS drives hold-and-retry**
- [x] ~~Notification recipient/channel model~~ → **Namespaced `targets` array; `namespace.key.channel` format; `*` wildcard; `people` and `areas` namespaces; each delivery group owns its own routing logic**
- [x] ~~Utility: Notifications~~ → **Built and tested; HA Companion delivery, Connection Gate, person lookup, clear notification path**
- [x] ~~Fan-out routing pattern~~ → **`link call` with dynamic `msg.target`; `resolveLinkTarget()` maps channel keys to `Link In` node names; delivery groups return via `link out` (return mode); catch node handles timeouts**
- [x] ~~Namespaced target addressing~~ → **Implemented; `Build Targets` and `Build Clear Call` both use namespace resolver with wildcard; `notifications.json` has `people` and `areas` sections**
- [x] ~~Television Delivery group~~ → **Built; HA state lookup, source → endpoint type resolution, dispatches to Android TV or WebOS via `link call`; unknown source and TV-off both log WARN and drop**
- [x] ~~Android TV Delivery group~~ → **Built; `nfandroidtv` via HA; severity maps to duration/color/interrupt; clears are no-ops**
- [x] ~~WebOS Delivery group~~ → **Built; WebOS notify via HA; clears are no-ops**
- [ ] **Echo Show / View Assist** — LineageOS Echo Show devices running View Assist; determine whether HA registers them as `mobile_app_*` (→ `ha_companion` channel, no new plumbing) or as Android TV devices (→ `android_tv` endpoint type); add to `notifications.json` accordingly after setup
- [ ] **Voice notifications** — Completely separate from visual notifications; different payload schema (`tts_text`, target speaker, voice/language, volume, interruptible vs queued); publish to `highland/event/speak`; handled by a future `Utility: Voice` flow; callers may publish to both `highland/event/notify` and `highland/event/speak` independently when both visual and spoken delivery is desired
- [ ] **Action responses** — deferred until actionable notifications are implemented
- [x] ~~Utility: Scheduler~~ → **Built and tested; three solar/fixed periods via schedex, midnight task event, startup recovery via send_state, retained state + non-retained events**
- [x] ~~Utility: Battery Monitor~~ → **Built and tested; Z2M wildcard subscription, device_catalog.json pattern, graceful degradation, auto-derived friendly names, 24hr critical repeat timer with restart recovery**
- [x] ~~Utility: Calendaring~~ → **Built and tested; OAuth 2.0, three Google sub-calendars (Appointments/Reminders/Trash & Recycling), 15-min poll, manual reload via MQTT command, rolling 7-day snapshot published to `highland/state/calendar/snapshot`**
- [x] ~~Utility: Weather Alerts~~ → **Built and tested; NWS alerts endpoint, 30-second poll, alert lifecycle tracking (new/updated/expired), severity-coded notifications with NWS deep link, `highland/state/weather/alerts` retained snapshot**

---

*Last Updated: 2026-03-25*
