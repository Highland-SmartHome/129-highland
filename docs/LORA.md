# LoRa Integration — Design &amp; Architecture

## Overview

LoRaWAN-based sensor network extending smart home coverage beyond the practical range of Zigbee and Z-Wave. Designed for low-power, outdoor, long-range use cases where the property's ~275ft driveway makes other protocols impractical.

---

## Infrastructure

### Gateway

**Device:** DFRobot DFR1093-915 Private LoRaWAN Gateway  
**Cost:** ~$99 (direct from DFRobot)  
**Frequency:** US915 (902–928 MHz)  
**Radio:** Semtech SX1302 8-channel chip  
**Compute:** Quad-core Cortex-A35, 512MB RAM, 16GB eMMC  
**Connectivity:** 100Mbps Ethernet + 2.4GHz WiFi  

**Architecture note:** The gateway runs an internal SIoT MQTT broker (not configurable to an external broker natively). Data path uses a lightweight relay flow on the gateway's built-in Node-RED instance:

```
LoRa Node → Gateway radio → SIoT broker (localhost:1883)
  → Gateway Node-RED relay flow
  → Mosquitto on Communication Hub (hub.local:1883)
  → Node-RED on NR box
```

**Relay flow pattern (on gateway):**
- MQTT In — subscribes to SIoT broker at `localhost:1883` (credentials: change from default `siot`/`dfrobot` before LAN connection)
- Function node — reshapes payload into `highland/` namespace, adds timestamp and source
- MQTT Out — publishes to `hub.local:1883` with highland MQTT credentials

**Security note:** Change SIoT default credentials before the gateway touches the LAN. Default `siot`/`dfrobot` is publicly documented.

---

## Use Cases

### 1. Trash &amp; Recycling Bin Monitoring

#### Problem Statement

Property is ~275ft from the road via a shared driveway with no line of sight to the street. Two bins (trash, recycling) need monitoring for:
1. **"Did I put them out?"** — bins may or may not go out each week; trash accrues faster than recycling so it's normal for trash to be out while recycling stays home
2. **"Were they picked up?"** — can't hear or see the truck from the house

Previous Bluetooth RSSI attempt failed due to range and signal instability.

#### Sensor Hardware

**Device:** Milesight EM320-TILT (US915)  
**Cost:** ~$84 each × 2 = ~$168  
**Quantity:** One per bin (trash, recycling independently tracked)  
**Key specs:**
- 3-axis MEMS accelerometer, ±1° accuracy, 0.01° resolution
- IP67 weatherproof enclosure
- 2 × ER14505 Li-SOCl2 batteries, ≥3 year life at 10-min reporting interval
- NFC configuration via Milesight ToolBox app
- Up to 36 configurable threshold trigger rules
- Immediate uplink on threshold crossing (does not wait for next scheduled report)
- Operating temperature: -30°C to +60°C

**Mounting:** Screw-mount to bin side panel (not lid) with consistent orientation. Verify axis mapping (which axis is X relative to mount orientation) via NFC app before deployment.

#### Zone Detection (Home vs. Away)

Uses **RSSI** (and SNR) from LoRaWAN uplink packets — reported by the gateway on every received transmission, no additional hardware required.

**Approach:**
- Calibrate once: note RSSI at street (~275ft), note RSSI at home position
- Rolling average of last N readings (3–5) to reduce noise before zone state change
- Use SNR alongside RSSI — SNR degrades more predictably with distance
- Threshold sits in the ~15–20dB delta between home and street positions

**Limitations:** RSSI is not GPS-grade. Orientation, weather, bin contents (empty vs. full of metal cans), and seasonal foliage all introduce variance. Sufficient for "probably on property" vs. "probably at the street" — which is all that's needed.

**Future option if RSSI proves unreliable:** Second LoRa anchor near driveway entrance provides two-point comparison for higher-confidence zone determination. Not planned for initial deployment.

#### Pickup Detection

**Trigger:** X-axis rotation ≥ 85° while bin is in `AWAY` state and past debounce window.

**Rationale:** Automated truck arm rotates bin 90–135° during dump cycle. This is unambiguous — no normal bin activity (rolling down driveway, wind, bumps) produces rotation anywhere near that magnitude.

**Configuration:** Threshold set directly on EM320-TILT via NFC app. Device fires immediate uplink on threshold crossing — zero detection latency.

#### Bin State Machine

```
HOME ──────────────────────────────────────────────────────┐
  │                                                         │
  │ RSSI crosses outbound threshold                         │
  ▼                                                         │
AWAY_SETTLING (15-min debounce)                             │
  │                                 │                       │
  │ 15 min elapsed,                 │ RSSI crosses back     │
  │ no return                       │ (changed mind)        │
  ▼                                 ▼                       │
AWAY ──────────────────────────► HOME                       │
  │                                                         │
  │ X-axis ≥ 85° threshold                                  │
  ▼                                                         │
PICKED_UP                                                   │
  │                                                         │
  │ RSSI crosses inbound threshold (retrieved)              │
  ▼                                                         │
RETURNED ────────────────────────────────────────────────► HOME
```

**Debounce rationale:** 15-minute AWAY_SETTLING window prevents rolling/jostling during bin placement from triggering false pickup events. Applies whether bins go out the night before or morning of pickup day.

#### Calendar Integration

Pickup schedule stored in household Google Calendar. Polled periodically by Node-RED (Daily Digest infrastructure).

`trash_day` boolean (or `days_until_pickup` integer) unlocks context-aware behavior:

| Condition | Action |
|-----------|--------|
| Trash day, bins still `HOME` at configurable morning time | Reminder notification |
| Trash day, bin transitions to `PICKED_UP` | "Trash picked up, bins at street" notification |
| Bin in `PICKED_UP` state past configurable evening time | "Don't forget to bring the bins in" reminder |
| Not trash day, bin transitions to `AWAY` | No action (bin being cleaned, moved around yard, etc.) |

**Independent tracking:** Trash and recycling run separate state machines. It is normal and expected for trash to be `AWAY` while recycling is `HOME` — no anomaly, no notification.

#### MQTT Topics

**State topics (retained)** — current truth for each bin. A restarting flow reads these immediately and knows where each bin is.

```
highland/state/driveway/trash_bin        ← RETAINED
highland/state/driveway/recycling_bin    ← RETAINED
```

```json
{
  "timestamp": "2026-03-09T07:45:00Z",
  "source": "lora_bin_monitor",
  "state": "AWAY",
  "rssi": -108,
  "snr": -4.2,
  "battery_pct": 94
}
```

`state` values: `"HOME"` | `"AWAY_SETTLING"` | `"AWAY"` | `"PICKED_UP"` | `"RETURNED"`

**Event topics (not retained)** — point-in-time transitions. Flows that missed the event use the state topic instead.

```
highland/event/driveway/trash_bin/zone_changed
highland/event/driveway/trash_bin/picked_up
highland/event/driveway/trash_bin/returned

highland/event/driveway/recycling_bin/zone_changed
highland/event/driveway/recycling_bin/picked_up
highland/event/driveway/recycling_bin/returned
```

```json
{
  "timestamp": "2026-03-09T14:30:00Z",
  "source": "lora_bin_monitor",
  "previous_state": "AWAY",
  "new_state": "PICKED_UP"
}
```

---

### 2. Mailbox Monitoring

#### Problem Statement

Mailbox is at the street (~275ft from house). No line of sight. Want to know:
1. When mail has arrived (delivery confirmation)
2. When mail has been retrieved (door activity after delivery)

USPS Informed Delivery provides morning email with scanned mail images and package tracking. Combined with a physical door sensor, this enables reliable delivery detection without needing to identify the mail carrier specifically.

#### Sensor Hardware

**Device:** Milesight EM300-MCS (Magnetic Contact Sensor)  
**Cost:** ~$60  
**Frequency:** US915  
**Key specs:**
- Reed switch + external magnet
- IP67 weatherproof enclosure
- 2 × ER14505 Li-SOCl2 batteries, ≥5 year life
- Configurable reporting interval + immediate uplink on state change
- Operating temperature: -30°C to +70°C

**Mounting:** Sensor body on mailbox post (sheltered side), magnet on mailbox door. Verify reliable triggering before finalizing placement.

#### Detection Logic

**Primary signal:** USPS Informed Delivery email (morning advisory + delivery confirmation)

**Flow:**
1. Morning: USPS sends "Informed Delivery Daily Digest" email with scanned mail images
2. Throughout day: USPS sends delivery confirmation emails as items are delivered
3. Mailbox door sensor fires on open/close events
4. Node-RED correlates: door event + delivery email within window = mail arrived

**State machine:**

```
IDLE ─────────────────────────────────────────────────────┐
  │                                                        │
  │ Morning advisory email received                        │
  ▼                                                        │
ADVISORY_RECEIVED                                          │
  │                                                        │
  │ Delivery confirmation email received                   │
  ▼                                                        │
MAIL_WAITING                                               │
  │                                                        │
  │ Door activity (mail retrieved)                         │
  ▼                                                        │
RETRIEVED ──────────────────────────────────────────────► IDLE
```

**Edge cases:**
- Door activity before delivery email = neighbor checking their box, ad stuffers, etc. → log but don't transition state
- Delivery email with no prior door activity = mail delivered, waiting for retrieval
- Multiple deliveries in one day = each confirmation email is a separate "delivery" event; state stays `MAIL_WAITING` until retrieved

#### Email Correlation Window

USPS delivery confirmation emails arrive *after* the carrier has physically delivered — sometimes minutes, sometimes an hour or more. The correlation window accounts for this lag.

**Approach:**
- When door activity is detected and state is `IDLE` or `ADVISORY_RECEIVED`, start a correlation timer
- If delivery confirmation email arrives within the window, transition to `MAIL_WAITING`
- If no email arrives within the window, treat as "unclassified door activity" (neighbor, ad stuffer, etc.)

**Window tuning:** Initial value ~30 minutes. Adjust based on observed lag for this specific route/carrier.

#### Example Timeline

```
06:03 AM — Informed Delivery morning email arrives (IDLE → ADVISORY_RECEIVED)
02:20 PM — Mailbox door opens/closes (door event logged, state unchanged)
02:21 PM — USPS delivery confirmation email arrives
           → Correlates with 02:20 door event
           → State: ADVISORY_RECEIVED → MAIL_WAITING
           → Notification: "Mail has arrived"
05:45 PM — Mailbox door opens/closes (state is MAIL_WAITING)
           → State: MAIL_WAITING → RETRIEVED → IDLE
           → No notification (retrieval is implicit success)
```

#### The 36-Hour Delivery Gap

The following scenario was flagged during design review as a potential edge case worth documenting. It is not expected to occur frequently, but the state machine handles it correctly.

**Scenario:**
- Day 1, 06:03 AM — Advisory email arrives (IDLE → ADVISORY_RECEIVED)
- Day 1, all day — No delivery (carrier skip, weather hold, operational delay)
- Day 1, midnight — State is still `ADVISORY_RECEIVED`
  - The advisory was received, but no delivery confirmation came
  - This is a *delivery exception* — mail was expected but did not arrive
- Day 2, 02:20 PM — Door opens/closes (carrier finally delivers)
- Day 2, 02:21 PM — Delivery confirmation email arrives (for Day 1's advisory)
  - State: `ADVISORY_RECEIVED` → `MAIL_WAITING`
  - This works correctly: the state machine was waiting for confirmation, and it got one

**Key points:**
- The advisory email transitions the state machine from `IDLE` to `ADVISORY_RECEIVED`, which means "expecting delivery"
- The state machine does not auto-reset at midnight — it waits for either (a) a delivery confirmation, or (b) a new day's advisory to override
- If mail arrives the *next day* after the advisory, the confirmation email still correlates correctly because the state machine is still in `ADVISORY_RECEIVED`
- **Midnight rollover** triggers the `DELIVERY_EXCEPTION` check: if `ADVISORY_RECEIVED` is still the current state at midnight, no confirmation came that calendar day. Exception declared, low-severity notification sent.
- `DELIVERY_EXCEPTION` is not terminal. If a delayed confirmation arrives (weather hold, next-day delivery, operational delay), it resolves normally to `MAIL_WAITING`. The next midnight tick only fires an exception if state is still `ADVISORY_RECEIVED`, not if it's already `DELIVERY_EXCEPTION`.
- The morning advisory is explicitly **not** load-bearing for delivery detection. On days USPS sends no advisory but delivers anyway, `DELIVERY_EXCEPTION` simply never gets set — the confirmation email transitions directly from `IDLE` or `UNCLASSIFIED` to `MAIL_WAITING`.
- Door activity with no email context (`UNCLASSIFIED`) is low-stakes. Neighbor interaction, ad stuffer, checking before mail arrives — all leave the state machine unchanged until a confirmation email arrives or midnight rolls over without one.

#### Scheduler Integration

The midnight check uses a dedicated task event:

```
highland/event/scheduler/midnight
```

This is a schedex node in the Scheduler flow fixed at 00:00. It is **distinct** from `highland/event/scheduler/overnight` (10pm, carries house-state semantics). It is the single calendar-day-boundary trigger — the Daily Digest flow also subscribes to this same event. Publishers don't name events after their consumers; `midnight` is the correct name and multiple flows may subscribe to it.

*Note: EVENT_ARCHITECTURE.md should be updated to add `midnight` to the defined task events when this flow is implemented.*

#### Failure Modes &amp; Contingencies

| Scenario | System Behavior |
|----------|----------------|
| Advisory arrives, confirmation arrives same day (normal) | `ADVISORY_RECEIVED` → `MAIL_WAITING` → `RETRIEVED` |
| Advisory arrives, no delivery that day | `ADVISORY_RECEIVED` → `DELIVERY_EXCEPTION` at midnight |
| Advisory arrives, confirmation arrives next day or later (weather hold) | `ADVISORY_RECEIVED` → `DELIVERY_EXCEPTION` at midnight → `MAIL_WAITING` when confirmation eventually arrives |
| Exception + following day is no-mail day | Exception state persists until confirmation eventually arrives |
| No advisory, delivery happens anyway (USPS system glitch) | Confirmation email transitions `IDLE`/`UNCLASSIFIED` → `MAIL_WAITING` directly |
| Door activity before delivery (neighbor, ad stuffer) | Logged as `UNCLASSIFIED`; resolved or ignored when confirmation arrives |
| Door activity after delivery, before retrieval (neighbor opens wrong box) | Transitions `MAIL_WAITING` → `RETRIEVED` prematurely — low probability, low stakes |
| No emails at all (USPS system outage) | State stays `IDLE`; no false positives, no notification |

**Notification behavior:**

| Event | Severity | DND Override | Notes |
|-------|----------|--------------|-------|
| `MAIL_WAITING` entered | `low` | No | "Mail has arrived" |
| `DELIVERY_EXCEPTION` entered | `low` | No | "Mail was expected but not confirmed delivered today" |
| `mail_waiting_reminder_time` reached, not yet retrieved | `low` | No | "Don't forget the mail" |
| `RETRIEVED` | Silent | — | No notification needed |

#### Email Management

Informed Delivery emails are processed by a dedicated IMAP poller in Node-RED (`node-red-node-email`), polling `highland@ferris.network` via Dynu's IMAP server. Credentials stored in `secrets.json`. After processing, emails are **moved to a mailbox folder** (`Highland/Mailbox` or similar) rather than deleted. This provides an audit trail for debugging edge cases and preserves the data needed to calibrate the lag window.

A fallback purge of processed emails older than 14 days keeps the folder from accumulating indefinitely.

**Rationale against deletion:** The 36-hour delivery gap scenario demonstrates that an email parsed one day may be relevant the next. Deleting on parse would have removed the advisory before the confirmation arrived. Move-and-age-purge handles this cleanly without requiring the flow to reason about whether a given email's arc is "complete."

**IMAP polling interval:** TBD during implementation. Probably every 5–10 minutes; could be reduced when state is `IDLE` with no advisory pending.

#### Configuration (thresholds.json)

```json
"mailbox": {
  "delivery_email_lag_window_minutes": 30,
  "mail_waiting_reminder_time": "18:00",
  "processed_email_retention_days": 14
}
```

`delivery_email_lag_window_minutes` is a tuning parameter — real-world lag for this carrier/route is unknown until data is collected. The flow logs both the door event timestamp and the email arrival timestamp on every delivery cycle to build a calibration dataset.

#### MQTT Topics

**State topic (retained)** — current truth for the mailbox state machine.

```
highland/state/mailbox/delivery    ← RETAINED
```

```json
{
  "timestamp": "2026-03-09T14:22:00Z",
  "source": "lora_mailbox",
  "state": "MAIL_WAITING",
  "last_door_event": "2026-03-09T14:20:00Z",
  "advisory_received_at": "2026-03-09T06:03:00Z",
  "confirmation_received_at": "2026-03-09T14:21:00Z"
}
```

`state` values: `"IDLE"` | `"UNCLASSIFIED"` | `"ADVISORY_RECEIVED"` | `"MAIL_WAITING"` | `"DELIVERY_EXCEPTION"` | `"RETRIEVED"`

**Event topics (not retained)** — point-in-time transitions and activity signals.

```
highland/event/mailbox/door_activity          # any open/close event (unclassified)
highland/event/mailbox/mail_expected          # morning advisory processed
highland/event/mailbox/mail_delivered         # delivery confirmed (email resolved event)
highland/event/mailbox/mail_retrieved         # door event while MAIL_WAITING
highland/event/mailbox/delivery_exception     # advisory received, no confirmation by midnight
```

```json
{
  "timestamp": "2026-03-09T14:22:00Z",
  "source": "lora_mailbox",
  "previous_state": "ADVISORY_RECEIVED",
  "new_state": "MAIL_WAITING"
}
```

#### Open Questions (Mailbox-Specific)

- [ ] Sensor pricing confirmation and purchase
- [ ] Physical installation — sensor body mount point on post, cable routing, magnet placement
- [ ] IMAP polling mechanism — interval, Dynu server details, email parsing (subject line, sender, body); credentials in `secrets.json`
- [ ] Real-world delivery email lag — calibrate from observed data; tune `delivery_email_lag_window_minutes`
- [ ] IMAP folder name convention for processed emails (`Highland/Mailbox` or similar)
- [ ] Evening mail retrieval reminder time — set based on household preference
- [ ] EVENT_ARCHITECTURE.md update — add `highland/event/scheduler/midnight` to defined task events

---

## Hardware Summary

| Item | Device | Qty | Unit Cost | Total |
|------|--------|-----|-----------|-------|
| Gateway | DFRobot DFR1093-915 | 1 | $99 | $99 |
| Bin sensors | Milesight EM320-TILT (US915) | 2 | $84 | $168 |
| Mailbox sensor | Milesight EM300-MCS (US915) | 1 | ~$60 | ~$60 |
| **Total (current scope)** | | | | **~$327** |

---

## Open Questions

**Infrastructure**
- [ ] SIoT topic structure — confirm `ProjectID/DeviceName` format during gateway setup and align with device naming conventions

**Trash &amp; Recycling**
- [ ] RSSI calibration procedure — document actual threshold values once sensors are deployed
- [ ] Optimal RSSI smoothing window (N readings) — tune during initial deployment
- [ ] Morning reminder time threshold ("bins not out by X:00am on trash day")
- [ ] Evening retrieval reminder time threshold
- [ ] EM320-TILT mount orientation per bin — verify axis mapping before finalizing pickup threshold config

**Mailbox**
- [ ] Sensor pricing confirmation and purchase
- [ ] Physical installation — sensor body mount point on post, cable routing, magnet placement
- [ ] Gmail polling mechanism — interval, authentication, email parsing approach
- [ ] Real-world delivery email lag — calibrate from observed data; tune `delivery_email_lag_window_minutes`
- [ ] Evening mail retrieval reminder time — set based on household preference

---

## Related Documents

| Document | Relevance |
|----------|-----------|
| **EVENT_ARCHITECTURE.md** | MQTT topic conventions; `highland/event/` namespace |
| **ENTITY_NAMING.md** | Entity naming if bin sensors are exposed to HA |
| **NODERED_PATTERNS.md** | Flow design, calendar integration, notification framework |
| **AUTOMATION_BACKLOG.md** | Trash/recycling automation backlog item |

---

*Last Updated: 2026-03-10 — Corrected Scheduler Integration section: `digest_daily` is not a separate topic; `midnight` is the single calendar-day-boundary event and the Daily Digest subscribes to it alongside the mailbox flow.*
