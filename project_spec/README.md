# GeoFleetic — Geo-distributed Fleet Intelligence Collaboration
## Technical Specification v0.1

**Date:** March 25, 2026
**Status:** Draft
**Author:** [&] Ampersand Box Design
**License:** MIT (open core)
**Stack:** Elixir · OTP · Phoenix · Ecto · PostgreSQL + PostGIS · Tile38

---

## 1. Overview

GeoFleetic is the **spatial intelligence layer** of the [&] Protocol ecosystem. It provides geo-distributed fleet tracking, federated route learning, and GNN-based route optimization — all running on the BEAM for fault tolerance and real-time concurrency. GeoFleetic is the `&space` primitive provider, exposing `&space.fleet`, `&space.route`, and `&space.geofence` capability contracts to the rest of the [&] stack.

Every vehicle, drone, or mobile asset maintains a **spatial digital twin** — a local replica synchronized via epoch-aware delta-CRDTs. Fleet learning happens federatively: each node trains on its own route data and shares only LoRA model deltas, never raw GPS traces. Over time, the fleet collectively learns spatial-temporal patterns that improve routing, capacity planning, and geofence accuracy — without centralizing sensitive location data.

### 1.1 The Problem

Fleet management systems today are centralized, batch-oriented, and privacy-hostile. A logistics company with 10,000 vehicles streams all GPS data to a cloud API, which returns routes computed on stale models that know nothing about the driver's local context. The fleet learns nothing from yesterday's deliveries. Privacy regulations (GDPR, CCPA) make centralizing raw location data increasingly untenable. And when the cloud API goes down, the fleet goes blind.

The industry needs spatial intelligence that:
- **Lives at the edge** — each vehicle reasons locally with sub-second latency
- **Learns continually** — routes improve from every delivery, not quarterly retraining
- **Preserves privacy** — shares model updates, never raw trajectories
- **Tolerates partitions** — vehicles operate offline and reconcile when reconnected
- **Composes with other intelligence** — spatial context enriches upstream reasoning

### 1.2 Design Principles

1. **Spatial digital twins** — Every tracked asset is a CRDT-backed twin. State converges eventually, even under partition.
2. **Federated learning, not centralized surveillance** — Share LoRA deltas. Raw data stays on-device.
3. **Edge-first** — SQLite + sqlite-vec on each vehicle. PostGIS at the fleet coordinator level.
4. **Continual, not batch** — Models adapt after every trip, not on a quarterly retrain cycle.
5. **Composable primitives** — `&space.fleet`, `&space.route`, `&space.geofence` are standalone contracts that compose with `&time` (TickTickClock), `&knowledge` (Graphonomous), and `&govern` (Delegatic).
6. **MCP-first API** — All capabilities exposed as MCP tools. Any MCP client can consume spatial intelligence directly.
7. **Explain, don't just optimize** — Every route decision can be explained via `route_explain`.

### 1.3 Why Elixir

Spatial fleet intelligence is a concurrent, distributed, real-time problem — exactly what the BEAM was built for. Each spatial digital twin is a GenServer process. Geofence boundary checks run as concurrent tasks across thousands of assets without thread pools. Delta-CRDT state synchronization maps naturally to distributed Erlang message passing. Phoenix PubSub provides real-time geofence event streaming with zero additional infrastructure. And OTP supervision trees mean a crashing twin process restarts in microseconds without affecting the rest of the fleet.

### 1.4 One-Liner

> "The fleet that learns where. Spatial digital twins, federated route learning, and GNN optimization for geo-distributed intelligence."

---

## 2. Architecture

```
┌──────────────────────────────────────────────────────────────────┐
│                          GEOFLEETIC                              │
│             Spatial Intelligence Layer (Elixir/OTP)              │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│   MCP Server (Hermes)            Phoenix LiveView (optional)     │
│   ├── tools/fleet_*              └── Fleet map dashboard         │
│   ├── tools/route_*              └── Route visualization         │
│   ├── tools/geofence_*           └── Geofence editor             │
│   ├── tools/spatial_*            └── Capacity heatmaps           │
│   ├── resources://fleet/*                                        │
│   └── resources://spatial/*                                      │
│                                                                  │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│   ┌──────────────┐  ┌──────────────┐  ┌──────────────────────┐  │
│   │  Fleet        │  │  Route       │  │  Geofence            │  │
│   │  Tracker      │  │  Optimizer   │  │  Engine              │  │
│   │               │  │              │  │                      │  │
│   │  Delta-CRDT   │  │  GNN-based   │  │  Tile38-inspired     │  │
│   │  spatial      │  │  continual   │  │  boundary eval       │  │
│   │  digital      │  │  route       │  │  with CL-adapted     │  │
│   │  twins        │  │  learning    │  │  thresholds          │  │
│   └──────┬───────┘  └──────┬───────┘  └──────┬───────────────┘  │
│          │                 │                  │                  │
│   ┌──────▼─────────────────▼──────────────────▼───────────────┐  │
│   │              Federated Learning Coordinator                │  │
│   │                                                            │  │
│   │  LoRA delta aggregation · Privacy budget enforcement       │  │
│   │  Model version management · Gradient compression           │  │
│   └────────────────────────┬───────────────────────────────────┘  │
│                            │                                     │
│   ┌────────────────────────▼───────────────────────────────────┐  │
│   │                  Spatial Index Layer                        │  │
│   │                                                            │  │
│   │  PostGIS (fleet coord) · Tile38 (geofencing)               │  │
│   │  sqlite-vec (edge)     · ETS (hot twin cache)              │  │
│   └────────────────────────────────────────────────────────────┘  │
│                                                                  │
├──────────────────────────────────────────────────────────────────┤
│   Storage Layer                                                  │
│   ├── PostgreSQL 16+ / PostGIS 3.4   (fleet coordinator)         │
│   ├── Tile38                          (geofence eval engine)     │
│   ├── SQLite + sqlite-vec             (edge/vehicle local)       │
│   └── ETS/DETS                        (hot twin state cache)     │
└──────────────────────────────────────────────────────────────────┘
```

### 2.1 Component Summary

| Component | Responsibility | OTP Pattern |
|-----------|---------------|-------------|
| `GeoFleetic.Fleet.Tracker` | Spatial digital twin lifecycle: create, update, locate, destroy | DynamicSupervisor + per-twin GenServer |
| `GeoFleetic.Fleet.Twin` | Individual asset twin state, delta-CRDT merge, local inference | GenServer (one per tracked asset) |
| `GeoFleetic.Fleet.CRDTSync` | Epoch-aware delta-CRDT synchronization across nodes | GenServer + distributed Erlang |
| `GeoFleetic.Route.Optimizer` | GNN-based route generation and optimization | GenServer + Task.async_stream |
| `GeoFleetic.Route.Explainer` | Produce human-readable explanations for route decisions | Stateless module |
| `GeoFleetic.Geofence.Engine` | Boundary membership, enter/exit event detection | GenServer + Tile38 client |
| `GeoFleetic.Geofence.EventStream` | Pub/Sub for geofence transition events | Phoenix.PubSub |
| `GeoFleetic.Learning.Coordinator` | Federated learning round orchestration, LoRA delta aggregation | GenServer |
| `GeoFleetic.Learning.LocalTrainer` | On-device model training on local route data | GenServer + Nx |
| `GeoFleetic.Spatial.Enricher` | Attach spatial context to upstream [&] artifacts | Stateless module |
| `GeoFleetic.MCP.Server` | MCP tool/resource exposure via Hermes | Hermes.Server |

---

## 3. Technology Stack

### 3.1 Core

| Layer | Technology | Rationale |
|-------|-----------|-----------|
| **Language** | Elixir 1.17+ / OTP 27 | Fault-tolerant, concurrent, distributed-native. Each twin is a process. Fleet-scale concurrency without thread pools. |
| **MCP Server** | `hermes_mcp` (v0.8+) | Mature Elixir MCP SDK. STDIO + streamable HTTP. JSON-RPC 2.0 compliant. |
| **Spatial DB (coordinator)** | PostgreSQL 16+ / PostGIS 3.4 via `ecto` + `postgrex` | Full ACID spatial queries. `ST_DWithin`, `ST_Contains`, `ST_Distance` for fleet-level analytics. |
| **Geofencing** | Tile38 via `tile38_ex` | In-memory geospatial database optimized for real-time geofencing. NEARBY, WITHIN, INTERSECTS queries at microsecond latency. |
| **Spatial DB (edge)** | SQLite + sqlite-vec via `exqlite` | Zero-config on-vehicle storage. sqlite-vec for local embedding similarity. Single-file, portable. |
| **Embeddings** | `bumblebee` + ONNX | Local spatial feature embeddings. Route fingerprinting for similarity search. |
| **Continual Learning** | `nx` + LoRA adapters | Gradient-free adaptation via low-rank adapter updates. Shared as compressed deltas. |
| **Delta-CRDTs** | Custom Elixir impl (GeoCoCo 2025) | Epoch-aware delta-CRDTs for spatial twin synchronization. Convergent under arbitrary partition. |
| **Hot Cache** | ETS | In-memory twin state for sub-millisecond position lookups. Configurable TTL. |
| **Admin UI** | Phoenix LiveView (optional) | Real-time fleet map, route visualization, geofence editor, capacity heatmaps. |
| **Telemetry** | `:telemetry` + `telemetry_metrics` | All spatial operations emit telemetry. Observable by default. |

### 3.2 Why PostGIS + Tile38 (Not Just One)?

PostGIS and Tile38 serve complementary roles:

- **PostGIS** handles durable spatial analytics: historical route queries, fleet-wide capacity aggregations, spatial joins across regions. It is the coordinator-level source of truth.
- **Tile38** handles real-time geofencing: point-in-polygon checks, NEARBY searches, and enter/exit event hooks at microsecond latency. It is an in-memory engine that Tile38 was purpose-built for.

Running both avoids forcing PostGIS into a real-time geofencing role it was not optimized for, and avoids forcing Tile38 into durable analytics it does not support.

### 3.3 Why Federated Learning (Not Centralized)?

Centralizing raw GPS traces from a fleet creates:
- **Privacy liability** — GDPR Article 9 treats precise location as sensitive data
- **Bandwidth cost** — streaming high-frequency GPS from 10K+ vehicles is expensive
- **Stale models** — centralized retraining happens on a cycle; the fleet learns nothing between cycles
- **Single point of failure** — cloud outage means no route optimization

Federated learning inverts this: each vehicle trains locally, shares only LoRA adapter deltas (kilobytes, not megabytes), and the coordinator aggregates deltas into a global model update. Privacy is preserved by construction. Bandwidth drops by orders of magnitude. Each vehicle learns immediately from its own routes. And the system degrades gracefully — a disconnected vehicle still has its local model.

---

## 4. Spatial Digital Twins

### 4.1 Twin State (Delta-CRDT)

Each tracked asset maintains a spatial digital twin as an epoch-aware delta-CRDT. The CRDT ensures convergence under concurrent updates and network partitions.

```elixir
defmodule GeoFleetic.Fleet.Twin do
  @type t :: %__MODULE__{
    asset_id: binary(),              # Unique asset identifier
    position: {float(), float()},    # {longitude, latitude}
    altitude: float() | nil,         # Meters above sea level
    heading: float(),                # Degrees (0-360)
    speed: float(),                  # m/s
    accuracy: float(),               # Position accuracy in meters
    status: atom(),                  # :active | :idle | :offline | :maintenance
    metadata: map(),                 # Vehicle type, capacity, fuel level, etc.

    # CRDT fields
    epoch: non_neg_integer(),        # Monotonic epoch counter
    vector_clock: map(),             # {node_id => counter} for causal ordering
    delta_buffer: [delta()],         # Unsent deltas awaiting sync
    last_sync: DateTime.t(),         # Last successful sync timestamp

    # Learning state
    local_model_version: binary(),   # Current LoRA adapter version hash
    route_history_size: non_neg_integer(),
    last_training_epoch: non_neg_integer(),

    # Lifecycle
    created_at: DateTime.t(),
    updated_at: DateTime.t()
  }

  @type delta :: %{
    field: atom(),
    value: term(),
    epoch: non_neg_integer(),
    node_id: binary(),
    timestamp: DateTime.t()
  }
end
```

### 4.2 Delta-CRDT Synchronization (GeoCoCo 2025)

GeoFleetic implements epoch-aware delta-CRDTs based on the GeoCoCo 2025 framework. Key properties:

1. **Epoch monotonicity** — Each update increments the twin's epoch counter. Merges always advance to the maximum observed epoch.
2. **Delta propagation** — Only state deltas (not full twin snapshots) are transmitted. A delta is a `{field, value, epoch, node_id}` tuple.
3. **Causal ordering** — Vector clocks disambiguate concurrent updates to the same field. Last-writer-wins (LWW) within an epoch; epoch ordering across epochs.
4. **Garbage collection** — Deltas older than the minimum acknowledged epoch across all peers are discarded.

```
Vehicle A                    Coordinator                   Vehicle B
    │                             │                             │
    │── position update ──►       │                             │
    │   δ = {pos, {-73.9, 40.7}, │                             │
    │        epoch=42, node=A}    │                             │
    │                             │── broadcast delta ─────────►│
    │                             │                             │── merge
    │                             │                             │   epoch=max(local, 42)
    │                             │◄── ack(epoch=42) ──────────│
    │◄── ack(epoch=42) ──────────│                             │
    │                             │                             │
    │   [GC deltas ≤ 42]         │   [GC deltas ≤ 42]         │
```

### 4.3 Twin Process Lifecycle

Each twin is a GenServer under a DynamicSupervisor. Twins are started on first observation and hibernated after a configurable idle period. Hibernated twins are swapped to ETS and restarted on next access.

```elixir
defmodule GeoFleetic.Fleet.Tracker do
  use DynamicSupervisor

  def start_twin(asset_id, initial_state) do
    DynamicSupervisor.start_child(__MODULE__, {
      GeoFleetic.Fleet.Twin, {asset_id, initial_state}
    })
  end

  def locate(asset_query) do
    # Resolve query to asset_ids, gather positions from twin processes
    asset_query
    |> resolve_assets()
    |> Task.async_stream(&Twin.get_position/1, max_concurrency: 500)
    |> Enum.into([])
  end
end
```

---

## 5. Federated Fleet Learning

### 5.1 Learning Architecture

```
┌─────────────────────────────────────────────────────────┐
│                   Fleet Coordinator                      │
│                                                          │
│   ┌──────────────────────────────────────────────────┐  │
│   │         LoRA Delta Aggregator                     │  │
│   │                                                    │  │
│   │   Collect vehicle deltas → FedAvg aggregate →     │  │
│   │   Produce global update  → Distribute to fleet    │  │
│   └──────────────────────────┬───────────────────────┘  │
│                              │                          │
└──────────────────────────────┼──────────────────────────┘
                               │
          ┌────────────────────┼────────────────────┐
          │                    │                    │
   ┌──────▼──────┐     ┌──────▼──────┐     ┌──────▼──────┐
   │  Vehicle A   │     │  Vehicle B   │     │  Vehicle C   │
   │              │     │              │     │              │
   │  Local route │     │  Local route │     │  Local route │
   │  data        │     │  data        │     │  data        │
   │      │       │     │      │       │     │      │       │
   │      ▼       │     │      ▼       │     │      ▼       │
   │  Train LoRA  │     │  Train LoRA  │     │  Train LoRA  │
   │  adapter     │     │  adapter     │     │  adapter     │
   │      │       │     │      │       │     │      │       │
   │      ▼       │     │      ▼       │     │      ▼       │
   │  Send Δ only │     │  Send Δ only │     │  Send Δ only │
   │  (not data)  │     │  (not data)  │     │  (not data)  │
   └──────────────┘     └──────────────┘     └──────────────┘
```

### 5.2 LoRA Adapter Deltas

Each vehicle maintains a small LoRA adapter (rank 4–16) on top of the shared route model. After each completed route, the local trainer:

1. Computes a LoRA update from the route's actual vs predicted performance
2. Compresses the delta via gradient quantization (INT8)
3. Buffers the delta for the next sync window

The coordinator aggregates deltas using **FedAvg** (weighted by route count) and produces a global model update that is distributed back to the fleet. Vehicles apply the global update as a base and continue local adaptation on top.

**Privacy guarantees:**
- Raw GPS traces never leave the vehicle
- LoRA deltas are low-rank matrix updates — they do not encode individual positions
- Optional differential privacy noise (configurable epsilon) can be applied before transmission

### 5.3 Continual Learning Cycle

```
Route Complete
    │
    ▼
┌─────────────────────┐
│ Feature Extraction   │──── Extract spatial-temporal features
│                     │      from route: segments, durations,
│                     │      delays, detours, conditions
└────────┬────────────┘
         │
         ▼
┌─────────────────────┐
│ Local LoRA Training  │──── Update adapter weights using
│                     │      actual vs predicted route metrics
└────────┬────────────┘
         │
         ▼
┌─────────────────────┐
│ Delta Compression    │──── INT8 quantize, buffer for sync
└────────┬────────────┘
         │
         ▼  (on sync window)
┌─────────────────────┐
│ Coordinator Agg.     │──── FedAvg across fleet deltas
│                     │      Produce global model v(n+1)
└────────┬────────────┘
         │
         ▼
┌─────────────────────┐
│ Fleet Distribution   │──── Push global update to all vehicles
│                     │      Vehicles merge and continue local CL
└─────────────────────┘
```

---

## 6. Route Optimization (GNN)

### 6.1 Graph Neural Network Architecture

Route optimization uses a spatial-temporal GNN where:
- **Nodes** represent road segments, intersections, or waypoints
- **Edges** represent traversal connections with learned weights
- **Node features** include historical travel time, current congestion, time-of-day encoding, weather
- **Edge features** include distance, road type, speed limit, turn complexity

The GNN is trained continually via the federated learning pipeline. Each vehicle's local route completions provide ground-truth traversal times that update the GNN's edge weight predictions.

### 6.2 Optimization Objectives

`route_optimize` accepts a multi-objective specification:

| Objective | Description | Default Weight |
|-----------|-------------|---------------|
| `min_time` | Minimize total travel time | 0.5 |
| `min_distance` | Minimize total distance | 0.2 |
| `min_fuel` | Minimize fuel/energy consumption | 0.15 |
| `max_reliability` | Maximize on-time arrival probability | 0.15 |
| `avoid_zones` | Hard constraint: avoid specified geofences | constraint |
| `prefer_zones` | Soft preference: prefer routes through specified geofences | 0.0 |

### 6.3 Temporal Integration (TickTickClock)

Route optimization is inherently spatial-temporal. GeoFleetic pairs with TickTickClock (`&time` primitives) to incorporate:
- **Time-of-day patterns** — rush hour vs off-peak learned from historical data
- **Temporal windows** — delivery windows, pickup deadlines, shift constraints
- **Predicted conditions** — weather, events, road closures from temporal forecasting

The pairing is "where AND when" — GeoFleetic handles the spatial graph, TickTickClock handles the temporal context, and the GNN fuses both for optimization.

---

## 7. Geofencing Engine

### 7.1 Tile38 Integration

GeoFleetic wraps Tile38 for real-time geofence evaluation. Geofences are defined as GeoJSON polygons, circles, or bounding boxes and stored in Tile38's in-memory spatial index.

```elixir
defmodule GeoFleetic.Geofence.Engine do
  use GenServer

  # Check if positions fall within geofences
  def contains(location_set) do
    location_set
    |> Task.async_stream(fn {asset_id, {lon, lat}} ->
      Tile38.within(asset_id, lon, lat)
    end, max_concurrency: 1000)
    |> Enum.map(&to_membership/1)
  end

  # Subscribe to enter/exit transitions
  def subscribe_events(geofence_id, callback) do
    Tile38.sethook(geofence_id, callback)
  end
end
```

### 7.2 Continual Learning for Geofences

Static geofences degrade over time as real-world boundaries shift (construction, new roads, seasonal access changes). GeoFleetic applies continual learning to geofence boundaries:

1. **Observation** — Track actual enter/exit patterns vs expected boundaries
2. **Anomaly detection** — Flag when >N% of transitions occur outside the defined boundary
3. **Boundary suggestion** — Propose adjusted geofence polygons based on observed patterns
4. **Human-in-the-loop** — Surface suggestions via MCP; humans approve or reject

This creates geofences that adapt to reality rather than requiring manual maintenance.

---

## 8. [&] Capability Contracts

GeoFleetic implements three [&] Protocol capability contracts. Each contract defines operations with typed inputs and outputs that any [&]-compatible client can invoke.

### 8.1 `&space.fleet`

Fleet asset tracking and spatial awareness.

| Operation | Input | Output | Description |
|-----------|-------|--------|-------------|
| `locate` | `asset_query` | `location_set` | Resolve current positions for queried assets. Query supports asset IDs, types, regions, or status filters. |
| `enrich` | `context` | `spatial_context` | Attach fleet spatial context (nearby assets, density, capacity) to an upstream artifact. |
| `capacity` | `region_query` | `capacity_snapshot` | Region-level capacity snapshot: active assets, utilization, coverage gaps. |
| `route` | `route_request` | `route_plan` | Generate a route plan for a fleet asset under constraints. Delegates to `&space.route`. |

### 8.2 `&space.route`

Route generation, optimization, and explainability.

| Operation | Input | Output | Description |
|-----------|-------|--------|-------------|
| `route` | `route_request` | `route_plan` | Generate a baseline route plan given origin, destination, waypoints, and constraints. |
| `optimize` | `route_plan` | `optimized_route` | Improve an existing route against multi-objective criteria (time, distance, fuel, reliability). |
| `explain` | `route_plan` | `route_explanation` | Produce a human-readable explanation of why this route was chosen, including alternatives considered and tradeoffs. |

### 8.3 `&space.geofence`

Boundary evaluation and transition events.

| Operation | Input | Output | Description |
|-----------|-------|--------|-------------|
| `contains` | `location_set` | `geofence_membership` | Evaluate which geofences each location belongs to. Returns membership set per location. |
| `enter_exit` | `trajectory_set` | `boundary_events` | Detect geofence boundary crossings from a trajectory. Returns timestamped enter/exit events. |
| `enrich` | `context` | `spatial_context` | Attach geofence context (zone names, policy tags, capacity) to an upstream artifact. |

---

## 9. MCP Tools

GeoFleetic exposes itself as a single MCP server via `hermes_mcp`. All spatial operations are MCP tools and resources.

### 9.1 Server Registration

```elixir
defmodule GeoFleetic.MCP.Server do
  use Hermes.Server,
    name: "geofleetic",
    version: "0.1.0",
    protocol_version: "2025-06-18"
end
```

### 9.2 MCP Tools

#### Fleet Operations

| Tool | Description | Input | Output |
|------|------------|-------|--------|
| `fleet_locate` | Resolve current positions for queried assets | `{query: string, types?: [string], region?: geojson, status?: string, limit?: int}` | `{locations: [{asset_id, position, heading, speed, status, updated_at}]}` |
| `fleet_capacity` | Region-level capacity snapshot | `{region: geojson, asset_types?: [string], time_window?: string}` | `{total_assets: int, active: int, idle: int, utilization: float, coverage_gaps: [geojson]}` |

#### Route Operations

| Tool | Description | Input | Output |
|------|------------|-------|--------|
| `route_generate` | Create a route plan under constraints | `{origin: {lon, lat}, destination: {lon, lat}, waypoints?: [{lon, lat}], constraints?: object, objectives?: object}` | `{route_id: string, segments: [Segment], total_distance_m: float, total_time_s: float, confidence: float}` |
| `route_optimize` | Improve an existing route for specified objectives | `{route_id: string, objectives: {min_time?: float, min_distance?: float, min_fuel?: float, max_reliability?: float}, avoid_zones?: [string], prefer_zones?: [string]}` | `{route_id: string, segments: [Segment], improvements: {time_delta_s, distance_delta_m, fuel_delta_pct, reliability_delta}, model_version: string}` |
| `route_explain` | Explain route decisions in human-readable form | `{route_id: string, detail_level?: "summary"\|"full"}` | `{explanation: string, factors: [{factor, weight, impact}], alternatives_considered: int, tradeoffs: [string]}` |

#### Geofence Operations

| Tool | Description | Input | Output |
|------|------------|-------|--------|
| `geofence_check` | Evaluate boundary membership for locations | `{locations: [{asset_id, lon, lat}], geofence_ids?: [string]}` | `{memberships: [{asset_id, geofences: [string], distances_m: [float]}]}` |
| `geofence_events` | Subscribe to enter/exit transitions | `{geofence_ids: [string], asset_types?: [string], callback_url?: string}` | `{subscription_id: string, status: "active"}` |

#### Spatial Enrichment

| Tool | Description | Input | Output |
|------|------------|-------|--------|
| `spatial_enrich` | Attach spatial context to an upstream artifact | `{context: object, enrichments: ["nearby_assets"\|"zone_membership"\|"capacity"\|"route_proximity"]}` | `{spatial_context: {nearby_assets?: [Asset], zones?: [Zone], capacity?: Snapshot, route_proximity?: [Route]}}` |

### 9.3 MCP Resources

```
resources://fleet/stats              → Fleet-wide statistics (count, active, idle, coverage)
resources://fleet/twin/{asset_id}    → Individual twin state snapshot
resources://fleet/region/{geojson}   → Assets within a region
resources://route/active             → Currently active route plans
resources://route/{route_id}         → Individual route details
resources://geofence/list            → All defined geofences
resources://geofence/{fence_id}      → Individual geofence definition + stats
resources://spatial/health           → Spatial system health (sync lag, model version, Tile38 status)
```

### 9.4 Example: Claude Desktop Integration

```json
{
  "mcpServers": {
    "geofleetic": {
      "command": "geofleetic",
      "args": ["--mode", "coordinator", "--db", "~/.geofleetic/fleet.db"],
      "env": {
        "GEOFLEETIC_TILE38_URL": "redis://localhost:9851",
        "GEOFLEETIC_POSTGIS_URL": "postgres://localhost/geofleetic_dev"
      }
    }
  }
}
```

Once configured, Claude (or any MCP client) can:
1. **Ask spatial questions:** Call `fleet_locate` to find where assets are
2. **Plan routes:** Call `route_generate` → `route_optimize` → `route_explain`
3. **Monitor boundaries:** Call `geofence_check` or subscribe via `geofence_events`
4. **Enrich decisions:** Call `spatial_enrich` to add spatial context to any reasoning

---

## 10. Integration Points

### 10.1 TickTickClock (`&time`)

GeoFleetic and TickTickClock are the spatial-temporal complement pair. "Where AND when."

- **Route optimization** queries TickTickClock for temporal patterns (rush hour, seasonal, event-based)
- **Geofence events** are timestamped via TickTickClock's consistent temporal model
- **Capacity snapshots** include temporal forecasts (predicted utilization in 1h, 4h, 24h)
- **Joint queries** like "which vehicles will be within 5km of warehouse W between 2pm-4pm?" compose `&space.fleet.locate` with `&time.predict`

### 10.2 Graphonomous (`&knowledge`)

Spatial knowledge persisted in the knowledge graph:

- **Semantic nodes** store learned spatial patterns ("Route 9 congests Mondays 8-9am", "Warehouse W is at capacity Fridays")
- **Procedural nodes** encode route strategies ("For downtown deliveries, approach from the east via Bridge St")
- **Episodic nodes** record specific fleet events ("Vehicle 42 was delayed 22min on Route 9 on March 12")
- GeoFleetic calls `graphonomous.learn_from_outcome` after each route completion to ground spatial learning

### 10.3 Deliberatic (`&deliberate`)

Spatial disputes resolved via argumentation:

- **Route conflicts** — two vehicles assigned to the same narrow street; Deliberatic resolves priority
- **Geofence disputes** — asset claimed in two overlapping zones; Deliberatic evaluates policy arguments
- **Capacity allocation** — competing requests for limited capacity in a region; Deliberatic runs a structured debate

### 10.4 Delegatic (`&govern`)

Geofence policies enforced via governance:

- **Zone access control** — Delegatic policies define which teams/agents can operate in which geofences
- **Route approval** — high-cost or hazardous routes require Delegatic governance gate
- **Privacy policies** — Delegatic enforces fleet-level privacy budgets (epsilon per sync round)
- **Audit trail** — every spatial operation is logged to Delegatic's append-only audit

### 10.5 FleetPrompt (`&publish`)

Spatial intelligence agents published to the marketplace:

- `geofleetic/logistics-optimizer` — Last-mile delivery route optimization
- `geofleetic/geofence-monitor` — Real-time boundary monitoring with alerts
- `geofleetic/fleet-capacity-planner` — Capacity forecasting and rebalancing
- `geofleetic/spatial-enrichment` — Generic spatial context enrichment for any agent

### 10.6 WebHost.Systems (`&host`)

Managed GeoFleetic instances:

- PostgreSQL + PostGIS backed (not SQLite)
- Managed Tile38 instances for geofencing
- Hosted MCP endpoints with API key auth
- Usage-based billing per twin/query/route
- Monitoring dashboard with fleet map visualization

---

## 11. Project Structure

```
geofleetic/
├── mix.exs
├── config/
│   ├── config.exs
│   ├── dev.exs
│   ├── prod.exs
│   └── runtime.exs
├── lib/
│   ├── geofleetic/
│   │   ├── application.ex              # OTP application + supervision tree
│   │   ├── cli.ex                      # CLI entrypoint (MCP server, coordinator, edge)
│   │   ├── fleet/
│   │   │   ├── tracker.ex              # DynamicSupervisor for twin processes
│   │   │   ├── twin.ex                 # Per-asset GenServer (delta-CRDT state)
│   │   │   ├── crdt_sync.ex            # Epoch-aware delta-CRDT synchronization
│   │   │   └── capacity.ex             # Region-level capacity aggregation
│   │   ├── route/
│   │   │   ├── optimizer.ex            # GNN-based route optimization
│   │   │   ├── generator.ex            # Baseline route generation
│   │   │   ├── explainer.ex            # Human-readable route explanations
│   │   │   └── gnn/
│   │   │       ├── model.ex            # GNN architecture (Nx/Axon)
│   │   │       ├── features.ex         # Spatial-temporal feature extraction
│   │   │       └── inference.ex        # Route scoring and ranking
│   │   ├── geofence/
│   │   │   ├── engine.ex              # Tile38-backed geofence evaluation
│   │   │   ├── event_stream.ex        # PubSub for enter/exit transitions
│   │   │   ├── boundary_learner.ex    # Continual geofence adaptation
│   │   │   └── definitions.ex         # Geofence CRUD (GeoJSON storage)
│   │   ├── learning/
│   │   │   ├── coordinator.ex         # Federated learning round orchestration
│   │   │   ├── local_trainer.ex       # On-device LoRA training
│   │   │   ├── delta_aggregator.ex    # FedAvg delta aggregation
│   │   │   └── privacy_budget.ex      # Differential privacy enforcement
│   │   ├── spatial/
│   │   │   ├── enricher.ex            # Spatial context enrichment
│   │   │   └── index.ex               # Spatial index abstraction (PostGIS/sqlite-vec)
│   │   ├── storage/
│   │   │   ├── behaviour.ex           # Storage behaviour (adapter pattern)
│   │   │   ├── postgis.ex             # PostgreSQL + PostGIS adapter
│   │   │   ├── sqlite.ex              # SQLite + sqlite-vec adapter (edge)
│   │   │   ├── tile38.ex              # Tile38 client adapter
│   │   │   └── ets_cache.ex           # ETS hot twin state cache
│   │   ├── mcp/
│   │   │   ├── server.ex             # Hermes MCP server definition
│   │   │   └── tools/
│   │   │       ├── fleet_tools.ex     # fleet_* tools
│   │   │       ├── route_tools.ex     # route_* tools
│   │   │       ├── geofence_tools.ex  # geofence_* tools
│   │   │       └── spatial_tools.ex   # spatial_* tools
│   │   └── contracts/
│   │       ├── space_fleet.ex         # &space.fleet contract impl
│   │       ├── space_route.ex         # &space.route contract impl
│   │       └── space_geofence.ex      # &space.geofence contract impl
│   └── geofleetic_web/               # Optional Phoenix app
│       ├── router.ex
│       ├── live/
│       │   ├── fleet_map_live.ex      # Real-time fleet map
│       │   ├── route_view_live.ex     # Route visualization
│       │   ├── geofence_editor_live.ex # Geofence polygon editor
│       │   └── capacity_live.ex       # Capacity heatmaps
│       └── components/
├── priv/
│   ├── migrations/                    # Ecto migrations (PostGIS mode)
│   └── sqlite/
│       └── schema.sql                 # SQLite schema (edge mode)
├── test/
└── rel/
    └── env.sh.eex
```

---

## 12. Supervision Tree

```elixir
defmodule GeoFleetic.Application do
  use Application

  def start(_type, _args) do
    children = [
      # Storage layer (starts first)
      {GeoFleetic.Storage, storage_config()},

      # Tile38 connection pool
      {GeoFleetic.Storage.Tile38, tile38_config()},

      # ETS hot cache for twin state
      GeoFleetic.Storage.ETSCache,

      # Fleet tracking (DynamicSupervisor for twin processes)
      GeoFleetic.Fleet.Tracker,

      # CRDT synchronization
      {GeoFleetic.Fleet.CRDTSync, crdt_config()},

      # Route optimization (GNN model + inference)
      {GeoFleetic.Route.Optimizer, optimizer_config()},

      # Geofence engine
      GeoFleetic.Geofence.Engine,

      # Geofence event stream (PubSub)
      {Phoenix.PubSub, name: GeoFleetic.PubSub},
      GeoFleetic.Geofence.EventStream,

      # Federated learning coordinator
      {GeoFleetic.Learning.Coordinator, learning_config()},

      # MCP Server (primary API)
      {GeoFleetic.MCP.Server, mcp_config()},

      # Optional: Phoenix endpoint (admin UI)
      maybe_start_web()
    ] |> List.flatten() |> Enum.reject(&is_nil/1)

    opts = [strategy: :rest_for_one, name: GeoFleetic.Supervisor]
    Supervisor.start_link(children, opts)
  end
end
```

**Supervision strategy: `:rest_for_one`** — If storage crashes, everything downstream restarts. If the fleet tracker crashes, CRDT sync and learning restart too. The MCP server is near the end of the chain so it restarts cleanly if any upstream dependency fails.

---

## 13. Configuration

```elixir
# config/runtime.exs
config :geofleetic,
  # Mode
  mode: System.get_env("GEOFLEETIC_MODE", "coordinator"),  # "coordinator" | "edge"

  # Storage (coordinator)
  postgis_url: System.get_env("GEOFLEETIC_POSTGIS_URL"),

  # Storage (edge)
  sqlite_path: System.get_env("GEOFLEETIC_DB", "~/.geofleetic/spatial.db"),

  # Tile38
  tile38_url: System.get_env("GEOFLEETIC_TILE38_URL", "redis://localhost:9851"),

  # Fleet tracking
  max_twins: 100_000,
  twin_idle_timeout: :timer.minutes(30),
  twin_hibernate_after: :timer.minutes(5),
  crdt_sync_interval: :timer.seconds(5),
  crdt_gc_epoch_lag: 10,

  # Route optimization
  gnn_model_path: System.get_env("GEOFLEETIC_GNN_MODEL", "~/.geofleetic/models/route_gnn.onnx"),
  route_max_waypoints: 50,
  route_default_objectives: %{min_time: 0.5, min_distance: 0.2, min_fuel: 0.15, max_reliability: 0.15},

  # Federated learning
  federated_learning_enabled: true,
  sync_round_interval: :timer.minutes(15),
  lora_rank: 8,
  lora_alpha: 16,
  gradient_quantization: :int8,
  differential_privacy_epsilon: 1.0,
  min_vehicles_per_round: 3,

  # Geofencing
  geofence_check_concurrency: 1000,
  boundary_learning_enabled: true,
  boundary_anomaly_threshold: 0.15,

  # MCP transport
  mcp_transport: :stdio,     # :stdio | :streamable_http
  mcp_port: 4300,            # Only for streamable_http

  # Web UI
  enable_web: false,
  web_port: 4400
```

---

## 14. Implementation Roadmap

### Pre-Phase: Feasibility Validation (Weeks 0–2)

Before committing to the full implementation, validate the three highest-risk technical assumptions:

**FV-1: Delta-CRDT convergence on spatial state**
- [ ] Prototype a minimal delta-CRDT (position + heading + speed) using the `delta_crdt` hex package or a custom Elixir implementation
- [ ] Simulate 100 concurrent twin updates with network partitions (delayed messages, reordering)
- [ ] Measure convergence time and final-state correctness under partition
- **Pass criteria:** 100% eventual consistency within 5 seconds of partition heal; zero lost updates

**FV-2: Federated learning without EXLA**
- [ ] Build a minimal LoRA adapter training loop using `Nx` + `Bumblebee` + ONNX runtime (no EXLA — consistent with portfolio EXLA exclusion)
- [ ] Train on a synthetic route dataset (100 routes, 10 features) on a single node
- [ ] Verify that LoRA delta extraction + compression + aggregation produces a valid merged adapter
- **Pass criteria:** Merged adapter performs within 5% of centrally-trained baseline; adapter delta size <100KB per node per round

**FV-3: GNN route optimization in Elixir**
- [ ] Evaluate whether `Nx` can express a basic GCN/GAT for route scoring (forward pass only — inference, not training)
- [ ] If Nx cannot express the GNN architecture efficiently, design a Python sidecar protocol (gRPC or STDIO) and prototype the boundary
- [ ] Benchmark inference latency for a 50-node road graph
- **Pass criteria:** Route scoring <500ms on commodity hardware; sidecar boundary adds <50ms overhead if needed
- **Decision gate:** If FV-3 requires a Python sidecar, update the spec's technology stack section and OTP supervision tree to include the sidecar process

**FV-4: SQLite-first for MVP (de-risk operational complexity)**
- [ ] Evaluate whether SQLite + sqlite-vec can serve as the sole storage layer for Phase 0 and Phase 1, deferring PostGIS and Tile38 to Phase 2+
- [ ] Prototype basic point-in-polygon and nearest-neighbor queries using sqlite-vec
- **Pass criteria:** Geofence checks <10ms for 1000 boundaries; if not, PostGIS is required from Phase 0

### Acceptance Test Criteria

**Spatial Digital Twins:**
- Given a twin GenServer with position (lon, lat) → `fleet_locate` returns position within 10ms
- Given 10K concurrent twins updating at 1Hz → BEAM node memory < 2GB; no process mailbox overflow
- Given a twin crash → DynamicSupervisor restarts it with last-known state from ETS cache within 100ms
- Given a twin with no updates for `stale_timeout` → status transitions to `:stale`

**Delta-CRDTs:**
- Given 2 nodes with divergent twin state after a partition → states converge within 5s of partition heal
- Given concurrent updates to the same twin from 2 nodes → delta merge produces correct final state (no lost updates)
- Given a delta buffer exceeding `max_delta_buffer_size` → garbage collection runs without dropping un-synced deltas

**Route Optimization:**
- Given a route request with start + end coordinates → `route_generate` returns a valid path within 2s
- Given a GNN-optimized route → travel time is at least 15% better than static shortest-path
- Given `route_explain` → response includes human-readable justification referencing spatial + temporal factors

**Geofencing:**
- Given a point inside a geofence boundary → `geofence_check` returns `{:inside, geofence_id}` within 10ms
- Given a twin crossing a geofence boundary → `geofence_events` emits `enter` or `exit` event within 1s
- Given 1000 geofence boundaries → `geofence_check` latency < 10ms p99

**`&govern` Integration:**

GeoFleetic emits telemetry via `&govern.telemetry.emit`:
- `twin.updated` — position change events (sampled at 1% for high-frequency streams)
- `route.generated` — route computation with duration and compute cost
- `geofence.event` — boundary crossing events for audit trails
- `federation.round` — federated learning round completion with delta sizes

When a route optimization exceeds `max_compute_ms_per_task` (from Delegatic policy), GeoFleetic escalates via `&govern.escalation.escalate` before returning a potentially suboptimal result.

### Phase 0: Foundation (Weeks 3–6)

- [ ] Project scaffold (mix new, supervision tree, config)
- [ ] PostgreSQL + PostGIS storage adapter with spatial schema (or SQLite-first per FV-4 outcome)
- [ ] SQLite storage adapter for edge mode
- [ ] Spatial digital twin GenServer with basic CRUD
- [ ] DynamicSupervisor for twin lifecycle
- [ ] ETS hot cache for twin state lookups
- [ ] **Proof:** 10K concurrent twin processes updating positions at 1Hz on single BEAM node

### Phase 1: Delta-CRDTs + Fleet Tracking (Weeks 5–10)

- [ ] Epoch-aware delta-CRDT implementation (GeoCoCo 2025)
- [ ] Vector clock causal ordering
- [ ] Delta propagation and garbage collection
- [ ] `fleet_locate` and `fleet_capacity` MCP tools
- [ ] Tile38 integration for geofence storage
- [ ] Basic `geofence_check` — point-in-polygon via Tile38
- [ ] `geofence_events` — enter/exit subscription via Tile38 hooks
- [ ] **Proof:** Two BEAM nodes syncing 1K twins with eventual convergence under simulated partition

### Phase 2: Route Optimization (Weeks 11–16)

- [ ] GNN model architecture in Nx/Axon
- [ ] Spatial-temporal feature extraction pipeline
- [ ] `route_generate` — baseline route planning
- [ ] `route_optimize` — multi-objective GNN-based optimization
- [ ] `route_explain` — explainable route decisions
- [ ] TickTickClock integration for temporal context
- [ ] **Proof:** GNN routes outperform static shortest-path by >15% on travel time in simulation

### Phase 3: Federated Learning (Weeks 17–22)

- [ ] Local LoRA trainer on edge devices
- [ ] Delta compression (INT8 quantization)
- [ ] FedAvg aggregation coordinator
- [ ] Privacy budget enforcement (differential privacy)
- [ ] Model version management and fleet distribution
- [ ] `spatial_enrich` MCP tool
- [ ] **Proof:** Fleet of 100 simulated vehicles collectively improves route quality by >20% over 1000 routes without sharing raw data

### Phase 4: MCP Server + Polish (Weeks 23–28)

- [ ] Hermes MCP server with all tools defined in section 9.2
- [ ] MCP resources defined in section 9.3
- [ ] STDIO transport (for Claude Desktop, Cursor, etc.)
- [ ] Streamable HTTP transport (for remote access)
- [ ] Phoenix LiveView fleet map dashboard
- [ ] [&] capability contract compliance validation
- [ ] Integration tests: Claude Desktop → GeoFleetic MCP → fleet operations

### Phase 5: Ecosystem Integration (Weeks 29–34)

- [ ] Graphonomous integration: persist spatial knowledge as graph nodes
- [ ] Deliberatic integration: spatial dispute resolution
- [ ] Delegatic integration: geofence access control policies
- [ ] FleetPrompt skill packaging
- [ ] WebHost.Systems managed hosting support
- [ ] Continual geofence boundary learning
- [ ] Release packaging (mix release, Docker, edge image)

---

## 15. Pricing

| Tier | Twins | Routes/mo | Geofences | Fed. Learning | Price |
|------|-------|-----------|-----------|---------------|-------|
| **Edge** (self-hosted) | Unlimited | Unlimited | Unlimited | Local only | Free (MIT) |
| **Starter** | 500 | 10,000 | 50 | 5 vehicles | $79/mo |
| **Pro** | 5,000 | 100,000 | 500 | 50 vehicles | $299/mo |
| **Fleet** | 50,000 | 1,000,000 | 5,000 | 500 vehicles | $999/mo |
| **Enterprise** | Custom | Custom | Custom | Custom | Contact |

All managed tiers include WebHost.Systems hosting, PostGIS + Tile38 infrastructure, monitoring dashboard, and 99.9% SLA.

---

## 16. Research Foundations

1. **Epoch-Aware Delta-CRDTs** (GeoCoCo 2025) — Causal consistency with bounded metadata for geo-distributed CRDT synchronization. Basis for GeoFleetic's twin sync protocol.

2. **Federated Continual Adaptation via LoRA** — Extending FedAvg to low-rank adapters for continual model updates without centralizing training data. Privacy-preserving by construction.

3. **Spatial Digital Twin Framework** (IEEE COMST 2025) — Reference architecture for maintaining synchronized virtual replicas of physical assets with real-time state propagation.

4. **MCP + A2A Agent Protocol Integration** — Model Context Protocol for tool exposure. Agent-to-Agent protocol for cross-fleet coordination. GeoFleetic implements both.

---

## 17. Open Questions

1. **GNN architecture selection:** Spatial-temporal graph networks (STGNN) vs Graph Attention Networks (GAT) vs simpler message-passing GNNs. Need to benchmark inference latency on edge devices.

2. **LoRA rank tradeoff:** Rank 4 is tiny but may underfit complex route patterns. Rank 16 captures more but increases delta transmission size. Needs empirical evaluation per fleet size.

3. **Tile38 vs pure PostGIS geofencing:** Tile38 adds operational complexity (another process to manage). For small fleets (<100 assets), PostGIS `ST_Contains` at 10ms may be sufficient. When does Tile38's microsecond latency become necessary?

4. **Delta-CRDT garbage collection policy:** Aggressive GC saves memory but risks data loss if a long-offline vehicle reconnects. Need to define maximum offline window before twin state is considered stale.

5. **Differential privacy epsilon:** Default epsilon=1.0 provides moderate privacy. Lower values (0.1) add more noise, potentially degrading route quality. Fleet operators should be able to configure this tradeoff per their regulatory requirements.

6. **sqlite-vec on edge devices:** sqlite-vec HNSW performance on ARM processors (Raspberry Pi, vehicle compute modules) needs benchmarking at 10K+ vectors.

---
