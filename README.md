# 129 Highland — Smart Home Infrastructure

Ground-up rebuild of a 6-year-old Home Assistant infrastructure. The goal is a distributed, resilient system where protocol coordinators and automations survive Home Assistant restarts — eliminating single points of failure.

## Architecture Overview

```
┌─────────────────────────┐
│   HAOS (Dedicated HW)   │
│   Dell OptiPlex SFF     │
│   • Home Assistant      │
│   • Frontend/UI         │
└───────────┬─────────────┘
            │ MQTT / WebSocket
            ▼
┌─────────────────────────┐     ┌─────────────────────────┐
│  Protocol Nerve Center  │     │   Node-RED / Utility    │
│  Dell OptiPlex MFF      │     │   Dell OptiPlex SFF     │
│   • MQTT Broker         │◄───►│   • Node-RED            │
│   • Zigbee2MQTT         │     │   • PostgreSQL          │
│   • Z-Wave JS UI        │     │   • Utility services    │
└─────────────────────────┘     └─────────────────────────┘
            │
            │ (Future)
            ▼
┌─────────────────────────┐
│   Edge AI Box           │
│   (SFF + Coral TPU)     │
│   • CodeProject.AI      │
│   • Ollama (LLM)        │
│   • Local inference     │
└─────────────────────────┘
```

**Key Design Principles:**
- **Resiliency** — Protocol coordinators and automations survive HA restarts
- **Separation** — Logical AND physical separation of concerns
- **Scalability** — Room to grow without architectural rework
- **Zero-baggage migration** — Rebuild from scratch, don't copy legacy

## Documentation

All planning and reference documentation lives in [`/docs`](./docs/):

### Core Architecture
| Document | Purpose |
|----------|---------|
| [ARCHITECTURE.md](docs/ARCHITECTURE.md) | Hardware, topology, backup strategy |
| [EVENT_ARCHITECTURE.md](docs/EVENT_ARCHITECTURE.md) | MQTT event patterns and philosophy |
| [MQTT_TOPICS.md](docs/MQTT_TOPICS.md) | Authoritative MQTT topic registry |
| [NODERED_PATTERNS.md](docs/NODERED_PATTERNS.md) | Flow organization, utilities, config management |
| [ENTITY_NAMING.md](docs/ENTITY_NAMING.md) | HA entity naming conventions |

### Domain-Specific
| Document | Purpose |
|----------|---------|
| [VIDEO_PIPELINE.md](docs/VIDEO_PIPELINE.md) | Camera motion detection → AI triage → notification |
| [WEATHER_FLOW.md](docs/WEATHER_FLOW.md) | Weather data synthesis (Tempest + Pirate Weather) |
| [CALENDAR_INTEGRATION.md](docs/CALENDAR_INTEGRATION.md) | Google Calendar bridge for automation triggers |
| [LORA.md](docs/LORA.md) | LoRaWAN sensors (mailbox, driveway bins) |
| [ASSIST_PIPELINE.md](docs/ASSIST_PIPELINE.md) | HA Assist voice pipeline with Marvin persona |

### Reference
| Document | Purpose |
|----------|---------|
| [RUNBOOK.md](docs/RUNBOOK.md) | Step-by-step implementation guide |
| [AUTOMATION_BACKLOG.md](docs/AUTOMATION_BACKLOG.md) | Future automation ideas |
| [README.md](docs/README.md) | Project context and working agreements |

## Repository Structure

```
129-highland/
├── docs/           # Planning &amp; reference documentation
├── node-red/       # Flows, package.json, utilities
├── pnc/            # Protocol Nerve Center configs
├── ha/             # Home Assistant configuration
└── scripts/        # Backup scripts, utilities
```

## Domain

`highland.ferris.network` — Remote access via Nabu Casa

## Status

**Phase:** Documentation complete, awaiting hardware (SSDs on order)

---

*Project epoch: 2026-03-03*
