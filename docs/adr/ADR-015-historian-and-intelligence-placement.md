# ADR-015: Historian Placement and Intelligence Locality

**Status**: Accepted
**Date**: 2026-07-27
**Decision**: The bundled edge historian is a first-class Qorel-owned component, distinct from the sink layer. The control plane stores operational data and never telemetry. Site intelligence is computed at the edge; fleet intelligence composes edge-computed findings.

---

## Context

Two questions had to be answered together before Phase 6 (AI / copilot) design
could start, because the answer to each constrains the other:

1. Where does telemetry history live, and who reads it?
2. Where is intelligence computed, and what crosses the network?

### What the scaling roadmap assumed

The internal scaling roadmap collapsed Phase 6's entire federation design, a proposed
context-router, memory-federation layer, and agent-coordinator, on the reasoning
that fleet-scope queries are not a fan-out problem but a filtered query against
data already in one place:

> "Aggregated telemetry already centralizes there… 'Regional' or 'fleet' scope
> for an AI query is not a fan-out-and-aggregate problem across independent
> nodes, it's a filtered query (`WHERE site_code IN (...)`) against data that's
> already in one place."

That premise rested on a queued design in which gateways POST aggregates to a
control-plane API. **That design was never built.** What shipped instead reads
telemetry from the destination configured on a TimescaleDB *sink*, by DSN.

### What is actually shipped

- The gateway aggregator (`gateway_runtime/aggregator.py`) emits `telemetry.1s`
  and `telemetry.1min` windows to gateway-local topics. Resolutions are
  config-driven, not hardcoded.
- The TimescaleDB sink writes those to a database, creating a hypertable where
  the extension is available.
- `control-plane/app/core/telemetry_reads.py` reads them back. It is
  **DSN-driven and deliberately topology-agnostic**, it groups read sources by
  DSN and opens a connection pool per DSN.
- The production edge compose bundles `timescale/timescaledb` on the edge host
  and describes it as the "onboard historian."

### The defect this combination produces

`grep -rn "historian"` across `control-plane/app` and `gateway_runtime` returns
**zero hits**. There is no first-class historian concept anywhere in the code;
the word exists only in a compose comment.

Instead, `_load_timescaledb_sources` powers every operator history view from:

```python
select(Sink).where(Sink.sink_type == "timescaledb").where(Sink.status == "active")
```

A **Qorel product capability is wired to a customer integration choice**. Three
concrete consequences, all reachable through supported configuration:

1. A customer deactivates or deletes their TimescaleDB sink and **operator
   history views go blank**, although the data is still in the edge historian.
2. A customer points that sink at central cloud Timescale, a supported and
   reasonable thing to do, and the operator UI reads **the cloud, not the edge
   historian**, which becomes invisible to the product that ships it.
3. A customer runs both an edge and a cloud TimescaleDB sink and both are
   returned and **merged**, mixing or double-counting rows.

Case 3 is the root cause of the table-collision incident during the live hardware
E2E (2026-07-15), in which two independently created sinks resolved to the same
table on the same host and corrupted it. That was diagnosed at the time as a
missing uniqueness validation; it is more accurately a coupling defect.

### The distinction that was being missed

Qorel **ships TimescaleDB as the default edge historian**. That is a product
component, always present. Sinks are a *separate* concern: customer-directed
egress to wherever the customer wants a copy, their own edge historian, their
central cloud store, Kafka, S3, Azure, a cloud Timescale, covering one site or
many.

Conflating the two is what produced both the defect above and the incorrect
premise in the scaling roadmap.

## Options Considered

### Option A: Push aggregates to a central control-plane store

Gateways POST `1min` and coarser aggregates to a control-plane API; a central
hypertable becomes the fleet-query surface. This was the queued design and the
literal reading of the scaling roadmap's premise.

**Pros**: Fleet queries become a single query against one store; the control
plane stops needing reachability to edge databases; scope-gating by resolved
query width becomes mechanically simple.

**Cons**: Makes the control plane a telemetry historian, breaking the
control-plane/data-plane separation the product is built on (README Core
Principle 2, ADR-006). Duplicates what the sink layer already provides. Adds a
central storage sizing and retention problem on customer-owned hardware.

### Option B: Keep the sink-DSN read path and design Phase 6 for partial results

Accept the shipped behaviour; treat fleet queries as best-effort across whichever
sites answer.

**Pros**: No migration work.

**Cons**: Leaves the historian/sink coupling defect in place, including the
blank-history and merged-source failure modes. Makes every fleet-wide result
silently incomplete when a site is offline, and offline sites are the target
market's normal operating condition, not an exception.

### Option C: First-class edge historian; sinks are egress only

Make the bundled historian a Qorel-owned concept in code, distinct from the sink
layer. Operator reads and site intelligence read *the historian*. Sinks revert to
pure egress and stop backing product reads.

**Pros**: Fixes the coupling defect at the root, including the E2E table
collision. Gives site intelligence a guaranteed substrate, since the historian
ships by default rather than depending on customer configuration. Preserves
control-plane/data-plane separation. Keeps central storage a customer choice
served by the mechanism that already exists.

**Cons**: Fleet-wide telemetry-depth analysis is no longer a single central
query; it requires edge-computed findings, or a customer-provided central store.

## Decision

**Option C.**

1. **The bundled edge historian becomes first-class.** A Qorel-owned component,
   present by default, addressed as a historian in code rather than discovered by
   scanning sink rows. Single hypertable with a `resolution_seconds` column.
2. **Sinks are customer egress only.** They stop being scavenged as read sources.
   A customer's sink configuration can no longer disable, redirect, or
   double-count a Qorel product capability.
3. **The control plane stores operational data and never telemetry.** Gateways,
   adapters, sinks, deployments, alarms, DLQ, audit, health, and, new, edge
   findings. Telemetry does not centralise.
4. **Site intelligence is computed at the edge**, against the edge historian.
   Guaranteed substrate; no dependency on customer configuration or on
   connectivity to the control plane.
5. **Fleet intelligence composes.** It reasons over centralised *operational*
   data plus **edge-computed findings**, scores, detected patterns, conclusions,
   reported up the existing operational channel. The conclusion travels; the
   evidence does not.
6. **Central storage remains a customer choice, served by sinks.** Customers
   wanting a central store configure one. New sink types (S3, warehouse,
   PostgreSQL) are the correct place for that convenience, additive under
   ADR-005.

## Access mechanism

*Added 2026-07-27, after the decision above exposed the gap.*

Removing sink-scavenging leaves an open question the decision above does not
answer: the `Gateway` model carries no field the control plane could use to reach
an edge historian, only `hostname` (informational, captured at enrollment) and
`hardware_info`. Today the sink's `db_dsn` is the *only* thing telling the control
plane how to reach edge data.

**Decision: the gateway serves historian reads over the existing poll-based RPC
channel.** The control plane records a query; the gateway polls for pending work,
executes it locally against its own historian, and posts results back. This is
the same mechanism already shipped and field-tested for gateway connection tests
(`gateway_connection_tests.py`), which runs on its own dedicated loop,
`GATEWAY_CONNECTION_TEST_INTERVAL`, default 5s, independent of the 30s config
poll.

**Why this over a Qorel-owned historian DSN per gateway:**

- No inbound reachability to the edge host. Gateways stay outbound-only, matching
  the network model described in `README.md` and the NAT'd reality of field
  sites.
- The control plane never holds edge database credentials, removing a
  credential blast radius that grows linearly with fleet size.
- The gateway remains the authority over its own data, consistent with ADR-006.

**Cost, accepted:** up to one poll interval (~5s default, tunable) of added
latency on history views. Acceptable for history and analytics surfaces; not
suitable for live-tailing, which should continue to use the existing log/heartbeat
paths rather than this channel.

### Constraint: structured query specs, never SQL

The query payload **must** be a bounded, structured specification, resolution,
time range, asset/parameter filter, limit, validated gateway-side before
execution. The control plane must never be able to send SQL for a gateway to
execute. Doing so would turn this channel into remote SQL execution on every
field device, which is unacceptable regardless of who holds control-plane
credentials.

### Scope: what the historian stores

*Added 2026-07-27.*

**Events and aggregates. Not raw telemetry.**

`telemetry_reads.py` exposes exactly two public read paths, `read_event_records`
and `read_aggregate_records`, called only from `routers/events.py` and
`routers/aggregates.py`. **There is no raw-telemetry read path in the control
plane at all.** (`_ensure_telemetry_table` exists in the TimescaleDB *sink*, but
that is customer egress; nothing in the control plane queries it.)

Storing raw clean telemetry in the historian would therefore be building storage
nothing reads, on the most disk-constrained hardware in the system, frequently a
Raspberry Pi on an SD card.

**Raw data does not become unavailable.** It remains at the edge in Redpanda
under its configured retention and the ADR-009 overflow policy, which is already
the designed buffer for exactly this. Site intelligence needing raw signal
consumes the topic directly at the edge, cheaper than a second copy in
PostgreSQL, and the correct shape for edge-local computation.

### Retention

Per-resolution, with operator-configurable defaults. Aggregate volume differs by
roughly 60× between `1s` and `1min`, so a single window either wastes storage on
high-resolution data or discards coarse history far earlier than necessary.

Defaults: events and `1min`-and-coarser retained for months; `1s` retained for
days. Exposed as gateway configuration, following the pattern already established
by `QOREL_DLQ_RETENTION_DAYS` and its Platform Settings control.

### Invariant: sink egress capability is unchanged

**Narrowing sinks is not a consequence of this decision and must not become one.**
Decoupling reads from sinks changes only what backs *operator views and site
intelligence*. The sink write path is untouched.

Sinks remain full multi-format egress. Verified against the current
implementation:

| Kind | Path |
|---|---|
| Telemetry | `_ensure_telemetry_table` (`sink_timescaledb`), plus `kafka` / `http` |
| Aggregates | `_ensure_aggregate_table`, `message_format: aggregate` |
| Events | `_ensure_event_table`, `event.avsc` |
| Alarms | `alert_router`, with escalation-policy step-walking |

`message_format` (default `auto`) is exposed on the `timescaledb`, `kafka`, and
`http` contracts and stays that way.

Specifically, H1b **must not**:

- change which topics sinks consume, or their `source_topic` semantics
- reduce or gate the `message_format` options
- limit how many sinks a deployment may carry, or which destinations they reach
- make the historian a prerequisite for any sink to function

A customer must remain able to forward every kind of data, telemetry, events,
aggregates, alarms, to their own edge historian, their central cloud store,
Kafka, S3, or anywhere else, for one site or many, exactly as before. The
historian serves Qorel's own read surface; it is not a chokepoint on the
customer's data.

### What this does not fix

RPC fan-out has the **same offline behaviour as the DSN fan-out it replaces**: a
query spanning N gateways issues N requests, and an offline site returns nothing.
Multi-gateway views therefore need explicit partial-result handling in the UI,
showing which sources answered and which did not, rather than silently rendering
an incomplete total.

This is inherent to edge-first. Data held at the edge is unavailable when the
edge is unreachable, under any mechanism short of keeping central copies, which
this ADR deliberately rejects. The benefits claimed above are the reachability,
credential, and coupling fixes, not availability.

## Consequences

### Supersedes

- The scaling roadmap's cross-cutting decision that "aggregated telemetry
  already centralizes." It does not, and under this decision it deliberately will
  not. **Operational** data centralises; telemetry does not.
- The queued-historian direction "gateway POSTs aggregates to the control-plane
  API." Dropped, for the Core Principle 2 reason. The other half of that note,
  bundled TimescaleDB, single hypertable, `resolution_seconds` column, is
  retained and becomes the work item.

### Phase 6 impact

The federation collapse that the scaling roadmap argued for still holds, but for a
different reason than stated. It is not that telemetry sits in one queryable
store, it is that the data Phase 6 tools actually reason over (gateway state,
alarms, DLQ patterns, config, audit, edge findings) is **operational data that
already centralises**. Most of ADR-010's proposed tool surface, `list_gateways`,
`get_alarms`, `diagnose_gateway`, `analyze_dlq`, needs no telemetry access at
all.

Telemetry-depth reasoning is served by edge-computed findings, or, where a
customer has directed a sink at a central store, by that store. Qorel exposes
capability; it does not become the warehouse. This is consistent with ADR-010's
tools-first stance that the customer's own agent performs the reasoning.

### Work items

Tracked in the project's internal pre-AI roadmap document (not included in this
showcase):

- **H1a** (Stage 1), introduce the first-class edge historian
- **H1b** (Stage 1), repoint `telemetry_reads.py` at the historian rather than
  `select(Sink)`; closes **B2b** at the root
- **H1′** (Stage 2), correct the scaling-roadmap premise and the queued-historian
  note
- **E2′** (Stage 4), document the connectivity precondition for the
  edge-local-historian topology: a remote control plane needs a network path to
  the edge host. This is a deployment requirement, not a defect,
  `telemetry_reads.py` is topology-agnostic by design and works unchanged
  wherever the historian is placed.

Stage 5 loses its central-hypertable migration entirely as a result of this
decision.

### Correction to an earlier finding

The audit of 2026-07-27 initially framed the DSN read path as contradicting the
outbound-only network model described in `README.md`. That was half wrong.
`telemetry_reads.py` is deliberately topology-agnostic; the inbound-reachability
requirement follows from one *topology choice* (edge-local historian plus a
remote control plane), not from the code. It is a documentation item (E2′), not a
defect. The real defect is the historian/sink coupling recorded above.

## Related

- [ADR-001](ADR-001-edge-buffering.md), edge buffering; raw telemetry never
  leaves the gateway. Unchanged by this decision.
- [ADR-005](ADR-005-sink-architecture.md), sink architecture; this ADR narrows
  sinks to egress only.
- [ADR-006](ADR-006-gateway-autonomy.md), gateway autonomy.
- [ADR-010](ADR-010-copilot-mcp.md), tools-first copilot direction; this ADR
  establishes the data surface those tools reason over.
- The project's internal pre-AI roadmap and licensing decision records track the
  surrounding Stage 0 structural work (not included in this showcase).
