# Deliveries — Design & Architecture

## Scope

The `Utility: Deliveries` flow owns the **informational layer** of every delivery that touches the property — letter mail, USPS packages, and eventually other carriers (UPS, FedEx, Amazon). It consumes normalized email events from `Utility: Email Ingress`, parses delivery-specific content, maintains authoritative delivery state, and publishes a clean consumer-facing surface for dashboards, notifications, and other flows.

The flow does **not** own IMAP connection management, folder conventions, or deduplication. Those concerns belong to `Utility: Email Ingress` (see `subsystems/EMAIL_INGRESS.md`).

Physical sensors also belong elsewhere. When a mailbox door sensor or driveway package sensor is installed, its raw events are published by `Area: Driveway`. `Utility: Deliveries` subscribes to those raw events and fuses them with the informational layer to produce richer state.

### Phases

| Phase | Scope | Status |
|-------|-------|--------|
| **Phase 1 — Baseline** | USPS Informed Delivery letter mail only (morning digest + delivery confirmation) | 📋 Planned |
| **Phase 2 — Packages** | Multi-carrier package tracking (USPS, UPS, FedEx, Amazon) | Backlog |
| **Phase 3 — Physical sensor fusion** | Consume `Area: Driveway` mailbox door events; maintain pending-retrieval queue; reconcile email vs. physical signals | Blocked on LoRa hardware |

This document covers Phase 1 in detail and establishes the architectural framework for Phase 3. Specific Phase 3 parameters (timing windows, thresholds, classification rules) will be calibrated from empirical data once hardware is installed. Phase 2 is captured in `AUTOMATION_BACKLOG.md`.

---

## Architectural Principles

Before diving into phases, a few principles that shape the whole design:

### Separation of claims from reality

- The **email state machine** is authoritative about USPS's claims — "USPS says mail is coming today," "USPS says mail was delivered." It knows nothing about physical reality.
- The **physical layer** (Phase 3) is authoritative about observable events — "the mailbox door opened." It knows nothing about USPS's claims.
- The **synthesis layer** (Phase 3) composes both to produce user-facing semantics — "there is probably mail waiting."

Each layer has a clean contract. This separation means each can evolve, be replaced, or be extended without disturbing the others.

### Affirmative signaling over inference

State changes happen when a positive signal arrives, not when time elapses without one. The email state machine defaults to `NOT_EXPECTING` and promotes on digest arrival, rather than defaulting to `UNKNOWN` and inferring `NO_MAIL_SCHEDULED` at some arbitrary cutoff time. This aligns with how humans naturally think about mail and eliminates cutoff-timer race conditions.

### Probabilistic by design (Phase 3)

No combination of hardware we can reasonably deploy will definitively identify *who* interfaces with the mailbox at a given moment. Phase 3 is an inherently probabilistic subsystem — its outputs should express confidence, not claim certainty, and a manual override is a first-class feature, not a fallback. This is a principled design choice, not a limitation to apologize for. Future enhancements should improve confidence, not chase certainty.

---

## Dependencies

- **`Utility: Email Ingress`** — Provides the normalized email stream via `highland/event/email/deliveries/+/received`. This flow must be live before Phase 1 can go into production. See `subsystems/EMAIL_INGRESS.md`.
- **USPS Informed Delivery registration** — Requires a mailed PIN for identity verification before email delivery begins. Done once, out-of-band, before Phase 1 goes live.

---

## Phase 1 — Informed Delivery Baseline

### Problem

USPS Informed Delivery sends scan-previews and delivery confirmations via email, free of charge. This provides enough signal to answer "is USPS expecting to deliver mail today?" and "has USPS reported the delivery?" without any physical infrastructure at the mailbox — a useful baseline that:

1. Delivers value immediately, before LoRa hardware arrives.
2. Establishes the consumer-facing `highland/state/deliveries/*` contract so downstream flows, cards, and notifications can be built once and never rewritten.
3. Remains useful *after* LoRa arrives — the email signal becomes one input to the fused state rather than the only input.

### Email Contract

USPS sends two relevant email types on mail days. Both originate from the Informed Delivery sender domain and are routed by the Gmail filter into the `Highland/Deliveries/USPS` label.

| Email | Typical Timing | Signal |
|-------|---------------|--------|
| **Daily Digest** | 6–9am local | Preview of letter mail expected that day, with scanned images. Piece count inferable from image count. |
| **Delivery Confirmation** | Shortly after physical delivery (observed range: ~3pm to ~7:30pm) | "Your mail has been delivered today." Confirms USPS reports letter mail in the box. |

Critical points:

- The delivery confirmation email is **reliable** for this address, per observed behavior. It is the load-bearing signal for the `DELIVERED` transition.
- The morning digest is **informational, not load-bearing** — it tells us what USPS expects to deliver but not when.
- On Sundays, federal holidays, and other no-mail days, neither email fires. No cutoff timer or inference logic is required — `NOT_EXPECTING` is simply the default state until a digest arrives to promote it to `EXPECTING`.

### Ingress Contract

This flow subscribes to `highland/event/email/deliveries/+/received` — a wildcard match for all sources under the `Highland/Deliveries/` Gmail label namespace. Payloads arrive in the standard shape defined in `subsystems/EMAIL_INGRESS.md § Payload Schema`. For every successfully processed (or deliberately rejected) message, this flow publishes `highland/ack/email` with the matching `message_id`.

Per the label convention in `EMAIL_INGRESS.md`, Gmail labels under `Highland/Deliveries/` identify the email's source (carrier/service), not the product name. For Phase 1, the only active source is `Highland/Deliveries/USPS`, which captures USPS Informed Delivery emails and any future USPS-originated automation mail.

The Gmail filters that route incoming mail to the appropriate label are manually configured in Gmail, not defined in this flow. Gmail filter setup is part of the Ingress operational runbook.

### State Machine

Three primary states plus one exception branch. Affirmative signals only — default is `NOT_EXPECTING`, and only an incoming email can promote to higher-information states.

```
NOT_EXPECTING ──(digest)──▶ EXPECTING ──(confirmation)──▶ DELIVERED
                                │                              │
                        (8pm, no confirmation)                 │
                                │                              │
                                ▼                              │
                            EXCEPTION                          │
                                │                              │
                                └──(midnight)──▶ NOT_EXPECTING ◀┘
```

**States:**

| State | Meaning |
|-------|---------|
| `NOT_EXPECTING` | No digest has arrived today. Either it's a no-mail day or the digest hasn't arrived yet. Default state; entered on startup and at midnight rollover. |
| `EXPECTING` | Daily digest received with piece count > 0. USPS reports mail is coming today. Delivery has not yet been confirmed. |
| `DELIVERED` | Delivery confirmation email received. USPS reports mail has been placed in the box. |
| `EXCEPTION` | In `EXPECTING` state when the exception threshold (8pm) is reached. USPS said mail was coming, but the delivery window has effectively closed without a confirmation. Resolves at midnight when the daily cycle resets. |

**Exception threshold rationale:** 8pm is chosen from ~8 months of historical delivery confirmation emails at this address. Delivery times cluster in a primary 3–5pm window with a tail extending to 7:30pm (latest observed). 8pm represents a 30-minute margin past the latest-observed delivery, covering >99% of normal deliveries. After 8pm with no confirmation, the probability that delivery will still occur today is vanishingly small, and the state reflects that by promoting to `EXCEPTION`. This threshold is a claim about USPS delivery patterns, not about operator behavior — it should recalibrate if ~6 months of operational data show a distribution shift (e.g., route change, carrier change, seasonal effects).

**Design notes:**

- Single state machine per day. State resets to `NOT_EXPECTING` at midnight, persistent context cleared.
- Only two time-based transitions: the 8pm exception check (if still `EXPECTING`) and the midnight rollover. No cutoff timer for the "no mail today" case — absence of a digest is already represented by remaining in `NOT_EXPECTING`.
- `EXCEPTION` is not an alarm or a notification in itself. It's a state that surfaces the anomaly in `highland/state/deliveries/mail` for dashboards and downstream consumers to react to as they see fit. Whether to fire a notification on entering `EXCEPTION` is a downstream decision — not all users will want one.
- The email state machine makes no claim about whether mail is physically present in the mailbox. It represents USPS's claims only. Physical reality is the concern of the Phase 3 synthesis layer.

### MQTT Topics

**State (retained):**

`highland/state/deliveries/mail`

```json
{
  "timestamp": "2026-04-21T14:15:00-04:00",
  "source": "informed_delivery",
  "state": "DELIVERED",
  "expected_pieces": 3,
  "digest_received_at": "2026-04-21T07:15:00-04:00",
  "delivered_at": "2026-04-21T14:15:00-04:00"
}
```

`state` values: `NOT_EXPECTING` | `EXPECTING` | `DELIVERED` | `EXCEPTION`

`expected_pieces`, `digest_received_at`, `delivered_at` may be null in states that haven't reached the corresponding transition.

**Events (not retained):**

| Topic | Fires When | Payload |
|-------|-----------|---------|
| `highland/event/deliveries/digest_received` | Daily digest parsed | `{ piece_count, has_packages, timestamp }` |
| `highland/event/deliveries/letter_delivered` | Delivery confirmation parsed | `{ timestamp }` |
| `highland/event/deliveries/exception` | 8pm threshold reached in `EXPECTING` state | `{ expected_pieces, digest_received_at, timestamp }` |

**ACK (not retained):**

`highland/ack/email` — published after each ingress message is processed. See `subsystems/EMAIL_INGRESS.md § ACK` for schema.

### Flow Outline — `Utility: Deliveries`

Per `nodered/OVERVIEW.md` conventions: groups are the primary organizing unit; link nodes connect groups; no node has more than two outputs.

**Group 1 — Ingress Subscription**
- MQTT In on `highland/event/email/deliveries/+/received`
- Route by source (extracted from topic or `label` field) → USPS parser pipeline (Phase 1); FedEx / UPS / Amazon parsers (future phases)
- On unrecognized source: publish `highland/ack/email` with `status: "rejected"` and log

**Group 2 — USPS Parser (Phase 1)**
- Sub-route by subject pattern → Daily Digest path / Delivery Confirmation path
- Daily Digest: extract piece count (image count heuristic), detect "no mail scheduled" variant, mutate flow context
- Delivery Confirmation: confirm sender + subject shape, timestamp the confirmation
- On parser success: publish `highland/ack/email` with `status: "ok"` and link-out to State Machine
- On parser failure: publish `highland/ack/email` with `status: "parse_error"`

**Group 3 — State Machine**
- Single transition engine; reads flow context, applies rules, emits new state
- Publishes retained `highland/state/deliveries/mail` on any transition
- Publishes corresponding event topics

**Group 4 — Scheduler Hooks**
- 8pm exception check (CronPlus, two-output per project convention): if current state is `EXPECTING`, transition to `EXCEPTION`
- Midnight rollover (CronPlus or via `Utility: Scheduling` period transition): reset to `NOT_EXPECTING`

**Group 5 — HA Discovery**
- Sensor: `sensor.mail_status` (string — current state)
- Sensor: `sensor.mail_expected_pieces` (int)
- Sensor: `sensor.mail_last_digest_received` (timestamp)
- Sensor: `sensor.mail_last_delivered` (timestamp)

Notably absent compared to earlier drafts: no IMAP group, no folder-management group, no digest-cutoff timer. `Utility: Email Ingress` owns IMAP; the cutoff timer is eliminated by the affirmative-signaling model.

### Configuration

Delivery-specific tunables only. All IMAP/folder/retention concerns live in `config/email_ingress.json`.

```json
{
  "informed_delivery": {
    "sender_domain": "email.informeddelivery.usps.com",
    "exception_time": "20:00"
  }
}
```

File location is TBD — depends on how configuration groups ultimately organize. Candidates include a dedicated `config/deliveries.json` or inclusion in a broader file once the shape stabilizes. Decision deferred to implementation.

---

## Phase 3 — Physical Sensor Fusion (Future)

When the LoRa mailbox door sensor is installed (per `subsystems/LORA.md`), the design adds a second data model — a **pending-retrieval queue** — and a synthesis layer that composes it with the Phase 1 email state machine.

**This is not an extension of the email state machine.** Two different concerns deserve two different models:

- The email state machine represents **USPS's claims** about today, and cycles daily.
- The pending-retrieval queue represents **mail that hasn't been retrieved yet**, which can persist across multiple days.

Mail accumulates. With Informed Delivery providing daily previews, the operator may not physically check the mailbox every day — a delivery on Monday that isn't retrieved is still in the mailbox on Tuesday when another delivery arrives. The queue model captures this; a single state machine cannot.

### Producer: `Area: Driveway`

Publishes raw physical events only — no interpretation. Published topics are defined in `subsystems/LORA.md § Use Case 2`:

- `highland/state/driveway/mailbox` (retained) — sensor telemetry including `door_state`, battery, env, signal.
- `highland/event/driveway/mailbox/opened` (not retained) — door opened.

No state machine; no delivery logic. The driveway flow doesn't know whether a door-open is a carrier, a retrieval, or a mail stuffer.

### Consumer: `Utility: Deliveries` — Pending-Retrieval Queue

Maintains a queue of `{ delivered_at, piece_count, source }` records. Each `DELIVERED` transition on the email state machine appends a record. Door-open events classified as retrievals clear the queue entirely (we cannot know which specific days' mail were retrieved; it's one mailbox).

**New topics:**

- `highland/state/deliveries/pending` (retained) — current queue contents, oldest-age, count, and confidence indicator
- `highland/event/deliveries/retrieved` (not retained) — fires when the queue transitions from non-empty to empty via a retrieval classification

Payload shape (provisional, to be refined during implementation):

```json
{
  "timestamp": "2026-04-21T19:15:00-04:00",
  "source": "deliveries_synthesis",
  "count": 3,
  "oldest_delivered_at": "2026-04-19T15:20:00-04:00",
  "oldest_age_hours": 52.0,
  "confidence": "high",
  "pending": [
    { "delivered_at": "2026-04-19T15:20:00-04:00", "piece_count": 2, "source": "informed_delivery" },
    { "delivered_at": "2026-04-20T16:05:00-04:00", "piece_count": 4, "source": "informed_delivery" },
    { "delivered_at": "2026-04-21T14:48:00-04:00", "piece_count": 1, "source": "informed_delivery" }
  ]
}
```

### Door-Open Classification

The central challenge of Phase 3 is classifying door-open events as **carrier activity** (do not clear queue) or **retrieval activity** (clear queue). The email state machine provides the primary signal for this classification.

**Sequence of events on a normal mail day:** carrier opens mailbox → carrier deposits mail → carrier closes box → carrier syncs device → USPS fires delivery confirmation email (typically 10-30+ minutes later). The door-open *precedes* the email, not the other way around.

**Classification rules:**

| Email state at door-open | Default classification | Notes |
|--------------------------|------------------------|-------|
| `NOT_EXPECTING` | Retrieval | No mail is coming; any door-open is retrieval of pre-existing mail or unrelated activity. Clear queue. |
| `DELIVERED` | Retrieval | Mail already came; carrier won't return today. Clear queue. |
| `EXCEPTION` | Retrieval | Delivery window has closed. Any door-open is retrieval. Clear queue. |
| `EXPECTING` | **Deferred** | Could be carrier or retrieval. Hold classification until `DELIVERED` arrives or a timeout expires. |

**Deferred classification during `EXPECTING`:**

1. Door-open during `EXPECTING` is recorded but does not immediately clear the queue.
2. If `DELIVERED` transition arrives within the classification window, the most recent door-open is reclassified as carrier activity (discarded); any earlier door-opens during the same `EXPECTING` window are reclassified as retrievals.
3. If the window expires with no `DELIVERED`, the door-open is reclassified as a retrieval.

**Classification window** is the time between the earliest door-open-during-`EXPECTING` and the arrival of `DELIVERED` (or its timeout). This is not a fixed value; it's bounded by the state transitions. Empirically this window is typically minutes to a few hours.

**Ambiguous sequences during `EXPECTING`:** when multiple door-opens occur before `DELIVERED` arrives, confidence drops. The system publishes with `confidence: "low"` and may request manual confirmation via the override topic (see below). These sequences are expected to be uncommon.

### Manual Override

Because classification is inherently probabilistic, a manual override is a first-class feature:

**`highland/command/deliveries/mark_retrieved`** — clears the pending queue regardless of current state or ambiguity. No payload required. Intended for:

- Voice commands ("I got the mail")
- HA dashboard button tap
- Scripted automation (e.g., reset when guests handle mail during an away period)

Manual override always succeeds silently — no confirmation dialog, no friction. Correcting the system should be as easy as interacting with it.

### Confidence Reporting

The `confidence` field on `highland/state/deliveries/pending` reflects how certain the system is about the current queue state:

| Value | Meaning |
|-------|---------|
| `high` | No recent ambiguous events; queue state derives from clean signals (DELIVERED transitions or clear retrievals) |
| `medium` | Some ambiguity but classification was resolvable (e.g., deferred door-open during EXPECTING that resolved normally) |
| `low` | Multiple ambiguous events; queue count may not reflect reality. Consumers should prompt for manual verification. |

Consumers (notifications, dashboards, voice responses) should phrase outputs according to confidence — "you have 3 pieces of mail waiting" for `high`, "you likely have mail waiting" for `medium`, "mail state is uncertain, please verify" for `low`.

### Future Enhancement: USPS Vehicle Detection

A camera positioned to observe the mailbox approach could detect USPS vehicle presence correlated with door-open events, providing a much stronger classification signal than email-state correlation alone:

- Door-open + USPS vehicle detected in snapshot → HIGH CONFIDENCE carrier
- Door-open + no vehicle in snapshot → HIGH CONFIDENCE retrieval

This would slot into the synthesis layer as an additional input, making door-open classification real-time and eliminating most of the deferred-classification logic around the `EXPECTING` state.

#### Proposed trigger architecture (battery-friendly)

Standard battery cameras burn most of their power on always-on motion detection, producing dozens of useless wakes per day (wind, cars, animals, lighting changes). A battery camera in a busy outdoor location typically drains in weeks.

Inverting the trigger pattern preserves battery:

1. Camera motion detection **disabled entirely**
2. Camera sits in deep sleep
3. LoRa mailbox contact sensor fires `highland/event/driveway/mailbox/opened`
4. Highland sends capture command to camera (API call)
5. Camera wakes, captures, returns to sleep

Expected wake events per day: 1–3 (carrier visits + occasional retrievals), rather than dozens. Battery life multiplies proportionally — "months" becomes plausible.

#### Latency is the central constraint

The approach works only if end-to-end latency from physical door-open to camera snapshot is **less than the carrier's vehicle-present window at the mailbox**.

Latency contributors:

| Stage | Estimate |
|-------|----------|
| LoRa uplink (sensor → MQTT on hub.local) | 500ms–2s (see `LORA.md § LoRaWAN configuration defaults`) |
| Node-RED flow processing + camera API call | sub-second |
| Camera wake from deep sleep + capture | 2–8s (varies by model) |
| **Total estimate** | **~5–10s** |

**Property-specific geometry favors feasibility.** Several factors extend the effective vehicle-present window well beyond a typical single-mailbox carrier stop:

- The mailbox is one of three on a shared bank (three properties share the driveway; all three mailboxes on one post).
- Our mailbox is the **first** accessed on approach — our door-open fires while the carrier still has two more boxes to service on the bank.
- We are the **last stop** on the carrier's route for this part of the drive, so the carrier turns around in place after the bank, adding repositioning time before departure.

Realistic vehicle-present window from our door-open to carrier departure: **~25–60 seconds**. A single snapshot at T+8s (well inside our latency budget) should reliably catch the vehicle in frame. Burst capture becomes redundancy rather than necessity. Generic guidance for other mailboxes (drop-and-go single-box stops with 5–15 second dwell) would require burst capture to be viable; our situation is easier.

#### Challenges to resolve before this is viable

- **Line of sight and distance.** ~275ft driveway with no line of sight to the road in the straight-ahead direction. Camera placement needs to see the mailbox approach clearly. Possibly a camera on a fence post or tree near the mailbox, or mounted on the house with zoom optics.
- **Camera wake latency** of candidate hardware. Varies massively by model; must measure before committing. Reolink Argus series, Wyze Cam Outdoor, Eufy SoloCam, and similar battery cameras are candidates.
- **Empirical LoRa latency validation** per `LORA.md` open question. If real-world latency is consistently >2s or highly variable, the whole approach needs reconsideration.
- **Power source for camera.** Battery is the constraint that makes the sensor-triggered approach interesting. PoE would eliminate the power question but requires cable pulls to the mailbox area — probably not justified for this use case alone.
- **Vision classification pipeline.** Distinguishing a USPS LLV from other vehicles. Distinctive silhouette and livery make this tractable with a reasonable model; exact pipeline (CodeProject.AI on edgeai.local, Gemini API, other) TBD.

#### Phased progression for Phase 3

Given these uncertainties, the sensible path is:

- **Phase 3a — LoRa door sensor only.** Build the pending-retrieval queue and email-state-based classification. Valuable on its own; confidence will be "high" most of the time, "low" in rare ambiguous EXPECTING cases handled by manual override. Deploy, observe for months, tune parameters.
- **Phase 3b — Add camera with contact-sensor-triggered capture** *if* empirical latency measurements show it's viable. Camera provides vehicle-detection input that upgrades confidence from "medium" to "high" in deferred-classification cases. If latency proves unworkable, shelve and accept Phase 3a's accuracy.
- **Phase 3c — Always-on PoE camera with continuous vision classification** — the "certainty" option, significantly more expensive (cable pulls, always-on power, persistent vision compute). Not warranted unless Phase 3a/3b prove insufficient for actual operational needs.

Blocked on:

- LoRa hardware deployment and latency characterization (`LORA.md`)
- Camera infrastructure build-out (issue #23)
- Line-of-sight survey of the mailbox approach
- Candidate camera selection and wake-latency measurement
- Vision classification pipeline design (tied to `edgeai.local` infrastructure, issue #22)

Not a near-term priority. The email-state-based classification (Phase 3a) delivers most of the Phase 3 value; vehicle detection is a confidence-improvement enhancement for later.

### Synthesis Layer Architecture — Summary

```
┌──────────────────────────┐          ┌──────────────────────────┐
│ Email State Machine      │          │ Physical Event Stream    │
│ (Phase 1)                │          │ (Area: Driveway)         │
│                          │          │                          │
│ NOT_EXPECTING            │          │ mailbox/opened events    │
│ EXPECTING                │          │ mailbox/state telemetry  │
│ DELIVERED                │          │                          │
│ EXCEPTION                │          │                          │
└──────────────┬───────────┘          └──────────────┬───────────┘
               │                                     │
               └──────────────┬──────────────────────┘
                              │
                              ▼
                 ┌──────────────────────────┐
                 │ Synthesis Layer          │
                 │ (Utility: Deliveries)    │
                 │                          │
                 │ Pending-retrieval queue  │
                 │ Door-open classification │
                 │ Confidence reporting     │
                 │ Manual override handling │
                 └──────────────┬───────────┘
                                │
                                ▼
                 ┌──────────────────────────┐
                 │ Consumer-facing topics   │
                 │                          │
                 │ state/deliveries/pending │
                 │ event/deliveries/*       │
                 │ HA Discovery sensors     │
                 └──────────────────────────┘
```

The email state machine and physical event stream are **inputs** to the synthesis layer. They remain independent and internally consistent — neither knows about the other. The synthesis layer is where they compose to produce user-facing semantics.

Additional inputs (future vehicle detection, package tracking APIs, etc.) slot into this architecture without structural change. The synthesis layer's internal logic gets richer as inputs accumulate; its external contract stays stable.

---

## Open Questions

**Phase 1 — Informed Delivery (calibrate after PIN verification and real emails arrive):**

- [ ] Confirm exact sender address format once PIN verification completes and real emails arrive
- [ ] Confirm digest "no mail scheduled" text variant (may require observation across a Sunday/holiday) — noting that under the affirmative-signaling model, we may not need to parse this variant at all, since remaining in `NOT_EXPECTING` is already correct
- [ ] Validate piece count heuristic (image count ≈ piece count) against real digests
- [ ] Confirm `exception_time: "20:00"` against longer-term observation — recalibrate if ~6 months of data show distribution shift
- [ ] Finalize configuration file location (dedicated `deliveries.json` vs. broader grouping)

**Phase 3 — Physical sensor fusion (calibrate after LoRa installation and empirical observation):**

- [ ] Calibrate deferred-classification window behavior. Typical time between door-open-during-EXPECTING and `DELIVERED` arrival; fallback timeout for cases where `DELIVERED` never arrives.
- [ ] Define behavior for door-open in `NOT_EXPECTING` on known mail days (e.g., package drops that don't trigger Informed Delivery). Does this warrant an unexpected-activity notification, or is silently clearing the queue sufficient?
- [ ] Decide whether entering `EXCEPTION` state should fire an operator notification. Arguments for: USPS said mail was coming, it's past the window, operator may want to know. Arguments against: might result in routine "mail was late again" notifications if there are periods of systematic carrier lateness. Decide after observing real EXCEPTION frequency.
- [ ] Calibrate confidence thresholds. What actually counts as "multiple ambiguous events"? Empirical tuning required.
- [ ] Decide HA surface for manual override — voice intent, dashboard button, both?

**Phase 2 — Multi-carrier packages:**

- Captured in `AUTOMATION_BACKLOG.md` — separate design session when scoped.

---

*Last Updated: 2026-04-23*
