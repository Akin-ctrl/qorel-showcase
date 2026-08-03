# ADR-005: Sink Architecture

**Status**: Accepted  
**Date**: 2026-01-29  
**Decision**: Deploy sinks as Docker containers on gateway

---

## Context

Data must flow from the gateway-local Kafka-compatible stream to customer destinations:
- TimescaleDB/PostgreSQL
- Customer-owned Kafka-compatible cluster
- Cloud storage (S3, GCS)
- HTTP APIs
- Alerting systems (PagerDuty, Slack)

We needed to decide where sinks run and how they're deployed.

## Options Considered

### Option A: Sinks on Customer Infrastructure
- Customer deploys their own consumers
- We only provide local stream access

**Pros**: Less for us to manage  
**Cons**: Poor UX, customer needs Kafka expertise, connectivity issues

### Option B: Sinks in Control Plane
- Centralized sinks connect to the gateway-local stream
- All data flows through Control Plane

**Pros**: Centralized management  
**Cons**: Control Plane becomes data plane, latency, single point of failure

### Option C: Sinks as Docker Containers on Gateway
- Same deployment model as adapters
- Consume from the gateway-local stream, write to external destination

**Pros**: Consistent model, gateway is self-contained, works offline (buffers)  
**Cons**: More containers on gateway, external sinks need network

### Option D: Sinks as Gateway Runtime Modules
- Built into runtime like validator
- Less overhead than containers

**Pros**: Lightweight  
**Cons**: Coupled, harder to add new sinks, version coupling

## Decision

**Option C: Docker containers on gateway** (same as adapters)

## Rationale

1. **Consistency**: Same model as adapters (container, config, health, metrics)
2. **Isolation**: Each sink isolated, one failure doesn't affect others
3. **Independence**: Add new sinks without modifying gateway
4. **Offline capable**: Sinks buffer when destination unavailable
5. **Resource limits**: Prevent one slow sink from starving others

## Sink Contract

Every sink must (same pattern as adapters):

```
Environment:
  SINK_CONFIG: JSON configuration

Endpoints:
  GET /health → {"status": "healthy", "lag": 0}
  GET /metrics → Prometheus format

Behavior:
  - Consume from specified Kafka-compatible topics
  - Write to configured destination
  - Commit offsets after successful write
  - Handle retries internally
  - Expose consumer lag in metrics
```

## Current and Planned Sinks

| Sink | Description |
|------|-------------|
| `sink-timescaledb` | Implemented. Write telemetry, events, and aggregates to TimescaleDB/PostgreSQL-compatible destinations. |
| `sink-kafka` | Implemented. Replicate to a customer's Kafka-compatible system. |
| `sink-http` | Implemented. HTTP POST to webhook or API destination. |
| `sink-alert-router` | Implemented. Route alarms to webhook-style alert targets such as Slack. |
| `sink-postgres` | Planned/deferred as a separate dedicated PostgreSQL sink. |
| `sink-s3` | Planned/deferred for object-store archive workflows such as Parquet. |

## Key Clarification: Customer-Owned Kafka-Compatible Systems

**Qorel does not manage any central Kafka-compatible backbone.**

When customers need multi-gateway aggregation:
1. Each gateway has `sink-kafka` configured
2. `sink-kafka` replicates to the customer's own Kafka-compatible system
3. The customer manages that destination system

This keeps Qorel focused on the gateway, not external broker operations.

## Consequences

### Positive
- Gateway is fully self-contained
- Consistent operational model
- Easy to add new sink types
- Bulkhead isolation

### Negative
- More containers per gateway
- Each sink needs destination connectivity

### Mitigations
- Lightweight base images
- Clear documentation per sink type
- Health checks surface connectivity issues

## Related Decisions
- [ADR-002: Protocol Adapters](ADR-002-protocol-adapters.md)
- [ADR-001: Edge Buffering](ADR-001-edge-buffering.md)
