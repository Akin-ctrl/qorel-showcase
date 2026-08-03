# ADR-006: Gateway Autonomy

**Status**: Accepted  
**Date**: 2026-01-29  
**Decision**: Full gateway autonomy with cached configuration

---

## Context

Industrial gateways often operate in environments with:
- Intermittent network connectivity
- Network outages lasting hours to days
- Air-gapped deployments
- High reliability requirements

We needed to decide how dependent gateways are on the Control Plane.

## Options Considered

### Option A: Full Control Plane Dependency
- Gateway requires Control Plane for all operations
- Config fetched on every restart

**Pros**: Always current config, centralized control  
**Cons**: Gateway fails if Control Plane unreachable

### Option B: Cached Config, Periodic Sync
- Gateway caches config locally
- Syncs when Control Plane available

**Pros**: Survives Control Plane outages  
**Cons**: Config may be stale

### Option C: Full Autonomy
- First boot requires Control Plane (one-time)
- Subsequent boots use cached config
- Gateway runs indefinitely offline

**Pros**: Maximum reliability, survives extended outages  
**Cons**: Config updates delayed until reconnection

## Decision

**Option C: Full autonomy**

## Rationale

1. **Reliability**: Gateway should never stop collecting data due to Control Plane issues
2. **Reality**: Industrial networks are unreliable
3. **Simplicity**: "First boot needs network, then runs forever" is easy to understand
4. **Buffered continuity**: The gateway-local Kafka-compatible stream buffers during outages

## Behavior Matrix

| Scenario | First Boot | Subsequent Boot |
|----------|------------|-----------------|
| Control Plane reachable | Pull config, cache, start | Check for updates, use cached if offline |
| Control Plane unreachable | FAIL (cannot start) | Use cached config, start normally |
| Config change while offline | N/A | Applied when reconnected |

## Cached State

What's cached locally:

| Item | Location | Purpose |
|------|----------|---------|
| Gateway config | `/data/config/gateway.json` | Full pipeline, adapter, sink config |
| Schema cache | `/data/schemas/` | Avro schemas for serialization |
| Container images | Docker cache | Adapter and sink images |
| Local broker data | Broker data volume | Buffered messages |

## First Boot Sequence

```
1. Gateway starts
2. No cached config found
3. Connect to Control Plane (required)
4. Download config, schemas, pull images
5. Cache everything locally
6. Start adapters and sinks
7. Report healthy to Control Plane
```

## Subsequent Boot Sequence

```
1. Gateway starts
2. Cached config found
3. Start with cached config immediately
4. In background: try to reach Control Plane
5. If reachable: check for config updates
6. If updates: apply atomically
7. If unreachable: continue with cached config
```

## Consequences

### Positive
- Extremely reliable data collection
- Survives extended network outages
- Simple mental model

### Negative
- Config updates delayed during outages
- First boot requires network
- Stale config possible

### Mitigations
- UI shows "last config sync" time
- Alert if gateway hasn't synced in N hours: implemented, a periodic control-plane maintenance pass (`app/core/gateway_maintenance.py`, default threshold 24h, `QOREL_GATEWAY_SYNC_STALE_HOURS`) raises `alarm_type = "gateway.config_sync_stale"` per gateway and clears it automatically once the gateway resyncs (remediation ledger item D1)
- Manual config push via USB (future feature)

## Related Decisions
- [ADR-001: Edge Buffering](ADR-001-edge-buffering.md)
- [ADR-008: Failure Modes](ADR-008-failure-modes.md)
