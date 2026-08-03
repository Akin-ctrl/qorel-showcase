# ADR-004: Validation and DLQ Workflow

**Status**: Accepted  
**Date**: 2026-01-29  
**Decision**: Validator as Gateway Runtime module with operator DLQ workflow

---

## Context

Industrial data requires quality validation:
- Range checks (temperature between -50°C and 500°C)
- Rate-of-change detection (sudden spikes)
- Duplicate detection
- Gap detection

Invalid data must be handled without blocking the pipeline.

## Options Considered

### Option A: Validation in Adapter
- Each adapter validates its own data
- No centralized rules

**Pros**: Simple, close to source  
**Cons**: Duplicated logic, no cross-adapter rules, harder to update

### Option B: Validation as Separate Container
- Dedicated validation service
- Consumes raw, produces clean

**Pros**: Isolated, independently scalable  
**Cons**: Container overhead, another component to manage

### Option C: Validation as Gateway Runtime Module
- Built into gateway runtime process
- Consumes raw topics, produces clean topics

**Pros**: Lightweight, shared config, single deployment  
**Cons**: Coupled to runtime, can't scale independently

### Option D: Kafka Streams Application
- Stream processing for validation
- Full Kafka Streams semantics

**Pros**: Powerful, exactly-once semantics  
**Cons**: Complex, JVM dependency, overkill for validation

## Decision

**Option C: Validation as Gateway Runtime module**

## Rationale

1. **Lightweight**: No additional container overhead
2. **Shared config**: Validation rules from same config as adapters
3. **Simple**: Python module, easy to understand and modify
4. **Sufficient**: Validation doesn't need independent scaling at edge

## Quality Codes

Based on IEC 61850 quality model:

| Code | Meaning | Action |
|------|---------|--------|
| GOOD | Passed all validation | Forward to `.clean` topic |
| SUSPECT | Anomalous but within valid range | Forward with flag |
| UNCERTAIN | Low confidence (e.g., clock skew, missing device_time) | Forward with warning |
| BAD | Failed validation | Route to DLQ |

## DLQ Workflow

Dead Letter Queue provides operator visibility and recovery:

```
1. Validator detects BAD data
2. Message written to dlq.telemetry (or dlq.events)
3. Message includes:
   - Original payload
   - Validation failure reason
   - Timestamp

4. Operator views in UI:
   - Filter by gateway, time, error type
   - See original value and failure reason
   - Preview what value would look like if approved

5. Operator actions:
   a) Approve single message → reprocess to .clean topic
   b) Bulk approve → reprocess batch
   c) Bulk discard / bulk annotate → same batch primitives as approve
   d) Discard → remove from DLQ
```

A "suggested fix" field was originally scoped as part of the DLQ message but
was dropped, the `reason` field already carries the failure cause, and a
determinable, machine-generated remediation suggestion turned out to be more
speculative than useful in practice (e.g. "value outside range" doesn't
imply a correct value). Not implemented, not currently planned.

**On step (c), "update validation rules → reprocess matching":** this is
implemented as two separate, composable capabilities rather than one atomic
workflow action. Validation rules are edited per-deployment (in the
deployment form's `validation_config`); DLQ bulk-reprocessing is a separate
endpoint (`/bulk/approve-filtered`) that reprocesses messages matching a
filter. An operator who wants the effect described above edits the rule,
then separately runs a filtered bulk-approve, the two steps are not wired
into a single guided action today.

## Consequences

### Positive
- Operators see all rejected data
- Recovery path for false positives
- Validation rules can be updated without data loss
- Clear separation: raw → validate → clean

### Negative
- DLQ can grow large if validation is too strict
- Operator must review (manual step)

### Mitigations
- DLQ retention limit: 7 days, enforced at two layers, the gateway-local `dlq.*` Kafka topics carry a `retention.ms` backstop (`gateway_runtime/kafka_manager.py`, independent of ADR-009's priority-eviction exemption for these topics), and the control-plane `dlq_messages` table is purged of rows older than the window by a periodic maintenance pass (`app/core/dlq_maintenance.py`) regardless of review status
- Alerts when DLQ grows beyond threshold: the same maintenance pass raises a per-gateway `Alarm` (`alarm_type = "dlq.backlog_high"`) when PENDING message count exceeds a configurable threshold (default 500), and clears it once the backlog is worked back down
- Bulk actions for common patterns

## Related Decisions
- [ADR-009: Overflow Handling](ADR-009-overflow-handling.md)
