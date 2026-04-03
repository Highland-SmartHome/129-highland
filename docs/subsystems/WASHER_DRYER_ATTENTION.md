# Washer & Dryer Attention State Machine

## Purpose & Scope

This document covers the **attention layer** for the washing machine and dryer — state machines that run above the power-based cycle detection documented in `subsystems/APPLIANCE_MONITORING.md`. Where cycle detection answers "is the appliance running?", the attention layer answers "does the appliance require action from a human?"

The washing machine and dryer share the same attention state machine architecture and are documented together. Where they diverge — primarily notification urgency — that is called out explicitly.

---

## Design Philosophy

### Independent state machines

The washing machine and dryer are **fully independent.** There is no logical relationship modeled between the two appliances. Valid real-world use cases include:

- Running the washer without the dryer (line dry or air dry items)
- Running the dryer without the washer (wrinkle removal, items dried separately)
- Either machine used by guests doing their own laundry independently

Physical causality (wet clothes must come from somewhere) is not logical dependency, and treating it as such would create edge cases without buying anything meaningful. Any termination of either machine's attention state requires a physical trip to the laundry room regardless — the button is the signal.

### Explicit intent over inference

Same philosophy as `DISHWASHER_ATTENTION.md`: **human confirmation via button press is the primary signal.** The button is not just the preferred path — given hardware constraints, it is the only reliable state confirmation path.

### Lighter instrumentation than the dishwasher

The dishwasher has a more complex door interaction profile (steam venting, partial unloads, loading throughout the day). The washer and dryer have simpler interaction profiles — you go to the laundry room, you deal with the load, you leave. This justifies a lighter sensor approach.

**What was considered and ruled out:**

| Sensor | Verdict | Reason |
|--------|---------|--------|
| Vibration sensor | Ruled out | Shared pedestal — vibration from one machine transmits to the other, no atomicity |
| Contact sensor (door) | Ruled out | Bubble door geometry — no flat mounting surface on either machine |
| Tilt sensor | Not applicable | No steam venting analog; same atomicity problem as contact sensor |

**What remains:**

| Sensor | Role |
|--------|------|
| PIR (laundry room, shared) | Notification suppression + guest heuristic input |
| Button — Aqara WXKG13LM (one per machine) | Hero action — state confirmation |

### Notification urgency is asymmetric

The washing machine has a genuine hygiene clock — a wet load sitting in a closed drum will go musty within a few hours, faster in warm weather. Notification cadence reflects this — more aggressive, shorter intervals.

The dryer is a comfort and convenience concern. Wrinkled clothes are annoying, not a health issue. Notification cadence is more relaxed, and the dryer never overrides DnD.

---

## Hardware

### PIR Presence Sensor (laundry room, shared)

A single Zigbee PIR sensor mounted inside the laundry room, aimed at the machines. One sensor serves both appliances.

**Why PIR over mmWave (FP300):** The laundry room is a transactional space. Nobody stands motionless in a laundry room for extended periods — loading, unloading, and transferring loads all involve active movement. PIR is sufficient and considerably cheaper for this use case.

**Mounting:** Aimed at the machines, not at the doorway. The laundry room has narrow double doors that are typically open during laundry activity and closed otherwise. Door state is **not** used as a signal — there is insufficient ritual around the doors to make their state meaningful. The PIR fires when someone is physically near the machines regardless of door position.

**Role:**
- Suppresses repeat nag notifications while someone is actively in the room
- Feeds the `LIKELY_ATTENDED` guest heuristic when presence is sustained

**Shared resource:** A single presence event suppresses notifications for both machines simultaneously and feeds both state machines.

### Manual Button — Aqara WXKG13LM (Zigbee, one per machine)

Two buttons, one per appliance. Mounted in accessible locations in or near the laundry room. Specific mounting locations TBD pending physical assessment.

**Button action mapping:**

| Action | Meaning |
|--------|---------|
| Single press | "I have dealt with this load" — primary action |
| Double press | Reserved |
| Hold | Reserved |

The two buttons should be clearly distinguishable from each other by position, label, or color. Pressing the wrong one is a harmless no-op but poor UX.

**Acknowledgment:** Brief HA notification confirms press registered ("Washing machine marked as empty" / "Dryer marked as empty"). Voice acknowledgment deferred until voice pipeline is integrated.

**Idempotency:** Button press in `IDLE` is a safe no-op.

---

## State Machine

### States

Both appliances use identical states.

| State | Meaning |
|-------|---------|
| `IDLE` | No load in progress. No action required. |
| `RUNNING` | Cycle in progress. No attention required yet. |
| `UNATTENDED` | Cycle complete. Load has not been confirmed dealt with. |
| `LIKELY_ATTENDED` | Guest heuristic: sustained presence after cycle suggests load was dealt with. Awaiting confirmation or timeout. |

### Transitions

```
IDLE
  ──(cycle_started event)───────────────────────────────► RUNNING

RUNNING
  ──(cycle_finished event)──────────────────────────────► UNATTENDED

UNATTENDED
  ──(button: single press)──────────────────────────────► IDLE
  ──(voice: "washer/dryer is empty") [future]───────────► IDLE
  ──(presence: sustained ≥ attended_presence_duration_s)► LIKELY_ATTENDED
  ──(presence: brief)────────────────────────────────────  [notification suppression + timer reset, no transition]
  ──(nag interval elapsed, no presence)──────────────────  [nag notification, no transition]

LIKELY_ATTENDED
  ──(button: single press)──────────────────────────────► IDLE
  ──(voice: "washer/dryer is empty") [future]───────────► IDLE
  ──(M hours elapsed, no confirmation)──────────────────► IDLE  [self-resolve timeout]

IDLE
  ──(button: single press)──────────────────────────────►  no-op
```

### Transition Logic Detail

**`UNATTENDED` → `IDLE` (primary path)**

Trigger: button single press or voice command (future).

The button is the authoritative signal. There is no sensor on either machine capable of detecting door interaction with sufficient atomicity — vibration sensing ruled out by shared pedestal, contact sensing ruled out by bubble door geometry. The button is therefore not just the preferred path but the only reliable state confirmation path.

**Presence suppression behavior**

Brief presence (below `attended_presence_duration_s`) while in `UNATTENDED`:
- Suppresses the current nag cycle — if a nag was about to fire, it is held
- Resets the nag timer — next nag fires N minutes/hours after presence clears, not from original cycle end time
- Applies to both machines simultaneously

**`UNATTENDED` → `LIKELY_ATTENDED` (guest / fallback path)**

Trigger: presence detected in laundry room for a sustained continuous duration ≥ `attended_presence_duration_s` (default 180s / 3 min).

Catches the scenario where a guest does their own laundry and doesn't press the button. Sustained presence near the machines is the only passive signal available. A single brief presence event is not sufficient — actually dealing with a load takes several minutes.

**`LIKELY_ATTENDED` self-resolve timeout**

If `LIKELY_ATTENDED` receives no confirmation for `likely_attended_timeout_hours`, it transitions to `IDLE` automatically.

**Button press in `IDLE`**

No-op. Safe, silent, harmless.

---

## Notification Strategy

### Washing Machine

Higher urgency — wet load hygiene concern.

| Trigger | Message | Timing |
|---------|---------|--------|
| Enter `UNATTENDED` | "Washing machine finished — move the load." | Immediately |
| Nag, no recent presence | "Washing machine has been done for N minutes." | Every `nag_repeat_minutes` |
| Presence detected (brief) | [suppressed] | While presence active + reset |
| Enter `LIKELY_ATTENDED` | "Looks like the washing machine was dealt with — press the button to confirm." | Immediately |
| Button press | "Washing machine marked as empty." | Immediately |
| `LIKELY_ATTENDED` self-resolve | "Washing machine marked as empty (auto-confirmed)." | Configurable |

**DnD behavior:** Cycle completion respects DnD. Nag notifications override DnD after `nag_dnd_override_minutes` — the hygiene concern is real enough to justify it.

### Dryer

Lower urgency — wrinkle concern only.

| Trigger | Message | Timing |
|---------|---------|--------|
| Enter `UNATTENDED` | "Dryer finished — load is ready." | Immediately |
| Nag, no recent presence | "Dryer has been done for N hours." | Every `nag_repeat_hours` |
| Presence detected (brief) | [suppressed] | While presence active + reset |
| Enter `LIKELY_ATTENDED` | "Looks like the dryer was dealt with — press the button to confirm." | Immediately |
| Button press | "Dryer marked as empty." | Immediately |
| `LIKELY_ATTENDED` self-resolve | "Dryer marked as empty (auto-confirmed)." | Configurable |

**DnD behavior:** All dryer notifications respect DnD. Wrinkles are not a reason to wake anyone up.

---

## Integration Points

### Upstream: Cycle Detection

Each attention state machine subscribes to its appliance's cycle events:
- `highland/event/appliance/washing_machine/cycle_finished` → `UNATTENDED`
- `highland/event/appliance/washing_machine/cycle_started` → `RUNNING`
- `highland/event/appliance/dryer/cycle_finished` → `UNATTENDED`
- `highland/event/appliance/dryer/cycle_started` → `RUNNING`

### PIR Presence (Z2M → MQTT)

Z2M publishes PIR motion events to its device topic. A shared handler in Node-RED normalizes these into presence events and fans out to both attention state machines.

Raw Z2M PIR data is not published to the `highland/` bus.

### Buttons (Z2M → MQTT)

Two WXKG13LM devices, one per machine. Z2M publishes to separate per-device topics. Each button's single press routes to its respective attention state machine only.

### MQTT Topics

**`highland/state/appliance/washing_machine/attention`** ← RETAINED
**`highland/state/appliance/dryer/attention`** ← RETAINED

```json
{
  "timestamp": "2026-04-03T10:00:00Z",
  "source": "washer_attention",
  "state": "UNATTENDED",
  "cycle_finished_at": "2026-04-03T09:30:00Z",
  "presence_last_detected_at": null
}
```

`state` values: `IDLE` | `RUNNING` | `UNATTENDED` | `LIKELY_ATTENDED`

**`highland/event/appliance/washing_machine/attention_changed`**
**`highland/event/appliance/dryer/attention_changed`**

```json
{
  "timestamp": "2026-04-03T10:00:00Z",
  "source": "washer_attention",
  "previous_state": "UNATTENDED",
  "new_state": "IDLE",
  "trigger": "button"
}
```

`trigger` values: `button` | `voice` | `presence_heuristic` | `timeout` | `cycle_started` | `cycle_finished`

### HA Entities (via MQTT Discovery)

Per appliance:

| Entity | Type | Notes |
|--------|------|-------|
| Attention State | `sensor` | `state` field |
| Needs Attention | `binary_sensor` | `ON` when state is `UNATTENDED` or `LIKELY_ATTENDED` |

---

## Node-RED Flow Architecture

One flow per appliance, structurally identical. A shared presence handler feeds both.

```
[MQTT In: cycle_finished / cycle_started]
        │
        ▼
[Function: Attention State Machine]
        │
        ├──► [MQTT Out: highland/state/appliance/{a}/attention]         retained
        ├──► [MQTT Out: highland/event/appliance/{a}/attention_changed]
        └──► [Function: Notification Builder]
                    │
                    ▼
             [MQTT Out: highland/event/notify]

[MQTT In: Z2M PIR topic]
        │
        ▼
[Function: Presence Handler]   ← shared; fans out to both state machines
        │
        ├──► [Link Out → Washer Attention State Machine]
        └──► [Link Out → Dryer Attention State Machine]

[MQTT In: Z2M washer button topic]
        │
        ▼
[Function: Button Handler]     ← filters single press
        │
        ▼
[Link Out → Washer Attention State Machine]

[MQTT In: Z2M dryer button topic]
        │
        ▼
[Function: Button Handler]     ← filters single press
        │
        ▼
[Link Out → Dryer Attention State Machine]

[Inject: nag timer — washer]
[Inject: nag timer — dryer]
        │
        ▼
[Link Out → respective Attention State Machine]
```

---

## Configuration Parameters

### Shared (both appliances)

| Parameter | Default | Notes |
|-----------|---------|-------|
| `attended_presence_duration_s` | 180 | Continuous presence to trigger `LIKELY_ATTENDED` (3 min) |
| `likely_attended_timeout_hours` | 2 | Hours before `LIKELY_ATTENDED` self-resolves |
| `notify_on_self_resolve` | false | Notification on auto-timeout resolution |

### Washing Machine

| Parameter | Default | Notes |
|-----------|---------|-------|
| `nag_first_minutes` | 30 | Minutes in `UNATTENDED` before first nag |
| `nag_repeat_minutes` | 30 | Minutes between subsequent nags |
| `nag_dnd_override_minutes` | 60 | DnD overridden after this duration — hygiene concern |

### Dryer

| Parameter | Default | Notes |
|-----------|---------|-------|
| `nag_first_hours` | 1 | Hours in `UNATTENDED` before first nag |
| `nag_repeat_hours` | 2 | Hours between subsequent nags |
| `nag_dnd_override` | false | Dryer never overrides DnD |

---

## Open Questions / Pending Actions

- **Button mounting locations** — Two buttons needed, clearly distinguishable. TBD pending physical assessment of laundry room options.
- **PIR sensor selection** — Any standard Zigbee PIR supported by Z2M. Mount aimed at machines, not doorway. Coverage angle to be confirmed at install.
- **PIR mounting position** — Confirm line of sight to machines without doorway in detection cone.
- **Nag timer calibration** — Starting points: 30-minute washer cadence, 1-hour dryer cadence. Tune based on actual usage patterns.
- **Presence duration threshold** — 3 minutes for `LIKELY_ATTENDED` is a reasoned estimate. Tune after observing guest laundry behavior in logs.
- **Voice integration** — "Marvin, washer/dryer is empty" as future input path. No state machine changes required when voice pipeline arrives.

---

## Changelog

| Date | Change |
|------|--------|
| 2026-04-03 | Initial commit. Washer and dryer attention layer above cycle detection. Vibration sensors ruled out (shared pedestal). Contact sensors ruled out (bubble door geometry). PIR presence established as shared notification suppression + guest heuristic signal. Button (WXKG13LM, one per machine) as sole state confirmation path. Notification cadence asymmetric: washer aggressive (hygiene clock), dryer relaxed (wrinkles only, never overrides DnD). |
