---
Status: Reviewed
Last reviewed: 2026-05-02
Owner: Matteo Casavecchia
Version: 1.3
Italian working draft: available on request
---

# Hub Architecture Specification
## Supply Open Sky

*Version 1.3 — April 2026*

---

> *This document is an adapted public version of an internal working specification maintained in the Supply Open Sky private repository. References to internal-only documents have been preserved as labels without links. The Italian working draft is available on request.*

---

## Table of Contents

1. [Scope and perimeter](#1-scope-and-perimeter)
2. [Architectural principles](#2-architectural-principles)
3. [sos-hub — Internal architecture](#3-sos-hub--internal-architecture)
   - 3.1 [Layer model](#31-layer-model)
   - 3.2 [State Layer — in-memory structures](#32-state-layer--in-memory-structures)
   - 3.3 [Inbound REST API](#33-inbound-rest-api-from-hardwaresimulator)
   - 3.4 [Outbound REST API](#34-outbound-rest-api-toward-hardwaresimulator)
   - 3.5 [Dashboard REST API](#35-dashboard-rest-api-operator)
   - 3.6 [WebSocket /ws/live](#36-websocket-wslive)
   - 3.7 [Scheduler Client](#37-scheduler-client)
   - 3.8 [Startup and configuration](#38-startup-and-configuration)
   - 3.9 [Event Logger](#39-event-logger)
   - 3.10 [Prometheus Metrics Exporter](#310-prometheus-metrics-exporter)
   - 3.11 [Reset Detection via Heartbeat (placeholder)](#311-reset-detection-via-heartbeat-placeholder)
4. [sos-simulator — Design](#4-sos-simulator--design)
   - 4.1 [Transparency principle](#41-transparency-principle)
   - 4.2 [Components](#42-components)
   - 4.3 [SimClock and time compression](#43-simclock-and-time-compression)
   - 4.4 [DroneAgent](#44-droneagent)
   - 4.5 [NodeAgent](#45-nodeagent)
   - 4.6 [ScenarioEngine and CLI](#46-scenarioengine-and-cli)
   - 4.7 [Scenario configuration format (superseded)](#47-scenario-configuration-format-superseded)
5. [sos-dashboard — Design](#5-sos-dashboard--design)
   - 5.1 [Technology stack](#51-technology-stack)
   - 5.2 [Frontend architecture](#52-frontend-architecture)
   - 5.3 [Panels](#53-panels)
   - 5.4 [Data flow](#54-data-flow)
6. [Interface payloads](#6-interface-payloads)
   - 6.1 [Drone telemetry](#61-drone-telemetry)
   - 6.2 [Node sensors](#62-node-sensors)
   - 6.3 [Events](#63-events)
   - 6.4 [WebSocket messages](#64-websocket-messages)
   - 6.5 [State snapshot](#65-state-snapshot)
7. [Revision log](#7-revision-log)

---

## 1. Scope and perimeter

This document specifies the architecture of three software components: `sos-hub`,
`sos-simulator` and `sos-dashboard`.

`sos-hub` is the central production component: it receives telemetry from the hardware
(or from the simulator), maintains the live state of the entire network, delegates
planning decisions to `sos-scheduler`, and serves the dashboard to the operator.

`sos-simulator` is a development and validation tool: it impersonates hardware (drones,
nodes, sensors) by calling the same `sos-hub` REST endpoints that real hardware would
use. It is not a production component.

`sos-dashboard` is the operational interface: a React app served by `sos-hub` as static
files, displaying the network state in real time and allowing the operator to plan
ON-DEMAND missions.

**Out of perimeter:**
- The LoRa Mesh protocol (`sos-lora`) — specified in [communication-protocol-spec](../04_communication-protocol/communication-protocol-spec.md)
- The Node firmware (`sos-node`) — *internal node firmware specification*, tracked in a future development phase
- The drone firmware (`sos-drone`) — *internal companion firmware specification*, tracked in a future development phase
- Scheduling algorithm details — specified in [scheduling-algorithm-spec](../03_scheduling-algorithm/scheduling-algorithm-spec.md)

---

## 2. Architectural principles

**P1 — Simulator transparency.**
`sos-hub` contains no conditional logic that distinguishes real hardware from the
simulator. The simulator calls the same endpoints the hardware would call. When real
hardware arrives, it will connect without modifying `sos-hub`.

**P2 — Centralised in-memory live state.**
`sos-hub` keeps the current snapshot of the entire network in memory: position and
state of each drone, sensor levels of each node, active missions. This state is
reconstructed on receipt of every telemetry message or event. It is not persisted to
DB by `sos-hub`; mission persistence is the responsibility of `sos-scheduler`.

**P3 — Real-time push via WebSocket.**
The dashboard does not poll. Every state update is immediately transmitted to
connected clients via WebSocket. HTTP polling is only used for the initial snapshot
load.

**P4 — Decisions delegated to the scheduler.**
`sos-hub` plans nothing. Every event requiring a scheduling decision (new planning
cycle, DELIVERY_COMPLETE, ON-DEMAND request) is forwarded to `sos-scheduler` via
the internal REST API.

**P5 — Transparent time compression.**
Accelerated simulation is obtained by starting `sos-scheduler` with a reduced
`pad_slot_duration`. `sos-hub` and `sos-dashboard` are unaware of the acceleration:
they always work with the real server time.

---

## 3. sos-hub — Internal architecture

### 3.1 Layer model

```
┌──────────────────────────────────────────────────────────────┐
│  Transport Layer                                             │
│  Express HTTP + WebSocket (ws library)                       │
│  Receives requests, routes to internal layers                │
└────────────┬──────────────────────┬──────────────────────────┘
             │                      │
             ▼                      ▼
┌────────────────────┐   ┌──────────────────────────────────┐
│  Telemetry         │   │  Event Processor                 │
│  Ingestor          │   │  Receives DELIVERY_COMPLETE,     │
│  Updates State     │   │  ON_DEMAND_REQUEST, NODE_ALERT   │
│  Layer with        │   │  → calls Scheduler Client        │
│  flight/sensor     │   │  → updates State Layer           │
│  data              │   │                                  │
└────────┬───────────┘   └──────────────┬───────────────────┘
         │                              │
         └────────────┬─────────────────┘
                      ▼
┌──────────────────────────────────────────────────────────────┐
│  State Layer  (in-memory)                                    │
│  droneState: Map<droneId, DroneState>                        │
│  nodeState:  Map<nodeId, NodeState>                          │
│  missionIndex: Map<missionId, MissionSummary>                │
│  alerts: Alert[]                                             │
└────────────┬──────────────────────────────────────────────── ┘
             │ on every modification
             ▼
┌─────────────────────────────┐    ┌──────────────────────────┐
│  WebSocket Broadcaster      │    │  Scheduler Client        │
│  Sends diff to all          │    │  Calls sos-scheduler     │
│  connected dashboard        │    │  REST API (internal)     │
│  clients                    │    │                          │
└─────────────────────────────┘    └──────────────────────────┘
             ▲
             │ initial polling + operator actions
┌─────────────────────────────┐
│  Dashboard API              │
│  GET /api/state             │
│  GET /api/missions          │
│  POST /api/missions/on-demand │
│  DELETE /api/missions/{id}  │
└─────────────────────────────┘
```

### 3.2 State Layer — in-memory structures

#### DroneState

```typescript
interface DroneState {
  droneId:        string;
  missionId:      string | null;       // null when drone at HUB on standby
  status:         DroneStatus;         // STANDBY | IN_FLIGHT | ON_PAD | ELZ_WAIT
  flightDirection: 'OUTBOUND' | 'RETURN' | null;   // null if not in flight
  onPadActivity:   'BATTERY_SWAP' | 'WATER_DISCHARGE' | 'CARGO_DROP' | 'FAULT' | null; // null if not on Pad
  currentLat:     number;
  currentLon:     number;
  batteryPct:     number;              // 0–100
  payloadKg:      number | null;       // null for WATER missions (uses water_l)
  waterRemainingL: number | null;      // null for non-WATER missions
  lastUpdatedAt:  number;              // Unix timestamp
}
```

#### NodeState

```typescript
interface NodeState {
  nodeId:          string;
  name:            string;
  status:          NodeStatus;         // ONLINE | DEGRADED | OFFLINE
  tankLevelRatio:  number;             // 0.0–1.0; null for tankless nodes
  batteryStockPct: number;             // charged batteries / total rack batteries
  padStatus:       PadStatus[];        // array of pad_count elements
  lastUpdatedAt:   number;
}

interface PadStatus {
  padIndex:  number;                   // 0 or 1
  occupied:  boolean;
  droneId:   string | null;            // drone present on the pad
}
```

#### MissionSummary

```typescript
interface MissionSummary {
  missionId:    string;
  flightMode:   'SCHEDULED' | 'ON_DEMAND';
  missionType:  'WATER' | 'MEDICAL' | 'POSTAL' | 'SUPPLY';
  status:       string;                // MissionStatus from sos-scheduler
  droneId:      string | null;
  leafNodeId:   string | null;
  slotLaunch:   number | null;
}
```

#### Alert

```typescript
interface Alert {
  alertId:    string;
  severity:   'INFO' | 'WARNING' | 'CRITICAL';
  sourceId:   string;                  // nodeId or droneId
  message:    string;
  timestamp:  number;
  resolved:   boolean;
}
```

### 3.3 Inbound REST API (from hardware/simulator)

These endpoints receive data from physical hardware in production, or from the
simulator during development/test. `sos-hub` does not distinguish between the two
sources.

#### POST /api/telemetry/drone

Position and state update for a drone in flight. Called at regular intervals during
flight (frequency configurable in the simulator; in production it depends on the RF
transmission frequency).

```
Body → see §6.1
Response: 200 { received: true }
```

#### POST /api/telemetry/node

Sensor readings for a node (tank level, Pad status, battery stock). In production
sent by the node firmware via LoRa Mesh; in the simulator by the NodeAgent.

```
Body → see §6.2
Response: 200 { received: true }
```

#### POST /api/events/delivery-complete

Notification of completed delivery. `sos-hub` forwards it to `sos-scheduler` for
return path planning.

```
Body → see §6.3.1
Response: 200 { scheduleId: string }
```

#### POST /api/events/on-demand-request

New ON-DEMAND request originated by a Field Transmitter or by the operator via
dashboard. `sos-hub` inserts it into the `sos-scheduler` queue.

```
Body → see §6.3.2
Response: 202 { requestId: string }
```

#### POST /api/events/node-alert

Anomaly reported by a node (Pad fault, tank offline, degraded communication).

```
Body → see §6.3.3
Response: 200 { alertId: string }
```

### 3.4 Outbound REST API (toward hardware/simulator)

These endpoints are called by `sos-hub` to send commands to hardware components
(drones, nodes). In production, delivery happens via LoRa/Reticulum; in simulation,
via REST toward the agents' mini servers.

Each outbound call is subject to retry with backoff (3 attempts, 2 s between
attempts) to model the unreliable nature of LoRa.

#### POST → `<deviceAddress>/node-prep` (CTL_NODE_PREP)

Sent by Hub to a Node when a mission traversing it is planned. Prepares the Node
with the hop instruction it shall deliver to the drone at touchdown.

```typescript
Body: {
  missionId:      string;
  droneId:        string;
  slotArrival:    number;        // estimated drone arrival slot
  hopInstruction: HopPayload;    // see below
}
Response: 200 { ack: true, nodeId: string }
```

#### POST → `<deviceAddress>/hop-instruction` (CTL_HOP_INSTRUCTION)

Sent by Hub directly to a Drone in two cases:
1. **Launch:** first hop of the mission, sent to the drone at the HUB before takeoff.
2. **Fallback:** when the Node has not delivered the hop to the drone within
   `HOP_INSTRUCTION_TIMEOUT_S` and the drone has sent `CTL_HOP_REQUEST`.

```typescript
Body: {
  droneId:        string;
  hopInstruction: HopPayload;
  source:         'hub';         // distinguishes from delivery via Node
}
Response: 200 { ack: true }
```

#### HopPayload — common structure

```typescript
interface HopPayload {
  missionId:         string;
  missionType:       'WATER' | 'MEDICAL' | 'POSTAL' | 'SUPPLY';
  hopSequence:       number;
  sourceNodeId:      string;
  sourceLat:         number;
  sourceLon:         number;
  destinationNodeId: string | null;   // null for ON-DEMAND hops toward GPS
  destinationLat:    number;
  destinationLon:    number;
  action:            'LAND' | 'DROP' | 'RETURN_TO_HUB';
  waterReleaseL:     number | null;
  slotDeparture:     number;
  direction:         'OUTBOUND' | 'RETURN';
  travelSlots:       number;          // pre-computed traversal slots
}
```

#### Device Address Registry

Hub maintains a `Map<deviceId, baseUrl>` registry populated at simulator boot via
`POST /api/sim/register-devices` (which now includes the `address` field for each
device). When real hardware arrives, it will pass its IP/port address to the same
endpoint — Hub does not distinguish the two sources.

```typescript
// In POST /api/sim/register-devices:
{ id: 'DRN-001', droneType: 'WATER', address: 'http://127.0.0.1:9101' }
{ id: 'N1',      address: 'http://127.0.0.1:9201' }
```

#### Automatic distribution at planning

After each call to `triggerScheduledBatch()` in `planning.ts`, Hub:
1. Reads the missions just planned from the scheduler.
2. For each intermediate node of the path (action=LAND): sends `CTL_NODE_PREP`
   with the hop instruction the node shall deliver to the drone.
3. For each launching drone: sends `CTL_HOP_INSTRUCTION` with the first hop.

The same mechanism activates after `plan_return()` (DELIVERY_COMPLETE):
Hub distributes `CTL_NODE_PREP` to nodes along the return path and
`CTL_HOP_INSTRUCTION` to the drone for the first return hop.

---

### 3.5 Dashboard REST API (operator)

#### GET /api/state

Complete system snapshot in a single payload. Used by the dashboard at initial
load and for reconciliation after WebSocket reconnection.

```
Response → see §6.5
```

#### GET /api/missions

List of active missions (status PLANNED, IN_FLIGHT, RETURN_IN_PROGRESS,
EARLY_RETURN). Proxy toward `sos-scheduler GET /schedule/active`.

#### GET /api/missions/:missionId

Detail of a specific mission. Proxy toward `sos-scheduler GET /missions/{id}`.

#### POST /api/missions/on-demand

Operator action: plan a new ON-DEMAND mission from the dashboard.
Inserts the request into the scheduler queue via `POST /api/events/on-demand-request`.

```typescript
Body: {
  missionType: 'MEDICAL' | 'POSTAL' | 'SUPPLY';
  targetLat:   number;
  targetLon:   number;
  payloadKg:   number;    // max 10.0
  priority:    number;    // 1 = highest priority
  notes:       string;    // operator notes (optional)
}
Response: 202 { requestId: string }
```

#### DELETE /api/missions/:missionId

Operator action: cancel a planned mission. Proxy toward
`sos-scheduler DELETE /missions/{id}`. Not applicable to IN_FLIGHT missions.

---

#### POST /api/fleet/drones

Registers a new drone in the fleet. Writes to `drone_devices` in the scheduler DB.

```typescript
Body: {
  droneId:   string;
  droneType: 'WATER' | 'SUPPLY' | 'BATTERY_SWAP';
  modelId:   string;    // reference to DroneSpec
}
Response: 201 { droneId: string }
```

#### DELETE /api/fleet/drones/:droneId

Removes a drone from the active fleet. Not applicable if the drone is `IN_FLIGHT`.
Releases the current battery (returns to availability in the rack).

```
Response: 200 { droneId: string, batteryReleased: string | null }
```

#### POST /api/fleet/batteries

Adds physical batteries to a node (e.g., after field delivery).
Writes to `batteries` and `battery_slots` in the scheduler DB.

```typescript
Body: {
  nodeId:   string;
  modelId:  string;    // reference to BatteryModel
  count:    number;    // number of batteries to add
  slotFrom: number;    // first rack slot to assign (0-indexed)
}
Response: 201 { batteryIds: string[] }
```

#### DELETE /api/fleet/batteries/:batteryId

Removes a battery from the fleet (status `FAULTY` or `DECOMMISSIONED`).
Not applicable if the battery is `IN_FLIGHT`.

```
Response: 200 { batteryId: string, slotFreed: string | null }
```

#### POST /api/topology/nodes

Adds a new node to the topology (append-only — see [scheduling-algorithm-spec](../03_scheduling-algorithm/scheduling-algorithm-spec.md) §1.4).
Calls `POST /topology/reload` on the scheduler to hot-reload the topology.

```typescript
Body: {
  nodeId:     string;
  name:       string;
  parentId:   string;
  gpsLat:     number;
  gpsLon:     number;
  distanceKm: number;
}
Response: 201 { nodeId: string, segmentAdded: boolean }
```

#### GET /health

Liveness check. Responds 200 if hub and scheduler are operational.

```typescript
Response: {
  status:         'ok' | 'degraded';
  schedulerOk:    boolean;
  activeClients:  number;        // connected WebSocket clients
  uptimeS:        number;
}
```

### 3.6 WebSocket /ws/live

Dashboard clients connect to `ws://<host>/ws/live`. On connection, `sos-hub`
immediately sends a `FULL_STATE` message with the current snapshot. Each subsequent
update is transmitted as a differential message.

The base format of every WebSocket message is:

```typescript
interface WsMessage {
  type:      WsMessageType;
  timestamp: number;             // Unix timestamp
  payload:   object;
}
```

Message types and their payloads are defined in §6.4.

The connection is unidirectional (server → client). Operator actions
(ON-DEMAND planning, mission cancellation) use the REST API, not the WebSocket.

The `LOG_EVENT` type is emitted whenever the Event Logger (§3.9) records a
significant event, allowing the dashboard to update the Logs panel
(§5.3) in real time without polling on `GET /api/logs`.

### 3.7 Scheduler Client

`sos-hub` communicates with `sos-scheduler` via internal HTTP (localhost). Calls
are synchronous (await) within event processing.

| Event received | Call to sos-scheduler |
|---|---|
| Periodic cycle (every N slots) | `POST /schedule { type: "SCHEDULED_BATCH", ... }` |
| `DELIVERY_COMPLETE` (SCHEDULED) | `POST /schedule { type: "RETURN", trigger: "DELIVERY_COMPLETE", ... }` |
| `DELIVERY_COMPLETE` (ON-DEMAND) | `POST /schedule { type: "RETURN", trigger: "ON_DEMAND_DELIVERY_COMPLETE", ... }` |
| New ON-DEMAND request in queue | `POST /schedule { type: "ON_DEMAND", ... }` |
| Mission cancellation | `DELETE /missions/{id}` |
| System startup | `GET /health` (availability check) |

The periodic batch planning cycle is run by a `setInterval` in `sos-hub`. The
interval in milliseconds corresponds to the `pad_slot_duration` configured in
`sos-scheduler` (read from `GET /health` at startup or configured externally).

### 3.8 Startup and configuration

`sos-hub` is started with the following environment variables:

| Variable | Default | Description |
|---|---|---|
| `HUB_PORT` | `3000` | HTTP/WebSocket port |
| `SCHEDULER_URL` | `http://127.0.0.1:8000` | sos-scheduler base URL |
| `PLANNING_INTERVAL_MS` | `6000` | Batch planning cycle interval (must match pad_slot_duration × 1000) |
| `LOG_LEVEL` | `info` | Log level (debug / info / warn / error) |
| `STATIC_DIR` | `./dist` | React dashboard static files directory |
| `EVENTS_DB_PATH` | `./events.db` | Event Logger SQLite database path (§3.9) |
| `GRAFANA_URL` | — | Grafana dashboard URL for the Statistics panel (§5.3). If not configured, the panel is hidden. |

---

### 3.9 Event Logger

The Event Logger is a `sos-hub` module responsible for **persistence of raw
telemetries and events** received from hardware or from the simulator. It writes
to a separate SQLite database (`events.db`), distinct from the `sos-scheduler` DB.

**Responsibilities:**
- Records every message received on `POST /api/telemetry/*` and `POST /api/events/*` endpoints
- Exposes `GET /api/logs` for historical query from the dashboard
- Emits `LOG_EVENT` messages via WebSocket to connected clients (§3.6, §6.4)

**`events.db` schema:**

```sql
CREATE TABLE telemetry_events (
  id           INTEGER PRIMARY KEY AUTOINCREMENT,
  event_type   TEXT NOT NULL,    -- DRONE_TELEMETRY | NODE_TELEMETRY |
                                 -- DELIVERY_COMPLETE | ON_DEMAND_REQUEST |
                                 -- NODE_ALERT | BATTERY_SWAP_EXECUTED |
                                 -- BATTERY_RECALLED | BATTERY_FAULTY
  source_id    TEXT NOT NULL,    -- droneId or nodeId
  data_quality TEXT NOT NULL,    -- CONFIRMED | INFERRED | CORRECTED
  payload      TEXT NOT NULL,    -- raw JSON of the original payload
  received_at  INTEGER NOT NULL  -- Unix timestamp in milliseconds
);

CREATE INDEX idx_evlog_type     ON telemetry_events(event_type);
CREATE INDEX idx_evlog_source   ON telemetry_events(source_id);
CREATE INDEX idx_evlog_received ON telemetry_events(received_at);
```

**GET /api/logs**

```typescript
Query params:
  sourceId?    string    // filter by drone or node
  eventType?   string    // filter by event type
  from?        number    // ms timestamp (inclusive)
  to?          number    // ms timestamp (inclusive)
  limit?       number    // max results (default: 100, max: 1000)
  offset?      number    // pagination

Response: {
  total: number;
  items: Array<{
    id:          number;
    eventType:   string;
    sourceId:    string;
    dataQuality: 'CONFIRMED' | 'INFERRED' | 'CORRECTED';
    payload:     object;
    receivedAt:  number;
  }>;
}
```

---

### 3.10 Prometheus Metrics Exporter

`sos-hub` exposes a `/metrics` endpoint in Prometheus text format (OpenMetrics
standard). Metrics are scraped by the deployment's Prometheus server and visualised
through Grafana in the dashboard's Statistics panel (§5.3).

**GET /metrics**

```
# COUNTERS
sos_missions_total{mode, type, status}
    # mode: SCHEDULED | ON_DEMAND
    # type: WATER | MEDICAL | POSTAL | SUPPLY
    # status: COMPLETE | FAILED | CANCELLED
sos_deliveries_total{node_id, mission_type}
sos_battery_swaps_total{node_id}
sos_on_demand_requests_total{status}
    # status: SCHEDULED | INFEASIBLE | CANCELLED

# GAUGES
sos_drones_active                        # drones currently in flight
sos_drones_standby                       # drones at HUB on standby
sos_nodes_online                         # nodes with status ONLINE
sos_nodes_degraded                       # nodes with status DEGRADED
sos_tank_level_ratio{node_id}            # tank level 0.0–1.0
sos_battery_stock_charged{node_id}       # charged batteries in the rack
sos_battery_stock_total{node_id}         # total batteries in the rack

# HISTOGRAMS
sos_flight_duration_slots{mission_type}  # flight duration in slots (outbound + return)
sos_pad_occupancy_slots{node_id}         # Pad occupancy in slots per swap
```

> **Note:** the scheduler in turn exposes `/metrics` with planning metrics
> (queue length, unschedulable count, planning cycle duration). The Prometheus server
> scrapes both endpoints independently.

---

### 3.11 Reset Detection via Heartbeat (placeholder)

> **Status:** placeholder — *tracked in internal firmware roadmap*.
> The section will be completed when the Node firmware emits `TEL_HEARTBEAT` (M-09)
> with the `uptime_s` field.

**Context.** The Node firmware does not persist the PadManager state to disk
(see *internal node firmware specification* — Boot Reconciliation). On restart,
it begins with all Pads in FREE state and relies on two external sources to
reconstruct state: GPIO (for physical drone presence on the Pad) and the HUB
(for prep messages still pending).

**HUB responsibility.** The HUB must detect a Node reset and re-emit the M-02
prep messages that the scheduler had planned for that Node but that have not
yet been consumed (drone not yet landed).

**Mechanism.** The HUB monitors the `uptime_s` field in `TEL_HEARTBEAT` (M-09)
payloads. When it receives from a Node a heartbeat with `uptime_s` significantly
lower than the previous one (e.g., < 60 s after the Node had been active for
hours), it concludes the Node has restarted. In that case:

1. It queries the scheduler for `pad_reservations` with `status = 'PLANNED'` or
   `'DISPATCHED'` destined for that Node and not yet confirmed by an
   `EVT_DRONE_ARRIVED`
2. It re-emits `CTL_NODE_PREP` (M-02) for each of them, with the same original
   `hop_instruction`
3. The Node's M-02 handler is idempotent on `mission_id` — see *internal node
   firmware specification* — so the re-send does not generate anomalies even if
   for some prep the Node had already received the original before the crash

**Not a Node requirement.** The Node does not change its behaviour: it emits
periodic heartbeats per spec, with `uptime_s` computed from process start.
The entire detection + re-push logic lives in the HUB.

**Recovery window coverage.** Between Node crash and the HUB receiving the first
post-reboot heartbeat, up to ~90 s may elapse (heartbeat interval). In this
window, a drone landing without an active prep triggers its own
`CTL_HOP_REQUEST` (M-20) fallback toward the HUB (see *internal companion
firmware specification*), which resolves independently of the reset detection
mechanism.

---

## 4. sos-simulator — Design

### 4.1 Transparency principle

The simulator impersonates hardware. It calls `sos-hub` via HTTP REST exactly as
physical hardware would: same endpoints, same payloads. `sos-hub` contains no
conditional logic that distinguishes the two sources.

When physical hardware becomes available, the simulator is simply switched off.
No modifications to `sos-hub`.

### 4.2 Components

```
sos-simulator/
├── sim_clock.py        SimClock — virtual clock with acceleration factor
├── sim_loop.py         SimLoop — main slot-by-slot loop
├── drone_agent.py      DroneAgent — simulates a drone (position, battery, payload)
├── node_agent.py       NodeAgent — simulates a node (sensors, battery rack)
├── scenario_engine.py  ScenarioEngine — loads scenario, manages programmed events
├── hub_client.py       HubClient — HTTP client toward sos-hub
├── config_loader.py    Loads and validates sim-config.yaml
└── cli.py              CLI interface (start / status / inject)
```

The simulator is a Python package in the existing uv workspace, alongside
`sos-scheduler` and `sos-messages`.

### 4.3 SimClock and time compression

Time compression is achieved by configuring `sos-scheduler` with a reduced value
of `pad_slot_duration`. For example:

| pad_slot_duration real | pad_slot_duration simulation | Acceleration factor |
|---|---|---|
| 360 s (6 min) | 6 s | 60× |
| 360 s (6 min) | 3 s | 120× |
| 360 s (6 min) | 1 s | 360× |

`SimClock` tracks **simulated time** (number of slots elapsed × configured slot
duration) and **real time** of startup. It exposes:

```python
class SimClock:
    slot_duration_s: float     # simulated duration of a slot (e.g., 6.0 s)
    slot_now: int              # current slot
    sim_time_s: float          # simulated time elapsed in seconds
    real_elapsed_s: float      # real time elapsed
    speedup: float             # effective acceleration factor
```

`SimLoop` advances one slot at a time, waiting `slot_duration_s` real seconds
between iterations:

```python
async def run(self):
    while not self.stopped:
        await self._tick()                # process all agents for current slot
        await asyncio.sleep(self.clock.slot_duration_s)
        self.clock.advance()
```

### 4.4 DroneAgent

One `DroneAgent` per simulated drone. Responsibilities:

- **GPS interpolation**: computes drone position between source and destination
  nodes proportionally to time progress within the segment.
- **Battery consumption**: decrements `batteryPct` proportionally to flight time.
- **Delivery simulation**: when the `slot_departure` of a hop is reached, it sends
  `POST /api/telemetry/drone` with the updated position. When the drone arrives
  at destination, it emits `POST /api/events/delivery-complete`.
- **Battery swap**: simulates the swap with a 1-slot pause on the node Pad,
  restoring `batteryPct` to 100%.
- **Periodic telemetry**: sends `POST /api/telemetry/drone` every slot (or every
  N configurable slots) during flight.

Drone state is driven by the `hop_instructions` the scheduler has written in the
DB. The `DroneAgent` does not know the complete Mission Plan: it reads instructions
directly from the `sos-scheduler` SQLite DB (read-only access to the SQLite DB),
exactly as the drone firmware would in the hop-by-hop model.

> **Implementation note:** direct access to the SQLite DB by the simulator is
> acceptable in development (they are on the same host). In production, the drone
> has no access to the DB: it receives instructions via RF from the node.

#### Event-driven drone telemetry

In addition to periodic telemetry, the DroneAgent emits additional transmissions
on specific events, replicating the expected behaviour of the real firmware:

| Event | Trigger | Key fields transmitted |
|---|---|---|
| **DEPARTURE** | Start of a new hop (`_start_hop`) | `status=IN_FLIGHT`, `flightDirection`, current position |
| **PRE_ARRIVAL** | GPS distance from destination < 750 m | `status=IN_FLIGHT`, updated position |
| **WATER_DISCHARGE** | Start of DROP action (water delivery) | `status=ON_PAD`, `onPadActivity=WATER_DISCHARGE` |

The PRE_ARRIVAL threshold of 750 m corresponds to about 1.3 minutes of advance
notice at the cruise speed of 35 km/h. In the real firmware, the trigger will be
based on GPS distance computed via the Haversine formula; the simulator uses the
same criterion.

### 4.5 NodeAgent

One `NodeAgent` per topology node. Responsibilities:

- **Tank level**: updates `tankLevelRatio` accounting for simulated consumption
  (configurable constant decrement per node) and water releases recorded in
  `hop_instructions`.
- **Battery stock**: updates the number of charged batteries based on simulated
  swaps and configured charging time.
- **Pad state**: marks the Pad as occupied when a drone is expected (slot_start of
  the `pad_reservation`) and free after the swap.
- **Periodic telemetry**: sends `POST /api/telemetry/node` every N slots, with the
  current values of tank level, battery stock and Pad state.
- **Hop instruction distribution**: at the `slot_departure` of an outbound hop,
  marks the instruction as `distributed` in the DB (simulation of the RF
  transmission to the drone).

#### Event-driven node telemetry

In addition to periodic telemetry (every 3 slots), the NodeAgent emits immediate
transmissions on relevant state changes:

| Event | Trigger | Key fields transmitted |
|---|---|---|
| **PAD_OCCUPIED** | Pad transitions from free to occupied (drone detected) | updated `padStatus` |
| **PAD_FREED** | Pad transitions from occupied to free (drone departed) | updated `padStatus` |
| **BATTERY_SWAPPED** | Battery swap completion (charged battery available) | updated `batterySlots` |
| **WATER_RECEIVED** | Water reception from DROP hop | updated `tankLevelRatio` |

The NodeAgent autonomously detects drone arrival by comparing the Pad state of
the current tick with the previous one. When a Pad transitions from free to
occupied, the node autonomously starts the battery swap sequence (via
`simulate_swap()`), replicating the firmware behaviour: the node receives no
explicit notification from the drone, it acts on the physical presence detected
by the Pad sensor.

### 4.6 ScenarioEngine and CLI

`ScenarioEngine` loads the `sim-config.yaml` file and manages the scenario's
programmed events:

- **Programmed ON-DEMAND missions**: at a specified slot, sends
  `POST /api/events/on-demand-request` with the configured coordinates.
- **Fault injection**: at a specified slot, can inject anomalous events:
  - `PAD_FAULT`: one of a node's Pads is marked as faulty (NodeAgent stops
    freeing that Pad).
  - `DRONE_LOST`: a DroneAgent stops sending telemetry and does not complete the
    flight (simulating loss of communication / crash).
  - `NODE_OFFLINE`: a NodeAgent stops sending telemetry.

**CLI** (`python -m sos_simulator`):

The `seed` command must be run once before starting the simulation, to load
topology and fleet into the scheduler DB from the 4 deployment YAML files.
Operational files reside in the `deployments/` directory (gitignored).

```
sos-simulator seed   --topology   deployments/topology.yaml        \
                     --drones     deployments/drone-devices.yaml    \
                     --batteries  deployments/battery-devices.yaml  \
                     --state      deployments/deployment-state.yaml   Initialise scheduler DB from deployment
sos-simulator start  --config sim-config.yaml             Start the simulation
sos-simulator status                                      Show current slot, active drones, alerts
sos-simulator inject pad-fault  --node N1.2 --pad 0 --slot 42
sos-simulator inject drone-lost --drone DRN-003 --slot 55
sos-simulator inject on-demand  --lat 12.3 --lon -8.1 --type MEDICAL --slot 30
sos-simulator stop                                        Stop the simulation
```

### 4.7 Scenario configuration format (superseded)

> **Note:** this section has been superseded by [simulator-architecture-spec](./simulator-architecture-spec.md) §7.3
> (version 1.1, April 2026), which describes the current format of the
> `sim-config.yaml` file and the updated simulator architecture.
>
> The previous format documented here (with fields `slot_duration_s`,
> `total_slots`, `telemetry_interval_slots` and a `scheduled_missions` section)
> is obsolete and does not correspond to the file present in the repository.

---

## 5. sos-dashboard — Design

### 5.1 Technology stack

| Layer | Technology | Version |
|---|---|---|
| UI Framework | React | 18 |
| Build tool | Vite | 5 |
| Language | TypeScript | 5 |
| Styling | Tailwind CSS | 3 |
| Map | Leaflet + react-leaflet | 1.9 / 2.x |
| WebSocket client | Native browser (WebSocket API) | — |
| HTTP client | Native Fetch API | — |
| State management | React Context + useReducer | — |

The dashboard is built with `vite build` and the `dist/` folder is served as
static files by `sos-hub` (Express `static` middleware). A single Node.js process
serves both the REST API and the dashboard.

> Project brand palette and typography are defined in internal brand guidelines
> (available on request).

### 5.2 Frontend architecture

```
src/
├── main.tsx              React entrypoint
├── App.tsx               Main layout, WebSocket management
├── context/
│   └── SystemContext.tsx  Global context — SystemState + dispatch
├── hooks/
│   ├── useWebSocket.ts   Connection and automatic reconnection
│   └── useSystemState.ts State access with selectors
├── components/
│   ├── NetworkMap/       Leaflet map (nodes, drones, segments)
│   ├── FleetStatus/      Fleet status (drone table)
│   ├── NodeGrid/         Node grid (sensors, Pads, batteries)
│   ├── MissionList/      Active mission list
│   ├── MissionTimeline/  Slot-Pad timeline (future)
│   ├── AlertPanel/       Active and historical alerts
│   └── OnDemandForm/     ON-DEMAND mission planning form
└── types/
    └── system.ts         TypeScript types mirroring §6 interfaces
```

### 5.3 Panels

#### NetworkMap

The map occupies the main portion of the screen. It shows:
- **Nodes**: markers with colour and icon based on status (ONLINE / DEGRADED /
  OFFLINE). Click on the marker opens a popup with node details.
- **Segments**: lines connecting each node to its parent, with thickness
  proportional to simulated traffic frequency.
- **Drones**: animated markers moving along segments based on the GPS position
  received via WebSocket. Colour based on status
  (IN_FLIGHT / ON_PAD / ELZ_WAIT / STANDBY).
- **ON-DEMAND coordinates**: temporary pins for GPS destinations of active
  ON-DEMAND missions.

#### FleetStatus

Table with one row per drone:
`Drone ID | Mission | Status | Battery % | Payload | Last update`

#### NodeGrid

Grid of cards, one per node. Each card displays:
- Node name and ID
- Tank level (progress bar)
- Battery stock (N/M batteries charged)
- Pad-1 and Pad-2 status (free / occupied — with drone ID if occupied)
- Status indicator (ONLINE / DEGRADED / OFFLINE)

#### MissionList

List of active missions with status, assigned drone, destination node,
launch slot. Link to mission detail (call `GET /api/missions/:id`).

#### AlertPanel

List of active alerts and the last N resolved alerts, with severity,
source, message and timestamp.

#### OnDemandForm

Operator form:
- Mission type (MEDICAL / POSTAL / SUPPLY)
- Destination GPS coordinates (lat/lon input or click on map)
- Payload weight (kg)
- Priority (1–5)
- Free-text notes

Submission via `POST /api/missions/on-demand`.

#### LogsPanel

Panel that displays raw events recorded by the Event Logger (§3.9). Queries
`GET /api/logs` with filters and updates in real time via `LOG_EVENT` messages.

Available filters:
- **Source** — dropdown: all drones and nodes in the network
- **Event type** — multiselect (DRONE_TELEMETRY, NODE_TELEMETRY, DELIVERY_COMPLETE, ...)
- **Time window** — last N minutes / hours
- **Data quality** — CONFIRMED / INFERRED / CORRECTED

Each row shows: timestamp, source_id, event_type, data_quality, payload summary.

#### Statistics

Grafana panel embedded in the dashboard via `<iframe>`. Visualises aggregated
metrics exposed by `sos-hub` (§3.10) and `sos-scheduler` via Prometheus/Grafana.

```
Grafana URL: GRAFANA_URL environment variable (configured in sos-hub)
```

The panel is read-only: the operator does not interact with Grafana directly from
the SOS dashboard. The Grafana URL points to a deployment-specific pre-configured
dashboard.

### 5.4 Data flow

```
1. Dashboard opens in browser
   └─▶ GET /api/state → loads initial SystemState into Context

2. WebSocket connection to /ws/live
   └─▶ FULL_STATE reception → overwrites SystemState (reconciliation)

3. Incoming WebSocket message
   └─▶ dispatch(action) → updates SystemState via reducer
   └─▶ Selective React re-render of affected components

4. Operator action (e.g., ON-DEMAND)
   └─▶ POST /api/missions/on-demand
   └─▶ sos-hub processes and sends MISSION_UPDATE via WebSocket
   └─▶ UI updates without explicit polling

5. WebSocket disconnection
   └─▶ useWebSocket reconnects with exponential back-off
   └─▶ on reconnection, GET /api/state to reconcile state
```

---

## 6. Interface payloads

### 6.1 Drone telemetry

**POST /api/telemetry/drone**

```typescript
{
  droneId:          string;
  missionId:        string | null;
  lat:              number;
  lon:              number;
  altitudeM:        number;
  batteryPct:       number;             // 0–100
  speedKmh:         number;
  waterRemainingL:  number | null;      // null if not WATER mission
  payloadKg:        number | null;
  status:           'IN_FLIGHT' | 'ON_PAD' | 'ELZ_WAIT' | 'STANDBY';
  flightDirection:  'OUTBOUND' | 'RETURN' | null;
  onPadActivity:    'BATTERY_SWAP' | 'WATER_DISCHARGE' | 'CARGO_DROP' | 'FAULT' | null;
  hopSequence:      number | null;      // current hop in the mission
  nextHopNodeId:    string | null;      // destination node of the current hop
                                        // (hop-by-hop model: the drone knows only
                                        // the next node, not the final destination)
  dataQuality:      'CONFIRMED' | 'INFERRED' | 'CORRECTED';
  timestampMs:      number;             // Unix timestamp in milliseconds
}
```

### 6.2 Node sensors

**POST /api/telemetry/node**

```typescript
{
  nodeId:            string;
  tankLevelRatio:    number | null;     // 0.0–1.0; null for tankless nodes
  batterySlots: {
    total:           number;
    charged:         number;
    charging:        number;
    depleted:        number;
    slots: Array<{                      // detail per single rack slot
      slotId:        string;           // format: "RACK-{nodeId}-{NN}", e.g., "RACK-N1-01"
      batteryId:     string | null;    // null if slot is empty
      status:        'CHARGED' | 'CHARGING' | 'DEPLETED' | 'EMPTY';
    }>;
  };
  infraBattery: {
    socPct:          number;           // state of charge 0–100
    cycleCount:      number;
  } | null;                            // null if data not available
  padStatus: Array<{
    padIndex:        number;
    occupied:        boolean;
    droneId:         string | null;
  }>;
  nodeStatus:        'ONLINE' | 'DEGRADED' | 'OFFLINE';
  dataQuality:       'CONFIRMED' | 'INFERRED' | 'CORRECTED';
  timestampMs:       number;
}
```

### 6.3 Events

#### 6.3.1 DELIVERY_COMPLETE

**POST /api/events/delivery-complete**

```typescript
{
  eventType:   'DELIVERY_COMPLETE' | 'ON_DEMAND_DELIVERY_COMPLETE';
  missionId:   string;
  droneId:     string;
  nodeId:      string | null;          // null for ON-DEMAND (GPS delivery)
  lat:         number | null;          // ON-DEMAND delivery coordinates
  lon:         number | null;
  slotNow:     number;                 // current slot (from SimClock or real clock)
  timestampMs: number;
}
```

#### 6.3.2 ON_DEMAND_REQUEST

**POST /api/events/on-demand-request**

```typescript
{
  eventType:   'ON_DEMAND_REQUEST';
  requestId:   string;                 // generated by the sender (FT or dashboard)
  missionType: 'MEDICAL' | 'POSTAL' | 'SUPPLY';
  targetLat:   number;
  targetLon:   number;
  payloadKg:   number;
  priority:    number;                 // 1 = highest priority
  notes:       string;
  timestampMs: number;
}
```

#### 6.3.3 NODE_ALERT

**POST /api/events/node-alert**

```typescript
{
  eventType:   'NODE_ALERT';
  nodeId:      string;
  alertType:   'PAD_FAULT' | 'TANK_SENSOR_FAULT' | 'BATTERY_LOW' |
               'COMM_DEGRADED' | 'NODE_OFFLINE';
  severity:    'WARNING' | 'CRITICAL';
  detail:      string;
  padIndex:    number | null;          // PAD_FAULT only
  timestampMs: number;
}
```

### 6.4 WebSocket messages

#### FULL_STATE

Sent on client connection. Contains the entire SystemState.

```typescript
{
  type:      'FULL_STATE';
  timestamp: number;
  payload:   SystemState;   // see §6.5
}
```

#### DRONE_UPDATE

```typescript
{
  type:      'DRONE_UPDATE';
  timestamp: number;
  payload:   DroneState;    // see §3.2
}
```

#### NODE_UPDATE

```typescript
{
  type:      'NODE_UPDATE';
  timestamp: number;
  payload:   NodeState;     // see §3.2
}
```

#### MISSION_UPDATE

```typescript
{
  type:      'MISSION_UPDATE';
  timestamp: number;
  payload:   MissionSummary; // see §3.2
}
```

#### ALERT

```typescript
{
  type:      'ALERT';
  timestamp: number;
  payload:   Alert;          // see §3.2
}
```

#### CLOCK_TICK

Sent every slot. Allows the dashboard to display the simulated time.

```typescript
{
  type:      'CLOCK_TICK';
  timestamp: number;
  payload: {
    slotNow:    number;
    simTimeS:   number;      // simulated seconds elapsed since epoch
    speedup:    number;      // current acceleration factor
  };
}
```

#### LOG_EVENT

Emitted by the Event Logger (§3.9) on every significant recorded event.
Allows the Logs panel (§5.3) to update in real time.

```typescript
{
  type:      'LOG_EVENT';
  timestamp: number;
  payload: {
    eventType:   string;     // event type (see §3.9 events.db schema)
    sourceId:    string;     // droneId or nodeId
    dataQuality: 'CONFIRMED' | 'INFERRED' | 'CORRECTED';
    summary:     string;     // short string for the row in the Logs panel
    payloadJson: object;     // raw event payload
  };
}
```

### 6.5 State snapshot

**GET /api/state** — complete response

```typescript
interface SystemState {
  drones:    Record<string, DroneState>;
  nodes:     Record<string, NodeState>;
  missions:  MissionSummary[];
  alerts:    Alert[];
  clock: {
    slotNow:  number;
    simTimeS: number;
    speedup:  number;
  };
}
```

---

## 7. Revision log

| Version | Date | Author | Notes |
|---|---|---|---|
| 1.0 | April 2026 | Matteo Casavecchia | First version. Architecture of sos-hub, sos-simulator, sos-dashboard. Interface payloads. Reference scenario. |
| 1.1 | April 2026 | Matteo Casavecchia | Added flightDirection and onPadActivity to DroneState and DroneTelemetryPayload |
| 1.2 | April 2026 | Matteo Casavecchia | Added event-driven drone telemetry (DEPARTURE, PRE_ARRIVAL 750 m, WATER_DISCHARGE) and node (PAD_OCCUPIED/FREED, BATTERY_SWAPPED, WATER_RECEIVED). Node battery swap is now autonomous on Pad detection. |
| 1.3 | April 2026 | Matteo Casavecchia | Fleet management API (POST/DELETE drones, batteries, topology/nodes) in §3.5. LOG_EVENT WebSocket in §3.6 and §6.4. New sections §3.9 Event Logger (events.db, GET /api/logs) and §3.10 Prometheus Metrics Exporter (/metrics). `seed` command in §4.6 (CLI). New scenario-only sim-config.yaml format in §4.7. Logs and Statistics panels in §5.3. data_quality field on DroneTelemetryPayload (§6.1) and NodeTelemetryPayload (§6.2). battery_id per rack slot, infra_battery on NodeTelemetryPayload (§6.2). |
