# Calendar Integration — Design & Architecture

## Overview

Google Calendar serves as the household's event scheduling layer. A dedicated Node-RED Calendar Bridge flow polls the household calendar and materializes relevant events as MQTT triggers.

The primary initial consumer is the video pipeline's camera kill switch — certain calendar events should automatically suppress camera monitoring for their duration. Other flows may consume calendar events in the future for unrelated purposes (reminders, away mode, etc.), but those are independent concerns designed separately when needed.

---

## Design Goals

- **Single integration point** — Calendar Bridge flow owns the calendar API relationship; consumers subscribe to MQTT events, never query the calendar directly
- **Zero-friction authoring** — household members use Google Calendar exactly as they do today; no new tools, no metadata syntax, no training beyond a one-time explanation
- **Loose coupling** — the bridge has no knowledge of consumers; flows subscribe to what they care about
- **Active event state** — retained MQTT topics allow restarting flows to immediately know what is currently active
- **Stateless re-derivation** — every poll cycle derives complete authoritative state from scratch; no incremental patching

---

## Camera Suppression: Attendee-Based Approach

### Core Concept

Each camera has a dedicated email address on the `your-domain.example` domain. To suppress a camera during a calendar event, **invite it as a guest**. That's the entire authoring workflow.

```
camera.driveway@your-domain.example
camera.rear_yard@your-domain.example
camera.rear_patio@your-domain.example
camera.front_porch@your-domain.example
```

The Calendar Bridge reads the attendee list of each event. Any event containing one or more camera addresses triggers camera suppression for those cameras from event start to event end. Events with no camera guests are invisible to the camera pipeline entirely.

### Why This Works

- **Self-documenting** — the guest list tells you exactly which cameras are affected; no separate reference needed
- **Native Google Calendar UX** — inviting a guest is a standard action on any platform (web, iOS, Android); no new tools or panels required
- **Granular by default** — invite the specific cameras you want suppressed; others remain active
- **One-time explanation** — "invite the camera to opt it out" is counterintuitive once, obvious forever
- **Event type is irrelevant** — the bridge doesn't need to know if it's a BBQ, a repair visit, or anything else; it only cares whether cameras are on the guest list

### What the Bridge Does NOT Care About

- Event title
- Event color
- Event description
- Event type or category
- Any metadata fields

A trash day reminder, an HVAC filter change reminder, and a backyard BBQ are all just calendar events. If no camera addresses are in the guest list, the bridge ignores them for camera purposes. The camera pipeline has no opinion about trash day.

### Email Notifications

Google Calendar will attempt to send invite emails to camera addresses. Since `your-domain.example` is already a managed domain, create a catch-all alias that discards silently. The bridge operates via Calendar API polling, not email; IMAP monitoring of these addresses adds no value.

---

## Core Operating Model

### Stateless Re-Derivation

Every poll cycle derives the complete authoritative picture from scratch. The bridge does not accumulate or patch state incrementally — it re-reads calendar reality and overwrites retained state with whatever is true right now.

This means:
- **Event ended while down** — next poll excludes it from active list; retained state is corrected
- **Event started while down** — next poll includes it as currently active; retained state reflects it
- **Overlapping events on the same camera** — bridge computes the union of all active camera suppressions and publishes the complete set; no consumer-side reference counting needed
- **Deleted or modified events** — next poll corrects everything automatically

### Startup = Immediate Poll

Node-RED startup is not a special codepath. On startup, the bridge:
1. Cancels all existing calendar timers (stale from before restart)
2. Immediately runs the full poll cycle
3. Re-derives state, overwrites retained topics, reschedules all timers fresh

This correctly handles events that started while down, events that ended while down, and timers that never fired. The scheduled periodic poll then takes over normally.

**Startup must cancel existing timers before polling** to prevent a stale pre-restart timer from firing alongside the fresh reconciliation.

### Poll Cadence

| Trigger | Cadence | Purpose |
|---------|---------|---------|
| Periodic | Every 30 minutes | Picks up events added same-day |
| On startup | Immediate | Reconciles state after any outage |
| Manual | `highland/command/calendar/reload` | On-demand for just-added events |

30-minute cadence is well within API limits (1M calls/day; effective rate ~5–10 req/sec). At 48 calls/day, consumption is negligible.

### Lookahead Window

**72 hours.** Weekend planning frequently happens by Thursday; a Friday addition for a Saturday event should land in the first poll after it's created. Extending beyond 72h provides diminishing returns given the 30-minute refresh cadence.

Multi-day events (e.g., a multi-day away period) are handled by daily re-evaluation — each poll picks up the ongoing span and refreshes retained state and timers. No single long-lived timer spanning multiple days.

### API Credentials

Shared with the Daily Digest query (same Google Calendar account). Single OAuth token, single config entry in `secrets.json`. The Calendar Bridge and Daily Digest are separate flows but share the credential.

---

## Two Classes of Consumers

Calendar events drive two fundamentally different response types. The bridge must distinguish them.

### Stateful Consumers

These just need to know "what is true right now." They don't care about ceremony — they read retained state, apply it, and move on.

**Example:** Camera kill switch. Whether the suppression started 10 seconds ago or the flow just restarted mid-event, the response is identical: enable the kill switch for the listed cameras.

**Pattern:** Subscribe to retained `highland/state/calendar/camera_suppression`. On receipt, apply current state directly. No event history needed.

### Ceremony Consumers

These have side effects that should fire exactly once per event, not on every reconciliation. A notification that "Backyard BBQ started — cameras suppressed" should fire when the event starts, not every time Node-RED restarts during the event.

**Pattern:** The bridge tracks whether it has already fired ceremony for a given event (keyed by event ID + event type), persisted in disk-backed flow context. On poll, if a start event is detected and ceremony has not been marked for that event ID, fire the start event and mark it. On end, same logic.

This keeps ceremony tracking in a single place (the bridge) rather than pushing the complexity to every consumer.

**Persistent MQTT sessions:** Ceremony consumers must use persistent MQTT sessions (`clean_session: false` with a stable client ID). This closes the race window where the bridge fires a ceremony event but the consumer is momentarily offline — the broker queues the message and delivers it exactly once on reconnect.

The remaining edge case (broker itself loses queue state) results in a missed notification, which is acceptable degradation. Stateful consumers are unaffected since they rely on retained topics, not queued delivery.

### `already_active` Flag

Start event payloads include `already_active: true` when the event was already in progress at poll time (i.e., event.start is in the past). This gives ceremony consumers a clean signal to skip ceremony and only reconcile state, if they choose to use it — though with persistent sessions, this is mainly a diagnostic/logging aid rather than a required guard.

---

## Calendar Bridge Flow

### Responsibilities

1. On startup and every 30 minutes: cancel existing timers, query Google Calendar API (next 72h), re-derive complete state
2. For each event containing camera guest addresses:
   - Extract camera list from attendee addresses
   - Map addresses to camera entity IDs via camera address registry
   - If currently active: publish state immediately, mark ceremony as already fired if applicable
   - If upcoming: schedule start and end timers
3. Overwrite retained active suppression state with current union of all active events
4. Fire ceremony events (start/end) for events not yet ceremonialized, tracking by event ID in flow context

### Poll Cycle

```
Trigger (startup | 30-min timer | manual reload)
        │
        ▼
Cancel all existing calendar timers
        │
        ▼
Query Google Calendar API (next 72h)
        │
        ├── For each event with camera attendees:
        │     ├── Compute currently active? (start <= now <= end)
        │     ├── If active and ceremony not fired → fire start event, mark ceremony
        │     ├── If active → include in retained state union
        │     └── If upcoming → schedule start/end timers
        │
        ├── Overwrite retained state: highland/state/calendar/camera_suppression
        │
        └── Log: "Calendar bridge: N active, M upcoming camera suppression events"
```

### MQTT Topics

**Camera suppression start/end (ceremony — single-fire per event):**
```
highland/event/calendar/camera_suppression/start
highland/event/calendar/camera_suppression/end
```

**Active suppression state (retained — authoritative current snapshot):**
```
highland/state/calendar/camera_suppression    ← retained
```

**Manual reload command:**
```
highland/command/calendar/reload
```

### Payloads

**Start event payload:**
```json
{
  "timestamp": "2026-03-07T16:00:00Z",
  "source": "calendar_bridge",
  "event_id": "google_calendar_event_id",
  "title": "Backyard BBQ",
  "start": "2026-03-07T16:00:00Z",
  "end": "2026-03-07T22:00:00Z",
  "cameras": ["rear_yard", "rear_patio"],
  "already_active": false
}
```

**Active suppression state payload (retained):**
```json
{
  "timestamp": "2026-03-07T16:05:00Z",
  "source": "calendar_bridge",
  "active": [
    {
      "event_id": "google_calendar_event_id",
      "title": "Backyard BBQ",
      "start": "2026-03-07T16:00:00Z",
      "end": "2026-03-07T22:00:00Z",
      "cameras": ["rear_yard", "rear_patio"]
    }
  ]
}
```

When no events are active, `active` is an empty array (not absent). This allows consumers to distinguish "no active events" from "retained state not yet received."

### Camera Address Registry

Maintained in a dedicated config file (or alongside `device_registry.json`). Maps email address → camera entity ID:

```json
{
  "camera_guests": {
    "camera.driveway@your-domain.example": "driveway",
    "camera.rear_yard@your-domain.example": "rear_yard",
    "camera.rear_patio@your-domain.example": "rear_patio",
    "camera.front_porch@your-domain.example": "front_porch"
  }
}
```

### Handling Edge Cases

**Overlapping events covering the same camera:**
- Bridge computes the union of all active events per camera
- Retained state reflects all overlapping entries
- Camera suppression remains active until all covering events have ended
- No consumer-side reference counting required

**Event spans poll boundary (still active at next poll):**
- Next poll detects event still active (start in past, end in future)
- Ceremony already marked for this event ID → no duplicate start event
- Retained state updated with current snapshot (no behavioral change)

**Event added after most recent poll:**
- Next 30-minute poll picks it up
- Manual `highland/command/calendar/reload` available for immediate refresh
- Manual kill switch in HA always available as fallback

**API unavailable:**
- Log error, do not overwrite retained state (leave last-known-good in place)
- Retry once after 5 minutes
- If still unavailable, notify (normal priority): "Calendar bridge unavailable — retained state may be stale, manual camera overrides may be needed"

---

## Consumer Pattern

### Stateful Consumer (Camera Flow)

```
On startup:
  Subscribe to: highland/state/calendar/camera_suppression (retained)
  → On receipt: apply entire active list — enable kill switches for all listed cameras

On start event:
  highland/event/calendar/camera_suppression/start
  → Enable kill switch for cameras in payload.cameras

On end event:
  highland/event/calendar/camera_suppression/end
  → Disable kill switch for cameras in payload.cameras
  → (Retained state has already been updated by bridge; this is belt-and-suspenders)
```

### Ceremony Consumer Requirements

Any consumer that should fire side effects exactly once per event (notifications, announcements, log entries):
- Use persistent MQTT session (`clean_session: false`, stable client ID)
- Subscribe to `highland/event/calendar/camera_suppression/start` and `/end`
- Optionally check `already_active` flag to adjust behavior on reconciliation vs. fresh start

---

## Relationship to Video Pipeline

Calendar-driven suppression complements rather than replaces the manual kill switch.

| Scenario | Mechanism |
|----------|-----------|
| Planned event (cameras invited) | Calendar bridge fires start → kill switch auto-enabled |
| Spontaneous event (not on calendar) | Manual kill switch from HA dashboard |
| Event ends | Calendar bridge fires end → kill switch auto-disabled |
| Override during calendar-driven suppression | Manual kill switch toggle always available |

See VIDEO_PIPELINE.md for kill switch design and state management.

---

## Future: Other Calendar Consumers

Other flows may consume calendar events for unrelated purposes. These are independent of the camera suppression design and will be designed separately when needed.

Candidate future use cases:
- **Away mode** — household away periods trigger security posture changes
- **Repair/service arrivals** — expected visitors, no camera suppression but context for other flows
- **Maintenance reminders** — HVAC filter changes, etc. (likely via a separate mechanism, not attendee-based)

If future flows need event type discrimination, mechanisms like event color or a separate Highland sub-calendar are options — but those decisions belong to those flows, not to this one.

---

## Future: One-Shot via AI Assistant

Once Marvin (HA Assist) has calendar write capability, the authoring workflow can collapse further:

*"Hey Marvin, add a BBQ Saturday 3 to 10pm, suppress the rear yard and driveway cameras."*

Marvin creates the event and adds the camera addresses to the guest list in one step. The Calendar Bridge Flow is unchanged — it still just reads attendee lists. The UX improves without any backend changes.

---

## Open Questions

- [ ] Camera address format — `camera.driveway@your-domain.example` confirmed as convention; verify no conflicts with other `your-domain.example` mail routing
- [ ] Ceremony state persistence — flow context (disk-backed) is the right store, but define eviction strategy: remove entries for events whose end time is in the past to prevent unbounded growth
- [ ] Whether `already_active` is sufficient as a ceremony guard or whether persistent sessions alone are relied upon — both are implemented, but the interaction should be validated during implementation

---

## Related Documents

| Document | Relevance |
|----------|-----------|
| **EVENT_ARCHITECTURE.md** | Calendar events follow standard MQTT payload conventions |
| **NODERED_PATTERNS.md** | Calendar Bridge is a utility flow; Config Loader handles API credentials |
| **VIDEO_PIPELINE.md** | Primary initial consumer — calendar events drive camera kill switch |
| **AUTOMATION_BACKLOG.md** | AI calendar assistant (natural language event creation) is a related backlog item |

---

*Last Updated: 2026-03-13*
