# Time Handling

## Purpose & Scope

This document defines how Highland handles timestamps across flows, storage, transport, scheduling, and presentation. The goal is a single unambiguous rule that eliminates timezone confusion in the event bus, PostgreSQL, logs, and Node-RED context, while keeping scheduling and user-facing displays intuitive.

**Relationship to other docs:**
- **`architecture/NETWORK.md`** — location is defined in `config/location.json`; that configuration drives local time resolution.
- **Subsystem docs** — reference this standard rather than restating timezone conventions locally.

---

## Core Principle

**UTC for storage and transport. Local time for scheduling and presentation.**

Every timestamp committed to persistent context, emitted on the MQTT bus, written to JSONL logs, or inserted into PostgreSQL is UTC. Scheduling operations that need to reason about "7am" or "midnight" interpret those as local time derived from `config/location.json`. Displays to the operator (HA dashboards, notification bodies, voice responses) show local time via the consumer's native timezone handling — flows do not explicitly format for display.

This convention removes any ambiguity about what a timestamp means. A stored or transported timestamp always refers to the same moment in time regardless of where the consumer is, what timezone the server is set to, or whether DST has shifted.

---

## Timestamp Format

ISO 8601 in all stored and transported contexts. Two equivalent representations are semantically valid:

| Form | Example | Status |
|------|---------|--------|
| **UTC with Z suffix** | `"2026-04-21T18:15:00Z"` | **Preferred for all new code and new doc examples.** Matches Node-RED's `new Date().toISOString()` default output. |
| UTC with explicit `+00:00` offset | `"2026-04-21T18:15:00+00:00"` | Equivalent, accepted. |
| Local with non-UTC offset | `"2026-04-21T14:15:00-04:00"` | Semantically unambiguous (the moment is fully specified), but non-preferred. Present in some existing docs predating this standard; no need to retrofit. |

Precision: second precision is always sufficient; millisecond precision (e.g., `2026-04-21T18:15:00.123Z`) is acceptable where the data source provides it but never required.

---

## Where UTC Applies

**Storage and transport — always UTC:**
- MQTT payloads (all namespaces: `event/`, `state/`, `status/`, `command/`, `ack/`, `homeassistant/`)
- Node-RED flow context (`default`, `initializers`, `volatile` stores)
- PostgreSQL `timestamptz` columns (Postgres stores UTC internally regardless of client timezone setting; no special handling needed)
- JSONL log entries per `nodered/LOGGING.md`
- External HTTP request/response bodies (when the API permits; otherwise normalize on boundary)

**Inbound normalization:**
- Email timestamps (RFC 2822 `Date:` header, etc.) parse to UTC at the ingress parser boundary.
- Third-party API timestamps parse to UTC immediately on receipt.
- Sensor-device reported timestamps (if a device ever reports a local time) convert to UTC before reaching flow context or the bus.

The boundary principle: **raw inputs may arrive in any timezone form; by the time a value enters Highland's storage or transport layer, it is UTC.**

---

## Where Local Time Applies

**Scheduling — always local:**
- CronPlus schedules (`0 20 * * *` means 8pm local, not 8pm UTC)
- Astronomical references (sunrise, sunset, solar noon — inherently local by definition)
- Period-based events (`scheduler/day`, `scheduler/evening`, `scheduler/overnight`) — fire at local-time triggers

Local time is resolved from `config/location.json` (latitude, longitude, timezone). The flow host's system timezone (via the Docker container's `TZ` environment variable) must be set to match.

**Presentation — always local:**
- HA Discovery sensors display timestamps in local time via HA's native timezone handling. Flows publish UTC; HA converts to local for display. No manual conversion in the flow.
- Notification bodies, dashboard labels, voice responses: same principle — the consumer handles localization from the UTC value Highland provides.

**"Today" semantics — local day boundaries:**

When code needs to reason about whether two timestamps fall within "the same day" (e.g., clearing today's packages at midnight, checking whether a retained state is from today for restart recovery), the comparison is against **local midnight**, not UTC midnight.

Example: at 23:30 local (03:30 UTC next day), a retained state with timestamp `2026-04-21T18:15:00Z` is still from "today" if today's local date is 2026-04-21. The comparison is performed by converting the stored UTC to local and comparing calendar dates.

---

## DST Transitions

DST transitions are handled by the underlying scheduler (CronPlus and Node-RED's scheduling layer are timezone-aware). Specific points:

- **Spring forward** (2am → 3am local, second Sunday of March in the US): the 2am-3am hour is skipped. Any CronPlus schedule within that window does not fire for that day. Highland's current schedules (midnight rollover, 8pm exception check, scheduler period transitions) are not affected.
- **Fall back** (2am → 1am local, first Sunday of November in the US): the 1am-2am hour repeats. CronPlus schedules fire per the scheduler's implementation — Node-RED typically fires once, at the first occurrence. No Highland schedule falls in this window.

If a new schedule is ever added in the 1am-3am range, validate its behavior across a DST boundary empirically before relying on it.

---

## Implementation Notes

### Node-RED

- `new Date().toISOString()` → UTC with Z. Use this for all "now" timestamps.
- `new Date(someUtcString)` → parses ISO 8601 correctly regardless of offset notation.
- `moment()` library: if used, configure UTC by default (`moment.utc()`).
- Docker container `TZ` env var must be set to the property's timezone. Highland convention: workflow-host `TZ` matches `config/location.json` timezone.
- CronPlus node: timezone is configurable per-node; default is the container timezone. Verify all CronPlus schedules use the correct timezone.

### Home Assistant

- HA has its own timezone configuration (Settings → System → General → Time Zone). Must match the property's local timezone.
- HA sensors with `device_class: timestamp` expect ISO 8601; UTC with Z works correctly.
- HA display converts UTC to local automatically. Flows should not pre-format for display.

### PostgreSQL

- Use `timestamptz` (timestamp with time zone), never `timestamp` (without). `timestamptz` stores UTC internally.
- Client session timezone setting affects only display in query results. Stored data is always UTC.
- `NOW()` returns current UTC wall time.

### Logging

- JSONL log entries use UTC per `nodered/LOGGING.md`. Most logger libraries produce UTC by default; verify your library's behavior.

---

## Rationale

Why UTC-for-storage rather than local-for-everything?

1. **Unambiguous semantics.** A UTC timestamp refers to one specific moment in time, globally. A local timestamp without explicit offset is ambiguous. A local timestamp *with* offset is semantically unambiguous but inconsistent with the rest of the stored data — different records could have different offsets (especially across DST), making sorting and comparison non-trivial.
2. **Operator intuition isn't affected.** Operators never see raw stored timestamps; they see presentation-layer values that have already been converted to local. The storage convention is invisible to them.
3. **Cross-infrastructure consistency.** MQTT, PostgreSQL, HA, and Node-RED all handle UTC cleanly. Mixing representations creates conversion bugs at boundaries.
4. **DST-safe.** UTC doesn't have DST. A timestamp recorded before spring-forward and one recorded after remain correctly ordered and directly comparable.

Why local for scheduling?

1. **Operator intent.** When an operator writes `"letter_mail_exception_time": "20:00"`, they mean "8pm local" — that's what they'd check against their watch. Interpreting it as UTC would produce a different firing time depending on DST and require mental math.
2. **Astronomical events are local by definition.** Sunrise and sunset are local observations.
3. **Alignment with operator daily rhythm.** Scheduling follows the operator's day, not a globally-synchronized clock.

---

*Last Updated: 2026-04-24*
