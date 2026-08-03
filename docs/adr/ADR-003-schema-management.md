# ADR-003: Schema Management Strategy

**Status**: Accepted  
**Date**: 2026-01-29  
**Decision**: Use Avro serialization with Schema Registry and offline caching

---

## Context

Industrial data must be serialized for transmission through Kafka and to sinks. Key requirements:
- Schema enforcement (prevent garbage data)
- Schema evolution (add fields over time)
- Compact binary format (bandwidth-constrained links)
- Offline operation (gateway may be disconnected)

## Options Considered

### Option A: JSON (No Schema)
- Human-readable
- No schema enforcement

**Pros**: Simple, debuggable  
**Cons**: No validation, verbose, no evolution support

### Option B: JSON Schema
- JSON with schema validation
- Schemas stored separately

**Pros**: Human-readable, validation  
**Cons**: Verbose, slower serialization, limited evolution

### Option C: Avro with Schema Registry
- Binary format with embedded schema ID
- Centralized schema registry
- Strong evolution support

**Pros**: Compact, fast, excellent evolution, industry standard  
**Cons**: Binary (harder to debug), registry dependency

### Option D: Protobuf
- Google's binary format
- Strong typing

**Pros**: Compact, fast, widely used  
**Cons**: Code generation required, less flexible than Avro for dynamic schemas

## Decision

**Option C: Avro with Schema Registry** with mandatory offline caching

## Rationale

1. **Compact binary**: Critical for bandwidth-constrained industrial links
2. **Schema evolution**: BACKWARD compatibility allows adding fields safely
3. **Schema Registry**: Single source of truth for data contracts
4. **Industry standard**: Well-supported by Kafka ecosystem
5. **Offline support**: Cache schemas locally for disconnected operation

## Offline Caching Strategy

The implemented strategy is lazy, on-demand caching rather than a startup bulk
pull. This is intentional: a bulk pull-all-schemas step adds first-boot
latency and complexity without a corresponding reliability benefit, since the
lazy path already tolerates outages once a schema has been seen once.

```
On gateway startup:
1. Load existing schema cache from disk (/data/schemas.cache), if present
2. Connect to Schema Registry in the background (non-blocking)

On produce:
1. Check local cache for schema
2. If missing and online → fetch and cache
3. If missing and offline → reject message (fail-safe)

On consume:
1. Schema ID is in message header
2. Look up in local cache
3. If missing and online → fetch and cache
4. If missing and offline → reject the message and surface the failure
   (there is no queue-and-replay-later path for this case)
```

## Consequences

### Positive
- Strong data contracts prevent garbage data
- Schema evolution allows non-breaking changes
- Compact format reduces bandwidth usage
- Offline operation supported

### Negative
- Binary format harder to debug
- Schema Registry is a required component
- Learning curve for Avro

### Mitigations
- Provide CLI tool to decode Avro messages for debugging
- Include Schema Registry in docker-compose
- Document common schema patterns

## Schema Evolution Rules

**Allowed changes**:
- Add optional field with default
- Add new enum value
- Widen numeric type (int → long)

**Breaking changes (rejected by registry)**:
- Remove required field
- Change field type incompatibly
- Rename field

## ISA-95 Telemetry Standard

Adapter output schemas support the **ISA-95 Hierarchical Model** as an
**optional** enrichment, not a required field. Many deployments (e.g. small
or informally-structured sites) will never populate it, and the wire format
must not force them to. When a site does have a known enterprise/site/area
hierarchy for an asset, that structure can be attached to give downstream
consumers (dashboards, RBAC-by-location, contextual routing) a consistent
place to read it from.

```json
{
  "asset_id": "pump-1",
  "gateway_time": "2026-07-08T10:00:00Z",
  "topology": {
    "enterprise": "string | null",
    "site": "string | null",
    "area": "string | null",
    "line": "string | null",
    "cell": "string | null",
    "equipment": "string | null"
  },
  "readings": [
    {
      "parameter": "string",
      "value": "double | long | boolean | null",
      "unit": "string",
      "quality": "GOOD | SUSPECT | UNCERTAIN | BAD",
      "device_time": "ISO8601 | null"
    }
  ]
}
```

`topology` defaults to `null` and every field inside it is independently
nullable, an adapter (or a site) that has no ISA-95 hierarchy simply omits
it, and the message is unaffected. The runtime-loaded schema
(`schemas/telemetry.avsc`) carries this field; earlier revisions of this ADR
described a `metrics` map keyed by parameter name, but the shipped wire
format uses a `readings` array instead (one entry per parameter, each
carrying its own quality and timestamp), the example above reflects what is
actually on the wire.

## Related Decisions
- [ADR-001: Edge Buffering](ADR-001-edge-buffering.md)
