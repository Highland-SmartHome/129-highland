# Matter Integration

Planned integration path for Matter devices into Highland via a standalone `matter.js`-based bridge.

**Status:** 📋 Planned — no Matter devices currently in scope. Architecture feasibility validated; build deferred until a specific device forces the issue.

---

## Scope

Highland does not currently have any Matter devices. This document captures the agreed-on integration path so that when a compelling device appears — most likely a lock, given market trend — we can move from design to implementation in a small number of focused sessions rather than re-litigating architecture.

The trigger is expected to be a specific device, not a general platform commitment. Matter-over-WiFi plugs and sensors rarely offer anything our existing Zigbee/Z-Wave ecosystem doesn't already cover. The compelling cases are:

- **Matter-native locks** (Aqara U200, Schlage Encode Plus, future Yale/Level variants) where the Matter version offers something the Z-Wave or proprietary equivalent does not
- **New device categories** where Matter is the only reasonable option
- **Multi-admin scenarios** where a device must be shared with another ecosystem (Apple Home, Google Home) alongside Highland

Until one of those triggers fires, this stays on the shelf.

---

## Approach

A standalone Node.js bridge service built directly on the [`matter.js`](https://github.com/project-chip/matter.js) library. `matter.js` is the official certified TypeScript implementation of the Matter protocol — part of the `project-chip` umbrella, same as the canonical CHIP SDK.

The bridge's single responsibility is protocol translation: wrap a `matter.js` `CommissioningController`, subscribe to its event stream, and republish into MQTT. Node-RED never sees Matter protocol concerns; it sees MQTT topics on the `matter/` namespace, the same way it sees Zigbee devices on `zigbee2mqtt/` and Z-Wave devices on `zwave/`.

A Highland Node-RED translation flow subscribes to `matter/#`, applies node-to-device mapping, and republishes into the `highland/` namespace following existing conventions. This is the same two-layer pattern used for Zigbee and Z-Wave: the protocol bridge owns its raw surface, Highland consumes at the boundary and normalizes from there.

---

## Architecture

```
Matter devices (Thread / Wi-Fi)
        ▲
        │  802.15.4 (Thread) / LAN (Wi-Fi)
        │
 Thread Border Router ◄──── SMLight SLZB-06Mg26 (PoE, dedicated appliance)
        │
        │  LAN (mDNS / SRP)
        ▼
  matter.js CommissioningController
        │
        │  in-process
        ▼
  matter-bridge (TypeScript service, Docker on Comm Hub)
        ▲
        │  MQTT
        ▼
      matter/#   ← raw protocol surface, owned by the bridge
        ▲
        │  MQTT
        ▼
  Node-RED translation flow (Subsystem: Matter Bridge)
        ▲
        │  MQTT
        ▼
      highland/# ← normalized, consumed by HA and the rest of Highland
```

### Service Placement

- **Bridge service** runs on the **Comm Hub** alongside Mosquitto, Zigbee2MQTT, Z-Wave JS UI, and (eventually) eufy-bridge. Deployed as a Docker container.
- **Thread Border Router** is a separate physical appliance — not a service on any host.

### Bridge Responsibilities

- Commission / decommission Matter nodes
- Maintain fabric credentials and node storage (persisted to a `/data` volume)
- Subscribe to all attributes and events on all commissioned nodes
- Publish raw Matter state and events to `matter/#`
- Accept commissioning and decommissioning commands via MQTT
- Report bridge liveness and commissioning results

### Translation Flow Responsibilities

- Subscribe to `matter/#`
- Maintain node-to-logical-device mapping (from Highland config, not bridge state)
- Normalize attribute and event payloads into `highland/state/...` and `highland/event/...` shapes
- Publish HA MQTT Discovery configs for each logical device
- Translate `highland/command/...` messages into `matter/bridge/command/...` calls when needed

---

## Thread Border Router

Matter locks and most battery-powered Matter devices communicate over Thread, not Wi-Fi. A Thread Border Router (OTBR) is required to bridge the Thread mesh to the LAN.

### Chosen Approach: Dedicated PoE Appliance

**SMLight SLZB-06Mg26** or equivalent — a PoE-powered network appliance running OpenThread Border Router firmware. Sits on the LAN, gets a static lease, has no software lifecycle to manage.

Selection rationale:
- **No host coupling** — not tied to HAOS, Comm Hub, or any Highland service. Survives any host reboot or rebuild.
- **No software maintenance** — firmware updates are vendor-managed and rare.
- **Same pattern as the network switch** — infrastructure, not a service.
- **Philosophically aligned with Highland** — HA is not in the path; the Comm Hub is not burdened with another daemon.
- **Thread mesh is range-tolerant** — mains-powered Thread devices act as routers and extend the mesh, so OTBR placement in the network closet is typically sufficient.

### Alternatives Considered

| Option | Why not |
|--------|---------|
| HA's OTBR add-on | Puts HA in the Thread network path; conflicts with "HA is consumer only" principle |
| `ot-br-posix` native on Comm Hub | Another daemon to maintain; potential mDNS/Avahi contention with existing services; IPv6 routing changes on the LAN |
| ESP32-C6 DIY | Custom firmware, ongoing maintenance burden, not worth the savings |
| Ecosystem byproduct (Apple TV, Nest Hub, Eero, etc.) | Thread 1.4 cross-vendor credential sharing still rolling out through 2026; we're not in any of those ecosystems anyway; creates multi-network fragmentation |

### Pre-Purchase Recommendation

Hardware lead time is the slow part of bootstrapping this subsystem. When next placing an order for other Highland hardware, include the SLZB-06Mg26 (or equivalent). Stash unpowered until needed. The software portion of the integration can be built in a weekend once the trigger arrives; hardware shipping should not be on the critical path.

---

## Bridge Topic Surface: `matter/#`

The bridge owns its own top-level namespace. `matter/` is not part of Highland's `highland/` namespace — it is the bridge's private protocol surface, consumed by a translation flow and no one else.

This mirrors how `zigbee2mqtt/` and `zwave/` are structured: raw protocol data lives under the bridge's own prefix, and Highland flows consume at the boundary.

### Topic Conventions

| Topic | Direction | Retained | Purpose |
|-------|-----------|----------|---------|
| `matter/bridge/status` | Publish | Yes | Bridge liveness — `online` / `offline` / `degraded` |
| `matter/nodes` | Publish | Yes | Commissioned node list with metadata |
| `matter/<node_id>/<endpoint>/<cluster>/<attribute>` | Publish | Yes | Current attribute value |
| `matter/<node_id>/<endpoint>/<cluster>/event/<name>` | Publish | No | Matter event (e.g. `DoorLock.LockOperation`) |
| `matter/<node_id>/status` | Publish | Yes | Per-node connection state (`connected` / `disconnected` / `reconnecting`) |
| `matter/bridge/commission/request` | Subscribe | No | Pairing code + optional friendly name |
| `matter/bridge/commission/result` | Publish | No | Commissioning outcome |
| `matter/bridge/decommission/request` | Subscribe | No | Remove node from fabric |
| `matter/bridge/decommission/result` | Publish | No | Decommission outcome |
| `matter/bridge/command/<node_id>/<endpoint>/<cluster>/<command>` | Subscribe | No | Invoke a Matter cluster command |
| `matter/bridge/command/result` | Publish | No | Command result (success, error, correlation ID) |

### Payload Shapes (sketch)

Attribute update:
```json
{
  "value": 85,
  "timestamp": "2026-04-20T14:30:00Z"
}
```

Event fire:
```json
{
  "event": "DoorLock.LockOperation",
  "data": { "operation": "Lock", "source": "Manual", "user_index": null },
  "timestamp": "2026-04-20T14:30:00Z"
}
```

Commission request:
```json
{
  "pairing_code": "12345-67890",
  "friendly_name": "Front Door Lock",
  "correlation_id": "commission-2026-04-20-01"
}
```

These are illustrative — exact shapes settle at implementation time.

---

## Translation to Highland Namespace

A Node-RED flow (`Subsystem: Matter Bridge`) subscribes to `matter/#` and applies Highland's existing conventions to republish into `highland/`.

### Responsibilities

- **Identity mapping** — Matter nodes are identified by node ID (an integer). Highland devices are identified by logical name. The flow maintains a map: `node_id 42 → front_door_lock, device_class=lock, area=foyer`. This map lives in a config file loaded by Config Loader, not in the bridge.
- **Namespace projection** — `matter/42/1/DoorLock/LockState` becomes `highland/state/lock/front_door` (retained JSON). Matter events become `highland/event/lock/front_door/...`.
- **Command translation** — `highland/command/lock/front_door` → `matter/bridge/command/42/1/DoorLock/LockDoor`, with correlation ID propagation for ACK tracking.
- **HA Discovery** — per-device Discovery configs published under `homeassistant/...` so HA auto-creates entities on Highland's terms, not Matter's.
- **Availability** — bridge `online/offline` and per-node `connected/disconnected` feed into Highland's availability model.

### What the Bridge Does Not Know

The bridge has no knowledge of:
- Highland's topic namespace or naming conventions
- Logical device names, areas, or classes
- HA Discovery
- Any consumer of its data

This separation means:
- The bridge is reusable outside Highland (it's just a `matter.js` → MQTT adapter)
- The bridge is replaceable — if we ever swap `matter.js` for something else, the blast radius is the translation flow, not downstream automations
- Schema changes in `matter.js` between versions are absorbed by the translation flow, not propagated through the house

---

## Commissioning UX

No standalone web UI. The control plane is MQTT, and existing Highland surfaces drive it.

### Common Path: Phone-Side Commissioning

New devices are commissioned from a phone, standing in front of the device being added.

1. HA Companion App QR scanner reads the Matter pairing QR code
2. Scanner writes the decoded pairing code to an `input_text` helper
3. HA automation publishes to `matter/bridge/commission/request` with the pairing code and a user-provided friendly name
4. Bridge attempts commissioning; publishes result to `matter/bridge/commission/result`
5. HA displays result on a Lovelace card

A manual fallback (text field for typing the 11-digit manual pairing code) handles cases where the QR isn't scannable.

HA's role here is strictly input collection and result display — the actual commissioning happens in the bridge, and HA being offline during commissioning would only affect the commissioning UI, not any already-commissioned device.

### Admin Path: Node-RED Dashboard

A Node-RED Dashboard 2.0 tab renders:
- Current node list (from retained `matter/nodes`)
- Per-node status and last-seen
- Decommission button per node
- Commissioning window controls (for sharing to another ecosystem)
- Basic diagnostics (attribute dumps, endpoint structure)

This is the admin surface for anything more complex than "add a new device."

### CLI Path

`mosquitto_pub` against the commission/decommission topics. Useful for scripting, initial setup, and fabric rebuild from a backup manifest.

---

## Node-to-Device Mapping

The mapping of Matter node IDs to logical Highland devices is a Highland concern, not a bridge concern.

**Mapping lives in Highland config** — likely an extension of `device_registry.json` or a parallel `matter_devices.json` loaded by Config Loader at startup. Format TBD at implementation time.

**Bridge persists only Matter-level state** — fabric credentials, node IDs, commissioned node metadata. Friendly names are not the bridge's concern.

**Mapping is created at commission time** — when the commissioning request includes a friendly name, that name is recorded in the Highland config (not the bridge). If the config is ever lost but the fabric is intact, the bridge can re-enumerate nodes and the config can be rebuilt.

---

## Alternatives Considered

### Why not `python-matter-server`?

`python-matter-server` is the Python wrapper around the CHIP SDK that powers Home Assistant's Matter integration. As of early 2026 it is explicitly in maintenance mode — the project is being rewritten on top of `matter.js`. New features are landing in the successor project only.

Building directly on `matter.js` skips the intermediate Python layer entirely. No reason to target a codebase that's on ice when the successor library is already what the rewrite will use. `matter.js` is also a more natural fit for Highland's Node.js-heavy stack than a Python FFI wrapper around a C++ SDK.

### Why not HA's Matter integration?

HA's Matter integration treats Matter devices as HA entities. Control flows through HA; outages take Matter devices with them. For most device categories this is an acceptable tradeoff; for security-critical devices like locks, it directly violates Highland's core principle that HA is a consumer, not a dependency.

The bridge approach keeps Matter devices out of the HA critical path while still allowing HA to see and interact with them via MQTT Discovery, same as every other Highland device.

### Why not a server/WebSocket architecture like `python-matter-server`?

`python-matter-server` exposes a WebSocket API so that multiple clients (HA, web dashboards, mobile apps) can share one fabric. Highland has one controller's worth of needs — no second client to accommodate. Removing the WebSocket layer removes an entire category of failure modes and reduces the bridge to a straightforward library-plus-MQTT service.

---

## Triggers for Implementation

This subsystem moves from Planned to Active when one of the following occurs:

1. **A specific Matter device lands in the procurement queue** — typically a lock, possibly a sensor in a category we can't easily source in Zigbee or Z-Wave.
2. **A multi-admin requirement emerges** — a device must be shared with another ecosystem alongside Highland.
3. **A Z-Wave or Zigbee device we already own fails and the best replacement happens to be Matter-only.**

In absence of any of these, speculative infrastructure work on this subsystem is explicitly deprioritized in favor of work with concrete current-day outcomes.

### Estimated Effort

Roughly 2–3 focused sessions once triggered:
- Bridge service (TypeScript, ~500–800 lines including MQTT client, commissioning RPC, Discovery of underlying cluster types)
- Node-RED translation flow (one tab, comparable in scope to Eufy or Zigbee translation)
- Commissioning UX (HA card + Node-RED admin panel)

Assumes PoE Thread dongle is already on hand.

---

## Implementation Notes

### BLE Requirement for Commissioning

Most Matter devices — especially Thread devices — require BLE for initial commissioning. The `matter.js` controller accepts Wi-Fi or Thread credentials to pass over BLE during onboarding.

The Comm Hub (OptiPlex MFF) is the intended bridge host. Verify Bluetooth hardware is present and exposed to the Docker container before committing. If BLE is not available, a USB BT dongle is a trivial addition. BLE is only required during commissioning, not at runtime.

### Thread Operational Dataset

Commissioning a Thread device requires the current operational dataset from the OTBR to hand to the device over BLE. Two approaches:

1. **Pre-load at startup** — extract the dataset from the SMLight web UI once at OTBR setup, stash in `secrets.json`, load at bridge startup. Simple; the dataset only changes if the Thread network is regenerated.
2. **Query at commission time** — have the bridge hit the OTBR's REST API at commission time to fetch the current dataset. More robust against dataset rotation but adds a runtime dependency on OTBR availability during commissioning.

Pre-load is the default choice. Revisit if dataset rotation becomes a real-world concern.

### Fabric Label

`matter.js` supports an `adminFabricLabel` on the controller. Set this to `Highland` explicitly. This label is visible to other ecosystems if a device is ever multi-admin'd, and makes the fabric identifiable in diagnostic tools.

### Docker Deployment

- `--network host` likely required for mDNS-based Matter device discovery on the LAN
- `/data` volume for fabric credentials and node storage (persists across container restarts)
- BLE access via `/run/dbus` mount (only relevant if BLE is needed)
- Runs alongside Mosquitto, Zigbee2MQTT, Z-Wave JS UI, and eufy-bridge on the Comm Hub

---

## Open Questions

These resolve at implementation time, not now.

- Exact `matter/` topic shapes (cluster name casing, event payload conventions) — settle against observed `matter.js` output from the first commissioned device
- Whether to extend `device_registry.json` or create a parallel `matter_devices.json` for node-to-device mapping
- Whether to run the bridge from a published Docker image (if one exists for `matter.js` bridges) or build in-house
- Exact commissioning QR scanning integration with HA Companion App — whether `input_text` + automation is clean enough or if a custom card is warranted
- Multi-admin handoff flow — if we ever want to share a Matter device with Apple Home, exactly how the "open commissioning window" command flows from request to completion

---

*Last Updated: 2026-04-20*
