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

---

## Flow Organization

### Flow Types

| Type | Purpose | Examples |
|------|---------|----------|
| **Area flows** | Own devices and automations for a physical area | `Garage`, `Living Room`, `Front Porch` |
| **Utility flows** | Cross-cutting concerns that transcend areas | `Scheduler`, `Security`, `Notifications`, `Logging`, `Backup`, `ACK Tracker`, `Battery Monitor`, `Health Monitor`, `Config Loader`, `Daily Digest` |

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

**Benefits:**
- Each group is a logical unit with a clear purpose
- Link nodes connect groups without spaghetti wires
- Flow reads top-to-bottom or left-to-right in sections
- Minimizes horizontal scrolling

---

## Subflows

### Use Sparingly, For Truly Reusable Components

**Good candidates for subflows:**
- `Await Initializers` — two-condition gate check with retry loop (see Startup Sequencing)
- Common transformations used identically across many flows

**Not good candidates:**
- Flow-specific logic (keep it visible)
- One-off utilities (just use a function node)
- Anything that hides important business logic

### Await Initializers Subflow

Reusable subflow that gates flow execution until the `initializers` context store is ready. Drop this into any flow's startup sequencing group after the echo probe returns.

**Inputs:** 1 (triggered by echo probe return)
**Outputs:** 2 — output 1: ready, output 2: timed out

**Environment variables (configurable per instance):**
- `RETRY_INTERVAL_MS` — delay between retries in milliseconds (default: 250)
- `MAX_RETRIES` — maximum retry attempts before timeout (default: 20)

Total timeout window at defaults: 250ms × 20 = 5 seconds.

The calling flow is responsible for handling both outputs:
- **Output 1 (ready):** open the gate, process buffered state
- **Output 2 (timed out):** set `flow.degraded = true`, clear buffer, log CRITICAL, set error node status

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

const formatStatus = global.get('utils.formatStatus', 'initializers');
node.status({ fill: 'green', shape: 'dot', text: formatStatus(`Registered: ${flowIdentity.devices.length} devices`) });

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
  3. Publish probe to highland/command/nodered/init_probe/{flow_name} (non-retained)
  4. Retained messages begin arriving → buffer or discard (gate is closed)
  5. Own probe returns → retained messages are done
     → Call Await Initializers subflow
     → If initializers.ready = true:  gate opens immediately
     → If initializers.ready = undefined: subflow polls with retry/timeout
  6. Gate opens → process buffered state → normal operation begins
```

### Why Non-Retained for the Ready Signal

Using a retained message for `highland/status/initializers/ready` introduces a stale session problem — a flow restarting at 09:00:02a would see the retained message from the previous session's 08:00:05a initialization and incorrectly open its gate before the current session's utilities are ready.

Using a **global flag** in the `initializers` memory store solves this cleanly:

- If Initializers runs first → sets `global('initializers.ready', true, 'initializers')` → `Await Initializers` subflow finds the flag immediately
- If a dependent flow starts first → `Await Initializers` polls until the flag is set or times out
- On restart → the `initializers` store clears (memory backend resets) → flag is gone → stale ready state is structurally impossible

### Initializers Startup Sequence

`Utility: Initializers` follows this sequence on every startup:

```javascript
// Step 1: Populate all utility functions
global.set('utils.formatStatus', function(text) { ... }, 'initializers');
// ... other utilities ...

// Step 2: Set the ready flag
global.set('initializers.ready', true, 'initializers');

// Step 3: Update node status
const formatStatus = global.get('utils.formatStatus', 'initializers');
node.status({ fill: 'green', shape: 'dot', text: formatStatus('READY') });
```

The `initializers` store is a memory backend — it resets on every Node-RED restart. There is no stale state from previous sessions.

### Await Initializers Subflow

The `Await Initializers` subflow encapsulates the readiness check with a retry loop:

```javascript
// Check function (runs on each attempt)
const maxRetries = env.get('MAX_RETRIES') || 20;
let retries = flow.get('retries') || 0;

if (global.get('initializers.ready', 'initializers') === true) {
    flow.set('retries', 0);
    return [msg, null];  // output 1: ready
}

if (retries >= maxRetries) {
    flow.set('retries', 0);
    msg.timedOut = true;
    return [null, msg];  // output 2: timed out
}

flow.set('retries', retries + 1);
return [null, msg];  // continue retry loop
```

A Switch node routes `msg.timedOut = true` to subflow output 2 (error path), otherwise to a Delay node that feeds back into the Check function. The Delay node reads `RETRY_INTERVAL_MS` from env (default 250ms).

### Gate Pattern in Sinks Groups

The gate check belongs in the Sinks group — at the point of ingress — before messages reach any processing logic. The gate handles three states:

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

### Degraded State and Recovery

When the `Await Initializers` subflow times out, the calling flow sets `flow.set('degraded', true)`. When entering the degraded state, the failure handler should also clear the buffer since those messages will never be processed:

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

---

## Error Handling

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

A single daily log file contains entries from ALL systems — Node-RED, Z2M, Z-Wave JS, HA, watchdog, etc.

**Log entry structure:**

| Field | Purpose | Examples |
|-------|---------|----------|
| `timestamp` | When it happened | `2025-02-24T10:00:00Z` |
| `system` | Which system generated the log | `node_red`, `ha`, `z2m`, `zwave_js`, `watchdog` |
| `source` | Component within that system | `garage`, `scheduler`, `coordinator` |
| `level` | Severity | `VERBOSE`, `DEBUG`, `INFO`, `WARN`, `ERROR`, `CRITICAL` |
| `message` | Human-readable description | `Failed to turn on carriage lights` |
| `context` | Structured additional data | `{"device": "...", "error": "..."}` |

### Log Levels

| Level | Value | When to Use |
|-------|-------|-------------|
| `VERBOSE` | 0 | Granular trace; active debugging only |
| `DEBUG` | 1 | Detailed info useful for troubleshooting |
| `INFO` | 2 | Normal operational events worth recording |
| `WARN` | 3 | Something unexpected but not broken |
| `ERROR` | 4 | Something failed but flow continues |
| `CRITICAL` | 5 | Catastrophic failure; intervention needed |

### Node Status Color Coding

Log-emitting function nodes use color-coded status to reflect the level of the last processed entry:

| Level | Fill | Shape | Rationale |
|-------|------|-------|----------|
| VERBOSE | grey | dot | Low signal |
| DEBUG | grey | dot | Low signal |
| INFO | green | dot | Normal |
| WARN | yellow | dot | Attention |
| ERROR | red | dot | Failed |
| CRITICAL | red | ring | Urgent — ring is more visually distinct than dot |

Status text uses `utils.formatStatus` for consistent formatting: `"{LEVEL} at: {TIME}"`

### Log Event Topic

```
highland/event/log
```

All systems publish here. QoS 2 for guaranteed delivery.

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

### Logging Utility Flow (`Utility: Logging`)

Centralized flow that:
1. Subscribes to `highland/event/log` (QoS 2)
2. Normalizes malformed entries rather than dropping them
3. Appends to today's JSONL file (`/var/log/highland/highland-YYYY-MM-DD.jsonl`)
4. If level = `CRITICAL` → forwards to `highland/event/notify`

**Groups:** Sinks, Log Writer, Critical Forwarder, Error Handling, Test Cases

**File node configuration:** `appendNewline: true`, `createDir: true`, filename from `msg.filename`

### Querying Logs

```bash
# All errors from any system
jq 'select(.level == "ERROR")' highland-2025-02-24.jsonl

# All Node-RED entries
jq 'select(.system == "node_red")' highland-2025-02-24.jsonl

# Last 10 entries
tail -10 highland-2025-02-24.jsonl | jq '.'
```

### Auto-Notify Behavior

**Only CRITICAL logs auto-notify.** ERROR and below do not. Escalation is the responsibility of the flow that owns the logic.

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

**File:** `/home/nodered/config/device_registry.json`

**Access:** `global.get('config.deviceRegistry')`

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

### Config Loader Utility Flow (`Utility: Config Loader`) ✅

Loads all config files into global context at startup.

**Triggers:** startup inject (0.5s delay), manual inject, `highland/command/config/reload`, `highland/command/config/reload/{name}`

**Stores under:** `global.config.{name}` (e.g. `global.config.secrets`, `global.config.thresholds`)

**Publishes:** retained status to `highland/status/config/loaded`

---

## Command Dispatcher

### Purpose

Translate high-level commands into protocol-specific MQTT messages. Flows say *what* they want; the dispatcher knows *how*.

### Common Actions (v1)

| Action | Applies To |
|--------|------------|
| `on` | lights, switches |
| `off` | lights, switches |
| `toggle` | lights, switches |
| `brightness` | dimmable lights |
| `lock` | locks |
| `unlock` | locks |
| `raw` | any (passthrough) |

---

## ACK Tracker Utility Flow

### Purpose

Centralized tracking of acknowledgment requests. Keeps ACK bookkeeping out of individual flows.

### Topics

| Topic | Purpose | Publisher |
|-------|---------|-----------|
| `highland/ack/register` | Register expectation for ACKs | Requesting flow |
| `highland/ack` | ACK responses | Responding flows |
| `highland/ack/result` | Outcome after timeout | ACK Tracker |

### Payloads

**Registration:**
```json
{
  "correlation_id": "abc123",
  "expected_sources": ["foyer_entry_door", "garage_entry_door"],
  "timeout_seconds": 30,
  "source": "security"
}
```

**Result:**
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

### Battery States

| State | Threshold | Notification |
|-------|-----------|--------------|
| `normal` | > 35% | None |
| `low` | 35% - 15% | Normal priority, once |
| `critical` | < 15% | High priority, repeats every 24h |

### Notification Behavior

| Transition | Action |
|------------|--------|
| normal → low | Notify once, normal priority |
| low → critical | Notify immediately, high priority; start 24hr repeat |
| critical → any | Cancel repeat; notify recovery |
| low → normal | Silent recovery |

---

## Notification Framework

### Notification Topic

```
highland/event/notify
```

### Severity Levels

| Severity | DND Override | Use Case |
|----------|--------------|----------|
| `low` | No | Informational |
| `medium` | No | Worth knowing soon |
| `high` | Yes | Needs attention now |
| `critical` | Yes | Emergency |

### Android Notification Channels

| Channel ID | DND Override |
|------------|--------------|
| `highland_low` | No |
| `highland_default` | No |
| `highland_high` | Yes |
| `highland_critical` | Yes |

---

## Health Monitoring

### Overview

Node-RED pings Healthchecks.io directly via HTTP for each service it monitors. No watchdog script needed — direct HTTP from Node-RED proves liveness independently of MQTT state.

### Status Values

| Status | Meaning |
|--------|---------|
| `healthy` | Responding AND all metrics within acceptable ranges |
| `degraded` | Responding BUT one or more metrics in warning territory |
| `unhealthy` | Not responding OR critical threshold exceeded |

### Healthchecks.io Configuration

| Check | Poll Frequency | Grace Period | Notes |
|-------|----------------|--------------|-------|
| Node-RED | 30 sec | 3 min | Pings every 30s; 3 min grace = ~6 missed pings |
| MQTT | 1 min | 3 min | 3 missed checks before alert |
| Z2M | 1 min | 3 min | 3 missed checks before alert |
| Z-Wave JS | 1 min | 3 min | 3 missed checks before alert |
| HA | 5 min | 15 min | 3 missed checks before alert |

> **Note:** Healthchecks.io only supports whole-minute grace periods. 3 minutes is the practical minimum for 30s/1min ping intervals.

### Watchdog Script

The original watchdog design (cron script subscribing to Node-RED's MQTT heartbeat) is superseded by direct HTTP pinging. Whether a watchdog script has a remaining role will be evaluated per-service as the Health Monitor is built.

---

## Daily Digest

**Trigger:** Midnight via Scheduler + 5 second delay

**Content:** Calendar events (next 24-48h), weather summary, battery status, system health

**Implementation:** Markdown → HTML via `node-red-node-markdown` → SMTP email

---

## Open Questions

- [x] ~~Pub/sub subflow implementation details~~ → **See Flow Registration pattern**
- [x] ~~Logging persistence destination~~ → **JSONL files, daily rotation**
- [x] ~~Mobile notification channel selection~~ → **HA Companion App (Android)**
- [x] ~~Should ERROR-level logs also auto-notify, or only CRITICAL?~~ → **CRITICAL only**
- [x] ~~Device Registry storage location~~ → **External JSON file, loaded to global.config.deviceRegistry**
- [x] ~~Device Registry population~~ → **Manual with discovery validation**
- [x] ~~Where to surface "devices needing batteries" data~~ → **Daily Digest + immediate notifications**
- [x] ~~ACK pattern design~~ → **Centralized ACK Tracker utility flow**
- [x] ~~Health monitoring approach~~ → **Node-RED direct HTTP pings to Healthchecks.io**
- [x] ~~Startup sequencing / race conditions~~ → **Two-condition gate: echo probe + Await Initializers subflow**
- [x] ~~Global utility functions~~ → **`initializers` memory context store; re-populated by Utility: Initializers on every startup**

---

*Last Updated: 2026-03-18*
