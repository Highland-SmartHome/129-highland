# Event Architecture — Node-RED Automation Framework

## Overview

Event-driven architecture for inter-flow communication using MQTT as the message transport. Publishers emit facts (not commands); consumers decide relevance and action.

---

## Core Principles

1. **Publishers emit facts, not commands**
   - ✓ "It's 30 minutes before sunset" (`evening`)
   - ✗ "Turn on exterior lights"

2. **Consumers decide relevance**
   - Each flow subscribes to events it cares about
   - Each flow decides what action to take (if any)

3. **Loose coupling**
   - Publishers have zero knowledge of subscribers
   - Flows can be added/removed without changing publishers

4. **Async acknowledgment when needed**
   - Critical flows can request confirmation via response events
   - ACKs reference a `correlation_id` from the original event

---

## Flow Types

### Area Flows
- Named after physical areas: `Garage`, `Living Room`, `Front Yard`
- Own their devices and know how to control them
- Subscribe to events and decide local actions
- Publish area-level semantic events (e.g., `presence_detected`)

### Utility Flows
- Functional concepts that transcend areas: `Scheduler`, `Security`, `Camera Monitor`
- Orchestrate cross-cutting concerns
- Publish global events (e.g., `evening`, `lockdown`)

---

## Scheduler Periods

Three core periods define the house's daily rhythm. These are retained messages — any flow can check current period on startup.

| Event | Trigger | Purpose |
|-------|---------|---------|
| `highland/event/scheduler/day` | Sunrise | House wakes up. Exterior lights off. |
| `highland/event/scheduler/evening` | 30 min before sunset | House winds down. Exterior lights on (100%). |
| `highland/event/scheduler/overnight` | 10:00 PM (fixed) | House goes to sleep. Exterior dims, ancillary off. |

**Naming rationale:**
- `overnight` rather than `night` — implies continuity through early morning until `day`
- Avoids collision with schedex astronomical/nautical constants
- Period-based (what time it is), not action-based (what to do)

**Retention:** All period events are retained. A restarting flow receives the current period immediately.

```
┌────────────────────────────────────────────────────────────────┐
│                                                                │
│   day ──────────────► evening ──────────────► overnight ───┐   │
│    ▲                                                       │   │
│    └───────────────────────────────────────────────────────┘   │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

---

## Scheduled Event Categories

The scheduler emits two distinct types of events:

### Period Events (Shared, Abstract)

| Characteristic | Description |
|----------------|-------------|
| **Purpose** | Represent what phase the house is in |
| **Retention** | Retained — current period always known |
| **Subscribers** | Multi-purpose — many flows may react |
| **Naming** | Abstract, phase-based (`day`, `evening`, `overnight`) |

Period events answer: *"What time of day is it, semantically?"*

Any flow can subscribe and decide its own response. When writing a new flow, these are your go-to for time-based behavior that aligns with house rhythms.

### Task Events (Bespoke, Purpose-Driven)

| Characteristic | Description |
|----------------|-------------|
| **Purpose** | Trigger a specific action at a specific time |
| **Retention** | Not retained — point-in-time trigger |
| **Subscribers** | Single-purpose — typically one flow cares |
| **Naming** | Descriptive, action-based (`backup_daily`, `health_check_hourly`) |

Task events answer: *"Is it time to do this specific job?"*

These have a named purpose. You wouldn't subscribe to `backup_daily` unless you're doing something related to backups.

### Defined Task Events

| Event | Trigger | Purpose |
|-------|---------|---------|
| `highland/event/scheduler/backup_daily` | TBD (e.g., 3:00 AM) | Trigger backup utility flow |
| `highland/event/scheduler/digest_daily` | Midnight | Trigger daily digest email |

*Additional task events added as needed.*

### When to Create a New Event

| Scenario | Approach |
|----------|----------|
| "I need to do something at sunset-ish" | Subscribe to `evening` |
| "I need to do something at a specific time unrelated to house rhythm" | Create a task event |
| "I need to do something 15 minutes after overnight starts" | Subscribe to `overnight`, add delay in flow |
| "Multiple unrelated flows need a 3am trigger" | Consider if it's actually a period, or create shared task event |

---

## Topic Structure

### Naming Convention
- **Style:** lowercase with underscores (e.g., `living_room`, `front_porch`)
- **Rationale:** Readable, straightforward, consistent

### Base Pattern
```
highland/event/{source}/{event_type}[/{entity}]
```

- `highland/` — Namespace for home automation (matches FQDN: `highland.ferris.network`)
- `{source}` — Area or utility name (lowercase, underscores)
- `{event_type}` — What happened (lowercase, underscores)
- `{entity}` — Optional, when consumers need entity-level granularity

*Note: `highland/` prefix reserves MQTT namespace for home automation, allowing other non-HA uses of the broker under different prefixes.*

### Examples

**Scheduled/Temporal Events (Utility → All)**
```
highland/event/scheduler/day              # sunrise
highland/event/scheduler/evening          # 30 min before sunset
highland/event/scheduler/overnight        # 10pm
```

**State Events (Utility → All)**
```
highland/event/security/lockdown          # house lockdown initiated
highland/event/security/away              # house set to away mode
```

**Area Semantic Events (Area → Interested Parties)**
```
highland/event/living_room/presence_detected
highland/event/garage/motion_detected
highland/event/front_door/opened
```

**Entity-Level Events (When entity matters to consumers)**
```
highland/event/basement/leak/sensor_washer
highland/event/kitchen/leak/sensor_dishwasher
highland/event/garage/motion/sensor_driveway
```

**Logging Events**
```
highland/event/log                            # All log entries (level in payload)
```

**Notification Events**
```
highland/event/notify                         # All notifications (severity in payload)
```

**ACK Events (See ACK Tracker in NODERED_PATTERNS.md)**
```
highland/ack/register                         # Register ACK expectations
highland/ack                                  # ACK responses (correlation_id in payload)
highland/ack/result                           # ACK results after timeout
```

**Status/Health Events**
```
highland/status/{service}/heartbeat           # Simple "I'm alive" ping
highland/status/{service}/health              # Detailed health + metrics

highland/status/node_red/heartbeat
highland/status/mqtt/health
highland/status/z2m/health
highland/status/zwave/health
highland/status/ha/health
```

**Command Topics**
```
highland/command/{target}/{action}            # Request an action from a service

highland/command/config/reload                # Reload all config files
highland/command/config/reload/{config_name}  # Reload specific config file
highland/command/backup/trigger               # Trigger backup on receiving host
highland/command/backup/trigger/{host}        # Trigger backup on specific host
```

Command topics are imperative (unlike events which are declarative facts). Services subscribe to commands they can handle.

---

## Wildcard Subscription Patterns

| Pattern | Use Case |
|---------|----------|
| `highland/event/scheduler/#` | Subscribe to all scheduled events |
| `highland/event/+/leak/#` | Any leak in any area (close main valve) |
| `highland/event/+/motion_detected` | Any motion in any area |
| `highland/event/garage/#` | Everything from garage |
| `highland/status/#` | All health/heartbeat events |
| `highland/status/+/health` | All health status (not heartbeats) |
| `highland/command/backup/#` | All backup commands |
| `highland/command/config/#` | All config reload commands |

---

## Message Payload

### Design Philosophy
- **Consumers care about the epoch, not the mechanics**
- Core payload is minimal and consistent
- Extended information is optional and unstructured
- Consumers can ignore `meta` entirely if they don't need it

### Standard Event Payload
```json
{
  "timestamp": "2025-02-23T22:00:00Z",
  "source": "scheduler",
  "correlation_id": "abc123",      // optional, for ACK correlation
  "meta": {}                        // optional, unstructured extended info
}
```

### Examples

**Minimal (most events):**
```json
{
  "timestamp": "2025-02-23T17:35:00Z",
  "source": "scheduler"
}
```

**With correlation ID (for ACK flows):**
```json
{
  "timestamp": "2025-02-23T22:00:00Z",
  "source": "security",
  "correlation_id": "lockdown_20250223_2200"
}
```

**With optional meta (if a consumer wants deeper context):**
```json
{
  "timestamp": "2025-02-23T17:35:00Z",
  "source": "scheduler",
  "meta": {
    "sunset_time": "18:05:00",
    "offset_minutes": -30,
    "next_event": "overnight",
    "next_event_time": "22:00:00"
  }
}
```

*Note: The `meta` field is a grab bag. No schema enforced. Consumers that care can inspect it; most won't.*

### ACK Payload
```json
{
  "timestamp": "2025-02-23T22:00:05Z",
  "source": "front_door",
  "correlation_id": "abc123",
  "result": "confirmed",            // confirmed | failed | timeout
  "message": "Lock engaged"         // optional detail
}
```

---

## Message Retention

### Retain: Period/State Events
Events representing ongoing state should be retained so restarting flows know current state:
- `highland/event/scheduler/evening` — retained, replaced by `overnight`, then `day`
- `highland/event/security/away` — retained until `home` published

### Do Not Retain: Point-in-Time Events
Events representing instantaneous occurrences:
- `highland/event/garage/motion_detected` — not retained
- `highland/event/front_door/opened` — not retained
- `highland/ack/*` — not retained

### Node-RED Context Persistence
- Flow context storage: **disk-based** (not memory)
- Provides secondary persistence for flow state across restarts/deployments
- Belt-and-suspenders with MQTT retention

---

## Design Guidelines

### When to include entity in topic
**Include entity** when consumers might need to:
- Filter to specific entities
- Include entity identity in actions (notifications, logging)
- Example: Leak sensors — "which sensor detected the leak?"

**Aggregate at area level** when:
- Consumers only care about the concept, not which entity
- Area flow handles internal sensor logic
- Example: Presence — "is someone in the room?" (not "which sensor saw them")

### ACK Pattern for Critical Operations
```
1. Utility publishes event with correlation_id
2. Utility starts timeout timer (e.g., 30 seconds)
3. Area flows perform action
4. Area flows publish ACK with same correlation_id
5. Utility collects ACKs:
   - All confirmed before timeout → success
   - Any failed or missing → escalate (notification, retry, etc.)
```

---

## Example: Exterior Lighting Flow

**Events consumed:**
- `highland/event/scheduler/evening` → Turn on all exterior lights (100%)
- `highland/event/scheduler/overnight` → Dim garage/porch to 20%, turn off patio
- `highland/event/scheduler/day` → Turn off all exterior lights
- `highland/event/+/motion_detected` → (if overnight) Restore 100% / turn on patio for 5 min

**Events published:**
- None (pure consumer in this example)

**Logic ownership:**
- Garage flow controls carriage lights (Zigbee group for synchronization)
- Front Porch flow controls entry light
- Back Yard flow controls patio lights

---

## Open Questions

- [x] ~~Naming convention for sources (areas/utilities)~~ → **lowercase with underscores**
- [x] ~~Topic prefix~~ → **`highland/`** (matches FQDN, reserves namespace)
- [x] ~~Payload structure for extended info~~ → **Optional `meta` field, unstructured**
- [x] ~~Standard set of period names~~ → **`day`, `evening`, `overnight`**
- [x] ~~Error/warning event patterns for system monitoring~~ → **See Health Monitoring in NODERED_PATTERNS.md**
- [x] ~~Should there be a `highland/status/` namespace for heartbeats/health checks?~~ → **Yes: `highland/status/{service}/heartbeat` and `highland/status/{service}/health`**

---

## Known Gaps: Wi-Fi Device Resiliency

These devices cannot currently be controlled via MQTT and may require HA and/or cloud connectivity. To be investigated during flow implementation.

| Device | Issue | Potential Path Forward |
|--------|-------|------------------------|
| **Reolink NVR + Cameras** | Cameras are Wi-Fi, NVR is hardwired | NVR supports ONVIF/RTSP and has an API. Investigate webhooks or polling for events without HA. |
| **Eufy Wi-Fi Locks** | Cloud-dependent, no official local API | Reverse-engineering efforts exist but are fragile. May need to accept HA-dependency or consider hardware replacement for security-critical role. |

*Note: These gaps mean that automations depending on these devices (e.g., `lockdown` ACK from door locks) may not survive HA or internet outages.*

---

*Last Updated: 2026-03-03*
