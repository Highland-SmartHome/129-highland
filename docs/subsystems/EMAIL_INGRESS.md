# Email Ingress — Design & Architecture

## Scope

The `Utility: Email Ingress` flow owns the **email intake surface** for the Highland system. It connects to the household Gmail account, watches for new messages via IMAP IDLE, normalizes them into a common schema, and publishes them to MQTT for consumption by downstream flows.

It is deliberately content-agnostic. It knows nothing about USPS, calendar invites, warranty notifications, or any other email source's meaning. Its only concerns are: hold a reliable IMAP connection, detect new mail, identify which consumer (if any) should receive it, publish in a consistent shape, and manage message lifecycle.

This is the email equivalent of `Utility: Device Registry` — a single owner for a cross-cutting concern so consuming flows don't each reimplement the same plumbing.

---

## Why a Dedicated Flow

Every email-driven subsystem would otherwise duplicate:

- IMAP connection management (credentials, TLS, reconnect handling, IDLE re-arming)
- Gmail rate-limit awareness (15 simultaneous IMAP connections per account)
- `Message-Id` deduplication across restarts
- Gmail label convention enforcement (`Highland/*` namespace)
- IMAP health monitoring and heartbeat
- App password revocation detection
- Processed-email retention and purge

Owning all of that in one place makes each consumer trivially simple — a consumer's entire job is "subscribe to my label's topic, parse content, publish semantic state." No IMAP code anywhere else in the system.

---

## Gmail Account

The flow connects to a single dedicated household Gmail account (referenced in `secrets.json` as `gmail.account`). The same account is used for Google Calendar and is the registered mailbox for USPS Informed Delivery. See `subsystems/DELIVERIES.md § Gmail Account` for rationale on the dedicated-account choice.

### Authentication

IMAP access requires 2FA on the account (Google deprecated "less secure app access" in 2022):

1. 2FA is enabled on the account.
2. A single app password is generated, scoped to "Mail," and named **"Node-RED Highland IMAP"** — an obvious name so a human reviewing the account's security page doesn't mistake it for suspicious activity.
3. The app password is stored in `secrets.json` under `gmail.app_password`.
4. The `Utility: Config Loader` flow populates `global.config.secrets.gmail` at startup, per the established project pattern.

App passwords are revocable independently of the account's primary credential.

---

## Architecture — Single Connection, All Mail Watch, Label Dispatch

### Why not one watcher per Highland folder?

Two options were considered and rejected:

**Per-folder IMAP connections** (one connection IDLE'd on each `Highland/*` folder) is simpler to reason about but burns connections against Gmail's 15-per-account ceiling. With multiple personal devices (phone, tablet, watch, computers) already consuming connections, leaving headroom matters.

**IMAP NOTIFY (RFC 5465) on multiple folders over a single connection** is the elegant IMAP-native answer. It was tested against Gmail on 2026-04-22 and found to be **not supported** — Gmail's IMAP server does not advertise `NOTIFY` in its post-auth capability list. Dead end.

### Why All Mail + X-GM-LABELS?

Gmail's `[Gmail]/All Mail` is a virtual folder containing every message in the account regardless of label. Combined with Gmail's `X-GM-EXT-1` IMAP extension, we can fetch `X-GM-LABELS` on every message — telling us, for each new mail, which user labels it has.

This gives us:
- **One IMAP connection** — low connection-budget impact
- **No dependency on a specific feature folder** — Informed Delivery ever changing or going away doesn't break the architecture
- **Auto-discovery of new consumers** — any mail labeled `Highland/<new_thing>` is automatically seen
- **Clean dispatch** — label hierarchy maps directly to MQTT topic hierarchy

Cost: the flow sees every email that hits the account, not just Highland-labeled ones. For each, it fetches envelope + labels, inspects the labels, and silently drops anything without a `Highland/*` label. Fetches are cheap; the "firehose" is trivial at personal-account volumes.

### Validated on Gmail (2026-04-22)

Confirmed working against the household Gmail account:
- IDLE on `[Gmail]/All Mail` fires `exists` events on new mail arrivals
- Gmail-side latency is typically 4–12 seconds between mail landing and event firing (Gmail's internal message-store propagation, not under our control)
- `X-GM-LABELS` fetch returns a mixed array of Gmail system labels (prefixed `\`, e.g. `\Inbox`, `\Important`, `\Sent`) and user labels (plain strings, e.g. `Highland/Deliveries/USPS`)
- Label-prefix filtering cleanly separates Highland-routable mail from everything else

---

## Label Namespace Convention

All automation-relevant email lives under a `Highland/` label prefix in Gmail, organized hierarchically by consuming flow. This establishes a clear boundary between "robot-managed mail" and "human-managed mail," and makes the flow that owns each email visible in the label itself.

```
Highland/
├── Deliveries/
│   ├── USPS                 (USPS Informed Delivery + any future USPS emails)
│   ├── FedEx                (future)
│   ├── UPS                  (future)
│   └── Amazon               (future)
└── Processed                (archival destination after ACK)
```

**Conventions:**

- Each consuming flow gets a namespace under `Highland/` (e.g., `Highland/Deliveries/`).
- Within each namespace, labels identify the **source** of the email (sender/carrier/service), not a product or feature name. `USPS`, not `Informed Delivery` — this survives vendors renaming or replacing product lines.
- A single flat `Highland/Processed` label serves as the archival destination for all consumed mail, regardless of origin namespace.
- Gmail filters (configured manually, once per source) apply the appropriate label to incoming mail and can optionally skip the inbox.
- Humans using the account for unrelated mail interact with regular Gmail; Highland labels are automation-facing.

**Adding a new source to an existing consumer** (e.g., adding FedEx to Deliveries):
1. Create a Gmail filter that applies `Highland/Deliveries/FedEx` to matching incoming mail
2. Add the new topic to the consumer flow's subscription (or rely on its existing wildcard subscription)

**Adding a new consumer entirely** (e.g., Warranties):
1. Decide the namespace name (e.g., `Highland/Warranties/`)
2. Create Gmail filters for each source under that namespace
3. Build the consumer flow, subscribed to `highland/event/email/<namespace>/+/received`

No ingress code or config changes required — the flow auto-discovers new labels as mail arrives carrying them.

### Label-to-topic derivation

The Gmail label is transformed into a consumer-facing MQTT topic suffix: strip the `Highland/` prefix, lowercase, replace whitespace with underscores. **Slashes in the label are preserved as MQTT topic separators.** This aligns Gmail's hierarchical label structure with MQTT's hierarchical topic structure, making wildcard subscriptions natural.

| Gmail Label | Derived Topic Suffix | Full MQTT Topic |
|-------------|---------------------|-----------------|
| `Highland/Deliveries/USPS` | `deliveries/usps` | `highland/event/email/deliveries/usps/received` |
| `Highland/Deliveries/FedEx` | `deliveries/fedex` | `highland/event/email/deliveries/fedex/received` |
| `Highland/Warranties/HVAC` | `warranties/hvac` | `highland/event/email/warranties/hvac/received` |

**Consumer subscription patterns:**

- Specific source: `highland/event/email/deliveries/usps/received`
- All sources for a flow: `highland/event/email/deliveries/+/received`
- Every Highland email event: `highland/event/email/#` (diagnostic only; not a typical production subscription)

---

## Message Lifecycle

```
[new message arrives in Gmail, picks up one or more labels]
         │
         ▼
[IDLE on All Mail fires 'exists' event]
         │
         ▼
[Ingress fetches new UID(s) with X-GM-LABELS]
         │
         ├── No Highland/* label? → silently drop
         │
         ▼
[Dedup check — Message-Id seen before?]
         │
         ├── Yes → silently drop
         │
         ▼
[Normalize → publish to MQTT]
         │
         ▼
[Consumer processes, publishes ACK]
         │
         ▼
[Ingress archives via Gmail label operations]
```

### Archival via Gmail label operations

In Gmail's IMAP model, there are no "folders" in the POP3 sense — everything is a label. Moving a message to a folder is equivalent to adding the destination label and removing the source label. Our archival operation after ACK:

1. Apply `Highland/Processed` label to the message
2. Remove the source `Highland/<namespace>/<source>` label from the message

After archival, the message still exists in All Mail (and in any Gmail system locations like `[Gmail]/All Mail` or `[Gmail]/Sent Mail`), but no longer carries a consumer-facing Highland label. It's out of scope for future dispatch but available for audit.

Implementation uses `X-GM-EXT-1`'s IMAP extensions for label modification. imapflow does not expose dedicated label methods; instead, labels are managed through the standard flag methods with `useLabels: true` passed in the options:

```javascript
// Add a Gmail label
await client.messageFlagsAdd(uid, ['Highland/Processed'], { uid: true, useLabels: true });

// Remove a Gmail label
await client.messageFlagsRemove(uid, ['Highland/Deliveries/USPS'], { uid: true, useLabels: true });
```

This is a Gmail-specific operation — it requires `X-GM-EXT-1` capability, which Gmail's IMAP server advertises.

### Key behaviors

- **State lives in Gmail labels, not in local context.** Flow-side tracking is just for dedup and in-flight ACK management. Gmail's label state is the authoritative record of what has been processed.
- **Idempotent.** A rolling record of recently-published `Message-Id` values in flow context (default store, disk-backed) prevents duplicate publishes if the same mail is seen twice. TTL: 24 hours, enforced by scheduled sweep.
- **ACK-gated archival.** After publishing, ingress waits for `highland/ack/email` with the matching `message_id` before archiving. This prevents loss if a consumer is down or restarting at publish time.
- **Fallback archival TTL.** If no ACK arrives within a configured window (default 24 hours), the message is archived anyway with a warning log. Prevents indefinite accumulation of unacked state when a consumer is genuinely broken or unconfigured.
- **Retention purge.** Messages carrying `Highland/Processed` older than a configurable retention (default 14 days) are permanently deleted by a scheduled sweep. This is the only destructive operation the flow performs.

---

## MQTT Topics

### Events (not retained)

One topic per derived label, with MQTT topic hierarchy mirroring Gmail label hierarchy. Topics exist only as consumers arrive — there is no static registry.

| Topic | Fires When |
|-------|-----------|
| `highland/event/email/<namespace>/<source>/received` | A new email with the corresponding Highland label is parsed and ready for consumption |

**Currently in use:**

| Gmail Label | Topic | Consumer |
|-------------|-------|----------|
| `Highland/Deliveries/USPS` | `highland/event/email/deliveries/usps/received` | `Utility: Deliveries` |

**Planned labels (as consumers come online):** captured in `AUTOMATION_BACKLOG.md`.

### Payload Schema

```json
{
  "message_id": "<CAxxxx@mail.gmail.com>",
  "label": "deliveries/usps",
  "gmail_label": "Highland/Deliveries/USPS",
  "from": "USPSInformeddelivery@email.informeddelivery.usps.com",
  "from_name": "USPS Informed Delivery",
  "to": "<household gmail>",
  "subject": "Your Daily Digest",
  "received_at": "2026-04-22T07:15:23-04:00",
  "body_text": "...",
  "body_html": "...",
  "attachment_count": 3
}
```

**Design notes:**

- `message_id` is the canonical identifier. Consumers use it for their own deduplication and must echo it in the ACK.
- `label` is the derived topic-suffix form (including slashes for hierarchy). `gmail_label` is the original Gmail label for reference/debugging.
- `body_text` and `body_html` are both included when available. Consumers pick whichever they prefer.
- **Attachments are not inlined.** `attachment_count` is metadata only. Binary attachments are deliberately excluded from the payload to keep MQTT traffic reasonable. No current consumer needs attachment bytes; if one ever does, a separate fetch-by-message-id mechanism will be added.
- HTML bodies for some senders (Informed Delivery digests with embedded scan references) can be large — several hundred KB. Mosquitto handles this fine at our volume (single-digit emails per day per label).

### ACK

Consumers publish an ACK after successful processing:

**`highland/ack/email`** — not retained

```json
{
  "message_id": "<CAxxxx@mail.gmail.com>",
  "consumer": "utility_deliveries",
  "processed_at": "2026-04-22T07:15:45-04:00",
  "status": "ok"
}
```

`status` values: `"ok"` | `"rejected"` | `"parse_error"`.

**On `status: "ok"`:** ingress archives the message (apply `Highland/Processed`, remove the source Highland label). Normal path.

**On `status: "rejected"`:** consumer saw the message but decided it wasn't theirs (e.g., sender mismatch for a shared label, or subject pattern the consumer doesn't handle). Ingress still archives — it has been "seen" — but tags the log entry. Current design assumes one consumer per label.

**On `status: "parse_error"`:** consumer tried to process but failed. Ingress leaves the message in its source label for retry, logs a warning, and publishes a notification on repeated failures (threshold configurable, default 3 attempts). This is the "something is broken" path — a parser bug or an unexpected email variant shouldn't silently vanish into `Highland/Processed`.

### Health

**`highland/status/email_ingress/health`** — retained

```json
{
  "status": "healthy",
  "imap_connection": "connected",
  "connected_at": "2026-04-22T07:20:00-04:00",
  "watching_folder": "[Gmail]/All Mail",
  "messages_in_flight": 0,
  "stale_messages": 0,
  "last_error": null,
  "timestamp": "2026-04-22T07:20:00-04:00"
}
```

`status` values: `"healthy"` | `"degraded"` | `"unhealthy"`.

Published on a periodic interval (60s) and on significant state changes.

Degraded conditions: messages past ACK TTL still in-flight, repeated parse failures from a consumer, recent but recovered IMAP errors.
Unhealthy conditions: IMAP authentication failure (likely app password revoked), connection closed and failing to re-establish, no new events in an unexpectedly long window.

Per `nodered/HEALTH_MONITORING.md`, this topic is a first-class health surface — Healthchecks.io pings from this flow monitor external availability.

---

## IMAP Connection Pattern

The flow holds a single long-lived IMAP connection via the `imapflow` npm package. This is the first flow in the Highland project to manage a persistent third-party library connection from a function node; the pattern is worth documenting explicitly.

### Module import

Per `nodered/ENVIRONMENT.md § Named Exports and the Destructure Pattern`, every function node that uses `imapflow` or `mailparser` declares them in its Setup tab with `Pkg` suffixes:

| Module name | Import as |
|---|---|
| `imapflow` | `imapflowPkg` |
| `mailparser` | `mailparserPkg` |

And destructures at the top of the function body:

```javascript
const { ImapFlow } = imapflowPkg;
const { simpleParser } = mailparserPkg;
```

### Context store usage

All internal state lives in the flow's own context — `flow.*`, not `global.*`. The flow's internal state is nobody else's business; consumers interact exclusively via MQTT. Only `global.config.secrets.gmail` (credentials from the Config Loader) and `global.utils.formatStatus` (shared status helper) are read from the global scope. Everything else is `flow.*`.

The IMAP client handle and the mailbox lock are stored in the `volatile` context store — in-memory, non-serializable, cleared on restart. Per `nodered/ENVIRONMENT.md § Context Storage`, `volatile` is the correct store for open connection handles. Note that `volatile` is a *store*, orthogonal to the `flow` vs `global` *scope* — `flow.get('key', 'volatile')` is valid and correct.

**Keys in use:**

| Key | Scope | Store | Purpose |
|-----|-------|-------|---------|
| `imap_client` | flow | volatile | The `ImapFlow` instance itself |
| `imap_connected_at` | flow | default | Timestamp of last successful connect (disk-persisted for health reporting across restarts) |
| `imap_last_error` | flow | default | Most recent connection error message, for degraded-status reporting |
| `imap_idle_lock` | flow | volatile | The mailbox lock held while IDLE is active |
| `imap_idle_running` | flow | volatile | Boolean guard preventing double-start of IDLE |
| `imap_last_uid_all_mail` | flow | default | High-water UID for All Mail; survives restarts to avoid backlog reprocessing |
| `imap_pending_ack` | flow | default | Map of `message_id` → `{ uid, gmail_label, published_at }` for in-flight ACK tracking |
| `imap_seen_<message_id>` | flow | default | Dedup entries with `{ seen_at }` timestamp for 24h TTL enforcement |

### Connection lifecycle events

`imapflow` emits `error` and `close` events when the connection drops. Handlers guard against races where a stale handler clobbers a newer connection reference:

```javascript
client.on('close', () => {
    if (flow.get('imap_client', 'volatile') === client) {
        flow.set('imap_client', null, 'volatile');
    }
});
```

"Only clear the handle if it's still pointing at me."

### Deploy cleanup

The IMAP-connection-managing function node uses the `On Stop` tab to gracefully log out on Node-RED deploy/restart, preventing connection leaks:

```javascript
// On Stop
const client = flow.get('imap_client', 'volatile');
if (client) {
    client.logout().catch(() => {});
    flow.set('imap_client', null, 'volatile');
}
```

---

## Flow Outline — `Utility: Email Ingress`

Per `nodered/OVERVIEW.md` conventions: groups are the primary organizing unit; link nodes between groups; no node has more than two outputs.

**Group 1 — Sinks**
- `inject` — Startup (once, ~3s delay after deploy)
- `inject` — Health ticker (every 60s)
- `inject` — ACK TTL sweeper (every 10min)
- `inject` — Retention sweeper (daily)
- `mqtt in` — `highland/ack/email`
- `mqtt in` — `highland/command/config/reload/secrets` (triggers disconnect + reconnect)
- Each sinks link-out to its respective downstream group

**Group 2 — Connection**
- `Ensure Connection` function node: returns existing client if healthy, else opens new one. Stores handle in `volatile`. Wires error/close handlers with identity guards.
- `On Stop` cleanup: graceful logout

**Group 3 — Watcher**
- `Start All Mail Watcher` function node: takes mailbox lock on `[Gmail]/All Mail`, records baseline UID, subscribes to `exists` events, fetches new UIDs with `X-GM-LABELS` on each event
- Emits one message per new mail (including those without Highland labels; filtering happens downstream)
- `On Stop` cleanup: release lock

**Group 4 — Label Filter**
- Function node: inspects `X-GM-LABELS` array, finds first `Highland/*` label (if any)
- Drops messages without a Highland label
- Derives the topic-suffix form (e.g., `deliveries/usps`) preserving slashes as MQTT hierarchy
- Attaches derived metadata to msg

**Group 5 — Dedup**
- Function node: checks `imap_seen_<message_id>` context key
- Drops duplicates, logs at debug level
- On new: sets the dedup key with current timestamp

**Group 6 — Normalize**
- Function node: fetches full message source with `mailparser`'s `simpleParser`
- Builds the standard payload schema
- Records in `imap_pending_ack` context

**Group 7 — Publish**
- `mqtt out` to `highland/event/email/<derived_label>/received`
- Non-retained, QoS 1

**Group 8 — ACK Handler**
- Function node: looks up `message_id` in `imap_pending_ack`
- On `ok`/`rejected`: dispatch to Label Archival
- On `parse_error`: increment retry counter, log, notify on threshold
- Removes the pending-ack entry

**Group 9 — Label Archival**
- Function node: applies `Highland/Processed`, removes the source `Highland/*` label on the UID
- Uses `client.messageAddLabels()` and `client.messageRemoveLabels()` from X-GM-EXT-1

**Group 10 — Lifecycle Sweepers**
- ACK TTL sweeper: scans `imap_pending_ack` for stale entries, force-archives, logs warning
- Retention sweeper: searches `Highland/Processed` for mail older than retention, deletes
- Dedup sweeper: scans `imap_seen_*` keys, removes entries older than TTL

**Group 11 — Health Publisher**
- Function node: assembles health payload from context
- Publishes retained to `highland/status/email_ingress/health`

**Group 12 — Error Handling**
- Standard project pattern: `catch` node + log/notify function node

---

## Configuration

Captured in `config/email_ingress.json` (pending final decision on file structure per `DELIVERIES.md` open question).

```json
{
  "imap": {
    "server": "imap.gmail.com",
    "port": 993,
    "tls": true
  },
  "watch_folder": "[Gmail]/All Mail",
  "label_prefix": "Highland/",
  "processed_label": "Highland/Processed",
  "ack_timeout_hours": 24,
  "processed_retention_days": 14,
  "parse_error_notification_threshold": 3,
  "dedup_ttl_hours": 24,
  "health_publish_interval_seconds": 60
}
```

Notably absent compared to earlier drafts: no `folders` array. The architecture is single-folder (All Mail) with dispatch-by-label.

---

## Startup Behavior

Per `nodered/STARTUP_SEQUENCING.md` patterns:

- The flow does **not** immediately process all mail in All Mail at startup. That would republish potentially years of backlog.
- The first thing the watcher does is read `imap_last_uid_all_mail` from persistent context. If present, it fetches any UIDs strictly greater than that value (catching up any mail that arrived while the flow was down). If absent, it records `uidNext - 1` as the baseline and begins watching fresh from that point.
- Existing mail with `Highland/*` labels that predates the flow's first run stays in place, untouched. Manual reprocessing, if ever needed, is a separate utility — not automatic startup behavior. (Likely implemented later as a `highland/command/email_ingress/reprocess` topic accepting a UID range.)

---

## Open Questions

- [ ] **Config file location.** `email_ingress.json` standalone vs folded into a broader file. Same question surfaced in `DELIVERIES.md`; settle both together.
- [ ] **Reprocessing mechanism.** A `highland/command/email_ingress/reprocess` topic for manual reprocessing of historical mail. Primary use case: filter criteria drift — if a Gmail filter stops matching (e.g., USPS changes sender domain), messages land unlabeled in the Inbox. After updating the filter and manually labeling the backlog, a reprocess command would flow those messages through the normal pipeline. Secondary use cases: operator-initiated reprocessing after parser fixes, or deliberate reclassification. This was explicitly considered vs. ambient label-change detection (via IMAP `flags` events) on 2026-04-22 — explicit reprocess was chosen for lower complexity, better signal-to-noise, and cleaner semantics. Payload shape TBD (candidates: `{uid_range}`, `{message_ids}`, `{label, since}`). Priority elevated from "future" to near-term.
- [ ] **Notification routing for ingress failures.** App-password revocation or IMAP auth failure needs to reach a human. Current thinking: tie into `Utility: Notifications` with severity `critical`, targeting the Daily Digest recipient. Confirm when notifications framework is wired into this flow.
- [ ] **Attachment fetch mechanism.** If a future consumer ever needs attachment bytes, a fetch-by-message-id pattern is the intended approach — but exact topic shape is deferred until a concrete consumer drives it.
- [ ] **Rejection semantics if multiple consumers share a label.** Current design assumes one consumer per label. If that assumption ever breaks, rework needed.
- [ ] **Gmail filter configuration tracking.** The manually-configured filters aren't version-controlled. Consider capturing them in a reference doc as they proliferate, so a rebuild isn't an archaeological dig through Gmail settings.

---

*Last Updated: 2026-04-22*
