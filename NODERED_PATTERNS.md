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

Node-RED context is configured with three named stores:

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

// Volatile store — timer handles, non-serializable values
flow.set('my_timer', timerHandle, 'volatile');
const timer = flow.get('my_timer', 'volatile');
```

The store name in `global.get` / `global.set` is what makes the naming self-documenting — seeing `'volatile'` tells you the value is transient, `'initializers'` tells you where it was defined.

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
- Gates — connection-aware routing (see Connection Gate)
- Common transformations used identically across many flows

**Not good candidates:**
- Flow-specific logic (keep it visible)
- One-off utilities (just use a function node)
- Anything that hides important business logic

### Latch Pattern

Latches are subflows that gate message flow until some condition is met. The naming convention is `{Condition} Latch` — e.g. `Initializer Latch`.

All latches share the same interface:
- **1 input** — any message from any source (inject, MQTT, HTTP, etc.)
- **Output 1 (OK)** — messages pass through once condition is met; buffered messages drain in order
- **Output 2 (TIMEOUT)** — single signal message when condition is never met within retry window

### Initializer Latch

Gates flow execution until `Utility: Initializers` has populated the `initializers` context store. Drop this into any flow's startup sequencing group.

**Environment variables (configurable per instance):**
- `RETRY_INTERVAL_MS` — delay between retries in milliseconds (default: 250)
- `MAX_RETRIES` — maximum retry attempts before timeout (default: 20)
- `CONTEXT_PREFIX` (UI label: **Scope**) — prefix for flow context keys; required when multiple instances on the same flow tab

Total timeout at defaults: 250ms × 20 = 5 seconds.

**Internal behavior:**
- Every incoming message is buffered immediately
- On first message, starts polling `global.get('initializers.ready', 'initializers')`
- If flag is `true` → sets `flow.{prefix}initialized = true`, clears degraded, drains buffer via Output 1
- If max retries exceeded → sets `flow.{prefix}degraded = true`, discards buffer, sets red ring node status on the subflow instance, emits signal via Output 2
- If already initialized → passes message through Output 1 directly (no buffering)
- If already degraded → drops message silently

**Output 2 is optional** — the subflow instance shows a red ring on timeout regardless of whether Output 2 is wired. Wire Output 2 only when programmatic handling of the failure is needed. For most flows, visual feedback is sufficient.

**Degraded state cause:** Always a bug in `Utility: Initializers` — introduced by a deploy. Not a random runtime failure. Recovery: fix Initializers, redeploy Initializers, redeploy affected flows.

---

## Flow Registration

### Purpose

Each area flow self-registers its identity and owned devices. This creates a queryable global registry that enables:
- Targeting messages by area
- Looking up devices by capability
- Knowing which area owns which device

### Storage

| Storage | Persistence | Purpose |
|---------|-------------|---------|
| `flow.identity` | Disk | This flow's identity and devices |
| `global.flowRegistry` | Disk | All flows' registrations |
| `global.config.deviceRegistry` | Disk | Device details (single source of truth for capabilities) |

### Registration Boilerplate

```javascript
const flowIdentity = {
  area: 'foyer',
  devices: ['foyer_entry_door', 'foyer_environment']
};
flow.set('identity', flowIdentity);
const registry = global.get('flowRegistry') || {};
registry[flowIdentity.area] = { devices: flowIdentity.devices };
global.set('flowRegistry', registry);
node.status({ fill: 'green', shape: 'dot', text: `Registered: ${flowIdentity.devices.length} devices` });
return msg;
```

---

## Startup Sequencing

### The Problem

On Node-RED startup or deploy, three things can conflict:

1. MQTT subscriptions deliver retained messages immediately
2. Config Loader and flow context restoration may still be in progress
3. `Utility: Initializers` may not have finished populating the `initializers` store

Node-RED makes no startup ordering guarantees between inject nodes across flows.

### Solution: Initializer Latch

The `Initializer Latch` subflow handles startup sequencing. Every flow that uses utilities from the `initializers` store should gate its startup path through an Initializer Latch instance.

### Bootstrapping Limitation

You cannot use infrastructure to report infrastructure failures. If MQTT is unavailable on startup:
- **Node-RED debug sidebar** — node status visible in editor regardless of MQTT
- **Node-RED console log** — `node.error()` writes to internal log, visible via `docker compose logs nodered`
- **Healthchecks.io** — Health Monitor pings via direct HTTP independently of MQTT

### Degraded State Recovery

Root cause is always in `Utility: Initializers`. Steps:
1. Fix the issue in `Utility: Initializers`
2. Deploy `Utility: Initializers`
3. Redeploy affected flows (required to reset persisted flow context)

---

## Error Handling

### Two-Tier Approach

1. **Targeted handlers** — Catch errors in specific groups where custom handling is needed
2. **Flow-wide catch-all** — Single Error node per flow catches anything unhandled, dispatches to `highland/event/log`

---

## Logging Framework

### Concept

Logging answers: *"How important is this for troubleshooting/audit?"* Separate from notifications, though CRITICAL logs auto-forward to `highland/event/notify`.

### Log Storage

**Format:** JSONL (JSON Lines) — one JSON object per line
**Location:** `/var/log/highland/highland-YYYY-MM-DD.jsonl`
**Rotation:** Daily, retain 30 days via cron

### Log Entry Structure

| Field | Purpose | Examples |
|-------|---------|----------|
| `timestamp` | When it happened | `2025-02-24T10:00:00Z` |
| `system` | Which system | `node_red`, `ha`, `z2m`, `zwave_js` |
| `source` | Component within system | `garage`, `connections` |
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

### Auto-Notify Behavior

**CRITICAL only** auto-notifies. Escalation beyond that is the responsibility of the flow.

### Querying Logs

```bash
jq 'select(.level == "ERROR")' highland-2025-02-24.jsonl
jq 'select(.source == "connections")' highland-2025-02-24.jsonl
tail -10 highland-2025-02-24.jsonl | jq '.'
```

---

## Device Registry

**File:** `/home/nodered/config/device_registry.json`
**Access:** `global.get('config').deviceRegistry`

Centralized knowledge about devices — protocol, topic structure, capabilities, battery metadata. Abstracts Z2M vs Z-Wave JS differences so flows don't need protocol details.

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
  "daily_digest": {
    "recipients": ["joseph@example.com"],
    "enabled": true
  },
  "defaults": {
    "admin_only": ["joseph"],
    "all": ["joseph", "spouse"]
  }
}
```

**Person-centric model:** Each person has an `admin` flag and a `channels` map of channel name → delivery address. When a new notification channel is added (e.g. Pushover, Telegram), add an entry under each person's `channels` object.

**`admin` flag** — Determines whether a person receives administrative notifications (system health, infrastructure alerts).

**`defaults`** — Named recipient lists for convenience. Callers copy these values into the payload `recipients` field explicitly — there is no implicit defaulting.

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

### Config Loader (`Utility: Config Loader`)

Loads all config files into `global.config` at startup and on reload commands (`highland/command/config/reload`).

---

## Notification Framework

### Concept

Notifications answer: *"How urgently does a human need to know about this?"* Separate from logging.

### Notification Topic

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
|-------|----------|--------------|
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

To programmatically dismiss a notification:

```yaml
service: notify.mobile_app_joseph_phone
data:
  message: "clear_notification"
  data:
    tag: "lockdown_20250224_2200"
```

**Use case:** Battery critical notification auto-clears when battery recovers.

### Action Responses

When user taps a notification action, HA fires `mobile_app_notification_action`. The Notification Utility normalizes and publishes to `highland/event/notify/action_response`. Action responses are a channel-specific feature (HA Companion only at present) — full design deferred until actionable notifications are implemented.

### Future Channels (Deferred)

Telegram is a strong candidate — rich features, HA-independent, two-way interaction. Each new channel adds an entry to each person's `channels` object in `notifications.json` and a new delivery group in the Notification Utility flow. The MQTT payload schema does not change.

---

## Utility: Connections

### Purpose

Tracks the live state of external service connections and exposes that state via global context for any flow that needs to make runtime decisions based on it. Distinct from `Utility: Health Checks` — Health Checks *reports* infrastructure health outward; Connections *exposes* connection state inward.

### Detection Mechanism

Uses Node-RED's built-in `status` node scoped to a connection-bearing node. Fires immediately on connection state change — no polling, no second connection, no additional palette dependencies.

**Signal mapping (via `msg.status.fill`):**
- `'red'` or `'yellow'` → `connections.{key} = 'down'`
- anything else → `connections.{key} = 'up'`

Works with any connection-bearing node — MQTT in/out, HA palette nodes, etc.

### Startup Settling

On restart, connections briefly drop before re-establishing, generating spurious log entries. The flow uses a **startup settling window**:

- `Startup Tasks` group fires on startup, sets `flow.timer_cadence` and starts a `setTimeout` that sets `flow.settled = true` in `volatile` store
- `Evaluate State` nodes read `flow.get('settled', 'volatile')` to gate logging

**During window (not settled):** `'down'` transitions start a debounce timer; `'up'` transitions cancel it silently.
**After window (settled):** All transitions logged immediately as real-time events.

**Single cadence value** — `flow.timer_cadence` drives both settling window and debounce timers. Change in one place in `Establish Cadence`.

### MQTT Availability — The Catch-22

When MQTT is down, the normal log path is unavailable. `State Change Logging` handles this:

```
Log Event link in → MQTT Available? switch (global.connections.mqtt == 'up')
    ↓ up                                         ↓ else
Format Log Message → MQTT out             Log to Console (node.error/warn)
```

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
const haAvailable = global.get('connections.home_assistant') !== 'down';
const mqttAvailable = global.get('connections.mqtt') !== 'down';
```

The `!== 'down'` guard handles startup case — `undefined !== 'down'` is `true`, defaulting to available.

### Future Additions

Additional flags follow the same pattern — `status` node scoped to any connection-bearing node, same `Evaluate State` structure with `NAME` and `KEY` changed, wired into shared `State Change Logging` group.

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
| `RETENTION_MS` | Retention (ms) | How long to poll for recovery before routing to Output 2. 0 = route to Output 2 immediately | `0` |
| `CONTEXT_PREFIX` | Scope | Prefix for flow context keys — required when multiple instances on the same flow tab | `''` |

### Behavior

| Scenario | Result |
|----------|--------|
| Connection up | Output 1 immediately |
| Connection down, `RETENTION_MS` = 0 | Output 2 immediately |
| Connection down, `RETENTION_MS` > 0, recovers within window | Output 1 when recovery detected |
| Connection down, `RETENTION_MS` > 0, window expires | Output 2 after window |

**Output 1 always means connected state** — either immediately or after recovery within the window. **Output 2 always means unrecovered down state** — the message is guaranteed to exit one output or the other, no silent drops.

**Latest-only retention** — if a new message arrives while a retention poll is in progress, the existing poll is cancelled and a new one starts. Earlier messages are discarded.

**Poll interval** is internalized at 500ms — not configurable per instance.

### Internal Structure

**Evaluate Gate** — single function node with two outputs. Reads `global.connections.{CONNECTION_TYPE}`, applies retention policy, sets `node.status()` for each decision path.

**Status Monitor** — `status` node scoped to `Evaluate Gate`, wired to `Set Status` function node, wired to subflow status output. Surfaces `Evaluate Gate` status onto the subflow instance in the parent flow without requiring a third output on `Evaluate Gate`.

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

## Health Monitoring

### Philosophy

Treat this as a line-of-business application. Degradation detection is as important as outage detection.

### Single Point of Failure Problem

Each service self-reports its own liveness independently of Node-RED.

**Healthchecks.io naming convention:**
- `{Service}` — service's own self-report
- `Node-RED / {Service} Edge` — Node-RED's connection check
- `Home Assistant / {Service} Edge` — HA's connection check

**Current implementation status:**
- `Node-RED` ✅ | `Home Assistant` ✅ | `Communications Hub` ✅
- `Node-RED / Home Assistant Edge` ✅ | `Node-RED / MQTT Edge` ✅
- `Node-RED / Zigbee Edge` ✅ | `Node-RED / Z-Wave Edge` ✅
- `Home Assistant / Zigbee Edge` ✅ | `Home Assistant / Z-Wave Edge` ✅

### Check Frequency

All checks: 1 minute period, 3 minute grace period.

---

## Daily Digest

**Trigger:** Midnight + 5 second delay
**Content:** Calendar (next 24–48h), weather, battery status, system health
**Implementation:** Markdown → HTML → SMTP email

---

## Open Questions

- [x] ~~Pub/sub subflow implementation details~~ → **Flow Registration pattern**
- [x] ~~Logging persistence destination~~ → **JSONL files, daily rotation**
- [x] ~~Mobile notification channel selection~~ → **HA Companion primary; explicit multi-channel via `channels` field; no implicit failover**
- [x] ~~Should ERROR-level logs also auto-notify?~~ → **CRITICAL only**
- [x] ~~Device Registry storage~~ → **External JSON, global.config.deviceRegistry**
- [x] ~~ACK pattern design~~ → **Centralized ACK Tracker**
- [x] ~~Health monitoring approach~~ → **Each service self-reports + Node-RED edge checks + HA edge checks + Healthchecks.io**
- [x] ~~Startup sequencing / race conditions~~ → **Initializer Latch subflow**
- [x] ~~HA connection state detection~~ → **`status` node pattern; `connections.home_assistant` and `connections.mqtt` global flags; startup settling window; `Utility: Connections` flow**
- [x] ~~Notification routing when HA is down~~ → **No implicit failover; sender specifies channels explicitly; Connection Gate handles per-channel availability**
- [x] ~~Connection-aware message routing~~ → **`Connection Gate` subflow; OUTPUT_1 = connected, OUTPUT_2 = unrecovered; RETENTION_MS drives hold-and-retry behavior**
- [x] ~~Notification recipient/channel model~~ → **Person-centric config; `channels` and `recipients` both required; graceful degradation per channel adapter; missing address → WARN log**
- [ ] **Utility: Notifications** — build out flow with HA Companion delivery, Connection Gate, person lookup
- [ ] **Action responses** — design deferred until actionable notifications are implemented
- [ ] **Utility: Scheduler** — period transitions and task events

---

*Last Updated: 2026-03-20*
