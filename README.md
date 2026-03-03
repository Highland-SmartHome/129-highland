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
```

**Key Design Principles:**
- **Resiliency** — Protocol coordinators and automations survive HA restarts
- **Separation** — Logical AND physical separation of concerns
- **Scalability** — Room to grow without architectural rework
- **Zero-baggage migration** — Rebuild from scratch, don't copy legacy

## Documentation

All planning and reference documentation lives in [`/docs`](./docs/):

| Document | Purpose |
|----------|--------|
| [ARCHITECTURE.md](docs/ARCHITECTURE.md) | Hardware, topology, backup strategy |
| [ENTITY_NAMING.md](docs/ENTITY_NAMING.md) | HA entity naming conventions |
| [EVENT_ARCHITECTURE.md](docs/EVENT_ARCHITECTURE.md) | MQTT topics and event patterns |
| [NODERED_PATTERNS.md](docs/NODERED_PATTERNS.md) | Flow organization and utilities |
| [AUTOMATION_BACKLOG.md](docs/AUTOMATION_BACKLOG.md) | Future automation ideas |
| [RUNBOOK.md](docs/RUNBOOK.md) | Step-by-step implementation guide |
| [README.md](docs/README.md) | Project context and working agreements |

## Repository Structure

```
129-highland/
├── docs/           # Planning & reference documentation
├── node-red/       # Flows, package.json, utilities
├── pnc/            # Protocol Nerve Center configs
├── ha/             # Home Assistant configuration
└── scripts/        # Backup scripts, utilities
```

## Domain

`highland.ferris.network` — Remote access via Nabu Casa

## Status

**Phase:** Documentation complete, hardware pending

---

*Project epoch: 2026-03-03*
