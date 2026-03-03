# Entity Naming Standards

## Overview

Consistent entity naming conventions for Home Assistant, designed to be intuitive, scalable, and resistant to breakdown as the system grows.

---

## Core Principles

1. **Broad to specific** — Start with area, end with the most specific identifier
2. **Underscores throughout** — Consistent with MQTT topic conventions
3. **Function over location** — Name by what it does when purpose is known
4. **Stable identifiers** — Name by the most permanent attribute
5. **Groups + individuals** — Both are addressable; groups for convenience, individuals for granular control

---

## Entity ID Pattern

```
{domain}.{area}_{device_or_function}[_{qualifier}]
```

| Component | Description | Examples |
|-----------|-------------|----------|
| `{domain}` | HA domain (fixed by integration) | `light`, `switch`, `sensor`, `binary_sensor`, `lock` |
| `{area}` | Physical area (lowercase, underscores) | `living_room`, `garage`, `front_porch`, `master_bedroom` |
| `{device_or_function}` | What it is or what it does | `overhead`, `carriage`, `environment`, `outlet` |
| `{qualifier}` | Optional disambiguation | `left`, `temperature`, `floor_lamp`, `west_1` |

---

## Disambiguation Hierarchy

When multiple similar devices exist in an area, disambiguate in this priority order:

| Priority | Method | Use When | Example |
|----------|--------|----------|---------|
| 1 | **Function** | Device has a dedicated purpose | `outlet_floor_lamp` |
| 2 | **Landmark** | Nearby stable reference point | `outlet_by_fireplace` |
| 3 | **Position** | Unique wall/corner/location | `outlet_east_wall` |
| 4 | **Greek letter** | Multiple at same position; general purpose | `outlet_west_alpha`, `outlet_west_beta` |

**Greek alphabet sequence:** alpha, beta, gamma, delta, epsilon, zeta, eta, theta, iota, kappa...

**Rationale:** Naming should reflect the most *stable* identifier. Functions change (lamp moves), landmarks are usually stable (fireplace stays), positions are permanent (outlet #1 is always physical outlet #1). Greek letters over numbers convey intentionality — a human made a deliberate choice rather than auto-assignment.

---

## Friendly Names

**Principle:** Omit area from friendly names. Area context is provided by HA's area assignment and inferred by voice assistants.

| Entity ID | Friendly Name | Voice Command (in room) |
|-----------|---------------|------------------------|
| `light.living_room_overhead` | Overhead Light | "Turn on the overhead light" |
| `light.garage_carriage` | Carriage Lights | "Turn on the carriage lights" |
| `sensor.garage_environment_temperature` | Environment Temperature | — |

**Cross-room voice commands** still work with full context:
- "Turn on the master bedroom overhead light" (from living room)

---

## Device Type Patterns

### Lights

**Single light:**
```
light.{area}_{function}
light.living_room_overhead
light.front_porch_entry
```

**Multiple lights (same function, grouped):**
```
light.{area}_{function}              → group entity
light.{area}_{function}_{position}   → individuals

light.garage_carriage                → group (use for normal control)
light.garage_carriage_left           → individual (granular control)
light.garage_carriage_right          → individual (granular control)
```

**Multiple lights (different functions):**
```
light.living_room_overhead
light.living_room_table_lamp
light.living_room_floor_lamp
```

---

### Switches (including smart outlets)

**Dedicated function:**
```
switch.{area}_outlet_{function}
switch.living_room_outlet_floor_lamp
switch.office_outlet_monitor
```

**By landmark:**
```
switch.{area}_outlet_{landmark}
switch.living_room_outlet_by_fireplace
switch.living_room_outlet_behind_tv
```

**Fallback (Greek letters):**
```
switch.{area}_outlet_{position}_{greek}
switch.living_room_outlet_west_alpha
switch.living_room_outlet_west_beta
```

---

### Sensors

**Multi-sensors (environment):**
```
sensor.{area}_environment                    → device grouping (if exists)
sensor.{area}_environment_temperature        → specific reading
sensor.{area}_environment_humidity           → specific reading
sensor.{area}_environment_illuminance        → specific reading
```

**Single-purpose sensors:**
```
sensor.{area}_{function}
sensor.outdoor_temperature
sensor.garage_air_quality
```

**Motion sensors:**
```
binary_sensor.{area}_motion
binary_sensor.{area}_motion_{qualifier}      → if multiple

binary_sensor.garage_motion
binary_sensor.garage_motion_driveway
binary_sensor.garage_motion_interior
```

---

### Contact Sensors (doors/windows)

The door/window is a logical grouping; `_contact` specifies the device type:

```
binary_sensor.{area}_{object}_contact
binary_sensor.foyer_entry_door_contact
binary_sensor.garage_side_door_contact
binary_sensor.master_bedroom_window_east_contact
```

---

### Locks

The `lock.` domain already indicates device type, so no suffix needed:

```
lock.{area}_{door}
lock.foyer_entry_door
lock.garage_side_door
lock.back_patio_door
```

**Door as logical grouping (full example with smart lock):**
```
lock.foyer_entry_door                         → control entity (send lock/unlock)
binary_sensor.foyer_entry_door_lock           → state sensor (locked/unlocked)
binary_sensor.foyer_entry_door_contact        → contact sensor (open/closed)
```

All three reference the same physical door; device type suffix differentiates.

---

### Leak Sensors

```
binary_sensor.{area}_leak_{location}
binary_sensor.basement_leak_water_heater
binary_sensor.kitchen_leak_dishwasher
binary_sensor.laundry_leak_washer
```

---

### Covers (blinds, shades, garage doors)

```
cover.{area}_{type}[_{qualifier}]
cover.living_room_blinds
cover.living_room_blinds_east
cover.garage_door
cover.garage_door_main
cover.garage_door_side
```

---

## Groups

Zigbee/Z-Wave groups follow the same pattern as their member devices, without a qualifier:

```
light.garage_carriage                → group
light.garage_carriage_left           → member
light.garage_carriage_right          → member
```

**Group behavior:**
- Groups are the primary control entity for normal use
- Individual members remain addressable for granular control (e.g., alternating brightness)
- Group entity controls timing synchronization

---

## System Entities

System entities (integrations, HA internals, etc.) are **excluded** from these standards.

- Don't rename them to match conventions
- If Node-RED needs to reference them consistently, create an abstraction layer (mapping) rather than fighting the entity ID

---

## Floors, Zones &amp; Areas

Home Assistant areas organized by floor and logical zone. The `area_id` is the lowercase/underscore form used in entity IDs and MQTT topics.

### Floor 0 — Exterior &amp; Virtual

| Zone | Area Name | Area ID | Notes |
|------|-----------|---------|-------|
| Outdoor Living | Front Porch | `front_porch` | |
| Outdoor Living | Rear Patio | `rear_patio` | |
| Yard Zone | Front Yard | `front_yard` | |
| Yard Zone | Driveway | `driveway` | |
| Yard Zone | Rear Yard | `rear_yard` | |
| Yard Zone | Side Yard | `side_yard` | |
| Virtual | Control Room | `control_room` | HA infrastructure devices |

### Floor 1 — First Floor

| Zone | Area Name | Area ID | Notes |
|------|-----------|---------|-------|
| First Floor | Dining Room | `dining_room` | |
| First Floor | Foyer | `foyer` | |
| First Floor | Garage | `garage` | |
| First Floor | Half Bathroom | `half_bathroom` | |
| First Floor | Kitchen | `kitchen` | |
| First Floor | Living Room | `living_room` | |
| First Floor | Pantry | `pantry` | |
| First Floor | Rear Hallway | `rear_hallway` | |
| First Floor | Stairway | `stairway` | Foyer to second floor hallway |
| First Floor | Utility Room | `utility_room` | |

### Floor 2 — Second Floor

| Zone | Area Name | Area ID | Notes |
|------|-----------|---------|-------|
| Second Floor | Guest Bathroom | `guest_bathroom` | |
| Second Floor | Guest Room One | `guest_room_one` | |
| Second Floor | Guest Room Two | `guest_room_two` | |
| Second Floor | Hallway | `hallway` | |
| Second Floor | Laundry Room | `laundry_room` | |
| Second Floor | Master Bathroom | `master_bathroom` | |
| Second Floor | Master Bedroom | `master_bedroom` | |
| Second Floor | Master Bedroom Closet One | `master_bedroom_closet_one` | Walk-in closet |
| Second Floor | Master Bedroom Closet Two | `master_bedroom_closet_two` | Walk-in closet |
| Second Floor | Office | `office` | |
| Second Floor | Office Bathroom | `office_bathroom` | |

### Floor 3 — Attic

| Zone | Area Name | Area ID | Notes |
|------|-----------|---------|-------|
| Attic | *(TBD)* | — | No defined areas yet |

### Area ID Conventions

- **Lowercase with underscores** — matches entity ID and MQTT topic conventions
- **Singular nouns** — `guest_room_one` not `guest_rooms`
- **No abbreviations** — `living_room` not `lr`
- **Stable identifiers** — rename rarely; if room purpose changes, consider whether area_id should change

---

## MQTT Topic Alignment

Entity naming aligns with the `highland/` MQTT event namespace:

| Entity | Semantic Event Topic |
|--------|---------------------|
| `binary_sensor.garage_motion` | `highland/event/garage/motion_detected` |
| `binary_sensor.basement_leak_water_heater` | `highland/event/basement/leak/water_heater` |
| `lock.front_door` | `highland/event/front_door/locked` |

The area and device/function components map directly to topic hierarchy.

---

## Examples: Full Room

**Living Room with:**
- Overhead light (single)
- Two table lamps (east and west)
- Three outlets on west wall (one for floor lamp, two general)
- Multi-sensor (environment)
- Motion sensor

```
# Lights
light.living_room_overhead
light.living_room_table_lamp_east
light.living_room_table_lamp_west

# Outlets
switch.living_room_outlet_floor_lamp
switch.living_room_outlet_west_alpha
switch.living_room_outlet_west_beta

# Sensors
sensor.living_room_environment_temperature
sensor.living_room_environment_humidity
sensor.living_room_environment_illuminance
binary_sensor.living_room_motion
```

---

## Open Questions

- [ ] Climate devices (thermostats, HVAC zones) — Deferred until installed. Likely ecobee Premium × 2 (upstairs/downstairs). Pattern TBD when evaluating hardware.
- [ ] Media players — Deferred. Not a day-one requirement. Pattern TBD when needed.

---

## Device Type Patterns (Continued)

### Cameras

Cameras expose multiple entities per physical device (different quality streams). The quality is the distinguishing qualifier.

**Pattern:**
```
camera.{area}_feed_{quality}
```

**Example (single camera in area):**
```
camera.driveway_feed_fluent          → Low-res, high-framerate stream
camera.driveway_feed_clear           → High-res, lower-framerate stream
```

**Friendly names:** Quality-focused, area omitted (per standard):
```
Feed (Fluent)
Feed (Clear)
```

**Future: Multiple cameras in same area:**

Use position or landmark to disambiguate:
```
camera.driveway_feed_front_fluent
camera.driveway_feed_front_clear
camera.driveway_feed_side_fluent
camera.driveway_feed_side_clear
```

Or Greek letters if position doesn't apply:
```
camera.driveway_feed_alpha_fluent
camera.driveway_feed_alpha_clear
camera.driveway_feed_beta_fluent
camera.driveway_feed_beta_clear
```

**Friendly names (multi-camera):**
```
Front Feed (Fluent)
Front Feed (Clear)
Alpha Feed (Fluent)
Alpha Feed (Clear)
```

**Related entities (motion, events):**

If the camera/NVR exposes motion or AI detection as separate entities:
```
binary_sensor.driveway_camera_motion
binary_sensor.driveway_camera_person_detected
binary_sensor.driveway_camera_vehicle_detected
```

Or with multi-camera:
```
binary_sensor.driveway_camera_front_motion
binary_sensor.driveway_camera_side_motion
```

---

*Last Updated: 2026-03-03*
