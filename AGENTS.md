# GeoFleetic — Agent Interface

GeoFleetic is the spatial intelligence layer for the [&] Protocol ecosystem. It provides `&space` capabilities to AI agents.

## Capabilities

### &space.fleet
- `locate` — resolve current fleet/asset locations
- `enrich` — attach spatial context to upstream artifacts
- `capacity` — return region/fleet capacity snapshots
- `route` — produce fleet-aware route recommendations

### &space.route
- `route` — generate feasible route plans from origin/destination/constraints
- `optimize` — improve existing routes for efficiency, cost, or SLOs
- `explain` — explain route selection and tradeoffs for auditing

### &space.geofence
- `contains` — evaluate location membership in geofence boundaries
- `enter_exit` — detect boundary crossing events for tracked assets
- `enrich` — attach geofence/compliance context to upstream artifacts

## Protocol Integration

- Accepts from: `&memory.*`, `&time.*`, raw data, context
- Feeds into: `&reason.*`, `&memory.*`, output
- A2A skills: fleet-state-enrichment, regional-capacity-lookup, route-feasibility-evaluation, route-generation, route-optimization, geofence-membership-evaluation, boundary-alerting
- Transport: MCP v1 (Streamable HTTP)

## Status

Spec complete. Implementation pending. See `project_spec/README.md`.
