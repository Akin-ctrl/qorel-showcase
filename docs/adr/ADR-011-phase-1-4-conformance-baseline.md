# ADR-011: Phase 1-4 Architecture Conformance Baseline

**Status**: Accepted  
**Date**: 2026-03-28  
**Decision**: Adopt a formal Phase 1-4 conformance baseline and use it as the source of truth for remediation planning.

---

## Context

During Phase 4 implementation work, architecture drift concerns were raised. A full documentation and code audit was performed across:

- Root and system docs (`README.md`, `PROJECT_PHASES.md`, `docs/ARCHITECTURE.md`, `docs/DATA_FLOW.md`, `docs/DEPLOYMENT.md`, `docs/SECURITY.md`)
- ADRs (`ADR-001` to `ADR-010`)
- Runtime, control-plane, adapters, sinks, deploy, and UI implementation

This ADR records what is currently aligned vs omitted for Phases 1-4 and defines a baseline for closing gaps.

Status interpretation note:
- **Baseline Findings** capture the historical audit snapshot from this ADR's adoption moment.
- **Current Status Update** reflects implementation state as of the 2026-05-30 documentation refresh.

---

## Current Status Update (2026-05-30)

### Phase Status

- **Phase 1 (Core Foundation): Completed** for the current committed core scope.
- **Phase 2 (Control Plane API): Substantially implemented** for the operator-driven model, with broader enterprise/OAuth and zero-touch onboarding still outside the completed set.
- **Phase 3 (Validation & Sinks): Completed** for the current committed telemetry, validation, events, aggregates, and sink scope.
- **Phase 4 (UI): Completed and expanded** beyond the original milestone scope (reusable adapters/sinks, deployment composition, validation/test/preflight, events, aggregates, fleet, and logs).

### Open Work Anchors

- Remaining open implementation items are now narrower and are tracked in P2/P3 checkboxes below plus the project's internal roadmap tracker (Cross-Cutting Open Issues, Production Readiness, and UI Product Action List sections).
- ADR-001 has been amended: the original Apache Kafka/KRaft implementation has moved to a Redpanda-first Kafka-compatible local dev/runtime broker path, while production packaging and image-pull templates remain pending.
- Completed issue items are moved to the internal roadmap tracker's resolved-issues ledger (not included in this showcase).

---

## Decision

1. The findings in this ADR are the **official conformance baseline** for Milestones/Phases 1-4.
2. Work planning after this date must reference this ADR when claiming architecture compliance.
3. Any closure of a listed omission must include:
   - code implementation,
   - docs update,
   - and explicit reference back to this ADR section.

---

## Conformance Findings

### Phase 1: Core Foundation

#### Followed
- Modbus TCP adapter implemented and publishing normalized telemetry.
- Local Kafka-compatible broker dependency wired and health-checked by runtime.
- Core schema files exist (`telemetry`, `event`, `alarm`).
- Dev compose stack exists for local demo/development.

#### Omitted / Partial
- Runtime can now provision/supervise the local embedded Kafka-compatible broker when it is not already reachable, but broader lifecycle hardening still applies elsewhere.
- Adapter runtime support is effectively single-protocol (`modbus_tcp`) while architecture describes broader protocol set.

---

### Phase 2: Control Plane API

#### Followed
- Gateway registration, approval, token issue/renew, and gateway config endpoint exist.
- Pipeline CRUD and Sink CRUD exist.
- User auth (JWT) and admin checks are implemented.

#### Omitted / Partial
- Runtime does not fully auto-register itself if missing; token request path assumes pre-registration.
- Control-plane config cache semantics for deterministic offline startup/fallback are not fully implemented as documented autonomy behavior.

---

### Phase 3: Validation & Sinks

#### Followed
- Validator module runs in gateway runtime (not containerized) with quality outcomes and DLQ emission.
- Sink manager lifecycle exists for sink containers.
- TimescaleDB sink implementation exists and writes validated telemetry.

#### Omitted / Partial
- Sink portfolio in architecture/ADR is broader than current implementation.

---

### Phase 4: UI

#### Followed
- Login/auth guard routing is implemented.
- Gateway and Health views are implemented.
- Pipeline Builder flow exists and compiles backend payload.

#### Omitted / Partial
- Wider post-milestone UI roadmap items remain open beyond original Phase 4 scope.

---

### Phase 5: Alerting, Escalation & Alarm Lifecycle (ADR-012 / ADR-013 / ADR-014)

This phase was introduced by ADR-012/013/014 (accepted 2026-06-30), after this
baseline's original adoption date. It was not previously tracked here, that
omission is itself corrected by this section as of 2026-07-08.

#### Followed
- `alerting_rules` and `escalation_policies` tables exist with full CRUD APIs, audited via `record_audit_event()`.
- The `escalation_policy_id` FK (ADR-013) and PolicyStep validation are both implemented, ADR-013's "planned, not yet applied" language has been corrected to reflect this.
- The four-state alarm lifecycle (ACTIVE/ACKNOWLEDGED/SUPPRESSED/CLEARED) is implemented with proper transition guards, exceeding the ADR's own diagram.
- Alerting rules bound to a deployment are evaluated in-process by the gateway validator via the existing cached-config hot-reload path (see ADR-012 amendment), this closes ADR-012's original "evaluation deferred" gap without a separate stream-processing component.
- Resource-pressure alarms (gateway heartbeat CPU/memory >90%) are implemented.
- Range, rate-of-change, and gap-detection validation breaches now also raise `Alarm` rows (`alarm_type`: `range_exceeded`/`rate_of_change`/`gap_detected`) alongside the existing DLQ entry, closed remediation ledger item B6.
- The `alert_router` sink now resolves an alarm's bound escalation policy (gateway-authenticated `GET /api/v1/escalation-policies/{policy_id}/resolve`) and walks its steps, checking current alarm state before firing delayed `escalate` steps (`GET /api/v1/alarms/{alarm_id}/gateway-view`), closed remediation ledger item B5. Falls back to the static webhook when no policy is bound.
- `Alarm` records now persist which escalation policy (if any) routed them (`escalation_policy_id`, `ON DELETE SET NULL`, part of the `0001_init` baseline migration). Previously this only existed in the gateway-local Kafka payload consumed by `alert_router` and was invisible in the control plane; `AlarmsPage` now shows it as a clickable detail row linking to Escalation Policies (closed as part of the same B5 UI-gap follow-up, 2026-07-08).

#### Omitted / Partial
- None remaining for this phase as of this update; all P0 items from the original Phase 5 audit (B1–B6) are closed.
- PolicyStep `channel_type` currently supports `slack` and `webhook` only; `email` and `pagerduty` are not implemented (ADR-013 has been revised to describe the shipped model rather than claim these exist).

---

## ADR-Level Drift Summary

### ADR-001 (Edge Buffering)
- **Aligned for local dev/runtime, pending production packaging**: The local Kafka-compatible stream pattern exists, the dev compose stack uses Redpanda, and the runtime-managed broker path is Redpanda-first. Production packaging, sizing, and field recovery guidance still need hardening.

### ADR-002 (Protocol Adapters)
- **Partial**: Containerized adapter pattern exists and implemented support now covers `modbus_tcp`, `modbus_rtu`, `mqtt`, and `opcua`; broader wireless/field protocols remain open.

### ADR-003 (Schema Management)
- **Aligned**: Avro + Schema Registry + offline schema-cache behavior are implemented in the current runtime/control-plane path.

### ADR-004 (Validation & DLQ)
- **Aligned**: Validation, DLQ routing, and operator DLQ workflows are implemented.

### ADR-005 (Sink Architecture)
- **Partial**: Containerized sink pattern is implemented with `timescaledb`, `kafka`, `http`, and `alert_router`; broader sink breadth remains open.

### ADR-006 (Gateway Autonomy)
- **Aligned**: Cached config semantics now support deterministic startup, offline reuse, and background refresh consistent with the autonomy decision.

### ADR-007 (Authentication)
- **Partial**: Gateway/user JWT implemented; broader enterprise path (OAuth/OIDC, richer RBAC model) is not complete.

### ADR-008 (Failure Modes)
- **Aligned**: Explicit circuit breaker behavior now protects control-plane requests and sink downstream writes, with health visibility and cooldown semantics matching ADR-008.

### ADR-009 (Overflow Handling)
- **Aligned**: Tiered overflow controls (compress/downsample/priority eviction/block) are implemented.

### ADR-010 (Copilot/MCP)
- **Not aligned for Phase 1-4 scope**: No implemented MCP tool surface matching ADR intent. This is intentional per ADR-010's own deferred status, not drift.

### ADR-012 (Alerting Rules)
- **Partial, ADR text corrected**: Storage, CRUD, and audit trail are implemented and aligned. Evaluation is live (in-process via the gateway validator), which is a deliberate amendment to the original "deferred, future stream evaluator" decision, see the ADR-012 amendment note. Data model section has been corrected to match the shipped schema.

### ADR-013 (Escalation Policies)
- **Aligned**: FK binding, step validation, and `alert_router` step-walking (with acknowledgement-aware delayed escalation, remediation ledger item B5, closed) are all implemented. PolicyStep schema has been corrected to describe the shipped `immediate|escalate` / `slack|webhook` model rather than the originally-proposed `webhook|slack|email|pagerduty` typing, `email`/`pagerduty` channel types remain unimplemented if needed later.

### ADR-014 (Alarm Lifecycle)
- **Aligned**: State machine, Path-1 "alarm rules" trigger, range/rate-of-change/gap-detection breaches raising `Alarm` rows (item B6, closed), and `alert_router` escalation-policy step-walking (item B5, closed) are all implemented. Resource-pressure field values and the alarm_type sentinel claim have been corrected to match the shipped code.

---

## Architecture Documentation Drift

There was a documentation inconsistency around central broker language:

- Some architecture text states Qorel manages only the local gateway Kafka-compatible broker.
- Some data-flow/deployment sections still describe central Kafka-compatible patterns in a way that can be interpreted as platform-managed.

Resolved direction as of the original baseline:

- Qorel owns only the gateway-local stream on the edge device.
- Any cloud Kafka-compatible broker mentioned in deployment patterns is an optional customer-owned sink target, not a Qorel-managed central backbone.

Amended direction as of 2026-05-30:

- Qorel owns only the gateway-local Kafka-compatible broker.
- Redpanda is the chosen embedded edge broker direction.
- The local dev/runtime path is Redpanda-backed; production packaging remains pending.
- Kafka-compatible topics, producers, consumers, partitions, offsets, and consumer-group semantics remain part of the platform contract.
- External Kafka-compatible systems remain optional customer-owned sink destinations.

---

## Additional Implementation Hardening Gaps (Cross-Review Addendum)

The following open implementation-level gaps remain attached to this baseline for remediation tracking:

The hardening issues originally tracked here are now resolved and have been moved to the internal roadmap tracker's resolved-issues ledger (not included in this showcase).

---

## Remediation To-Do List (Execution Order)

The checklist below converts the findings in this ADR into a practical fix sequence. Items are grouped by suggested priority and ordered so foundational blockers are addressed before broader architecture-compliance work.

### P1: Immediate Scope, Security, and Operator Readiness

- [x] Implement backend alarm endpoints required to support Phase 4 alarm workflows.
- [x] Implement Alarm UI view required by milestone scope.
- [x] Implement operator DLQ workflow in control-plane API: list, inspect, approve/reprocess, discard.
- [x] Implement DLQ UI view for operators.
- [x] Add first-user self-registration/bootstrap flow in API/UI.
- [x] Harden bootstrap/admin credentials so weak defaults are blocked outside explicit dev-only profiles.
- [x] Add targeted token refresh and immediate retry behavior when runtime config fetch fails due to auth expiry/invalid token.
- [x] Replace startup-only table creation assumptions with a migration-managed schema evolution path.
- [x] Reconcile architecture docs describing local-vs-central Kafka-compatible broker ownership to eliminate conflicting platform direction.
- [x] Complete the Redpanda migration across dev compose, runtime broker management, deployment docs, and tests while retaining Kafka-compatible client semantics.
- [x] Ship a production compose packaging path for the Redpanda-backed runtime (split 2026-07-10 into `deploy/prod/docker-compose.control-plane.yml` + `docker-compose.edge.yml` for a two-host deployment: pinned Redpanda image, TLS-terminating nginx, network isolation, versioned `qorel/*` images) plus a `Makefile` build/push/cross-build flow and a Production Deployment section in `docs/DEPLOYMENT.md`.
- [ ] Complete registry-publishing and image-pull-secret templates for orchestrators beyond compose (e.g. Kubernetes), this is the only piece of the original packaging item still open.

### P2: Runtime Reliability and Contract Enforcement

- [x] Enforce `BaseAdapter` lifecycle structurally with a concrete template run loop/shared contract instead of documentation-only hooks.
- [x] Centralize Kafka-compatible publishing behavior so delivery semantics, error handling, and observability are consistent across adapters.
- [x] Improve Modbus adapter polling efficiency with contiguous register batching where possible.
- [x] Add explicit Modbus reconnect/retry/backoff handling for connection and read failures.
- [x] Implement runtime-managed embedded Kafka-compatible broker lifecycle rather than reachability wait-only behavior.
- [x] Amend runtime-managed broker lifecycle to be Redpanda-first or broker-neutral.
- [x] Defer zero-touch gateway auto-registration/bootstrap. Current architecture keeps gateway creation and pipeline/sink composition operator-driven in the UI until multi-adapter/multi-sink onboarding is explicitly designed.
- [x] Implement deterministic cached control-plane config semantics for offline startup/fallback autonomy.
- [x] Expand failure-mode implementation to include explicit circuit-breaker strategy described by ADR-008.
- [x] Enforce manual offset commit semantics in validator so Kafka-compatible consumer offsets are committed only after clean-topic or DLQ publish durability is confirmed.
- [x] Harden sink SQL identifier handling by validating/quoting configured table names before executing DDL/DML.
- [x] Enforce gateway state invariants by reconciling or rejecting conflicting `status`/`approved` updates server-side.

### P3: Architecture Conformance Expansion

- [x] Decide and codify final schema strategy: align implementation/docs on Avro + Schema Registry or formally revise ADR/docs around the current JSON path.
- [x] Implement the chosen schema path consistently, including offline schema-cache behavior if Avro/Registry remains the target architecture.
- [ ] Expand remaining adapter support beyond the current implemented set (`modbus_tcp`, `modbus_rtu`, `mqtt`, `opcua`), or narrow architecture claims if wireless/field protocols are deferred.
- [ ] Expand remaining sink portfolio beyond the current implemented set (`timescaledb`, `kafka`, `http`, `alert_router`), or narrow architecture claims if broader cloud/archive sinks are deferred.
- [x] Implement tiered overflow handling from ADR-009: compress, downsample, priority eviction, and blocking behavior.
- [ ] Complete authentication roadmap gaps for richer RBAC and/or OAuth/OIDC if ADR-007 remains in scope for this phase set.
- [ ] Implement MCP/Copilot tool surface only if ADR-010 remains part of the committed architecture roadmap; otherwise explicitly defer/re-scope it.

### Tracking Rule

- [ ] Each completed item must land with code, docs updates, and an explicit reference back to this ADR in the PR/commit trail.

---

## Consequences

### Positive
- Team now has an explicit, versioned baseline of what is done vs missing.
- Future claims of “architecture complete” can be validated against concrete criteria.

### Negative
- Existing progress narratives may need correction where they over-claimed phase completion.

---

## Required Follow-up

1. Create a remediation backlog mapped to this ADR (P1/P2/P3 priorities).
2. Keep active backlog aligned with current reality (Phase 4 baseline scope already completed).
3. Decide and codify final direction for:
   - schema strategy (Avro+Registry vs current JSON path),
   - gateway autonomy cache semantics,
   - overflow handling implementation.
4. Keep architecture docs aligned on local broker ownership: gateway-local Kafka-compatible broker only; external Kafka-compatible systems only as optional sink destinations.
5. Keep gateway onboarding aligned with the current operator-driven model: the UI/control plane remains the source of truth for gateway records and pipeline composition until richer multi-adapter/multi-sink onboarding is designed.
6. Track and close the "Additional Implementation Hardening Gaps" section with explicit linked PRs.

---

## Related ADRs

- [ADR-001](ADR-001-edge-buffering.md)
- [ADR-002](ADR-002-protocol-adapters.md)
- [ADR-003](ADR-003-schema-management.md)
- [ADR-004](ADR-004-validation-dlq.md)
- [ADR-005](ADR-005-sink-architecture.md)
- [ADR-006](ADR-006-gateway-autonomy.md)
- [ADR-007](ADR-007-authentication.md)
- [ADR-008](ADR-008-failure-modes.md)
- [ADR-009](ADR-009-overflow-handling.md)
- [ADR-010](ADR-010-copilot-mcp.md)
- [ADR-012](ADR-012-alerting-rules.md)
- [ADR-013](ADR-013-escalation-policies.md)
- [ADR-014](ADR-014-alarm-lifecycle.md)
