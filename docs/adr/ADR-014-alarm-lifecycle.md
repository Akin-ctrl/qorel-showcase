# ADR-014: Alarm Lifecycle

**Status**: Accepted, corrected 2026-07-08  
**Date**: 2026-06-30  
**Last Updated**: 2026-07-08  
**Decision**: Four-state lifecycle (ACTIVE → ACKNOWLEDGED → SUPPRESSED → CLEARED); two distinct trigger paths; delivery via alert_router sink

---

## Context

An alarm represents a detected deviation from expected operating conditions at an industrial asset. The system must define:

1. What states an alarm can be in and what transitions are allowed
2. How alarms are created (trigger paths)
3. How alarms are delivered to operators (notification delivery)
4. How delivery is escalated when alarms are not acknowledged

The alarm model must be simple enough for operators to understand at a glance but expressive enough to capture the operational reality of industrial environments where alarms can persist across shifts and require acknowledgement by specific personnel.

## Options Considered

### State machine complexity

**Option A: Two states (ACTIVE / CLEARED)**, minimal, mirrors a simple boolean alert  
**Cons**: Operators cannot record that they have seen an alarm without clearing it; no concept of deliberate suppression

**Option B: Four states (ACTIVE / ACKNOWLEDGED / SUPPRESSED / CLEARED)**, matches IEC 62443 and ISA-18.2 alarm management recommendations  
**Pros**: Acknowledged = "I have seen this and am investigating"; Suppressed = "I know about this and am deliberately silencing it for now"; Cleared = "condition resolved"  
**Cons**: Slightly more state to manage in the UI

**Option C: Full ISA-18.2 state machine** (UNACKNOWLEDGED, ACKNOWLEDGED, RETURNED-TO-NORMAL, SHELVED, SUPPRESSED-BY-DESIGN...)  
**Pros**: Complete standards compliance  
**Cons**: Far more complexity than the current deployment scale requires; can be adopted incrementally later

### Trigger paths

**Option A: Single path, all alarms from stream evaluation**  
A future stream evaluator is the only source of `Alarm` records.  
**Cons**: Blocks alarm creation until the stream evaluator is built; operator has no alarm visibility from deployment validation rules today

**Option B: Two paths, deployment validation rules + alerting rules**  
The gateway runtime validator (which already runs for every deployment) writes `Alarm` records directly when a validation rule is breached. Alerting rules (ADR-012) provide a second, future trigger path via a stream evaluator.  
**Pros**: Operators get alarm records today from the validator; the model is consistent regardless of which path raised the alarm

## Decision

**Option B for both:** Four-state lifecycle; two trigger paths

## Alarm States

```
ACTIVE ──────────────► ACKNOWLEDGED ──────────────► CLEARED
   │                        │                           ▲
   │                        ▼                           │
   └───────────────► SUPPRESSED ───────────────────────►┘
```

| State | Meaning | Who transitions |
|---|---|---|
| ACTIVE | Condition detected, no operator action yet | System (validator / evaluator) |
| ACKNOWLEDGED | Operator has seen the alarm and is investigating | Operator via UI |
| SUPPRESSED | Operator has deliberately silenced this alarm | Operator via UI |
| CLEARED | Condition resolved or manually cleared | System or operator |

Transitions must be recorded with actor (`acked_by`, `suppressed_by`) and timestamp (`acked_at`, `suppressed_at`, `cleared_at`) for auditability.

## Trigger Paths

### Path 1: Deployment Validation Rules, two sub-paths, both now write Alarm rows

Deployment validation rules cover range, rate-of-change, gap-detection, and
"alarm rules" checks configured in the deployment form.

- **Range / rate-of-change / gap-detection breaches**: the validator detects
  these and writes a DLQ entry (`reason` describing the breach) as before,
  **and now also mirrors the same breach into the alarms table**
  (`ValidatorModule._process_validation_breach_alarm`, `validator.py`),
  closing the gap tracked as remediation ledger item B6. `alarm_type` is
  `range_exceeded` (severity `HIGH`), `rate_of_change` (`MEDIUM`), or
  `gap_detected` (`LOW`); the alarm is keyed by `(asset_id, parameter,
  alarm_type)` and auto-clears (state `CLEARED`) the next time that
  parameter reports a clean reading, mirroring the same active/clear
  lifecycle the alarm-rules sub-path already used. An operator working from
  `AlarmsPage` now sees these directly; the DLQ entry still exists too, as a
  separate record with the raw payload for reprocessing.
- **"Alarm rules" (threshold-style, configured per-deployment)**: these
  write an `Alarm` row via `POST /api/v1/alarms` (gateway-to-control-plane
  call, authenticated with gateway JWT), visible in `AlarmsPage` immediately.
  This sub-path was already operational.

### Path 2: Alerting Rules, operational, evaluated in-process (amended from "planned stream evaluator")

Per the ADR-012 amendment, this path no longer waits on a future stream
evaluator:

1. Operator creates an `AlertingRule` via `AlertingRulesPage` (`parameter`,
   `operator`, `threshold`, `on_delay_seconds`, `deadband`, `severity`,
   `escalation_policy_id`) and attaches it to one or more deployments.
2. The rule ships to the gateway inside the deployment's config payload
   (the same cached-config/hot-reload path used for everything else).
3. The gateway runtime validator evaluates it in-process against incoming
   telemetry; on a sustained breach (respecting `on_delay_seconds` and
   `deadband`), it writes an `Alarm` row the same way Path 1's alarm-rules
   sub-path does, both now go through the same mechanism.
4. **Correction**: the `alarm_type` field is **not** a fixed sentinel. ADR-012
   originally proposed `alarm_type = 'threshold'` as a distinguishing value;
   the shipped model instead makes `alarm_type` a unique, operator-defined
   string set when the rule is created. Path 1 (alarm-rules sub-path) and
   Path 2 alarms are therefore **not** distinguishable by a fixed
   `alarm_type` pattern, they share one evaluation mechanism and one free-
   text identifier space.

This path is **operational today**.

## Notification Delivery

The `alert_router` sink (ADR-005) is the delivery mechanism for alarm notifications:

1. alert_router container subscribes to `alarms.raw` Kafka topic
2. Gateway runtime publishes each new `Alarm` record to `alarms.raw`, carrying the alarm-rule's `escalation_policy_id` if one is bound (`gateway_runtime/validator.py`, `_alarm_payload`)
3. If `escalation_policy_id` is present, alert_router calls the gateway-authenticated `GET /api/v1/escalation-policies/{policy_id}/resolve` endpoint (includes `channel_url`, which is never exposed to user-session-authenticated routes, see ADR-013)
4. alert_router walks the resolved policy steps in array order: `step_type: immediate` delivers right away; `step_type: escalate` is scheduled after `delay_minutes`
5. Before firing a scheduled `escalate` step, alert_router checks `GET /api/v1/alarms/{alarm_id}/gateway-view` (also gateway-authenticated) and skips delivery if the alarm is no longer `ACTIVE`, this is the only way alert_router can learn an operator acknowledged/cleared/suppressed the alarm, since the gateway's local Kafka stream never receives that event back from the Control Plane
6. Alarms with no `escalation_policy_id` bound (e.g. resource-pressure alarms, or alarm-rules with no policy configured) fall back to the sink's statically configured webhook/Slack destination, unchanged from before

Implemented, closed remediation ledger item B5. Requires the alert_router
container to be started with `CONTROL_PLANE_URL`/`CONTROL_PLANE_TOKEN`
(injected by `gateway_runtime.sink_manager` from the gateway's own
control-plane connection); without it, policy resolution is skipped and
every alarm uses the static fallback, matching pre-B5 behavior.

## Resource Pressure Alarms (System-Generated)

The gateway heartbeat endpoint (`POST /api/v1/gateways/{id}/heartbeat`) checks for high CPU or memory utilisation in the heartbeat payload. When CPU > 90% or memory > 90%, the control plane generates an `Alarm` with:
- `alarm_type = 'gateway.high_cpu'` or `'gateway.high_memory'` (namespaced by resource, not a flat `resource_pressure` value)
- `severity = 'critical'` (lowercase, matching the rest of the severity values this code path emits)
- `asset_id = gateway_id`
- `message` describing the resource and current value

These alarms are visible in `AlarmsPage` alongside data-quality alarms.

## Data Model Reference

```
alarms
├── alarm_id        VARCHAR(128) PK
├── gateway_id      VARCHAR(128)
├── asset_id        VARCHAR(128)
├── alarm_type      VARCHAR(128)   -- operator-defined for alerting/alarm rules (not a fixed sentinel); "gateway.high_cpu" / "gateway.high_memory" for resource pressure; "range_exceeded" / "rate_of_change" / "gap_detected" for validation breaches (Path 1).
├── severity        VARCHAR(16)    -- "LOW", "MEDIUM", "HIGH", "CRITICAL"
├── state           VARCHAR(32)    -- "ACTIVE", "ACKNOWLEDGED", "SUPPRESSED", "CLEARED"
├── classification  VARCHAR(32)    -- "ALARM" | "WARNING" | "EVENT"
├── message         VARCHAR(1024)
├── value           FLOAT nullable
├── threshold       FLOAT nullable
├── unit            VARCHAR(64) nullable
├── metadata        JSON nullable
├── raised_at       TIMESTAMPTZ
├── acked_at        TIMESTAMPTZ nullable
├── acked_by        VARCHAR(128) nullable
├── cleared_at      TIMESTAMPTZ nullable
├── suppressed_at   TIMESTAMPTZ nullable
├── suppressed_by   VARCHAR(128) nullable
├── duration_seconds INTEGER nullable
├── created_at      TIMESTAMPTZ
└── updated_at      TIMESTAMPTZ
```

## Consequences

### Positive
- Operators have alarm visibility today from both the alarm-rules sub-path and alerting rules, neither waited on a separate stream evaluator, per the ADR-012 amendment
- Four-state lifecycle gives operators the vocabulary they need for shift handover and investigation
- All transitions are attributed and timestamped, satisfying industrial audit requirements
- Alarm-rules and alerting-rule alarms are now raised by the same in-process mechanism, which is more consistent than originally designed (two separate future paths never had to be built)

### Negative
- The `alert_router` sink still delivers to a static webhook; step-based escalation is the one remaining implementation gap from this ADR set (tracked as item B5)
- DLQ messages and alarms are separate records for the same breach: range/rate/gap validation failures now exist as both a DLQ entry (raw payload, for reprocessing) and an `Alarm` row (for operator visibility/lifecycle), not a single unified record, this is an intentional trade-off (each serves a different workflow), not a gap

## Related Decisions
- [ADR-004: Validation and DLQ Workflow](ADR-004-validation-dlq.md)
- [ADR-005: Sink Architecture](ADR-005-sink-architecture.md)
- [ADR-012: Alerting Rules](ADR-012-alerting-rules.md)
- [ADR-013: Escalation Policies](ADR-013-escalation-policies.md)
