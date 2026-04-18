# Stair Lighting — Design & Architecture

## Overview

Continuous wall-side LED accent lighting along the 14-step main staircase. Motion-triggered with direction-aware cascade animation. Active window gated by a combination of solar schedule and outdoor lux override. No dependency on HA for runtime logic — HA is dashboard-only.

**Design philosophy:** Node-RED owns decisions (mode, active window, traversal FSM). WLED executes choreography via a preset catalog. ToF sensor nodes publish motion events. Each layer has one job.

---

## Implementation Status

✅ **Designed — not yet implemented.** Hardware partially on hand (WLED controller). Remaining BOM to be acquired before build.

---

## Hardware

### WLED Controller — GLEDOPTO GL-C-015WL-D

ESP32-based WLED controller. FCC Part 15 compliant, flame-retardant PC enclosure, rated indoor-only.

| Spec | Value |
|------|-------|
| Input voltage | DC 5–24V |
| Total output current | 15A max |
| Per-channel output | 10A max |
| Output channels | 3 (GPIO16 default, GPIO2 configurable, IO33 extended) |
| Firmware | WLED (ships flashed; reflashable via Micro-B UART port) |
| Dimensions | 108 × 45 × 18 mm |

**Features relevant to Highland:**
- Native MQTT support with LWT
- JSON REST API (also accessible via MQTT `/api` topic)
- MOSFET relay cuts strip power when WLED output is off — idle quiescent current approaches zero
- Onboard microphone (sound-reactive modes) — **disabled** in Highland config, not used

**Phase 1 uses a single output channel.** With the shift to a continuous wall-side strip, the split-channel option originally considered is no longer advantageous — data runs are short enough that channel-splitting provides no meaningful integrity benefit. Channels 2 and 3 remain available for future subsystems.

### LED Strips — RGB IC FCOB (WS2811 Protocol)

Twelve-volt addressable RGB IC FCOB (flexible chip-on-board) strip, single continuous run mounted along the wall side of the staircase following the stair slope. BTF-Lighting branded.

| Spec | Value |
|------|-------|
| Voltage | 12V DC |
| Physical LED density | 630–720 LEDs/m (vendor SKU dependent) |
| Addressable pixel density | ~90 pixels/m (one IC group per ~11 cm) |
| Protocol | WS2811 (IC FCOB standard at 12V) |
| Run length | ~15 ft (stairs only) to ~19 ft (stairs + upper landing extension) |
| Addressable pixels (total) | ~410–520 |
| Theoretical max draw | ~60–70W (full white, full brightness, 720/m) |
| Realistic working draw | 15–30W (motion-triggered operation) |

**Choice rationale:** RGB IC FCOB over SMD strip (e.g. WS2815) because the installation is viewed at close range. COB construction places hundreds of tiny LEDs per meter under a continuous phosphor coating, producing a dotless line of light. SMD strips at 60 LEDs/m show visible "string of pearls" dots at close viewing distance, which is undesirable for a stair-adjacent accent where people pass within a few feet of the strip.

**Protocol and controller compatibility:** WS2811 at 12V is natively supported by the GL-C-015WL-D. WLED handles WS2811 IC FCOB out of the box with no special configuration beyond selecting the correct LED type and count.

**Tradeoff vs WS2815 accepted:** WS2815 was the original spec primarily for its dual data line redundancy over a longer discrete-segment run. With the shift to a short continuous strip, that robustness advantage no longer applies meaningfully, and the visual quality advantages of COB dominate. RGB IC FCOB is a single-data-line protocol — a dead pixel IC kills pixels downstream of it — but failure rates are low and the entire strip would be a single replacement unit if ever necessary.

**Density choice (630 vs 720):** Both densities produce dotless light at close viewing distance — human perception is already saturated at 630/m. Let availability, price, and IP rating drive the SKU decision rather than density.

**Landing inclusion is an open question** — whether the strip terminates at the top step or extends across the upper landing is a design decision pending further evaluation. See Open Questions.

### Power Supply — Meanwell LRS-150-12

150W, 12V DC. Sized for realistic working load with comfortable headroom for full-brightness emergency scenarios at the RGB IC FCOB's higher physical LED density.

**Location:** Adjacent bedroom. No usable outlet exists at the top or bottom of the stairs, and the stair underside is a finished closet (no access). PSU in the bedroom is Phase 1 solution; an electrician-installed outlet at the top of the stairs is a possible future refinement.

**12V DC distribution run:** 15–20 ft from PSU to top-of-stairs controller location, 14 AWG minimum. Voltage drop at 14 AWG × 20 ft × 6A (realistic emergency-bright draw) is well under 4%, well within tolerance. Low-voltage DC wiring is Class 2 and has no electrical code restrictions — routable through wall channels without permits or licensed trades.

### Motion Sensors — M5Stack Atom Stack

Two identical sensor nodes, one at the top of the stairs and one at the bottom. Each runs ESPHome with direct MQTT publishing (no HA auto-discovery).

| Component | Purpose | Approx. dimensions |
|-----------|---------|--------------------|
| M5Stack Atom Lite (or AtomS3 Lite) | ESP32/ESP32-S3 controller | 24 × 24 × 10–13 mm |
| M5Stack Atomic RS485 Base | Dock providing 12V→5V DC-DC step-down, terminal block input, and enclosure | ~24 × 24 × ~15 mm (stacks under Atom) |
| M5Stack ToF Unit — U010 (VL53L0X) *primary* or U172 (VL53L1X) *fallback* | Time-of-Flight distance sensor with Grove connector | ~30 × 24 × 13 mm |

**Per-node stack form factor:** ~24 × 24 × 20–25 mm (Atom + RS485 Base combined). ToF Unit tethers via Grove cable, positioned at the channel's optical port.

**Node symmetry:** Both nodes run identical hardware and firmware, differing only in MQTT client ID. This simplifies spares and BOM.

**Sensing approach:** ToF distance threshold crossing generates a motion event. Thresholds are applied in ESPHome itself — MQTT sees boolean events, not raw distance values. Mounting geometry aims the sensor beam across the first/last step at ankle-to-knee height.

**Why ToF over PIR/mmWave:** PIR has sluggish warmup/cooldown (tens of seconds to reliable detection); the motion cascade needs to fire in under 100 ms. mmWave is geometrically hostile to staircases — the sensors want flat-plane coverage from above, not a vertical slope.

#### Why this assembly

The three-component stack was chosen specifically to eliminate breadboarding:

- **No discrete buck converter.** The Atomic RS485 Base has a built-in 12V→5V DC-DC step-down regulator marketed for powering the Atom from RS485's 12V supply rail. For our purposes the RS485 functionality is unused — we consume the Base purely for its packaged 12V→5V step-down, terminal block input, and enclosure.
- **No buck calibration.** The RS485 Base's regulator output is fixed at 5V, eliminating the risk of accidentally over-volting the Atom during bench assembly.
- **No soldering.** Atom stacks onto Base via pin header, ToF Unit plugs into Grove port, 12V enters via terminal block. Bench assembly is plug-and-play.

**Unused RS485 chip consideration:** The RS485 transceiver on the Base is electrically connected to the Atom's UART pins (typically G19/G22). Since we're not driving RS485, those pins remain idle — no conflict with our GPIO needs (I²C is on G26/G32 via the Grove port).

#### Sensor SKU selection: U010 vs U172

The ToF Unit has two variants, both with identical Grove plug-and-play form factors and both lacking externally-accessible XSHUT pins:

| SKU | Sensor | Range | Field of view | ESPHome support |
|-----|--------|-------|---------------|-----------------|
| **U010** | VL53L0X | 2 m | 25° fixed | Native first-class |
| **U172** | VL53L1X | 4 m | 15–27° configurable ROI | External community component |

**Primary choice: U010.** Native ESPHome support makes it the lower-maintenance option. The 25° FoV and 2 m range are adequate for stair traversal detection in the expected mounting geometry.

**Fallback: U172.** If bench POC reveals that the U010's wider fixed FoV picks up unwanted reflections from the opposite wall or other objects, pivoting to the VL53L1X's narrower configurable ROI is straightforward. The VL53L1X requires an `external_components:` reference in the ESPHome YAML (community component, e.g. `soldierkam/vl53l1x_sensor`), which is well-trodden but adds a community dependency.

#### GPIO assignment

Atom Lite exposes eight usable GPIO pins — G26, G32 via the Grove port, plus G19, G21, G22, G23, G25, G33 on edge headers. Our needs are modest:

| Function | Atom Lite GPIO | Connection |
|----------|----------------|------------|
| I²C SDA (ToF Unit) | G26 | Grove port pin 1 |
| I²C SCL (ToF Unit) | G32 | Grove port pin 2 |
| RS485 TX (unused, tied to transceiver) | G19 | RS485 Base internal |
| RS485 RX (unused, tied to transceiver) | G22 | RS485 Base internal |

Remaining free: G21, G23, G25, G33 (G25 is shared with the onboard IR LED and wants care).

### Sensor Recovery

Neither the U010 nor U172 exposes the VL53Lxx's XSHUT pin externally in the M5Stack packaging, so we cannot power-cycle just the sensor chip from ESPHome. Recovery from a hung sensor follows a layered approach:

**Primary: Atom-level reboot via ESPHome watchdog.** If the sensor stops producing readings for a configurable interval (provisional: 5 minutes), ESPHome triggers a full Atom reboot via its built-in `restart` component. The Atom goes offline for ~10 seconds during boot — acceptable for a non-safety-critical motion sensor. LWT publishes `offline` during boot and `online` on recovery; Highland's standard device-monitoring logic applies.

**Secondary: Node-RED-initiated reboot.** If longer-term observability reveals pattern failures the ESPHome watchdog misses (e.g., sensor returns stuck values rather than stopping entirely), Node-RED's device monitoring flow can publish to `highland/command/sensor/stairs_*/reboot`. ESPHome subscribes and executes `restart`. Same practical outcome, centrally orchestrated.

**Tertiary (planned refinement): MOSFET-switched sensor power.** If bench POC or production experience reveals that Atom reboots do not reliably recover the sensor (i.e., the sensor maintains its hung state across the Atom's I²C re-init), add a small GPIO-driven MOSFET between the Atom's 5V rail and the Grove port's VCC. ESPHome then controls sensor power independently of Atom state. Cost: one transistor plus a dedicated GPIO. Not required for Phase 1.

---

## Physical Installation

### Channel Profile — Dual-Chamber ("H-Profile")

**Approach:** Aluminum LED channel with two chambers, mounted along the **wall side of the staircase** following the stair slope as a single continuous run.

- **Front chamber** (visible): houses the LED strip and frosted diffuser. Produces the continuous line of light.
- **Rear chamber** (concealed, wall-side): houses the Atom + ToF + buck electronics bricks at each end, plus the 12V/GND/data trunk conductors along the full run.

Commonly marketed as "dual-chamber," "H-profile," "architectural LED extrusion with raceway," or "LED profile with cable channel." Terminology is inconsistent across vendors — shop by cross-section, not name.

**Why dual-chamber over single-chamber U:**

- The Atom + RS485 Base stack plus Grove-tethered ToF Unit would not fit cleanly in a standard single-chamber U alongside the strip. Dual-chamber allows the front chamber to carry a continuous uninterrupted light line edge-to-edge while the electronics hide behind it.
- Cable routing for the full 15–19 ft trunk (12V, GND, data) lives invisibly in the rear raceway — no zip ties, clips, or tacky wire hiding along the strip itself.
- Lateral ToF optical port through the front face of the channel is short (just the channel wall thickness), not through a deeper single-chamber volume.

**Trade-off:** Dual-chamber profiles run $15–25/m versus $5–8/m for basic U-channel. For the ~5–6 m run, the delta is $50–100 — accepted for the aesthetic result.

**Target profile dimensions (ranges for sourcing):**

| Chamber | Approximate interior dimensions |
|---------|-------------------------------|
| Front | ≥ 10 mm wide, ≥ 8 mm deep; accommodates ~10 mm strip + snap-in frosted PC diffuser |
| Rear | ≥ 30 mm wide, ≥ 25 mm deep; accommodates the ~24 × 24 × 25 mm Atom+Base stack plus ToF Unit and service loop |
| Overall profile | ~40 mm wall-face width × ~30 mm protrusion from wall |

**Sourcing direction:** Klus ("PAC" and "GIP" series have rear cavities; custom orders available), Alumbrera, LED Profiles Europe, Alcon Lighting, BudgetLightingOnline (stocks Klus). Most commercial channels ship in 1 m or 2 m lengths — splicing will be required for a 15–19 ft run. Clean joints are worth sourcing carefully.

**Final profile selection is deferred to bench POC.** Small-scale testing with sample lengths of candidate profiles will verify assembly fit, diffuser appearance, and mounting practicality before committing to the production SKU.

### Mounting Height

Open question — three candidate positions on the wall:

- **Top edge of the skirt board** (light skims across the treads; closest visual analog to under-nosing lighting)
- **Mid-wall above the skirt** (decorative horizontal line; illuminates less of the stair surface itself)
- **Recessed into the skirt board** (most invisible hardware, requires finish woodwork on the channel-wall side)

Decision pending bench POC and in-situ evaluation. See Open Questions.

### Rationale for Wall-Side Mounting

Two independent constraints ruled out both under-tread and riser-face mounting:

- **Under-tread:** The stair underside is a finished closet with no access. True under-tread mounting would require opening the closet ceiling.
- **Riser-face:** Molding installed beneath each nosing leaves insufficient clearance for an LED channel, ruling out the shadow-line placement originally considered.

Wall-side mounting sidesteps both constraints and — with the dual-chamber profile — delivers a cleaner installation than either under-tread alternative: a continuous light line that follows the stair geometry, with all electronics and cabling hidden behind it.

### Sensor Integration

Both sensor nodes (top and bottom) are fully integrated into the channel itself. No separate sensor enclosures or visible mounting hardware.

**Assembly placement:** Each Atom + RS485 Base stack sits inside the rear chamber at the end of the channel — one at the top of the run, one at the bottom. The Grove-tethered ToF Unit sits slightly forward, positioned so its optical face aligns with the fabricated port in the channel wall. Location within the rear chamber is dictated by where the ToF sensor needs to see out.

**ToF optical port:** The VL53L0X's (or VL53L1X's) 940 nm laser cannot see through a frosted diffuser. Optical path is provided by:

- A hole drilled through the channel wall at the ToF location (side-facing or front-facing, depending on desired beam direction)
- A small clear window bonded over the hole — thin clear acrylic or polycarbonate, 10–15 mm square, CA-glued into a shallow recess cut into the channel exterior
- ToF Unit positioned in the rear chamber with its optical face aligned against the window

Fabrication is minor: drill, shallow countersink with a Dremel, cut acrylic square to fit, glue. Estimated 15 minutes per node at the workbench.

**Power entry:** 12V and GND from the strip's trunk conductors land at the RS485 Base's terminal block screws. The Base's internal regulator steps down to 5V for the Atom; no external buck or filtering components are needed.

**Thermal considerations:** At motion-triggered duty cycles (seconds of full brightness at a time, not continuous), ambient temperature inside the channel during operation is not expected to approach the Atom's or Base's limits. If prolonged emergency-bright operation became common, additional thought would be warranted. Placement of the assemblies at the *ends* of the channel puts them further from the main length of the active LED strip, which additionally mitigates thermal buildup.

### Cable Routing

The rear chamber doubles as a cable raceway. The trunk carries three conductors (12V, GND, data) the full length of the run:

- Top end: terminates at the WLED controller and top Atom (both taps share the 12V/GND feed; data comes from the controller output)
- Bottom end: terminates at the bottom Atom (taps 12V/GND; data line continues no further)

No external wire channels, no visible cabling outside the LED channel itself.

### Power Injection

Single-end feed at the top of the stairs (at the controller) is expected sufficient at 15–19 ft with the RGB IC FCOB's characteristics at realistic brightness levels. Bottom-end injection is available as a commissioning-time refinement if visible brightness drop manifests at the far end. Injection cabling, if added, routes through the rear chamber alongside the primary trunk.

### Sensor Placement

**Top node:** Atom stack in rear chamber at top end of channel. ToF optical port angled to sweep across the upper landing and top step at ankle-to-knee height. Stairlift parks "around the corner" from the top landing and does not occlude the ToF beam.

**Bottom node:** Atom stack in rear chamber at bottom end of channel. ToF optical port angled to sweep across the bottom step similarly. Stairlift has a parking position at the bottom that is also around a corner and does not occlude the ToF beam.

**Stairlift consideration:** Non-factor. The lift has parking positions at both the top and bottom of the stairs, but both are around corners and out of ToF line-of-sight. No current clamp, dry-contact, or other instrumentation of the lift is needed.

### Bench POC

Ahead of production installation, a small-scale bench proof-of-concept is planned to validate:

- Actual stacked dimensions of the Atom + RS485 Base + ToF Unit assembly
- Fit of the assembly inside candidate rear-chamber profiles
- Sample lengths of 2–3 candidate channel profiles for visual and mechanical evaluation
- Diffuser appearance at candidate mounting heights
- ToF optical port fabrication technique (drill, cut, clear window bond)
- ESPHome U010 native-component behavior against live sensor; validate detection performance with bench-top simulated traversal
- Escalation path to U172 with external ESPHome component if U010 FoV proves problematic
- Atom-reboot recovery behavior if sensor is deliberately hung

Assembly is intentionally plug-and-play — no breadboarding. Bench build is:

1. Connect 12V bench supply to Atomic RS485 Base's terminal block
2. Stack Atom Lite onto RS485 Base
3. Plug ToF Unit (U010) into Atom's Grove port
4. Flash Atom with ESPHome YAML
5. Confirm MQTT traffic on the broker; validate threshold crossing triggers events

Findings from POC feed final channel SKU selection, mounting height decision, U010-vs-U172 confirmation, and any wiring or housing refinements.

---

## Architecture

```
Tempest station  ──► highland/state/weather/station  ──┐
                                                        │
Schedex (dusk/dawn + offset)  ──► schedule state  ─────┤
                                                        ▼
                                             Active Window Gate
                                              (schedule OR lux)
                                                        │
Top ToF node  ──► highland/event/motion/stairs_top  ───┤
                                                        ▼
Bottom ToF node  ──► highland/event/motion/stairs_bottom  ──►  Traversal FSM
                                                        │
                                                        ▼
                                              Mode Resolver
                                             (priority layers)
                                                        │
                                                        ▼
                                         highland/command/stair_lights/preset
                                                        │
                                                        ▼
                                          WLED (GL-C-015WL-D)
                                                        │
                                                        ▼
                                               LED strip output
```

**Layer responsibilities:**

| Layer | Owns |
|-------|------|
| Sensor nodes (ESPHome) | Raw detection → boolean motion events |
| Tempest flow | Normalized weather state including outdoor lux |
| Active Window Gate (Node-RED) | Combining schedule + lux into on/off gate |
| Mode Resolver (Node-RED) | Priority resolution across mode inputs |
| Traversal FSM (Node-RED) | Direction inference and cascade sequencing |
| WLED | Per-pixel choreography (preset execution) |
| HA | Dashboard display and manual mode overrides |

---

## Active Window Gating

The FSM responds to motion events only during the active window. Outside the active window, motion events are logged but do not drive lighting.

### Two-input gate

```
schedule_active  = schedex dusk→dawn (with configurable offset)
lux_override     = outdoor_lux < threshold (with hysteresis)

stairs_active = schedule_active OR lux_override
```

Either input being true enables motion response. Both being false gates motion events out.

### Schedule input

Driven by schedex solar elevation calculation. The schedule *is* the default; the lux reading handles cases where the schedule disagrees with reality (storm darkness at 2pm, long dim dusks, etc.).

**Configurable offset:** Minutes before/after civil dusk/dawn to begin/end the active window. Starting value: 0 offset. Tune against observed behavior.

### Outdoor lux override

Uses the Tempest station's solar radiation / brightness reading from `highland/state/weather/station`. Threshold logic is applied in the stair lighting flow, not in the Tempest flow (Tempest publishes normalized data; consumers apply their own thresholds).

**Provisional thresholds (tune from real data once collected):**

| Transition | Lux value |
|------------|-----------|
| Enable (darkening) | < 200 |
| Disable (brightening) | > 500 |

Hysteresis prevents flapping during transitional lighting.

**Graceful degradation:** If `highland/status/weather/station` indicates Tempest is offline or lux data is stale (> 10 min), the lux override input is treated as `false` and the gate falls back to schedule-only. Failing to schedule-only is the correct behavior — we can't read outdoor lux, so we trust the clock.

### Thresholds storage

Values live in `config/thresholds.json` under the `stair_lighting` key:

```json
"stair_lighting": {
    "outdoor_lux_enable": 200,
    "outdoor_lux_disable": 500,
    "lux_stale_minutes": 10
}
```

See #45 for ongoing config taxonomy work.

---

## Mode Hierarchy

The stair lighting subsystem operates in one of several modes. Modes are resolved by priority — higher-priority modes override lower.

| Priority | Mode | Behavior |
|----------|------|----------|
| 1 (highest) | `emergency` | Full bright white, overrides all other modes. Triggered by smoke/CO alarm or power recovery. |
| 2 | `manual_override` | User-commanded state from HA dashboard (forced on, forced off, or forced accent). Overrides schedule and motion. |
| 3 | `effect` | Holiday/seasonal/party choreography. Triggered by calendar or scheduled. Phase 2. |
| 4 | `auto` (default) | Schedule-gated motion response. Normal operation. |
| 5 (lowest) | `off` | Disabled. No response to motion events. |

**Mode state:** `highland/state/stair_lights/mode` (retained).

**Mode command:** `highland/command/stair_lights/mode` (not retained).

The Mode Resolver subscribes to all mode inputs and publishes the resolved effective mode. The Traversal FSM only engages when effective mode is `auto` or `effect`.

---

## Motion Detection

### Sensor node topology

Each ESP32 sensor node publishes directly to MQTT via ESPHome's native MQTT component. HA auto-discovery is disabled on these nodes — Node-RED is the authoritative consumer.

**Published topics per node:**

| Topic | Type | Retained | Notes |
|-------|------|----------|-------|
| `highland/event/motion/stairs_top` | Event | No | Rising edge on detection |
| `highland/event/motion/stairs_bottom` | Event | No | Rising edge on detection |
| `highland/status/sensor/stairs_top` | LWT | Yes | `online` \| `offline` |
| `highland/status/sensor/stairs_bottom` | LWT | Yes | `online` \| `offline` |

### ESPHome threshold configuration

ToF distance threshold is set in ESPHome itself (not in Node-RED). The node publishes a motion event only when distance crosses the threshold from above. Hysteresis and debouncing are handled locally for fastest response.

**Provisional ToF configuration:**

| Parameter | Value |
|-----------|-------|
| Detection threshold | 150 cm |
| Debounce time | 100 ms |
| Minimum event spacing | 500 ms (rate limit) |

Tune after installation against real traversal patterns.

### Direction Inference FSM

Direction is inferred from **which sensor fires first**:

| First trigger | Direction |
|---------------|-----------|
| `stairs_bottom` | Ascending |
| `stairs_top` | Descending |

The second sensor triggering confirms traversal (both sensors have seen the same person pass through). If the second sensor does not trigger within the traversal timeout, the FSM returns to idle assuming the person turned back or stopped.

---

## Traversal FSM

### States

| State | Description |
|-------|-------------|
| `idle` | No active traversal. Subscribed to motion events. |
| `ascending` | Bottom sensor fired first; cascade running bottom→top. |
| `descending` | Top sensor fired first; cascade running top→bottom. |
| `traversing` | Both sensors have fired; full-brightness hold until exit timeout. |
| `fading` | Cooldown fade-out after traversal timeout. |

### Transitions

```
idle ──(bottom_motion)──► ascending
idle ──(top_motion)─────► descending

ascending ──(top_motion)──► traversing
ascending ──(ascending_timeout)──► fading

descending ──(bottom_motion)──► traversing
descending ──(descending_timeout)──► fading

traversing ──(traversing_timeout)──► fading

fading ──(fade_complete)──► idle
```

### Timeouts (provisional)

| Timeout | Value | Notes |
|---------|-------|-------|
| `ascending_timeout` | 8 s | Max time from bottom trigger to top trigger. Calibrate against stairlift traversal time if that's ever a factor. |
| `descending_timeout` | 8 s | Same, top→bottom |
| `traversing_timeout` | 5 s | Hold-at-full after both sensors have fired |
| `fade_time` | 2 s | Smooth fade back to idle/accent state |

### Edge cases

**Simultaneous traversal (two people, opposite directions):** Both sensors fire nearly together. The FSM will see whichever fired microseconds first and cascade in that direction, then immediately enter `traversing` when the other fires. The cascade animation may feel "wrong" for one of the two people, but the lighting itself is correct (full brightness during traversal). Acceptable.

**Person stops mid-staircase:** Ascending/descending timeout expires before the far sensor fires. FSM enters `fading` and returns to idle. If the person resumes motion, the nearer sensor fires again and a fresh cascade starts.

**Pets on stairs:** ToF beam mounted at ankle-to-knee height will miss most cats and small dogs. Tall dogs may trigger — acceptable false positive, cats get a light show occasionally. If tuning becomes necessary, raise the sensor mounting height.

---

## WLED Preset Catalog

Choreography lives as WLED presets defined in the WLED UI. Node-RED references presets by ID — it does not manipulate pixels directly.

### Preset IDs

| ID | Name | Purpose |
|----|------|---------|
| 1 | `off` | All pixels off |
| 2 | `accent_dim_warm` | Steady low-level warm glow (manual always-on mode) |
| 3 | `cascade_up` | Ascending animation — light pixels bottom→top |
| 4 | `cascade_down` | Descending animation — light pixels top→bottom |
| 5 | `hold_full` | Full brightness, warm white — traversal hold state |
| 6 | `fade_out` | Smooth fade from current to off |
| 7 | `cascade_up_dim` | Late-night dimmed ascending (reduced brightness, warmer color) |
| 8 | `cascade_down_dim` | Late-night dimmed descending |
| 9 | `hold_dim` | Late-night dimmed hold |
| 10 | `emergency_bright_white` | Full bright cool white, no animation |
| 20+ | `effect_*` | Holiday/seasonal presets (Phase 2) |

Preset IDs are referenced from Node-RED; preset definitions (colors, speeds, brightness) are tuned in the WLED UI during commissioning.

### Time-of-day variants

The FSM selects which preset set to use based on time of day:

| Window | Ascending | Descending | Hold |
|--------|-----------|------------|------|
| Evening (active window start → 22:00) | 3 | 4 | 5 |
| Late night (22:00 → 05:00) | 7 | 8 | 9 |
| Pre-dawn (05:00 → active window end) | 3 | 4 | 5 |

Windows are configurable. Schedex publishes time-of-day state; the FSM reads it at traversal start.

---

## MQTT Topics

### State Topics (Retained)

| Topic | Payload |
|-------|---------|
| `highland/state/stair_lights/mode` | Effective mode: `auto` \| `off` \| `manual_override` \| `effect` \| `emergency` |
| `highland/state/stair_lights/fsm` | Traversal FSM state: `idle` \| `ascending` \| `descending` \| `traversing` \| `fading` |
| `highland/state/stair_lights/active_window` | Active window state: `true` \| `false` |
| `highland/state/stair_lights/preset` | Current WLED preset ID (informational) |

### Event Topics (Not Retained)

| Topic | Fires on |
|-------|----------|
| `highland/event/motion/stairs_top` | Top ToF detection |
| `highland/event/motion/stairs_bottom` | Bottom ToF detection |
| `highland/event/stair_lights/traversal` | Completed traversal (ascending or descending) |

### Command Topics (Not Retained)

| Topic | Payload |
|-------|---------|
| `highland/command/stair_lights/mode` | `{"mode": "auto" \| "off" \| "manual_override" \| "effect"}` |
| `highland/command/stair_lights/preset` | `{"preset_id": N}` — direct preset trigger (debug/testing) |

### Status Topics (LWT-Retained)

| Topic | Source | Payload |
|-------|--------|---------|
| `highland/status/sensor/stairs_top` | Top ESP32 | `online` \| `offline` |
| `highland/status/sensor/stairs_bottom` | Bottom ESP32 | `online` \| `offline` |
| `highland/status/wled/stairs` | WLED controller | WLED native LWT |

### WLED Native Topics

WLED publishes to its own native topic namespace by default. Node-RED bridges between Highland's namespace and WLED's:

- `wled/stairs/api` ← Node-RED publishes preset commands here (WLED API string format)
- `wled/stairs/state` ← WLED state (JSON)
- `wled/stairs/g` ← WLED brightness
- `wled/stairs` ← WLED online/offline LWT

The bridging flow subscribes to Highland topics and translates to WLED topics.

---

## Configuration

### `config/thresholds.json`

Subsystem decision thresholds under the `stair_lighting` key (see #45 for convention).

```json
"stair_lighting": {
    "outdoor_lux_enable": 200,
    "outdoor_lux_disable": 500,
    "lux_stale_minutes": 10
}
```

### Subsystem parameters (location TBD)

Timing parameters, preset mappings, and active window offsets. These are not thresholds and may land in a dedicated subsystem config file, or in an environment variables block on the Node-RED tab, pending resolution of the broader config taxonomy in #45.

```json
{
    "timeouts": {
        "ascending_seconds": 8,
        "descending_seconds": 8,
        "traversing_seconds": 5,
        "fade_seconds": 2
    },
    "active_window": {
        "schedule_offset_minutes": 0
    },
    "time_of_day": {
        "late_night_start": "22:00",
        "late_night_end": "05:00"
    },
    "presets": {
        "off": 1,
        "accent_dim_warm": 2,
        "cascade_up": 3,
        "cascade_down": 4,
        "hold_full": 5,
        "fade_out": 6,
        "cascade_up_dim": 7,
        "cascade_down_dim": 8,
        "hold_dim": 9,
        "emergency": 10
    }
}
```

---

## HA Integration

HA exposes a dashboard tile for manual mode control. Entities are created via MQTT Discovery published by Node-RED (not by WLED's native HA integration — which would violate the Node-RED-as-authoritative principle).

| Entity | Type | Purpose |
|--------|------|---------|
| `select.stair_lights_mode` | `select` | Dropdown for mode selection (off/auto/manual_override) |
| `sensor.stair_lights_fsm` | `sensor` | Informational FSM state |
| `binary_sensor.stair_lights_active_window` | `binary_sensor` | Whether motion is currently armed |
| `sensor.stair_lights_preset` | `sensor` | Current preset (informational) |

WLED's native HA integration is **not enabled** for this controller. All HA visibility is through Node-RED-published Discovery entities subscribed to Highland topics.

---

## Phase Plan

### Phase 1 — Core Subsystem (This Design)

- GLEDOPTO GL-C-015WL-D controller, single output channel
- Single continuous RGB IC FCOB strip (WS2811 protocol, 12V)
- Dual-chamber H-profile aluminum channel with frosted diffuser
- Two M5Stack Atom sensor nodes (Atom Lite + Atomic RS485 Base + ToF Unit U010), fully integrated into channel rear cavity
- ToF optical ports fabricated in channel walls with clear windows
- Motion-triggered operation with schedule + lux gating
- HA dashboard for manual mode control
- No physical switches

### Phase 2 — Enhancements

- **ZEN37 magnetic wall remotes** at top and bottom of stairs. Magnetic fake-wall-plate mount — no switched wiring required. Pattern matches the garage bay remote. Gives the user manual bump-to-full and mode cycling without requiring phone access.
- **Effect presets** — holiday/seasonal choreography triggered by calendar or scheduled. Halloween, Christmas, birthdays, etc.
- **Emergency triggers** — wire in smoke/CO alarm state, power recovery, and security alarm state to drive `emergency` mode. Requires the security subsystem to be live.

### Phase 3 — Speculative

- Electrician-installed outlet at top of stairs, relocating PSU from bedroom
- Second staircase (if applicable)

---

## Open Questions

- [ ] Landing inclusion — strip terminates at top step, or extends across upper landing?
- [ ] Wall-side mounting height — skirt-top, mid-wall, or recessed into skirt board?
- [ ] Dual-chamber channel SKU selection — confirm interior dimensions fit the Atom+Base stack plus ToF Unit, evaluate diffuser appearance
- [ ] ToF optical port fabrication technique — refine at bench POC (drill size, countersink depth, window material and bond)
- [ ] Splice plan for the full run — most commercial channels ship in 1 m / 2 m lengths
- [ ] Atom variant — Atom Lite vs AtomS3 Lite; confirm RS485 Base compatibility with whichever is selected
- [ ] Confirm U010 FoV is adequate in actual install geometry; pivot to U172 if opposite-wall reflections cause false triggers
- [ ] Validate Atom-reboot recovery as sufficient for hung-sensor scenarios; promote MOSFET-switched power to Phase 1 if not
- [ ] Finalize ToF aiming geometry against real sensor performance — cat false positives vs. missed detections at knee height
- [ ] Calibrate outdoor lux thresholds from observed Tempest data over several weeks
- [ ] Decide home for subsystem parameters (timeouts, preset mappings) — dependent on #45 outcome
- [ ] Validate PSU headroom against measured full-brightness draw at commissioning
- [ ] Determine whether schedex time-of-day state deserves its own utility flow or lives in the stair lighting flow directly

---

*Last Updated: 2026-04-18*
