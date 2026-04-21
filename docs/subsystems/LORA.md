# LoRa Integration — Design & Architecture

## Overview

LoRaWAN-based sensor network extending smart home coverage beyond the practical range of Zigbee and Z-Wave. Designed for low-power, outdoor, long-range use cases where the property's ~275ft driveway makes other protocols impractical.

---

## Infrastructure

### Gateway

**Device:** DFRobot DFR1093-915 Private LoRaWAN Gateway (~$99)
**Frequency:** US915 (902–928 MHz)
**Radio:** Semtech SX1302 8-channel chip

**Architecture note:** The gateway runs an internal SIoT MQTT broker (not configurable to an external broker natively). Data path uses a lightweight relay flow on the gateway's built-in Node-RED instance:

```
LoRa Node → Gateway radio → SIoT broker (localhost:1883)
  → Gateway Node-RED relay flow
  → Mosquitto on Communication Hub (hub.local:1883)
  → Node-RED on Workflow host
```

**Relay flow pattern (on gateway):**
- MQTT In — subscribes to SIoT broker at `localhost:1883`
- Function node — reshapes payload into `highland/` namespace, adds timestamp and source
- MQTT Out — publishes to `hub.local:1883` with highland MQTT credentials

> **Security note:** Change SIoT default credentials (`siot`/`dfrobot`) before the gateway touches the LAN.

---

## Use Case 1: Trash & Recycling Bin Monitoring

### Problem

Property is ~275ft from the road via a shared driveway with no line of sight to the street. Two bins (trash, recycling) need monitoring for: "Did I put them out?" and "Were they picked up?"

### Sensor Hardware

**Device:** Milesight EM320-TILT (US915), ~$84 each × 2
- 3-axis MEMS accelerometer, ±1° accuracy
- IP67 weatherproof
- 2× ER14505 Li-SOCl2 batteries, ≥3 year life at 10-min reporting
- NFC configuration via Milesight ToolBox app
- Immediate uplink on threshold crossing

**Mounting:** Screw-mount to bin side panel (not lid) with consistent orientation. Verify axis mapping via NFC app before deployment.

### Zone Detection (Home vs. Away)

Uses **RSSI** (and SNR) from LoRaWAN uplink packets — reported by the gateway on every received transmission, no additional hardware required.

- Calibrate once: note RSSI at street (~275ft) vs. home position
- Rolling average of last N readings (3–5) to reduce noise before zone state change
- Threshold sits in the ~15–20dB delta between home and street positions

**Limitation:** RSSI is sufficient for "probably on property" vs. "probably at the street" — which is all that's needed.

### Pickup Detection

**Trigger:** X-axis rotation ≥ 85° while bin is in `AWAY` state and past debounce window.

**Rationale:** Automated truck arm rotates bin 90–135° during dump cycle. No normal bin activity (rolling, wind, bumps) produces rotation anywhere near that magnitude.

**Configuration:** Threshold set on EM320-TILT via NFC app. Device fires immediate uplink on threshold crossing — zero detection latency.

### Bin State Machine

```
HOME ──────────────────────────────────────────────────────┐
  │                                                         │
  │ RSSI crosses outbound threshold                         │
  ▼                                                         │
AWAY_SETTLING (15-min debounce)                             │
  │                                 │                       │
  │ 15 min elapsed                  │ RSSI crosses back     │
  ▼                                 ▼                       │
AWAY ──────────────────────────► HOME                       │
  │                                                         │
  │ X-axis ≥ 85° threshold                                  │
  ▼                                                         │
PICKED_UP                                                   │
  │                                                         │
  │ RSSI crosses inbound threshold                          │
  ▼                                                         │
RETURNED ────────────────────────────────────────────────► HOME
```

**Debounce rationale:** 15-minute AWAY_SETTLING window prevents rolling/jostling during bin placement from triggering false pickup events.

### Calendar Integration

Pickup schedule stored in household Google Calendar. `trash_day` boolean unlocks context-aware behavior:

| Condition | Action |
|-----------|--------|
| Trash day, bins still `HOME` at configurable morning time | Reminder notification |
| Trash day, bin transitions to `PICKED_UP` | "Trash picked up, bins at street" notification |
| Bin in `PICKED_UP` state past configurable evening time | "Don't forget to bring the bins in" reminder |
| Not trash day, bin transitions to `AWAY` | No action |

**Independent tracking:** Trash and recycling run separate state machines. It is normal for trash to be `AWAY` while recycling is `HOME`.

### MQTT Topics

**State (retained):**

| Topic | Purpose |
|-------|---------|
| `highland/state/driveway/trash_bin` | Current trash bin state |
| `highland/state/driveway/recycling_bin` | Current recycling bin state |

```json
{
  "timestamp": "...",
  "source": "lora_bin_monitor",
  "state": "AWAY",
  "rssi": -108,
  "snr": -4.2,
  "battery_pct": 94
}
```

`state` values: `"HOME"` | `"AWAY_SETTLING"` | `"AWAY"` | `"PICKED_UP"` | `"RETURNED"`

**Events (not retained):**

`highland/event/driveway/{bin}/zone_changed` | `picked_up` | `returned`

`{bin}` values: `trash_bin` | `recycling_bin`

---

## Use Case 2: Mailbox Activity Sensor

### Problem

Mailbox is ~275ft from the house at the end of the driveway with no line of sight. Three mailboxes share one post.

The physical sensor's job is simple: report when the mailbox door opens. That's it. A single reed switch cannot, on its own, distinguish delivery from retrieval from an ad stuffer — all interactions look the same at the sensor. Classification of those events into meaningful delivery state (mail expected, delivered, retrieved) belongs in `Utility: Deliveries`, which fuses door-open events with USPS Informed Delivery email signals.

See `subsystems/DELIVERIES.md` for the delivery state machine, email parsing, and authoritative mail state. This subsystem's responsibility ends at "the door opened at time T."

### Sensor Hardware

**Device:** Milesight EM300-MCS (US915), ~$55–65
- Magnetic reed switch on 1.5m lead cable — sensor body and radio mount separately from the switch
- IP67 weatherproof
- 4000mAh battery (5-year life)
- Includes temperature and humidity sensors (bonus)

**Mounting:** Sensor body mounts on the back/side of the mailbox post — outside the metal enclosure, radio fully unobstructed. 1.5m cable runs to the door mechanism. Small magnet epoxied to inside of mailbox door; reed switch element fixed to door frame.

**Rationale for split mount:** Metal mailbox enclosures attenuate RF and experience extreme temperature swings. Keeping the radio outside avoids both problems.

### MQTT Topics

Published by the `Area: Driveway` flow after the gateway relay reshapes the raw LoRa uplink into the `highland/` namespace. Raw physical signals only — no interpretation.

**State (retained):** `highland/state/driveway/mailbox`

```json
{
  "timestamp": "...",
  "source": "lora_mailbox",
  "door_state": "closed",
  "battery_pct": 96,
  "temperature_c": 12.4,
  "humidity_pct": 58,
  "rssi": -102,
  "snr": -3.1
}
```

`door_state` values: `"open"` | `"closed"`.

**Events (not retained):** `highland/event/driveway/mailbox/opened`

```json
{ "timestamp": "...", "source": "lora_mailbox" }
```

Primary consumer: `Utility: Deliveries` (see `subsystems/DELIVERIES.md`). The battery field is also consumed by `Utility: Battery Monitor`.

---

## Hardware Summary

| Item | Device | Qty | Unit Cost | Total |
|------|--------|-----|-----------|-------|
| Gateway | DFRobot DFR1093-915 | 1 | $99 | $99 |
| Bin sensors | Milesight EM320-TILT (US915) | 2 | $84 | $168 |
| Mailbox sensor | Milesight EM300-MCS (US915) | 1 | ~$60 | ~$60 |
| **Total** | | | | **~$327** |

---

## Open Questions

**Infrastructure**
- [ ] SIoT topic structure — confirm `ProjectID/DeviceName` format during gateway setup

**Trash & Recycling**
- [ ] RSSI calibration procedure — document threshold values once sensors are deployed
- [ ] Optimal RSSI smoothing window (N readings)
- [ ] Morning reminder time threshold
- [ ] Evening retrieval reminder time threshold
- [ ] EM320-TILT mount orientation per bin — verify axis mapping before finalizing pickup threshold config

**Mailbox**
- [ ] Physical installation — sensor body mount point, cable routing, magnet placement
- [ ] Confirm door-state polarity in payload (open/closed mapping on this specific hardware) once installed

---

*Last Updated: 2026-04-21*
