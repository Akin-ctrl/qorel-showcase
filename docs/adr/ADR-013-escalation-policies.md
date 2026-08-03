# ADR-013: Escalation Policy Structure and Rule Binding

**Status**: Accepted, corrected 2026-07-08  
**Date**: 2026-06-30  
**Last Updated**: 2026-07-08  
**Decision**: JSON steps column for policy flexibility; FK binding rules to policies via escalation_policy_id

---

## Context

When an alarm fires, the system needs to know *who to notify* and *in what order*. An escalation policy is an ordered list of notification steps, each targeting a channel (Slack webhook, email, PagerDuty endpoint) with an optional delay before escalating to the next step if the alarm remains unacknowledged.

Two design questions must be answered:
1. How should policy steps be stored, as a typed relational table or as a flexible JSON column?
2. How should an alerting rule reference its escalation policy, via a runtime lookup, a UI-side join, or a database FK?

## Options Considered

### Step storage

**Option A: Relational step table** (`escalation_steps` with `policy_id` FK, `step_order`, `target_type`, `target_uri`, `delay_minutes`)

**Pros**: Strongly typed, queryable by step target  
**Cons**: Every policy change requires multiple row mutations; step ordering requires an extra `step_order` column and careful update logic; migration cost when step schema evolves

**Option B: JSON column on the policy row**

Steps stored as an ordered JSON array on `escalation_policies.steps`. Each element is a plain object with fields `step_type`, `target_label`, `channels`, and `delay_minutes`.

**Pros**: Schema flexibility, a new step type requires only a frontend change and validation logic update, not a migration; atomic policy updates (one row write); consistent with the `deployment.config` and `deployment.validation_config` precedent already in the codebase  
**Cons**: Cannot query by step field without JSON operators; step schema must be validated in application code, not at DB level

### Rule-to-policy binding

**Option A: No FK, operator configures both independently, alert_router resolves at runtime**

The alert_router sink looks up the policy by name or ID from its own configuration rather than from the rule.

**Pros**: Loose coupling  
**Cons**: No visible link between rule and policy in the UI; easy to misconfigure; policy changes are invisible to rule owners

**Option B: FK on alerting_rule → escalation_policy**

`alerting_rules.escalation_policy_id` is a nullable VARCHAR FK to `escalation_policies.policy_id` ON DELETE SET NULL. A rule with a non-null `escalation_policy_id` knows its escalation path explicitly.

**Pros**: Explicit, auditable, UI can show the binding in one place; ON DELETE SET NULL prevents orphaned rules when a policy is deleted  
**Cons**: Adds a migration; policy must be created before the rule can reference it

## Decision

**Option B for both:** JSON steps column + explicit `escalation_policy_id` FK on alerting rules

## Rationale

1. **JSON steps match the platform precedent**: `deployment.validation_config` uses JSON for similarly structured operator-defined nested configuration. Staying consistent reduces cognitive overhead.
2. **Flexibility without migration cost**: Escalation step formats will evolve (new channel types, retry logic, conditional escalation). A JSON column absorbs these changes without schema migrations.
3. **Explicit FK is operator-friendly**: Operators should not have to mentally track which policy applies to which rule. An explicit FK displayed in the UI makes the relationship unambiguous and auditable.
4. **ON DELETE SET NULL is safe**: If an operator deletes a policy, the rules that referenced it become un-bound (policy_id = NULL) rather than being deleted or raising an error. The operator gets an incomplete-config warning, not a data loss event.

## Data Model

```
escalation_policies
├── policy_id   VARCHAR(128) PK
├── name        VARCHAR(256)
├── tier        VARCHAR(32) default "DEFAULT"   -- e.g. "L1", "L2", "CRITICAL"
├── steps       JSON         -- ordered array of PolicyStep objects
├── created_by  VARCHAR(128) nullable
├── created_at  TIMESTAMPTZ
└── updated_at  TIMESTAMPTZ

alerting_rules (addition)
└── escalation_policy_id  VARCHAR(128) nullable FK → escalation_policies(policy_id) ON DELETE SET NULL
```

### PolicyStep schema (enforced in application layer)

The schema below is the one actually implemented and validated. It differs
from the original proposal on both axes: `step_type` describes *escalation
timing behavior* rather than a channel/protocol, and channel typing is
narrower than first proposed.

```json
{
  "step_type": "immediate | escalate",
  "name": "Step name, required",
  "target_label": "Human-readable description, e.g. 'Ops Slack #alerts'",
  "channel_type": "slack | webhook",
  "channel_url": "<url>, write-only, never returned by the read API",
  "channel_display_label": "Human-readable channel label shown in the UI",
  "delay_minutes": 0
}
```

`step_type: immediate` fires as soon as the alarm is raised; `step_type:
escalate` fires after `delay_minutes` has elapsed with the alarm still
unacknowledged. `channel_type` is currently `slack` or `webhook` only,
`email` and `pagerduty` from the original proposal are not implemented as
channel types. `channel_url` is intentionally write-only: it is accepted on
create/update but never included in read responses, so operators viewing a
policy see `channel_display_label` rather than the raw destination URL.
Steps are evaluated in array order.

## Step Validation

Step validation has shipped. `EscalationPolicyStepCreate` (Pydantic) enforces
`step_type` against `^(immediate|escalate)$`, `channel_type` against
`^(slack|webhook)$`, and requires `name` and `channel_url` to be non-empty,
invalid step objects are rejected at the API layer with a 422, not persisted
silently. The originally-planned "Phase 3 Wave 4" caveat no longer applies.

## Consequences

### Positive
- Operators can define multi-tier escalation paths with any combination of targets
- Policy step format can evolve without schema migrations
- The rule→policy binding is explicit, visible in the UI, and auditable
- Deletion safety: ON DELETE SET NULL prevents cascading data loss

### Negative
- Step schema is not enforced at the DB level; application-layer validation must be rigorous (it now is, see Step Validation above)
- Channel types are narrower than originally scoped: only `slack` and `webhook` are implemented; `email` and `pagerduty` would need new delivery-integration work if needed later
- The `alert_router` sink now resolves an alarm's bound policy (via the gateway-authenticated `GET /api/v1/escalation-policies/{policy_id}/resolve` endpoint, which includes `channel_url`) and walks its steps: `immediate` steps deliver right away, `escalate` steps are scheduled after `delay_minutes` and skipped if the alarm was acknowledged/cleared/suppressed in the meantime (checked via `GET /api/v1/alarms/{alarm_id}/gateway-view`). Closed remediation ledger item B5. Alarms with no bound policy still use the static webhook/Slack config, unchanged.

## Related Decisions
- [ADR-005: Sink Architecture](ADR-005-sink-architecture.md)
- [ADR-012: Alerting Rules](ADR-012-alerting-rules.md)
- [ADR-014: Alarm Lifecycle](ADR-014-alarm-lifecycle.md)
