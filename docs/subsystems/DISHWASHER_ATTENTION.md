# Dishwasher Attention State Machine

## Purpose & Scope

This document covers the **attention layer** for the dishwasher — a state machine that runs above the power-based cycle detection documented in `subsystems/APPLIANCE_MONITORING.md`. Where cycle detection answers "is the dishwasher running?", the attention layer answers "does the dishwasher require action from a human?"

The two layers are independent and cooperating. The attention state machine subscribes to events published by the cycle detection flow; it does not duplicate or replace any of that logic.

---

## Design Philosophy

### Explicit intent over inference

The central design principle of this system is that **human confirmation is the primary signal**, not sensor inference. Door sensors, tilt sensors, and timing heuristics can observe *behavior* but cannot reliably determine *intent*. The gap between the two is where edge cases live, and there are always more edge cases than anticipated.

This is particularly true for the dishwasher, where legitimate interactions with the door include:

- Venting steam post-cycle (door cracked, not unloading)
- Grabbing a single item (door fully open, but not unloading)
- Partial unloading over multiple sessions
- Loading dirty dishes throughout the day
- Checking whether the cycle finished
- Guests interacting with it differently than the household norm

Rather than building increasingly elaborate heuristics to discriminate between these cases, the system **defaults to explicit confirmation** and uses sensor data as a supporting signal for notifications and fallback detection only.

### The button is the hero action

A dedicated physical button (Aqara WXKG13LM, Zigbee) serves as the primary confirmation mechanism. Pressing it means "I have unloaded the dishwasher." This is:

- Unambiguous — no inference required
- Low-friction — single press, natural gesture immediately after unloading
- Idempotent — pressing it when already in `IDLE_DIRTY` is a safe no-op
- Consistent — the same button pattern applies to the washing machine and dryer, establishing a household habit

Sensor data (tilt angle, timing) informs the system and drives notifications. It does **not** drive the primary state transition in the common case.

### The tilt sensor's actual role

A contact sensor cannot distinguish a door cracked 10° to vent steam from a door dropped fully open. A tilt sensor provides continuous angle data, enabling that discrimination structurally. However, even with full angle visibility, the tilt sensor **cannot** discriminate between unloading and any other full-open interaction. A full door open is a *necessary precondition* for unloading (the bottom rack physically requires the door fully flat to extend) but is not a *sufficient* indicator that unloading occurred. The close-after-open sequence adds no information.

**The tilt sensor's role is therefore:**
- Suppress "door opened" notifications when the door is only in the vent-angle range (< ~15°)
- Feed the guest heuristic: sustained full open beyond the duration threshold → `LIKELY_EMPTY`
- Trigger first-open notification when door reaches unload angle, first time after cycle completion

It does not drive any state transition in the primary user workflow.

---

## Hardware

### Tilt Sensor — Aqara DJT11LM (Zigbee)

Mounted on the dishwasher door. Reports 3-axis angle data and vibration events via Zigbee2MQTT. Reports on change, not on a fixed interval — every message is a meaningful signal.

**Calibration required at install:** Mount the sensor, open the door to various positions, record the axis readings from Z2M. Document the results as config values. One-time 5-minute exercise.

**Angle ranges (illustrative — update after calibration):**

| Range | Interpretation |
|-------|----------------|
| 0° – ~15° | Closed or minimally cracked. Venting territory. Suppress notifications. |
| ~15° – ~45° | Partially open. Ambiguous. |
| ~45°+ | Substantially open. Unload-candidate range. Guest heuristic timer starts. |

**Sensitivity setting:** Medium or low via Z2M. The dishwasher motor running and nearby appliances should not generate spurious angle reports.

**Reporting behavior:** Reports on change. While the door is in motion, reports arrive. When the door stops, reports stop. Last received angle is the resting position. Silence means stillness.

### Manual Button — Aqara WXKG13LM (Zigbee)

Surface or wall mounted in a convenient location relative to the dishwasher. Specific mounting location TBD — counter underside and adjacent cabinetry present physical constraints; further evaluation needed.

**Button action mapping (Z2M exposes single, double, hold):**

| Action | Meaning |
|--------|---------|
| Single press | "I have unloaded the dishwasher" — primary action |
| Double press | Reserved |
| Hold | Reserved |

**Acknowledgment:** Brief HA notification ("Dishwasher marked as empty") confirms the press registered. Voice acknowledgment deferred until voice pipeline is integrated.

**Idempotency:** Button press in `IDLE_DIRTY` is a safe no-op. No state change, no error, no notification.

**Note:** The WXKG13LM is the logical choice for the washing machine and dryer as well. Same hardware, same single-press gesture, same philosophy.

---

## State Machine

### States

| State | Meaning |
|-------|---------|
| `IDLE_DIRTY` | Dishwasher contains dirty dishes or is empty and ready to load. No action required. |
| `RUNNING` | Cycle in progress. No attention required yet. |
| `CLEAN_UNATTENDED` | Cycle complete. Dishes are clean. Human action (unloading) has not been confirmed. |
| `LIKELY_EMPTY` | Guest heuristic: sustained full-open after cycle suggests unloading occurred. Awaiting confirmation or timeout. |

### Transitions

```
IDLE_DIRTY
  ──(cycle_started event)───────────────────────────────► RUNNING

RUNNING
  ──(cycle_finished event)──────────────────────────────► CLEAN_UNATTENDED

CLEAN_UNATTENDED
  ──(button: single press)──────────────────────────────► IDLE_DIRTY
  ──(voice: "dishwasher is empty") [future]─────────────► IDLE_DIRTY
  ──(tilt: door at unload angle ≥ guest threshold)──────► LIKELY_EMPTY
  ──(tilt: door enters unload angle, first time)─────────  [notification only, no transition]
  ──(tilt: door in vent range only)──────────────────────  [notification suppressed, no transition]
  ──(N hours elapsed, no action)─────────────────────────  [nag notification, no transition]

LIKELY_EMPTY
  ──(button: single press)──────────────────────────────► IDLE_DIRTY
  ──(voice: "dishwasher is empty") [future]─────────────► IDLE_DIRTY
  ──(M hours elapsed, no confirmation)──────────────────► IDLE_DIRTY  [self-resolve timeout]

IDLE_DIRTY
  ──(button: single press)──────────────────────────────►  no-op
  ──(tilt: any door activity)────────────────────────────  [no state change — loading dirty dishes]
```

### Transition Logic Detail

**`CLEAN_UNATTENDED` → `IDLE_DIRTY` (primary path)**

Trigger: button single press or voice command (future).

The button is the authoritative signal. The tilt sensor cannot discriminate between "I opened the door and unloaded" and "I opened the door and changed my mind." A full door open is a *necessary precondition* for unloading — the bottom rack physically requires the door fully flat to extend — but it is not a *sufficient* indicator. The close-after-open sequence therefore adds no information.

**`CLEAN_UNATTENDED` → `LIKELY_EMPTY` (guest / fallback path)**

Trigger: door at unload-angle (≥ ~45°) for a continuous duration ≥ `guest_open_duration_s` (configurable, default 360s / 6 min).

Catches the scenario where a guest unloads the dishwasher without pressing the button. Extended continuous full-open is the only passive signal available and is reasonably discriminating — a genuine unload keeps the door down for several minutes, whereas grabbing one item or venting steam does not.

Cumulative open time is **not** used. Multiple short opens do not sum. A single continuous open of sufficient duration is required.

**`LIKELY_EMPTY` self-resolve timeout**

If `LIKELY_EMPTY` receives no confirmation for `likely_empty_timeout_hours` (default 2h), it transitions to `IDLE_DIRTY` automatically. The heuristic fired with reasonable confidence; the timeout avoids the system staying stuck indefinitely.

**Button press in `IDLE_DIRTY`**

No-op. No state change, no notification, no error. Idempotent by design.

---

## Notification Strategy

Notifications are informational. They do not change state.

| Trigger | Message | Notes |
|---------|---------|-------|
| Enter `CLEAN_UNATTENDED` | "Dishwasher finished — dishes are clean." | Always |
| Door enters unload angle, first time in `CLEAN_UNATTENDED` | "Reminder — dishes are clean." | First open only |
| Door in vent range only | [suppressed] | Not a meaningful interaction |
| `nag_first_hours` elapsed in `CLEAN_UNATTENDED`, door never opened | "Dishwasher has been done for N hours." | Configurable |
| `nag_first_hours` elapsed in `CLEAN_UNATTENDED`, door opened but not confirmed | "Dishwasher still waiting to be unloaded." | Configurable |
| Enter `LIKELY_EMPTY` | "Looks like the dishwasher was emptied — press the button to confirm." | Always |
| Button press in `CLEAN_UNATTENDED` or `LIKELY_EMPTY` | "Dishwasher marked as empty." | Always |
| `LIKELY_EMPTY` self-resolve | "Dishwasher marked as empty (auto-confirmed)." | Configurable — may be too chatty |

**DnD behavior:** Cycle completion notification and first-open reminder respect DnD. Nag notifications after `nag_dnd_override_hours` override DnD — their value is specifically in surfacing a forgotten task.

---

## Integration Points

### Upstream: Cycle Detection

The attention state machine subscribes to:
- `highland/event/appliance/dishwasher/cycle_finished` → trigger `RUNNING → CLEAN_UNATTENDED`
- `highland/event/appliance/dishwasher/cycle_started` → trigger to `RUNNING` (resets attention state if somehow mid-cycle)

### Tilt Sensor (Z2M → MQTT)

Z2M publishes the DJT11LM's axis data to its device topic. A normalization function in Node-RED translates raw axis readings into a single `door_angle_deg` value using the calibration mapping. The normalized value is what the state machine consumes. Raw Z2M data is not published to the `highland/` bus.

### Button (Z2M → MQTT)

Z2M publishes WXKG13LM press actions to its device topic. A handler maps `single` press to the attention state machine's confirm-empty input.

### MQTT Topics

**`highland/state/appliance/dishwasher/attention`** ← RETAINED

```json
{
  "timestamp": "2026-04-03T10:00:00Z",
  "source": "dishwasher_attention",
  "state": "CLEAN_UNATTENDED",
  "cycle_finished_at": "2026-04-03T09:30:00Z",
  "last_door_event_at": "2026-04-03T09:45:00Z",
  "last_door_angle": 12.3
}
```

`state` values: `IDLE_DIRTY` | `RUNNING` | `CLEAN_UNATTENDED` | `LIKELY_EMPTY`

**`highland/event/appliance/dishwasher/attention_changed`**

```json
{
  "timestamp": "2026-04-03T10:00:00Z",
  "source": "dishwasher_attention",
  "previous_state": "CLEAN_UNATTENDED",
  "new_state": "IDLE_DIRTY",
  "trigger": "button"
}
```

`trigger` values: `button` | `voice` | `tilt_guest_heuristic` | `timeout` | `cycle_started` | `cycle_finished`

### HA Entities (via MQTT Discovery)

| Entity | Type | Notes |
|--------|------|-------|
| Dishwasher Attention State | `sensor` | `state` field |
| Dishwasher Needs Attention | `binary_sensor` | `ON` when state is `CLEAN_UNATTENDED` or `LIKELY_EMPTY` |

---

## Node-RED Flow Architecture

Separate flow from the cycle detection flow. Tab name: `Appliance: Dishwasher Attention` (or grouped under a broader appliance tab).

```
[MQTT In: cycle_finished / cycle_started]
        │
        ▼
[Function: Attention State Machine]
        │
        ├──► [MQTT Out: highland/state/appliance/dishwasher/attention]   retained
        ├──► [MQTT Out: highland/event/appliance/dishwasher/attention_changed]
        └──► [Function: Notification Builder]
                    │
                    ▼
             [MQTT Out: highland/event/notify]

[MQTT In: Z2M tilt sensor topic]
        │
        ▼
[Function: Tilt Normalizer]    ← applies calibration mapping → door_angle_deg
        │
        ▼
[Link Out → Attention State Machine]

[MQTT In: Z2M button topic]
        │
        ▼
[Function: Button Handler]     ← filters for single press
        │
        ▼
[Link Out → Attention State Machine]

[Inject: nag timer]
        │
        ▼
[Link Out → Attention State Machine]
```

The attention state machine is a single Function node with state in flow context, consistent with the pattern in `APPLIANCE_MONITORING.md`.

---

## Configuration Parameters

Stored in flow context `config`, set by Config Loader on startup.

| Parameter | Default | Notes |
|-----------|---------|-------|
| `unload_angle_threshold_deg` | 45 | Door angle above which we consider door "fully open." Calibrate at install. |
| `vent_angle_threshold_deg` | 15 | Door angle below which we consider door "cracked/venting." Calibrate at install. |
| `guest_open_duration_s` | 360 | Continuous open at unload angle to trigger `LIKELY_EMPTY` (6 min) |
| `nag_first_hours` | 4 | Hours in `CLEAN_UNATTENDED` before first nag |
| `nag_repeat_hours` | 2 | Hours between subsequent nag notifications |
| `nag_dnd_override_hours` | 6 | Hours before nag overrides DnD |
| `likely_empty_timeout_hours` | 2 | Hours before `LIKELY_EMPTY` self-resolves |
| `notify_on_self_resolve` | false | Notification on `LIKELY_EMPTY` auto-timeout |
| `tilt_axis` | TBD | Which axis from DJT11LM maps to door angle. Set at calibration. |
| `tilt_closed_value` | TBD | Axis reading when door is closed. Set at calibration. |
| `tilt_open_value` | TBD | Axis reading when door is fully open. Set at calibration. |

---

## Open Questions / Pending Actions

- **Button mounting location** — Counter underside and cabinetry are constrained due to drawer clearance. Evaluate adjacent cabinet side panel or other location that survives daily use without accidental activation.
- **Tilt sensor calibration** — Mount DJT11LM, record axis readings at closed / venting crack / fully open positions, update `tilt_axis`, `tilt_closed_value`, `tilt_open_value` in config.
- **DJT11LM reporting behavior** — Verify whether it reports continuously during door travel or only on settling. Affects timer granularity in the state machine.
- **Guest threshold tuning** — 6 minutes is a reasoned estimate. Review actual guest interactions in logs and adjust `guest_open_duration_s` accordingly.
- **Voice integration** — "Marvin, dishwasher is empty" as additional path to `IDLE_DIRTY`. Deferred until voice pipeline is built. No state machine changes required when it arrives.

---

## Changelog

| Date | Change |
|------|--------|
| 2026-04-03 | Initial commit. Dishwasher attention layer above cycle detection. Tilt sensor (DJT11LM) selected over contact sensor for vent-angle discrimination. Button (WXKG13LM) established as hero action / primary confirmation. Guest heuristic via continuous door-open duration retained as fallback. Close-after-open inference explicitly ruled out — full open is necessary but not sufficient for unloading. |
