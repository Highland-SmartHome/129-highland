# Highland Home Automation — Project Context

## What This Is

Ground-up rebuild of an existing Home Assistant infrastructure. The goal is a distributed, resilient system where protocol coordinators and automations survive Home Assistant restarts — eliminating single points of failure.

**Replaces:** A monolithic HAOS installation with Node-RED as an add-on, Zigbee coordinator on USB passthrough, and automations tightly coupled to HA availability.

**Domain:** `your-domain.example` (Nabu Casa for remote access)

---

## Document Map

| Document | Purpose | When to Reference |
|----------|---------|-------------------|
| **ARCHITECTURE.md** | Hardware allocation, topology, backup strategy, migration approach | System-level questions, hardware decisions, recovery scenarios |
| **EVENT_ARCHITECTURE.md** | MQTT topic structure, payloads, periods, command/event patterns | Inter-flow communication, adding new topics, message formats |
| **MQTT_TOPICS.md** | Authoritative registry of all `highland/` topics | Adding/modifying topics, payload schemas, publisher/consumer mapping |
| **ENTITY_NAMING.md** | Naming conventions for HA entities, disambiguation rules | Adding new devices, renaming entities, ensuring consistency |
| **NODERED_PATTERNS.md** | Flow organization, utilities, config management, logging, notifications, health monitoring | Building/modifying flows, implementing new utilities |
| **APPLIANCE_MONITORING.md** | Power-based cycle detection for washer/dryer/dishwasher | ZEN15 integration, state machine logic, energy-gate thresholds |
| **VIDEO_PIPELINE.md** | Camera motion detection → AI triage → notification | Video analysis architecture, NVR integration, cooldown/kill switch |
| **WEATHER_FLOW.md** | Weather data synthesis (Tempest + Pirate Weather) | Weather automation, precipitation events, polling behavior |
| **CALENDAR_INTEGRATION.md** | Google Calendar bridge for automation triggers | Camera suppression, planned events, kill switch UX |
| **LORA.md** | LoRaWAN sensors (mailbox, driveway bins) | Mailbox state machine, bin tracking, RSSI-based zone detection |
| **GARAGE_DOOR.md** | Konnected GDO blaQ integration via Node-RED bridge | SSE state streaming, REST commands, MQTT Discovery |
| **ASSIST_PIPELINE.md** | HA Assist voice pipeline with Marvin persona | Voice control, STT/TTS, conversation agents, wake word |
| **PERSISTENT_MEMORY.md** | AI assistant memory architecture and defense layers | Memory poisoning defense, tool autonomy, voice identity |
| **RUNBOOK.md** | Step-by-step implementation guide | Building infrastructure, phase-by-phase setup |
| **AUTOMATION_BACKLOG.md** | Ideas for future automations | Capturing new ideas, reviewing what's planned |

---

## Key Decisions (Don't Re-Litigate)

These have been discussed and decided. Reference the relevant doc for rationale.

| Decision | Rationale |
|----------|-----------|
| **Four-box architecture** | HAOS, Communication Hub (MQTT/Z2M/Z-Wave), Workflow (Node-RED), Edge AI — physical separation for resiliency |
| **MQTT as control plane** | Node-RED subscribes directly to Z2M/Z-Wave topics; critical automations work without HA |
| **MQTT-triggered backups** | Each host owns its backup via MQTT command; no SSH between hosts |
| **File-based config** | External JSON files in `/home/nodered/config/`; `secrets.json` gitignored |
| **Schedex for time triggers** | Period configuration lives in schedex nodes, not external config |
| **node-red-contrib-home-assistant-websocket** | For HA integration (backups, notifications, entity state) |
| **JSONL logging** | Unified daily log files, 30-day retention, jq for querying |
| **HA Companion App** | Primary notification channel (Android); future channels deferred |
| **Google Calendar** | For scheduled events (recurring + one-time); queried once daily for digest |
| **Markdown → HTML** | Daily digest built as markdown, converted via node-red-node-markdown |
| **Healthchecks.io** | External monitoring; watchdog script pings on Node-RED heartbeat |

---

## Hardware

| Role | Hardware | Status |
|------|----------|--------|
| **HAOS** | Dell OptiPlex 7050 SFF (i7-7700, 16GB) | Ready |
| **Workflow** | Dell OptiPlex 7050 SFF (i7-7700, 16GB) | Ready |
| **Communication Hub** | Ryzen 5 3550H MFF (16GB, 512GB SSD) | Ready |
| **Edge AI Box** | Dell OptiPlex 7050 SFF (i7-7700, 32GB, Coral TPU) | Ready |

---

## Current State

**Phase:** Documentation complete, hardware in-hand — ready to build

**What's done:**
- Architecture finalized (hardware, topology, backup strategy)
- Event architecture defined (topics, payloads, periods)
- MQTT topic registry established (authoritative reference)
- Entity naming standards established
- Node-RED patterns documented (flows, config, logging, notifications, health monitoring)
- Domain-specific designs complete (video pipeline, weather, calendar, LoRaWAN, voice)
- Implementation runbook complete
- GitHub repository synced (`Highland-SmartHome/129-highland`)
- All open questions resolved

**What's next:**
1. Carve out time to begin build
2. Build order: Communication Hub first, then HAOS, then Workflow, then Edge AI
3. Commit working configs to GitHub as baseline after each machine

---

## Working Style

**Communication:**
- Peer-level, informal — we're well-acquainted colleagues
- Colorful language acceptable in moderation
- Direct and pragmatic over cautious and verbose

**Preferences:**
- Prefer pragmatic over perfect
- Avoid over-engineering; solve for today's problems
- Separation of concerns is a core value
- Intentional design choices over convenience (e.g., Greek letters over numbers)
- "Zero-baggage" migration — rebuild from scratch, don't copy legacy

**When uncertain:**
- Ask clarifying questions rather than assuming
- Present options with tradeoffs when decisions are needed
- Reference existing docs before proposing something that may conflict

---

## Protocols

**Modifying project-level files:**

Project files (`/mnt/project/`) are read-only from Claude's perspective. To modify them:

1. **Create a session copy** — Copy the file to `/mnt/user-data/outputs/` for editing
2. **Make changes** — Edit the session copy as needed
3. **Always present the file** — Use `present_files` after every modification (this busts a caching bug that can return stale content)
4. **User promotes** — Manually delete the old project-level file, then promote the session artifact to take its place

This workflow avoids fragile `str_replace` operations and provides a clean review checkpoint before changes land in project context.

**IMPORTANT: Always update the "Last Updated" timestamp** at the bottom of any document when making changes. This is easy to forget — make it habitual.

**Privacy & Security in Documentation:**

This repository is public. When creating or updating documents, always redact identifying information:
- **GPS coordinates** — Use `secrets.json` references, not literal lat/lon values
- **Domain names** — Use `your-domain.example` as a placeholder for actual FQDNs
- **Email addresses** — Reference `secrets.json` or use generic descriptions; never include actual addresses
- **Internal hostnames/IPs** — Use `.local` mDNS names or RFC1918 example ranges (`192.168.x.x`)
- **API keys, tokens, passwords** — Always use placeholders (`YOUR_API_KEY`, `GENERATE_NEW_KEY`, etc.)

Example patterns already in use:
- `secrets.json` stores credentials and is `.gitignore`d
- `thresholds.json` and other config files contain no secrets
- Healthchecks.io UUIDs shown as `uuid-1`, `uuid-2`, etc.

When in doubt, ask: *"Could someone use this information to locate or access the system?"* If yes, redact it.

**Project files vs GitHub:**

- Project files (`/mnt/project/`) are the **working copy** — always current, always in context
- GitHub (`Highland-SmartHome/129-highland`) is the **versioned baseline** — synced at milestones
- Work happens against project files; GitHub commits are intentional checkpoints
- Sync to GitHub at: end of significant sessions, before/after implementation phases, or on request
- This approach keeps docs available in both Claude Desktop and claude.ai without MCP dependency

**Capturing ideas:**
- New automation ideas → AUTOMATION_BACKLOG.md
- Don't derail current work; capture and move on

---

*Last Updated: 2026-03-13*
