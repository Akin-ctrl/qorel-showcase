# ADR-012: Alerting Rules Storage and Evaluation

**Status**: Accepted, amended 2026-07-08  
**Date**: 2026-06-30  
**Last Updated**: 2026-07-08  
**Decision**: Store alerting rules as control-plane records; evaluate them in-process on the gateway via the existing cached-config path (amended from the original "defer to a future stream evaluator" decision)

---

## Context

Operators need to define threshold-based conditions that should trigger alarms, for example, "temperature above 80°C for 5 consecutive minutes on pump_skid_01". The system must record these rules, make them visible and editable via the UI, and eventually evaluate them against live telemetry to produce `Alarm` records.

Two concerns are separable: **rule storage** (where and how rules are persisted) and **rule evaluation** (how and when the system checks incoming data against rules).

## Options Considered

### Option A: Rules stored at the gateway runtime; evaluated in-process

Rules are pushed to the gateway as configuration. The gateway runtime evaluates each reading against applicable rules as data flows through the validator module.

**Pros**: Low latency, works offline, no round-trip to control plane  
**Cons**: Rules are not operator-visible in the control plane UI without syncing; hard to update rules without restarting the gateway runtime; duplicated rule logic with the validation-rules configuration in deployments

### Option B: Rules stored in control plane; evaluation by a dedicated stream processor

Rules are persisted in the control-plane database. A separate Kafka Streams or Faust application subscribes to `telemetry.clean`, evaluates each record against active rules, and publishes to an `alarms.raw` topic.

**Pros**: Centralised management, independent scaling of the evaluation engine  
**Cons**: Additional component with its own deployment and failure mode; overkill for the current deployment scale; not yet implemented

### Option C: Rules stored in control plane; evaluation deferred

Rules are persisted in the control-plane PostgreSQL database (`alerting_rules` table). Full CRUD is exposed via `GET/POST/PUT/DELETE /api/v1/alerting-rules`. Evaluation is an explicit future step, rules currently serve as operator-visible policy documents consulted by the alert_router sink and any future evaluator. The `Alarm` table stores runtime alarm instances raised by the gateway validator (via deployment validation rules) or by future rule evaluation.

**Pros**: Consistent data model, immediate operator visibility, no premature component complexity, auditable  
**Cons**: Rules are not evaluated automatically until a stream evaluator is built; operators must currently rely on deployment validation rules for real-time checking

## Decision

**Option C: Control-plane storage.** Evaluation, however, did not end up
deferred to a future stream evaluator, see the Amendment below for what
actually shipped and why.

## Amendment (2026-07-08): Evaluation happens in-process, not via a stream evaluator

When it came time to make alerting rules actually evaluate, the system did
not build the "future stream evaluator" this ADR originally proposed to
defer to (Option B). Instead, `AlertingRule` records are attached directly to
a `Deployment` (via a `deployment_alarm_rules` join table) and are shipped to
the gateway inside the same deployment-config payload the gateway already
polls/hot-reloads for deployment validation rules. The gateway runtime
validator evaluates them in-process, exactly as Option A ("rules stored at
the gateway runtime; evaluated in-process") originally described, the
option this ADR explicitly rejected.

This is a deliberate amendment, not silent drift, recorded here because the
three downsides Option A was rejected for turned out not to apply against
how the rest of the platform had already evolved:

- **"Not operator-visible without syncing"**, doesn't apply. Rules are full
  CRUD with a dedicated UI page (`AlertingRulesPage`), same visibility Option
  C was chosen for.
- **"Hard to update without restarting the gateway runtime"**, doesn't
  apply. The gateway's config-sync/hot-reload path (ADR-006) already diffs
  and reloads `alarm_rules` on every config sync; no restart is needed.
- **"Duplicated rule logic with the validation-rules configuration in
  deployments"**, doesn't apply in practice. `alarm_rules` is evaluated as
  its own distinct rule list by the validator, separate from range/rate-of-
  change/gap-detection validation logic.

In short: storage stayed centralized (the reason Option A was rejected),
while evaluation moved to the edge (the reason Option A was attractive) by
reusing infrastructure ADR-006 had already built, rather than standing up
the dedicated stream-processing component Option B itself called "overkill
for the current deployment scale." **The dedicated stream evaluator
described in Option B is no longer planned.** If a future scale point
genuinely requires independent evaluation-engine scaling, that would need
its own ADR revisiting this decision; it is not presently on the roadmap.

## Rationale

1. **Single source of truth for operator intent**: Alerting rules belong with other operator-managed configuration (adapters, sinks, deployments) in the control plane. Operators should not need to look in multiple places.
2. **Separation of concerns**: Storage and evaluation are independent problems. Solving storage correctly now does not foreclose any evaluation approach later.
3. **Avoids premature architecture**: A dedicated stream evaluation engine introduces a new operational component. It should be introduced when the workload justifies it, not speculatively.
4. **Audit trail**: All rule create/update/delete actions are recorded to `audit_events` via `record_audit_event()`, which is consistent with all other control-plane mutations.

## Data Model

The schema below reflects the shipped `AlertingRule` model, not the original
proposal, every column changed name or shape at least once during
implementation. See the Amendment above for why evaluation-related fields
(`on_delay_seconds`, `deadband`) exist instead of `duration_minutes`.

```
alerting_rules
├── rule_id                   VARCHAR(128) unique, indexed
├── name                      VARCHAR(256)
├── description               VARCHAR(1024) nullable
├── status                    VARCHAR(16) default "enabled"  -- "enabled" | "disabled"
├── parameter                 VARCHAR(255) indexed            -- e.g. "temperature"
├── asset_id                  VARCHAR(255) nullable, indexed  -- scope to one asset
├── adapter_id                VARCHAR(128) nullable, indexed  -- scope to one adapter
├── operator                  VARCHAR(4)                      -- ">", "<", ">=", "<=", "==", "!="
├── threshold                 FLOAT
├── unit                      VARCHAR(64) nullable
├── deadband                  FLOAT nullable
├── on_delay_seconds          INTEGER nullable                -- replaces duration_minutes
├── severity                  VARCHAR(16) default "MEDIUM", indexed
├── alarm_type                VARCHAR(255) unique, indexed    -- operator-defined, not a fixed sentinel
├── active_message             VARCHAR(1024)
├── clear_message              VARCHAR(1024)
├── requires_acknowledgement  BOOLEAN default true
├── escalation_policy_id      VARCHAR(128) nullable FK → escalation_policies(policy_id) ON DELETE SET NULL
├── created_by                VARCHAR(128)
├── updated_by                VARCHAR(128)
├── created_at                TIMESTAMPTZ
└── updated_at                TIMESTAMPTZ
```

`AlertingRule` also joins to `Deployment` via a `deployment_alarm_rules`
association table, a rule is only evaluated once it is attached to at least
one deployment (this is how it reaches the gateway).

## Relationship to Deployment Validation Rules

Deployment validation rules (range, rate-of-change, gap-detection, 
configured per-deployment in `DeploymentCreatePage`) and alerting rules (this
ADR) are both evaluated **inside the gateway runtime**, in-process, per the
Amendment above. They remain distinct rule lists with different trigger
conditions and different current output completeness:

| Trigger path | Configured where | Evaluated where | Output |
|---|---|---|---|
| Deployment validation rule (range/rate/gap) | Per-deployment form | Gateway runtime (in-process) | DLQ message only, does not yet raise an `Alarm` row (tracked gap, see ADR-014) |
| Alerting rule (this ADR) | Alerting Rules page, attached to a deployment | Gateway runtime (in-process, via deployment config sync) | `Alarm` |
| alert_router sink | Sink config | Sink container consuming `alarms.raw` | External delivery |

## Consequences

### Positive
- Alerting rules are immediately manageable via UI and API
- Rule lifecycle is audited
- No new runtime component required, now or planned, evaluation reuses the existing gateway config-sync/validator path
- Rules evaluate automatically once attached to a deployment, with no separate stream-processing component to operate

### Negative
- Deployment validation rules (range/rate/gap) still only produce DLQ entries, not `Alarm` rows, an inconsistency with alerting rules (this ADR), which do raise `Alarm` rows. Tracked as an open gap in ADR-014.
- `escalation_policy_id` FK is implemented (see ADR-013); `alert_router` does not yet resolve or walk the linked policy's steps, tracked as an open gap in ADR-013/014.

## Related Decisions
- [ADR-004: Validation and DLQ Workflow](ADR-004-validation-dlq.md)
- [ADR-005: Sink Architecture](ADR-005-sink-architecture.md)
- [ADR-013: Escalation Policies](ADR-013-escalation-policies.md)
- [ADR-014: Alarm Lifecycle](ADR-014-alarm-lifecycle.md)
