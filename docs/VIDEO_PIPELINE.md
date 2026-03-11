# Video Pipeline — Design Document

## Overview

The video pipeline processes camera motion events through a multi-tier analysis system: local triage via CodeProject.AI (CPAI) with a Coral TPU, followed by selective deep analysis via Google Gemini for events that pass triage. The goal is to minimize cloud API costs and latency while maintaining high-quality notifications for meaningful events.

---

## Architecture

```
┌─────────────────────┐
│   Reolink Cameras   │
│   (Wi-Fi + NVR)     │
└──────────┬──────────┘
           │ Motion event
           ▼
┌─────────────────────┐
│   Motion Sidecar    │
│   (Python/reolink_aio)│
│   TCP Baichuan or   │
│   ONVIF SWN push    │
└──────────┬──────────┘
           │ MQTT: highland/event/camera/{camera}/motion
           ▼
┌─────────────────────────────────────────────────────────────┐
│                    Node-RED                                 │
│   ┌─────────────────────────────────────────────────────┐   │
│   │ Camera Pipeline Flow                                │   │
│   │                                                     │   │
│   │  [Motion Event] → [Check Kill Switch] → [Cooldown] │   │
│   │                           │                         │   │
│   │                           ▼                         │   │
│   │                  [Fetch Still/Clip]                 │   │
│   │                           │                         │   │
│   │                           ▼                         │   │
│   │                  [CPAI Triage]                      │   │
│   │                           │                         │   │
│   │            ┌──────────────┼──────────────┐          │   │
│   │            ▼              ▼              ▼          │   │
│   │        [Animal]      [Vehicle]      [Person]        │   │
│   │            │              │              │          │   │
│   │            ▼              ▼              ▼          │   │
│   │      [Auto-cooldown] [Auto-cooldown] [Deep Analysis]│   │
│   │                                          │          │   │
│   │                                          ▼          │   │
│   │                                    [Gemini API]     │   │
│   │                                          │          │   │
│   │                                          ▼          │   │
│   │                                    [Notification]   │   │
│   └─────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
           │
           ▼
┌─────────────────────┐
│   Edge AI Box       │
│   (CPAI + Coral)    │
└─────────────────────┘
```

---

## Motion Event Source

### Reolink Motion Events

Motion events are captured via a Python sidecar using `reolink_aio`, a library that interfaces with Reolink cameras/NVR.

**Connection options:**
- **TCP Baichuan protocol** — Direct connection to camera/NVR, lower latency
- **ONVIF SWN push** — Standard ONVIF subscription, camera pushes events

**Deployment:** Docker container on Workflow, alongside Node-RED.

**Output:** Publishes to MQTT `highland/event/camera/{camera}/motion` with minimal payload (timestamp, camera ID). The sidecar does not fetch images — that's the pipeline's job.

**HA independence:** The sidecar connects directly to cameras/NVR via their native protocols. It does not depend on HA or the Reolink HA integration. This ensures motion detection continues if HA is down.

---

## Kill Switch

### Purpose

Suppress camera notifications during planned activities (e.g., backyard party, contractors working). The kill switch is zone-aware — only specified cameras are suppressed.

### Data Source

Google Calendar events with structured metadata. See **CALENDAR_INTEGRATION.md** for the calendar bridge design.

**State topic:** `highland/state/calendar/camera_suppression` (retained)

The Camera Pipeline flow reads this state on startup and subscribes for updates. When processing a motion event, it checks if the triggering camera is in the active suppression list.

### Manual Kill Switch

For unplanned extended activity (e.g., kids playing in yard), a manual kill switch is available:

- Triggered via HA UI button or notification action
- Publishes to `highland/command/camera/suppress`
- Payload includes: `cameras`, `duration_minutes`, `reason`
- Pipeline flow merges manual suppression with calendar suppression
- Auto-expires after duration

### Smart Suppression Prompting (Future)

Backlog item: The system detects extended low-threat outdoor presence and proactively asks whether monitoring should be suppressed via notification action or Assist.

---

## Cooldown

### Purpose

Prevent notification spam from continuous motion (e.g., tree branches, persistent animal).

### Zone-Aware Cooldown

Each camera has an independent cooldown timer. Cooldown state is stored in Node-RED flow context.

### Classification-Based Cooldown

| Classification | Cooldown Behavior |
|----------------|-------------------|
| **Animal** | Auto-cooldown (configurable duration, default 15 min) |
| **Vehicle** | Auto-cooldown (configurable duration, default 5 min) |
| **Person** | No auto-cooldown — every person event proceeds to deep analysis |
| **Unknown** | No auto-cooldown — treat as potentially significant |

### Composition/Count Tracking

Within an active cooldown window, track what's been seen:
- If a new classification appears (e.g., person after animal), reset cooldown and process
- If count increases significantly (e.g., 1 person → 3 people), reset cooldown and process

### Cooldown State

Owned by Node-RED, exposed to HA via MQTT Discovery for dashboard visibility:
- `highland/state/camera/{camera}/cooldown` — current cooldown status, time remaining
- Manual reset available via `highland/command/camera/{camera}/cooldown/reset`

---

## Image/Clip Acquisition

### NVR-Managed Recording

Target state: NVR handles all recording. The pipeline fetches stills or clips from the NVR on demand.

**Pending NVR arrival:** Validate API capabilities:
- Can we fetch a still frame from a specific timestamp?
- Can we extract a clip (5-10 seconds) around an event?
- How do dual-lens 180° cameras expose their channels?
- Does recording lock interfere with API access?

### Fallback: Direct Camera Fetch

If NVR API is insufficient, fall back to direct camera snapshot via `reolink_aio`.

---

## Local Triage — CodeProject.AI

### Purpose

Fast, local classification to filter out low-value events before expensive cloud analysis.

### Deployment

- Runs on Edge AI Box (Dell OptiPlex SFF)
- Docker container: `codeproject/ai-server`
- Hardware acceleration: Google Coral TPU (PCIe)
- Model: YOLO-based object detection

### Classification Targets

| Target | Action |
|--------|--------|
| `person` | Proceed to deep analysis (Gemini) |
| `car`, `truck`, `motorcycle` | Log as vehicle, apply cooldown |
| `cat`, `dog`, `bird`, `deer`, etc. | Log as animal, apply cooldown |
| No detection / low confidence | Proceed to deep analysis (unknown = potentially significant) |

### API Integration

CPAI exposes a REST API. Node-RED sends image, receives JSON with detections:

```json
{
  "success": true,
  "predictions": [
    {
      "label": "person",
      "confidence": 0.92,
      "x_min": 120,
      "y_min": 80,
      "x_max": 220,
      "y_max": 380
    }
  ]
}
```

### Confidence Threshold

Configurable in `thresholds.json`. Start with 0.6 — adjust based on false positive/negative rates.

---

## Deep Analysis — Google Gemini

### Purpose

Rich contextual analysis for events that pass local triage. Gemini provides:
- Detailed scene description
- Activity classification (delivery, visitor, suspicious, etc.)
- Person description (if applicable)
- Suggested notification text

### When to Invoke

- Local triage detected `person`
- Local triage returned no/low-confidence detection (unknown = investigate)
- Manual request via notification action ("Analyze this")

### API Integration

Use Gemini Vision API with a well-crafted prompt:

```
Analyze this security camera image from {camera_name} at {timestamp}.

Describe:
1. What is happening in the scene?
2. If people are present, describe their appearance and apparent activity.
3. Is this a delivery, visitor, or potentially suspicious activity?
4. Rate the urgency: routine, notable, or urgent.

Respond in JSON format:
{
  "description": "...",
  "activity_type": "delivery|visitor|passerby|suspicious|animal|vehicle|other",
  "urgency": "routine|notable|urgent",
  "notification_text": "..."
}
```

### Cost Management

- Local triage filters ~80% of events (estimate)
- Gemini only sees person detections and unknowns
- Monitor API costs; add daily/monthly caps if needed

---

## Notification

### Channels

- **Primary:** HA Companion App (Android)
- **Future:** Additional channels as needed

### Notification Content

| Field | Source |
|-------|--------|
| Title | Camera name + urgency indicator |
| Body | Gemini `notification_text` or fallback template |
| Image | Thumbnail from CPAI analysis |
| Actions | "View Live", "Suppress 30min", "Mark as Known" |

### Actionable Notifications

Notification actions route back to Node-RED via `highland/event/notify/action_response`:

- **View Live:** Deep link to HA camera view
- **Suppress 30min:** Trigger manual kill switch for this camera
- **Mark as Known:** Add to known-face/vehicle list (future)

---

## PTZ Cameras

**Scoped out of detection pipeline.** PTZ cameras are used for manual viewing and recording, not automated detection. Their field of view changes make consistent motion detection impractical.

---

## Dual-Lens 180° Cameras

**Pending NVR validation:** How does the NVR expose channels for dual-lens cameras? Likely two separate streams/channels. Pipeline may need to handle as two logical cameras or correlate events across lenses.

---

## MQTT Topics

| Topic | Purpose |
|-------|---------|
| `highland/event/camera/{camera}/motion` | Motion event from sidecar |
| `highland/state/camera/{camera}/cooldown` | Current cooldown status |
| `highland/command/camera/{camera}/cooldown/reset` | Manual cooldown reset |
| `highland/command/camera/suppress` | Manual kill switch |
| `highland/state/calendar/camera_suppression` | Calendar-based suppression |

---

## Configuration

### thresholds.json

```json
{
  "camera": {
    "cpai_confidence_threshold": 0.6,
    "cooldown_animal_minutes": 15,
    "cooldown_vehicle_minutes": 5,
    "gemini_daily_cap": 100
  }
}
```

### secrets.json

```json
{
  "gemini": {
    "api_key": "..."
  },
  "cpai": {
    "base_url": "http://edgeai.local:32168"
  }
}
```

---

## Open Items

| Item | Notes |
|------|-------|
| NVR API validation | Pending hardware arrival |
| Dual-lens camera topology | Validate channel structure |
| CPAI model selection | Test YOLO variants for accuracy/speed tradeoff |
| Gemini prompt tuning | Iterate based on real-world results |
| Known-face/vehicle database | Future feature — local embedding storage |

---

*Last Updated: 2026-03-11*
