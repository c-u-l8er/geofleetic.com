# GeoFleetic — Spatial Intelligence Layer

Geo-distributed fleet intelligence collaboration. The `&space` primitive provider for the [&] Protocol ecosystem.

## Source-of-truth spec

- `project_spec/README.md` — GeoFleetic technical specification

## [&] Capabilities provided

| Capability | Contract | Operations |
|---|---|---|
| `&space.fleet` | `AmpersandBoxDesign/contracts/v0.1.0/space.fleet.contract.json` | locate, enrich, capacity, route |
| `&space.route` | `AmpersandBoxDesign/contracts/v0.1.0/space.route.contract.json` | route, optimize, explain |
| `&space.geofence` | `AmpersandBoxDesign/contracts/v0.1.0/space.geofence.contract.json` | contains, enter_exit, enrich |

## Key technologies

- Epoch-aware delta-CRDTs for convergent fleet state
- Federated continual learning (LoRA adapters, no raw data sharing)
- Spatial digital twins (IEEE COMST 2025 research)
- GNN route optimization with continual spatial learning
- MCP + A2A agent protocols

## Paired with

- **TickTickClock** — temporal intelligence (when + where = complete situational awareness)
- **Graphonomous** — continual learning substrate
- **Delegatic** — governance enforcement for fleet operations

## Status

This is a spec + marketing site. No implementation code yet. Implementation will be Elixir/OTP.
