# LoRaWAN Sensors — Design Document

## Overview

LoRaWAN sensors provide long-range, low-power connectivity for outdoor and hard-to-reach locations. The Highland deployment uses a **Milesight UG65 gateway** connected to The Things Network (TTN), with sensor data flowing through Node-RED for processing.

**Sensors planned:**
- Mailbox door sensor (EM300-MCS)
- Driveway bin tilt sensors (EM320-TILT × 2)

---

## Architecture

```
┌─────────────────┐     LoRaWAN     ┌─────────────────┐
│   EM300-MCS     │ ─────────────►  │   Milesight     │
│   (Mailbox)     │                 │   UG65 Gateway  │
└─────────────────┘                 └────────┬────────┘
                                             │
┌─────────────────┐     LoRaWAN              │ MQTT
│  EM320-TILT ×2  │ ─────────────►           │
│  (Bins)         │                          ▼
└─────────────────┘                 ┌─────────────────┐
                                    │   TTN MQTT      │
                                    │   Integration   │
                                    └────────┬────────┘
                                             │
                                             ▼
                                    ┌─────────────────┐
                                    │   Node-RED      │
                                    │   (LoRaWAN      │
                                    │   Processing)   │
                                    └────────┬────────┘
                                             │
                                             ▼
                                    ┌─────────────────┐
                                    │   Mosquitto on  │
  → Mosquitto on Communication Hub (hub.local:1883)
                                    │   highland/...  │
                                    └─────────────────┘
```

**Data path:**
1. Sensor transmits LoRaWAN uplink
2. Gateway receives, forwards to TTN
3. TTN MQTT integration publishes to `v3/{app_id}/devices/{device_id}/up`
4. Node-RED subscribes (MQTT Out — publishes to `hub.local:1883` with highland MQTT credentials)
5. Node-RED decodes payload, runs state machine, publishes to `highland/` topics
6. HA consumes via MQTT Discovery

---

## Gateway Configuration

**Milesight UG65:**
- Connected to TTN (The Things Network) Community Edition
- Region: US915
- Packet forwarder: Semtech UDP or Basic Station (TBD at implementation)

**TTN Application:**
- Application ID: `highland-sensors` (or similar)
- MQTT Integration enabled
- Payload formatters: Use Milesight-provided decoders (JavaScript)

---

## Mailbox Sensor — EM300-MCS

### Purpose

Detect mailbox door open/close events. Combined with USPS Informed Delivery email parsing to determine delivery state.

### State Machine

```
┌─────────────────────────────────────────────────────────────┐
│                        IDLE                                 │
│  • No delivery expected today                               │
│  • Waiting for morning advisory email                       │
└─────────────────────────────────────────────────────────────┘
           │
           │ Morning advisory email received
           ▼
┌─────────────────────────────────────────────────────────────┐
│                   ADVISORY_RECEIVED                         │
│  • Delivery expected today                                  │
│  • Waiting for door event or confirmation email             │
└─────────────────────────────────────────────────────────────┘
           │                              │
           │ Door event                   │ Midnight (no door event)
           ▼                              ▼
┌─────────────────────────┐    ┌─────────────────────────────┐
│     UNCLASSIFIED        │    │    DELIVERY_EXCEPTION       │
│  • Door opened          │    │  • Advisory but no delivery │
│  • Waiting for          │    │  • Non-terminal — can       │
│    confirmation email   │    │    resolve if confirmation  │
│    (lag window)         │    │    arrives later            │
└─────────────────────────┘    └─────────────────────────────┘
           │                              │
           │ Confirmation email           │ Late confirmation
           │ within lag window            │ email received
           ▼                              ▼
┌─────────────────────────────────────────────────────────────┐
│                      MAIL_WAITING                           │
│  • Confirmed delivery                                       │
│  • Mail is in the box                                       │
└─────────────────────────────────────────────────────────────┘
           │
           │ Door event (retrieval)
           ▼
┌─────────────────────────────────────────────────────────────┐
│                       RETRIEVED                             │
│  • Mail collected                                           │
│  • Resets to IDLE at midnight                               │
└─────────────────────────────────────────────────────────────┘
```

### USPS Informed Delivery Integration

**Email types:**
- **Advisory email** — arrives ~6:00 AM, contains scanned images of expected mail
- **Confirmation email** — arrives after delivery, confirms mail was delivered

**Email source:** `highland@ferris.network` (Dynu-hosted, standard IMAP/SMTP)

**Processing:**
- Node-RED polls IMAP for new emails from `USPSInformedDelivery@usps.gov`
- Parse subject line to determine email type
- Move processed emails to a `processed` folder (not delete)
- 14-day purge of processed folder

**Lag window:** Confirmation email may arrive minutes to hours after physical delivery. The state machine waits in `UNCLASSIFIED` for a configurable window (default: 2 hours) before assuming non-delivery.

### Midnight Boundary

The `DELIVERY_EXCEPTION` state is triggered by the midnight calendar boundary (`highland/event/scheduler/midnight`), not by the `OVERNIGHT` period transition. This is a discrete scheduler task event, distinct from period events.

**Rationale:** The exception check needs to run at a specific time (00:00:00), not "sometime after overnight begins." Period transitions are for behavior changes; the exception check is a scheduled audit.

### MQTT Topics

See **MQTT_TOPICS.md §Mailbox** for full topic definitions.

---

## Driveway Bin Sensors — EM320-TILT

### Purpose

Track trash and recycling bin location (at house vs. at curb) and pickup status. Two independent sensors, two independent state machines.

### Zone Detection via RSSI

No dedicated zone sensors. Instead, use RSSI/SNR from the gateway uplink metadata to infer zone:

- **HOME zone:** Strong signal (bin near house, close to gateway)
- **AWAY zone:** Weak signal (bin at curb, far from gateway)

**Thresholds:** Calibrate during implementation. Expected pattern:
- HOME: RSSI > -90 dBm
- AWAY: RSSI < -100 dBm
- Hysteresis band: -90 to -100 dBm (no transition in this range)

### State Machine (per bin)

```
┌─────────────────────────────────────────────────────────────┐
│                          HOME                               │
│  • Bin is at house                                          │
│  • Strong RSSI                                              │
└─────────────────────────────────────────────────────────────┘
           │
           │ RSSI drops below AWAY threshold (debounced)
           ▼
┌─────────────────────────────────────────────────────────────┐
│                     AWAY_SETTLING                           │
│  • Bin moved toward curb                                    │
│  • Debounce period (avoid false triggers from              │
│    momentary signal dips)                                   │
└─────────────────────────────────────────────────────────────┘
           │
           │ RSSI remains low for debounce period
           ▼
┌─────────────────────────────────────────────────────────────┐
│                          AWAY                               │
│  • Bin is at curb                                           │
│  • Waiting for pickup                                       │
└─────────────────────────────────────────────────────────────┘
           │
           │ X-axis rotation ≥ 85° (bin tipped)
           ▼
┌─────────────────────────────────────────────────────────────┐
│                       PICKED_UP                             │
│  • Bin was serviced (emptied)                               │
│  • Still at curb                                            │
└─────────────────────────────────────────────────────────────┘
           │
           │ RSSI rises above HOME threshold (debounced)
           ▼
┌─────────────────────────────────────────────────────────────┐
│                        RETURNED                             │
│  • Bin brought back to house                                │
│  • Transitions to HOME on next uplink                       │
└─────────────────────────────────────────────────────────────┘
```

### Tilt Detection

The EM320-TILT reports X/Y/Z axis angles. Pickup detection uses X-axis (rotation when truck mechanism tips the bin):

- **Threshold:** X-axis ≥ 85° from vertical
- **Trigger:** Rising edge (transition from <85° to ≥85°)
- **State guard:** Only fire `picked_up` event if current state is `AWAY`

### Independent State Machines

Trash and recycling bins have separate state machines. It is normal for:
- One bin to be `AWAY` while the other is `HOME` (e.g., trash day but not recycling day)
- Both to be `AWAY` (both taken to curb)
- Pickup events to occur at different times (separate truck schedules)

### MQTT Topics

See **MQTT_TOPICS.md §Driveway Bins** for full topic definitions.

---

## Sensor Configuration

### EM300-MCS (Mailbox)

| Parameter | Value | Notes |
|-----------|-------|-------|
| Reporting interval | 15 minutes | Heartbeat; door events are immediate |
| Door event mode | On change | Transmit on open and close |
| Confirmed uplinks | No | Class A, unconfirmed |

### EM320-TILT (Bins)

| Parameter | Value | Notes |
|-----------|-------|-------|
| Reporting interval | 15 minutes | Heartbeat with current angles |
| Angle threshold | 85° | Tilt event trigger |
| Angle axis | X | Rotation axis for pickup detection |
| Confirmed uplinks | No | Class A, unconfirmed |

---

## Node-RED Flow Structure

### LoRaWAN Decoder Flow

Receives raw TTN uplinks, decodes Milesight payloads, routes to device-specific handlers.

```
[TTN MQTT In] → [Decode Payload] → [Route by Device]
                                         │
                       ┌─────────────────┼─────────────────┐
                       ▼                 ▼                 ▼
                [Mailbox Handler]  [Trash Handler]  [Recycling Handler]
```

### Mailbox Flow

- Subscribes to decoded mailbox sensor events
- Subscribes to USPS email events (from Email Poller flow)
- Subscribes to `highland/event/scheduler/midnight`
- Runs state machine in flow context
- Publishes to `highland/state/mailbox/delivery` (retained)
- Publishes events to `highland/event/mailbox/*`

### Bin Monitor Flow

- Subscribes to decoded bin sensor events
- Runs two independent state machines in flow context
- Publishes to `highland/state/driveway/{trash|recycling}_bin` (retained)
- Publishes events to `highland/event/driveway/{trash|recycling}_bin/*`

### Email Poller Flow

- Polls IMAP for USPS Informed Delivery emails
- Parses subject line to classify email type
- Publishes internal events for Mailbox flow consumption
- Moves processed emails to `processed` folder
- Runs 14-day purge on `processed` folder

---

## Home Assistant Entities

All entities registered via MQTT Discovery.

### Mailbox

| Entity | Type | Notes |
|--------|------|-------|
| Mailbox Delivery State | sensor | State machine state |
| Mailbox Door | binary_sensor | Open/closed |

### Driveway Bins

| Entity | Type | Notes |
|--------|------|-------|
| Trash Bin State | sensor | State machine state |
| Trash Bin RSSI | sensor | Signal strength (diagnostic) |
| Trash Bin Battery | sensor | Battery percentage |
| Recycling Bin State | sensor | State machine state |
| Recycling Bin RSSI | sensor | Signal strength (diagnostic) |
| Recycling Bin Battery | sensor | Battery percentage |

---

## Open Items

| Item | Notes |
|------|-------|
| TTN application setup | Create application, add devices, configure MQTT integration |
| RSSI threshold calibration | Test with bins at house and curb to determine thresholds |
| Email polling interval | Balance responsiveness vs. IMAP load (start with 5 minutes) |
| Confirmation email lag window | Start with 2 hours, adjust based on observed USPS timing |
| Gateway placement | Optimize for coverage of both mailbox and curb locations |

---

*Last Updated: 2026-03-11*
