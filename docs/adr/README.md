# Architecture Decision Records

This directory contains Architecture Decision Records (ADRs) for key design decisions in Qorel.

## What is an ADR?

An ADR documents a significant architectural decision, including:
- **Context**: Why the decision was needed
- **Options**: What alternatives were considered
- **Decision**: What was chosen
- **Consequences**: Trade-offs and implications

## ADR Index

| ADR | Title | Status |
|-----|-------|--------|
| [ADR-001](ADR-001-edge-buffering.md) | Edge Buffering Strategy | Accepted, amended |
| [ADR-002](ADR-002-protocol-adapters.md) | Protocol Adapter Architecture | Accepted |
| [ADR-003](ADR-003-schema-management.md) | Schema Management Strategy | Accepted |
| [ADR-004](ADR-004-validation-dlq.md) | Validation and DLQ Workflow | Accepted |
| [ADR-005](ADR-005-sink-architecture.md) | Sink Architecture | Accepted |
| [ADR-006](ADR-006-gateway-autonomy.md) | Gateway Autonomy | Accepted |
| [ADR-007](ADR-007-authentication.md) | Authentication Model | Accepted |
| [ADR-008](ADR-008-failure-modes.md) | Failure Modes and Recovery | Accepted |
| [ADR-009](ADR-009-overflow-handling.md) | Overflow Handling | Accepted |
| [ADR-010](ADR-010-copilot-mcp.md) | Copilot Tools-First Approach | Accepted, deferred |
| [ADR-011](ADR-011-phase-1-4-conformance-baseline.md) | Phase 1-4 Architecture Conformance Baseline | Accepted |
| [ADR-012](ADR-012-alerting-rules.md) | Alerting Rules Storage and Evaluation | Accepted, amended |
| [ADR-013](ADR-013-escalation-policies.md) | Escalation Policy Structure and Rule Binding | Accepted, corrected |
| [ADR-014](ADR-014-alarm-lifecycle.md) | Alarm Lifecycle | Accepted, corrected |

## Key Decisions Summary

### Data Path
- **Edge Buffering**: Embedded Kafka-compatible broker per gateway, with Redpanda as the chosen edge direction and production packaging still pending
- **Serialization**: Avro with Schema Registry
- **Adapters & Sinks**: Docker containers with standard contract
- **Overflow**: Tiered strategy (compress → downsample → evict by priority)

### Architecture
- **Gateway Autonomy**: Full offline capability after first boot
- **Validation**: Module in Gateway Runtime, BAD → DLQ with operator workflow
- **Failure Handling**: Restart + bulkhead isolation + circuit breaker

### Operations
- **Authentication**: JWT for gateways and built-in users; OAuth/OIDC remains later enterprise work
- **AI Copilot**: MCP tools-first direction is deferred until core production-readiness work is closed
- **Conformance Baseline**: Phase 1-4 audit baseline for remediation tracking
- **Alerting, Escalation & Alarm Lifecycle**: rule-based alarm evaluation (ADR-012), escalation policies with step-walking (ADR-013), and the four-state alarm lifecycle ACTIVE/ACKNOWLEDGED/CLEARED/SUPPRESSED (ADR-014), status maintained in [ADR-011 Phase 5](ADR-011-phase-1-4-conformance-baseline.md#phase-5-alerting-escalation--alarm-lifecycle-adr-012--adr-013--adr-014)

Current implementation progress and issue movement are tracked in:
- [ADR-011](ADR-011-phase-1-4-conformance-baseline.md) for open conformance gaps and active checklist items.
- Resolved Issues Ledger for items moved out of active queues after completion.

## Creating New ADRs

When making a significant architectural decision:

1. Copy the template:
   ```bash
   cp ADR-TEMPLATE.md ADR-0XX-short-title.md
   ```

2. Fill in all sections

3. Add to the index above

4. Get team review

## ADR Statuses

| Status | Meaning |
|--------|---------|
| Proposed | Under discussion |
| Accepted | Decision made, implementing |
| Deprecated | Superseded by another ADR |
| Rejected | Considered but not chosen |
