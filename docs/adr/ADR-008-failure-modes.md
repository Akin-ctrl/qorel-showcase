# ADR-008: Failure Modes and Recovery

**Status**: Accepted  
**Date**: 2026-01-29  
**Decision**: Hybrid model with restart, bulkhead isolation, and circuit breaker

---

## Context

Industrial systems require high availability. Component failures are inevitable:
- Adapter crashes
- Network timeouts
- Disk full
- Downstream systems unavailable

We needed a consistent failure handling model.

## Options Considered

### Option A: Simple Restart
- Restart failed components automatically
- Fixed retry limits

**Pros**: Simple, predictable  
**Cons**: Not adaptive, no isolation

### Option B: Hierarchical Supervision (Erlang/Akka style)
- Supervision tree with restart strategies
- one-for-one, one-for-all, rest-for-one

**Pros**: Fine-grained control, proven  
**Cons**: Complex, overkill for our use case

### Option C: Health-Based Adaptive
- Health scoring per component
- Actions based on score thresholds

**Pros**: Adaptive, prevents thrashing  
**Cons**: Complex scoring, tuning required

### Option D: Hybrid (Restart + Bulkhead + Circuit Breaker)
- Simple restart for crashes
- Docker resource limits for isolation
- Circuit breaker for external connections

**Pros**: Practical combination, uses Docker's isolation  
**Cons**: Multiple patterns to understand

## Decision

**Option D: Hybrid model**

## Rationale

1. **Docker provides bulkhead**: Memory/CPU limits per container prevent cascade
2. **Simple restart is sufficient**: Most failures are transient
3. **Circuit breaker for external**: Prevents hammering unreachable services
4. **Graceful degradation**: Gateway continues with partial failures

## Failure Matrix

| Component | Failure | Detection | Recovery |
|-----------|---------|-----------|----------|
| Adapter | Crash | Docker event | Auto-restart (max 5 in 5 min) |
| Adapter | Protocol timeout | Health check | Retry with exponential backoff |
| Sink | Downstream unreachable | Write failure | Circuit breaker + local stream buffering |
| Sink | Repeated failures | 5 consecutive | Mark FAILED, alert, stop retrying |
| Validator | Crash | Process exit | Auto-restart, messages queue in the local stream |
| Local broker | Crash | Process exit | Auto-restart, data preserved on disk |
| Local broker | Disk full | Disk monitor | Tiered eviction (see ADR-009) |
| Gateway Runtime | Crash | Systemd/Docker | Restart, children restart |
| Control Plane | Unreachable | API timeout | Continue with cached config |

## Bulkhead Isolation

Each adapter and sink container has resource limits:

```yaml
# docker-compose example
services:
  adapter-modbus:
    image: adapter-modbus:1.0.0
    deploy:
      resources:
        limits:
          memory: 256M
          cpus: '0.5'
```

**Effect**: One adapter consuming excessive memory cannot starve others.

## Circuit Breaker

For external connections (sinks to destinations, gateway to Control Plane):

```
States:
  CLOSED: Normal operation
  OPEN: After 5 consecutive failures, stop trying for 30 seconds
  HALF_OPEN: Try one request, success → CLOSED, fail → OPEN

Timeouts:
  Open duration: 30 seconds
  Half-open test: 1 request
  Failure threshold: 5 consecutive
```

## Recovery States

| State | Meaning | Action |
|-------|---------|--------|
| HEALTHY | All OK | Normal operation |
| DEGRADED | Some non-critical failed | Alert, continue operating |
| UNHEALTHY | Critical component failed | Alert, attempt recovery |
| FAILED | Max retries exceeded | Requires manual intervention |

## Consequences

### Positive
- Predictable failure handling
- Cascade failures prevented
- External services protected from hammering
- Gateway stays up with partial failures

### Negative
- Multiple patterns to understand
- Circuit breaker adds latency variance

### Mitigations
- Clear documentation of failure states
- Health dashboard shows all component states
- Alerts on state transitions

## Related Decisions
- [ADR-009: Overflow Handling](ADR-009-overflow-handling.md)
- [ADR-006: Gateway Autonomy](ADR-006-gateway-autonomy.md)
