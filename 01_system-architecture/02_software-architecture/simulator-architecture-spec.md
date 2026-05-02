---
Status: Reviewed
Last reviewed: 2026-05-02
Owner: Matteo Casavecchia
Version: 1.1
Italian working draft: available on request
---

# Simulator Architecture Specification
## Supply Open Sky

*Version 1.1 — April 2026*

---

> *This document is an adapted public version of an internal working specification maintained in the Supply Open Sky private repository. References to internal-only documents have been preserved as labels without links. The Italian working draft is available on request.*

---

## Table of Contents

1. [Architectural principles](#1-architectural-principles)
2. [Architectural schema](#2-architectural-schema)
3. [The SimClock and the speedup parameter](#3-the-simclock-and-the-speedup-parameter)
4. [Deployment flow](#4-deployment-flow)
5. [Simulation start/reset cycle](#5-simulation-startreset-cycle)
6. [Simulator startup flow](#6-simulator-startup-flow)
7. [Device impersonation](#7-device-impersonation)
8. [New sos-hub APIs](#8-new-sos-hub-apis)
9. [New sos-scheduler APIs](#9-new-sos-scheduler-apis)
10. [Complete startup sequence](#10-complete-startup-sequence)
11. [Implementation status](#11-implementation-status)

---

## §1 Architectural principles

### §1.1 Roles

The Supply Open Sky simulator is a **mock device layer** that replaces physical devices (drones, nodes) during development and testing. It is not an autonomous system with its own decision logic: it is a realistic counterpart for `sos-hub` and `sos-scheduler`.

| Component | Role |
|---|---|
| `sos-hub` | Operational brain. Receives telemetry, calls the scheduler, manages the dashboard. |
| `sos-scheduler` | Plans missions, writes `hop_instructions` to the DB. |
| `sos-simulator` | Impersonates drones and nodes. Reads `hop_instructions` from the DB. Responds with realistic telemetry. |
| Dashboard | Operator interface. Starts the simulation, displays state in real time. |

### §1.2 Hub-agnostic simulation

`sos-hub` **does not know and must not know** the simulation mode. It receives the same REST calls it would receive from physical hardware (or from the LoRa Adapter). It has no "sim mode" flag, no special endpoints for the simulator.

The only difference from production is the clock speed: in simulation, the system runs faster than real time thanks to the `speedup` parameter.

### §1.3 Fidelity to the real communication channel

**Inbound channel (device → hub):**
In production: `[Physical device] → [LoRa mesh] → [LoRa Adapter] → [REST POST] → [sos-hub]`
In simulation: `[SimAgent] → [REST POST] → [sos-hub]`

The simulator calls the same REST endpoints that the LoRa Adapter would call. Same payload structure, same frequency.

**Outbound channel (hub → device):**
In production: `[sos-hub/scheduler] → [DB hop_instructions] → [LoRa Adapter reads DB] → [LoRa mesh] → [Device firmware reads plan]`
In simulation: `[sos-hub/scheduler] → [DB hop_instructions] → [SimAgent DroneAgent reads DB]`

The `DroneSimAgent` reads `hop_instructions` from the shared SQLite DB — exactly as the drone firmware would receive its Mission Plan before takeoff. This is not a workaround: it is the correct model.

---

## §2 Architectural schema

```
┌─────────────────────────────────────────────────────────────┐
│  OPERATOR (Dashboard)  — Simulation Panel                   │
│                                                             │
│  PRE-START state:                                           │
│    [  6×  ]  [ 10×  ]  [ 60×  ]    [ Restart (disabled) ]   │
│                                                             │
│  RUNNING state (after clicking one of the three buttons):   │
│    [6× dis] [10× dis] [60× dis]    [    Restart           ] │
│    "Simulation active — 60×"                                │
└──────────────┬──────────────────────────────────────────────┘
               │ HTTP REST / WebSocket
               ▼
┌─────────────────────────────────────────────────────────────┐
│  sos-hub  (TypeScript/Express)                              │
│                                                             │
│  GET  /api/clock              → {slotNow, slotDurationS}    │
│  GET  /api/deployment         → {topology, devices}         │
│  POST /api/sim/start          → configures speedup + starts │
│  POST /api/sim/reset          → stops + clears data         │
│  POST /api/sim/register-devices → registers virtual devices │
│  POST /api/telemetry/drone    ← from SimAgent / LoRa Adapter│
│  POST /api/telemetry/node     ← from SimAgent / LoRa Adapter│
│  POST /api/events/*           ← from SimAgent / LoRa Adapter│
│                                                             │
│  ┌─────────────────────┐                                    │
│  │ sos-scheduler       │ (Python/FastAPI)                   │
│  │ POST /schedule      │                                    │
│  │ pad_slot_duration = │                                    │
│  │   360 / speedup     │                                    │
│  └──────────┬──────────┘                                    │
│             │ writes                                        │
│             ▼                                               │
│  ┌─────────────────────┐                                    │
│  │  sos_scheduler.db   │  (SQLite — shared)                 │
│  │  - hop_instructions │  ← cleared by /api/sim/reset       │
│  │  - pad_reservations │  ← cleared by /api/sim/reset       │
│  │  - missions         │  ← cleared by /api/sim/reset       │
│  │  - nodes, segments  │  persist (deployment data)         │
│  └─────────────────────┘                                    │
└─────────────┼───────────────────────────────────────────────┘
              │ reads (same DB connection)
              ▼
┌─────────────────────────────────────────────────────────────┐
│  sos-simulator  (Python)                                    │
│                                                             │
│  sim-config.yaml → hub_url, scheduler_db, impersonation     │
│  GET /api/clock  → slot_duration_s (synchronisation)        │
│  GET /api/deployment → topology + devices to impersonate    │
│                                                             │
│  SimLoop (tick every slot_duration_s real seconds)          │
│  ├── DroneSimAgent × N  (physics, GPS, battery)             │
│  └── NodeSimAgent × M   (sensors, rack, Pad)                │
│                                                             │
│  → POST /api/telemetry/drone  (to sos-hub)                  │
│  → POST /api/telemetry/node   (to sos-hub)                  │
│  → POST /api/events/*         (to sos-hub)                  │
│                                                             │
│  WS ← SIM_RESET broadcast → SimLoop stops                   │
└─────────────────────────────────────────────────────────────┘
```

---

## §3 The SimClock and the speedup parameter

### §3.1 Constants and derivations

```
PRODUCTION_SLOT_DURATION_S = 360   # physical constant (6 minutes = 1 slot in production)

speedup             = configured by the operator (6 | 10 | 60; default: 1 in production)
slot_duration_s     = 360 / speedup
planningIntervalMs  = slot_duration_s * 1000
```

Examples:

| Speedup | slot_duration_s | planningIntervalMs | Note |
|---|---|---|---|
| 1× | 360 s | 360 000 ms | Production — real time |
| 6× | 60 s | 60 000 ms | Slow test, visible cycle |
| 10× | 36 s | 36 000 ms | Intermediate test |
| 60× | 6 s | 6 000 ms | Fast demo |

### §3.2 Clock synchronisation

`sos-hub` is the authoritative clock. It exposes `GET /api/clock` which returns:

```json
{
  "slotNow": 42,
  "slotDurationS": 6.0,
  "speedup": 60,
  "simEpochMs": 1743799200000
}
```

The simulator calls this endpoint **once at startup** to:
1. Obtain `slot_duration_s` to use in its own `SimClock`
2. Align the initial slot with the hub's current one

After initial alignment, hub and simulator advance autonomously using the same formula:

```python
slot_now = floor((time.time() * 1000 - sim_epoch_ms) / (slot_duration_s * 1000))
```

**No continuous IPC for the clock is needed.** Both use the same formula with the same parameters.

### §3.3 Simulation states and speedup immutability constraint

The simulation has two states relevant for the dashboard:

**PRE-START:** no planning cycle active. The three speedup buttons `[6×]` `[10×]` `[60×]` are enabled. The `[Restart]` button is disabled. Clicking one of the three speedup buttons starts the simulation with that value.

**RUNNING:** planning cycle active. The three speedup buttons are disabled. The `[Restart]` button is enabled.

The `speedup` parameter **cannot be modified after start**. The slot numbers in the `hop_instructions` in the DB have an absolute meaning: `slot_launch=100` means "100 × slot_duration_s seconds after sim_epoch". If `slot_duration_s` changed mid-session, all already-written plans would become inconsistent.

### §3.4 Restart button behaviour

The `[Restart]` button calls `POST /api/sim/reset`. This endpoint:

1. Stops the planning cycle (`clearInterval`)
2. Clears operational data from the DB (tables `missions`, `hop_instructions`, `pad_reservations`; resets `last_scheduled_slot` in `node_schedule_configs`)
3. **Does not clear** deployment data (topology, devices, batteries — they persist)
4. Resets in-memory state on hub (`droneState`, `nodeState`, `missionIndex`)
5. Resets the clock (`slotNow=0`, `speedup=1`, `slotDurationS=360`, `simEpochMs=0`)
6. WebSocket `SIM_RESET` broadcast

Upon receipt of `SIM_RESET`, the simulator stops its `SimLoop`. The operator must manually restart the simulator process for the next session (or the simulator can reconnect and wait for the next `SIM_STARTED`).

Dashboard after `SIM_RESET`: returns to **PRE-START** state (speedup buttons re-enabled, Restart disabled).

### §3.5 Scheduler synchronisation

The scheduler internally uses `pad_slot_duration` for `current_slot()`, `travel_slots()` and `try_reserve()`.

**Implementation choice:** hub includes `slot_duration_s` in every `POST /schedule` call. The scheduler uses the received value for that specific cycle. If not present in the body, it uses the default from its own config (360 = production). This avoids scheduler restarts on speedup changes between sessions.

---

## §4 Deployment flow

The "deployment" is the act of configuring a real (or test) site before starting any process. It is a **one-off** operation (or to be repeated only if topology/fleet change). It is not affected by the simulation start/reset cycle.

```
1. sos-hub seed --deployment deployments/<site>/
   ├── reads topology.yaml        → seeds nodes, segments, node_configs
   ├── reads drone-devices.yaml   → seeds drone_devices
   ├── reads battery-devices.yaml → seeds battery_models, batteries, battery_slots
   └── reads deployment-state.yaml → places batteries in racks and on drones

2. start sos-scheduler (uvicorn)
3. start sos-hub (node)
```

At this point, hub and scheduler are ready. The DB contains deployment data but no planned missions.

---

## §5 Simulation start/reset cycle

```
Dashboard PRE-START → click [60×]
   → POST /api/sim/start {speedup: 60}
   → hub: slot_duration_s=6, simEpochMs=now, starts setInterval(6000ms)
   → broadcast WS: SIM_STARTED {speedup: 60, slotDurationS: 6, simEpochMs: ...}
   → Dashboard: disables [6×][10×][60×], enables [Restart], shows "60×"

sos-simulator start --config sim-config.yaml
   → GET /api/clock → slot_duration_s=6, syncs SimClock
   → GET /api/deployment → loads topology+devices
   → starts SimLoop (tick every 6s real)

[simulation in progress — drones and nodes send telemetry]

Dashboard RUNNING → click [Restart]
   → POST /api/sim/reset
   → hub: stops interval, clears operational DB data, resets state
   → broadcast WS: SIM_RESET
   → simulator receives SIM_RESET → stops SimLoop
   → Dashboard: enables [6×][10×][60×], disables [Restart]

[operator restarts sos-simulator for new session]
[operator picks new speedup and clicks]
```

---

## §6 Simulator startup flow

```
sos-simulator start --config sim-config.yaml

1. Loads sim-config.yaml
   - hub_url, scheduler_db
   - impersonation: mode, extra_nodes, extra_drones

2. GET /api/clock (hub)
   - If slotNow == 0 and simEpochMs == 0: hub in PRE-START, wait for SIM_STARTED via WS
   - Otherwise: initialise SimClock with slot_duration_s and current slot_now

3. GET /api/deployment (hub)
   → complete topology + list of active devices

4. Filter devices to impersonate (based on impersonation.mode)

5. If extra_nodes/extra_drones present:
   → POST /api/sim/register-devices

6. Create DroneSimAgent × N, NodeSimAgent × M

7. Start SimLoop (tick every slot_duration_s real seconds)

8. Listen on WS: on SIM_RESET → stop SimLoop + exit
```

---

## §7 Device impersonation

### §7.1 Modes

**`mode: "all"`** — impersonates all devices in the deployment. Standard use case.

**`mode: "select"`** — impersonates only the listed devices. Useful for:
- Partial test (e.g., only 1 drone)
- Real-hardware companion: physical devices in the field + simulator for the rest

### §7.2 Extra devices (virtual devices)

Defined in `sim-config.yaml`, not present in the real deployment. Use cases:
- **Network expansion test:** virtual node to verify scheduler behaviour
- **Fleet capacity test:** virtual drone to verify batch planning
- **Demo:** "future" network with additional nodes

Virtual devices are inserted in the DB with `is_virtual=true` and visually marked in the dashboard.

### §7.3 sim-config.yaml format

```yaml
# sim-config.yaml — Supply Open Sky
# Simulator configuration. Does not contain topology nor fleet.
# gitignored — master kept in internal documentation.

simulation:
  hub_url: "http://127.0.0.1:3000"
  scheduler_db: "../scheduler/sos_scheduler.db"

impersonate:
  mode: "all"          # "all" | "select"
  # If mode == "select":
  # nodes:  ["N1", "N1.1", "N1.1.1"]
  # drones: ["DRN-001"]
  extra_nodes: []
  extra_drones: []

scenario:
  on_demand_events: []
  fault_events: []
```

---

## §8 New sos-hub APIs

### `POST /api/sim/start`

Starts the simulation with the given speedup. Corresponds to the click on one of the speedup buttons in the dashboard.

**Body:** `{ "speedup": 60 }`

**Behaviour:**
- Computes `slot_duration_s = 360 / speedup`, `planningIntervalMs = slot_duration_s × 1000`
- Records `sim_epoch_ms = Date.now()`
- Starts `setInterval(runPlanningTick, planningIntervalMs)`
- Updates in-memory clock
- Broadcasts WS `SIM_STARTED`

**Response:** `{ "speedup": 60, "slotDurationS": 6.0, "simEpochMs": 1743799200000 }`

**Errors:** HTTP 409 if simulation already active.

---

### `POST /api/sim/reset`

Stops the simulation and resets all operational data. Corresponds to clicking `[Restart]` in the dashboard.

**Body:** none.

**Behaviour:**
1. `clearInterval` of the planning cycle
2. Clears from DB: `DELETE FROM hop_instructions`, `DELETE FROM pad_reservations`, `DELETE FROM missions`
3. Resets: `UPDATE node_schedule_configs SET last_scheduled_slot = NULL`
4. Empties in-memory: `droneState`, `nodeState`, `missionIndex`
5. Resets clock: `slotNow=0, speedup=1, slotDurationS=360, simEpochMs=0`
6. Broadcasts WS `SIM_RESET`

**Response:** `{ "ok": true }`

**Note:** nodes, segments, node_configs, drone_devices, batteries, battery_slots **are not cleared**. These are deployment data and persist across sessions.

---

### `GET /api/clock`

**Response:**
```json
{
  "slotNow": 42,
  "slotDurationS": 6.0,
  "speedup": 60,
  "simEpochMs": 1743799200000
}
```

In PRE-START state: `slotNow=0, speedup=1, slotDurationS=360, simEpochMs=0`.

---

### `GET /api/deployment`

Returns topology and active devices from the DB.

**Response:**
```json
{
  "hub": { "id": "HUB", "name": "Main Hub", "lat": 0.000, "lon": 0.000 },
  "nodes": [
    {
      "id": "N1", "name": "N1", "parentId": "HUB",
      "lat": -0.050, "lon": 0.020,
      "padCount": 2, "rackSlots": 22,
      "tankCapacityL": 25.0, "minLevelRatio": 0.10,
      "isLeaf": false, "isVirtual": false
    }
  ],
  "drones": [
    { "id": "DRN-001", "droneType": "WATER", "status": "ACTIVE", "isVirtual": false }
  ]
}
```

> **Note:** the GPS coordinates above are illustrative placeholders. Real deployment coordinates are loaded from the deployment YAML files described in §4.

---

### `POST /api/sim/register-devices`

Registers virtual devices in the DB (extra_nodes/extra_drones from sim-config).

**Body:**
```json
{
  "nodes": [{ "id": "N1-VIRTUAL", "parentId": "N1", "lat": -0.070, "lon": 0.040,
               "distanceKm": 5.0, "travelTimeMin": 8.6, "padCount": 2,
               "rackSlots": 22, "tankCapacityL": 25.0, "minLevelRatio": 0.10,
               "isLeaf": true, "isVirtual": true }],
  "drones": []
}
```

---

## §9 New sos-scheduler APIs

### `POST /schedule` — additional `slot_duration_s` field

```json
{
  "type": "SCHEDULED_BATCH",
  "slot_now": 42,
  "slot_duration_s": 6.0,
  "sensor_levels": { "N1": 0.72 },
  "drone_ids_available": ["DRN-001"]
}
```

The scheduler uses `slot_duration_s` for that cycle. If absent: uses the default from its own config (360).

---

## §10 Complete startup sequence (60× regional deployment)

```bash
# One-off: seed DB
node hub/dist/cli.js seed \
  --deployment deployments/<site> \
  --db scheduler/sos_scheduler.db

# Terminal A: scheduler
cd scheduler && uvicorn sos_scheduler.api:app --port 8000

# Terminal B: hub
cd hub && npm run dev

# Browser: http://localhost:3000
# → Dashboard shows [6×] [10×] [60×] enabled, [Restart] disabled
# → Click [60×] → POST /api/sim/start {speedup:60} → planning starts
# → Dashboard: [6×][10×][60×] disabled, [Restart] enabled

# Terminal C: simulator
cd simulator
python -m sos_simulator start --config sim-config.yaml
# → syncs clock, creates agents, starts SimLoop

# To reset:
# → Dashboard: click [Restart] → POST /api/sim/reset
# → simulator receives SIM_RESET and stops
# → restart simulator for new session
```

---

## §11 Implementation status

All items below are currently in design phase. Implementation is tracked in the internal development roadmap.

| Component | Status |
|---|---|
| DB schema (fleet tables + is_leaf/is_virtual) | ⬜ Design — not yet implemented |
| `sos-hub seed` CLI (reads 4 deployment YAML files) | ⬜ Design — not yet implemented |
| `GET /api/deployment` | ⬜ Design — not yet implemented |
| `GET /api/clock` | ⬜ Design — not yet implemented |
| `POST /api/sim/start` | ⬜ Design — not yet implemented |
| `POST /api/sim/reset` | ⬜ Design — not yet implemented |
| `POST /api/sim/register-devices` | ⬜ Design — not yet implemented |
| Dynamic speedup in hub (parameterised `setInterval`) | ⬜ Design — not yet implemented |
| `slot_duration_s` field in `/schedule` | ⬜ Design — not yet implemented |
| Simulator refactor (impersonation) | ⬜ Design — not yet implemented |
| Dashboard: simulation panel (4 buttons) | ⬜ Design — not yet implemented |

---

*Reference: [NOMENCLATURE](../../NOMENCLATURE.md), [hub-architecture-spec](./hub-architecture-spec.md), [scheduling-algorithm-spec](../03_scheduling-algorithm/scheduling-algorithm-spec.md).*

*Revision log:*

| Version | Date | Notes |
|---|---|---|
| 1.0 | April 2026 | First draft — simulator architecture redesign |
| 1.1 | April 2026 | Added dashboard simulation panel (4 buttons + Restart); `POST /api/sim/reset`; §3.3–3.4 state behaviour; §5 start/reset cycle; §6 startup flow updated |
