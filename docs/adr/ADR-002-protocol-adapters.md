# ADR-002: Protocol Adapter Architecture

**Status**: Accepted  
**Date**: 2026-01-29  
**Decision**: Deploy protocol adapters as Docker containers

---

## Context

Qorel must support multiple industrial protocols over time. The current
implemented set covers Modbus TCP, Modbus RTU, MQTT, and OPC UA; XBee, LoRa,
and other field/wireless protocols remain expansion targets. Each protocol has different:
- Library dependencies
- Runtime requirements
- Update cycles
- Security considerations

We needed to decide how to package and deploy protocol adapters.

## Options Considered

### Option A: Monolithic Runtime
- All adapters compiled into gateway runtime
- Single process handles all protocols

**Pros**: Simple deployment, shared memory  
**Cons**: Version coupling, one crash affects all, large binary, security concerns

### Option B: Plugin Architecture
- Dynamic loading of adapter plugins
- Shared process, separate modules

**Pros**: Lighter weight than containers  
**Cons**: Still version coupled, complex plugin interface, shared memory security

### Option C: Docker Containers
- Each adapter runs in its own container
- Publishes through the Kafka-compatible protocol to the gateway-local edge broker

**Pros**: Full isolation, independent versioning, security boundaries, standard tooling  
**Cons**: Higher resource overhead, container orchestration complexity

### Option D: Separate Processes (no containers)
- Each adapter is a separate OS process
- Managed by gateway runtime

**Pros**: Isolation without container overhead  
**Cons**: No standard packaging, dependency conflicts, harder to manage

## Decision

**Option C: Docker containers**

## Rationale

1. **Isolation**: Crashes in one adapter don't affect others
2. **Security**: Each adapter runs with minimal privileges
3. **Versioning**: Update adapters independently
4. **Resource limits**: Docker enforces memory/CPU limits (bulkhead pattern)
5. **Portability**: Same container works on any gateway
6. **Standard interface**: Config via environment, health via HTTP, output via the Kafka-compatible local stream

## Consequences

### Positive
- Clear security boundaries between protocols
- Independent release cycles per adapter
- Easy to add new protocols without touching core
- Bulkhead pattern prevents resource starvation

### Negative
- Container orchestration complexity
- ~50MB overhead per container
- Network latency between containers (minimal for localhost)

### Mitigations
- Gateway runtime handles container lifecycle
- Use lightweight base images (python:slim, alpine)
- All communication over localhost

## Adapter Contract

Every adapter must:
1. Accept config via `ADAPTER_CONFIG` environment variable (JSON)
2. Implement protocol-specific `connect`, `poll`, `transform`, and `disconnect` hooks via the shared `BaseAdapter` lifecycle template
3. Use the shared adapter Kafka-compatible publisher for local edge-topic writes so delivery semantics, asset-based partition keys, and publish health reporting stay consistent
4. Publish to Kafka-compatible topics specified in config
5. Support graceful shutdown on `SIGTERM` so the container can stop cleanly without data corruption
6. Expose `GET /health` endpoint
7. Expose `GET /metrics` endpoint (Prometheus format)
8. Log structured JSON to stdout
9. Support **Flexible Polling Groups** (e.g., fast/medium/slow) to optimize industrial network bandwidth, moving away from single global poll intervals.
10. Enrich every emitted reading with an explicit **Data Quality** indicator (`GOOD`, `BAD`, `UNCERTAIN`) to prevent downstream analytics from misinterpreting stale or faulted sensor data.

The shared lifecycle and publisher keep adapter behavior consistent while preserving ADR-002's container isolation model: `gateway_runtime` owns adapter containers, and `BaseAdapter` plus the shared adapter Kafka-compatible publisher own the in-container execution contract. In the local dev/runtime stack, that compatible broker is Redpanda.

## Related Decisions
- [ADR-005: Sink Architecture](ADR-005-sink-architecture.md)
