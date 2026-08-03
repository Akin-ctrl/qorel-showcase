# ADR-001: Edge Buffering Strategy

**Status**: Accepted, amended for Redpanda direction
**Date**: 2026-01-29  
**Last Updated**: 2026-05-30
**Decision**: Use an embedded Kafka-compatible broker for edge buffering, with Redpanda as the chosen edge direction

---

## Context

Industrial gateways must buffer data locally when network connectivity to sinks is unavailable. This is critical for:
- Offshore installations with satellite links
- Remote sites with intermittent connectivity
- Network outages lasting hours to days

We needed to decide how to implement reliable local buffering.

## Options Considered

### Option A: Custom File-Based Buffer
- Write messages to local files
- Custom replay logic
- Custom durability guarantees

**Pros**: Lightweight, no dependencies  
**Cons**: Reinventing the wheel, error-prone, no standard tooling

### Option B: SQLite Queue
- Store messages in SQLite
- FIFO queue pattern

**Pros**: Simple, embedded  
**Cons**: Not designed for streaming, no consumer offset tracking, limited throughput

### Option C: Embedded Kafka-Compatible Broker
- Single-node Kafka-compatible broker per gateway
- Local dev/runtime path uses Redpanda
- Production packaging and image-pull templates are still pending
- Redpanda keeps Kafka protocol semantics with a lighter edge operational profile
- Standard Kafka protocol

**Pros**: Proven durability, standard tooling, replay capability, consumer offset tracking  
**Cons**: Higher resource usage (~512MB RAM)

### Option D: Redis Streams
- Use Redis as a message broker
- Lightweight streaming

**Pros**: Fast, lightweight  
**Cons**: Not designed for long-term persistence, less mature than Kafka

## Decision

**Option C: Embedded Kafka-compatible broker**

The original implementation used embedded Kafka in KRaft mode. The project is
now Redpanda-first for the local dev/runtime broker path while retaining Kafka
protocol, client, topic, partition, and consumer-group semantics. Production
packaging is still pending and should be documented separately when the
image/template workflow is implemented.

## Rationale

1. **Proven durability model**: Kafka-compatible logs are well suited to local buffering and replay
2. **Standard protocol**: Sinks use standard Kafka-compatible consumers, no custom code
3. **Replay capability**: Any sink can replay from any offset
4. **Consumer offsets**: Track exactly what each sink has processed
5. **No central broker required**: Each gateway is fully independent
6. **Backpressure handling**: Native flow control
7. **Edge operational fit**: Redpanda is better aligned with constrained gateway deployments than operating a heavier Kafka distribution

The broker must still be sized explicitly for the gateway hardware. Redpanda
should reduce operational complexity, but it does not remove the need to plan
disk, memory, and retention behavior.

## Consequences

### Positive
- Reliable buffering with minimal custom code
- Sinks can use Kafka-compatible client libraries
- Standard CLI tools for debugging
- Natural fit for multi-sink architecture

### Negative
- Higher resource footprint than a custom file queue or SQLite
- Kafka-compatible broker operational knowledge still required
- Disk I/O requirements for durability

### Mitigations
- Configure appropriate memory limits
- Document Redpanda/Kafka-compatible tuning for edge deployment
- Provide default configurations for common hardware

## Related Decisions
- [ADR-009: Overflow Handling](ADR-009-overflow-handling.md)
