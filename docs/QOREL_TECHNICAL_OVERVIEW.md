# Qorel: Technical Walkthrough

A concrete look at what a Qorel deployment does, how the engineering
underneath it works, and what is actually shipped today. Real numbers and
real scenarios, pulled directly from the system's own design documentation.

Last updated: 2026-07-08

---

## 1. What a Qorel deployment actually looks like

Picture an offshore oil platform fifty kilometers from shore, where the
only connectivity is a satellite or 4G link that goes down roughly a third
of the time. A PLC on the platform monitors pressure and temperature on a
pump skid over Modbus TCP. This is exactly the kind of site Qorel is built
for, and walking through it concretely shows how the pieces fit together.

An engineer configures, through Qorel's web application, a gateway for this
site, an adapter pointed at the PLC's Modbus registers (pressure on one
register, temperature on another, each tagged with its physical unit and
scaling), and a destination, in this case a TimescaleDB instance for
historical storage. That whole configuration, which registers map to which
physical quantities, what to validate, where the clean data should end up,
is sent to the gateway once, and the gateway caches it.

From there, the gateway takes over completely. Every second, it reads the
configured registers, tags each reading with both the moment the sensor
took the measurement and the moment the gateway received it (useful later
for spotting clock drift or network lag), and writes it immediately to its
own local, durable data log, before anything else happens to that reading.
A local validator checks it against a configured plausible range, flags
duplicates, and forwards clean readings onward. A local aggregator computes
rolling one-second and one-minute summaries. All of this happens whether or
not the satellite link is currently up.

Now say a storm knocks out the satellite link. This isn't a hypothetical,
it's the exact scenario this architecture is designed against, and it plays
out like this: the gateway keeps reading the PLC every second, keeps
writing to its local log, and simply keeps buffering, for 41.5 hours in
one documented outage scenario, accumulating roughly 149,400 buffered
messages (120,000 raw telemetry readings, 28,000 discrete events, 1,200
alarm records) and growing local disk usage from 20GB to 52GB. Nothing is
lost. When the link comes back, the gateway automatically drains the
backlog to its destinations at roughly 2,000 messages per second, the
entire 41.5-hour backlog fully synced in about 75 seconds. No operator has
to do anything. No data is missing from the historical record. This is not
a best-case demo scenario; it's the documented, designed-for behavior of
the system.

---

## 2. The engineering underneath: why this works the way it does

### 2.1 A real, replayable log, not a purpose-built buffer

Most tools in this space solve "what if the network is down" with some
version of a retry queue: write locally, retry sending, delete once
delivered. That's a reasonable starting point, but it has a structural
ceiling: it's built for one consumer and one delivery attempt. The moment
a site needs a real-time dashboard *and* a historical database *and* an
alert pipeline all consuming the same stream independently, a simple retry
queue starts needing workarounds bolted onto it.

Qorel instead runs a genuine distributed streaming log on every gateway,
Kafka-compatible, specifically an implementation called Redpanda, chosen
because it keeps the same protocol and client guarantees as the broader
Kafka ecosystem while being lighter to run on constrained edge hardware.
Concretely, this buys real properties a retry queue doesn't have:

- Every reading lands in an ordered, append-only record that isn't removed
  the moment one consumer reads it, it persists until it's explicitly aged
  out, which means multiple independent systems can each read the same
  data at their own pace without interfering with each other.
- Any consumer can rewind and replay from an earlier point in time. If a
  downstream system needs to reprocess a window of history, say, a
  correction to how a value gets interpreted, it replays rather than
  needing a separate backfill process.
- Data is organized and partitioned by the physical asset it came from, so
  throughput scales with the number of independent pieces of equipment
  reporting, rather than being one single serialized stream that everything
  has to funnel through.
- The underlying data contracts (what shape a reading, an event, or an
  alarm takes) are formally versioned, so the format can evolve, new
  fields added, for instance, without breaking anything already consuming
  the older shape.

The honest cost: a real log-based broker needs meaningfully more memory
(roughly half a gigabyte as a floor) than a bare-bones retry queue would.
On any reasonably capable gateway that's a non-issue; on the cheapest
available edge hardware it's a real, acknowledged constraint, and tuning
for that segment is active near-term work.

### 2.2 Full edge autonomy

The gateway needs the central control plane exactly once, the very first
time it powers on, to pull down its configuration. After that:

- It keeps everything it needs cached locally: configuration, the schemas
  it needs to serialize data correctly, and of course the buffered data
  itself.
- Every subsequent restart starts immediately from that cache. There is no
  waiting on permission from anywhere else.
- It checks in the background for configuration updates and applies them
  automatically when it can reach the control plane, a convenience, not a
  dependency.
- It can run fully offline indefinitely. The 41.5-hour outage above is a
  documented test case, not a theoretical ceiling, the actual constraint is
  local disk space, not time.

This is the single most load-bearing decision in the entire product.
Everything else, the tiered storage protection, the fault isolation, the
alarm system, exists to protect this one property: the gateway never stops
working because something external to it failed.

### 2.3 Protecting data when local storage runs out

If a site stays disconnected long enough that local disk starts filling up,
the response is staged, not a single cutoff:

| Disk usage | Response |
|---|---|
| Normal range | No change in behavior. |
| Elevated | Alert raised; older data compressed to reclaim space with minimal information loss. |
| High | Raw, high-frequency readings older than a short window are dropped in favor of the rolling summaries already computed, trend data survives even when individual samples don't. |
| Critical | Oldest data evicted outright, by priority, least operationally important first. |
| Exhausted | New data is refused rather than corrupting what's already safely stored, with a critical alert raised immediately. |

One rule holds at every single stage of this: **alarm and fault data is
never evicted by this mechanism.** Ordinary sensor readings are the
cheapest thing to sacrifice under real storage pressure; safety-relevant
history never is.

### 2.4 Fault isolation, described in terms a controls engineer already trusts

- **Every connection to a piece of equipment, and every connection to a
  destination system, runs in its own isolated, resource-limited process.**
  A misbehaving or crashed connection cannot consume resources that starve
  the others on the same gateway, the direct software analogue of a
  bulkhead.
- **A crashed component restarts automatically**, with a bounded retry
  count rather than an infinite crash loop.
- **Connections to unreachable destinations use a circuit-breaker pattern**,
  after a run of consecutive failures, the gateway stops actively
  hammering the dead connection for a cooldown period, then cautiously
  probes it again before resuming full traffic. This is functionally
  identical to how a control system avoids continuing to drive into a
  stalled actuator, the same "detect, back off, carefully retry" logic,
  applied to a network connection instead of a motor.

### 2.5 Data quality and the alarm lifecycle, walked through concretely

Every single reading gets an explicit confidence tag: **GOOD** (passed
every check), **SUSPECT** (unusual but plausible, forwarded with a flag,
not discarded), **UNCERTAIN** (something about its timing or context is
questionable, for instance, if the device's own timestamp is missing and
the gateway has to fall back to its own receipt time), or **BAD** (fails
validation outright and is set aside for a human decision, never silently
dropped and never silently trusted).

Fault conditions move through a real four-state lifecycle that mirrors how
an actual operations team works, not a simplified boolean flag. Here's an
actual walkthrough, exactly as the system is designed to run it: a pressure
sensor on that same offshore platform reads 205 PSI against a configured
critical threshold of 200 PSI. The moment that reading crosses the
threshold, the gateway raises an alarm, tagged CRITICAL, with the exact
value, the threshold it breached, and a timestamp. Within half a second,
that alarm is routed to a configured Slack channel. Two minutes later, an
operator acknowledges it in the web application, that acknowledgement is
recorded with who did it and exactly when. Fifteen minutes after that, the
pressure reads back down to 198 PSI, below threshold, and the alarm
automatically transitions to cleared, with a recorded duration of 900
seconds from raise to clear. Every one of those transitions, raised,
acknowledged, cleared, is timestamped and attributed, which matters both
for shift handover between operators and for after-the-fact review of
exactly what happened and when.

Fault conditions can also be routed through multi-step **escalation
policies**, for example: notify a Slack channel immediately; if nobody has
acknowledged it within fifteen minutes, automatically escalate to a second
channel. The escalation logic is acknowledgement-aware: if an operator
handles the alarm before an escalation step fires, that step is
automatically skipped rather than firing anyway.

---

## 3. Protocol depth: this is not a shallow integration layer

It's easy to describe "supports Modbus" as a checkbox. The actual
engineering involved in doing this correctly is considerably deeper, and
it's worth walking through because it's a real differentiator in field
reliability, not a marketing point:

- **Modbus TCP and Modbus RTU** (the network and serial variants of one of
  the oldest and most widespread industrial protocols) are both supported,
  including the full range of practical data encodings a real device might
  use, signed and unsigned integers of different widths, 32-bit and
  64-bit floating point values, and critically, the byte-and-word ordering
  quirks that vary by manufacturer. Schneider/Modicon devices, for
  instance, use a different word order than the default big-endian
  convention most other vendors use, Qorel handles that explicitly rather
  than assuming one universal convention and quietly misreading values from
  devices that don't match it. Contiguous or overlapping register requests
  are automatically batched into shared reads, minimizing round trips to
  real hardware that often can't tolerate being hammered with requests.
  Connection and read failures are handled with bounded retry and backoff
  before the system falls back to a full container restart, a graceful
  degradation path, not an immediate hard failure.
- **OPC UA**, a more capable, subscription-based protocol common on modern
  SCADA systems, is supported in its push mode, the server notifies the
  gateway the instant a value changes, rather than the gateway having to
  poll and potentially miss fast-changing values between polls. In a
  documented example, a reactor's temperature and pressure are sampled at
  100 milliseconds, generating 10 readings per second per parameter, the
  gateway's own aggregation immediately reduces that to a dashboard-ready 2
  messages per second at one-second resolution and roughly 0.03 messages
  per second at one-minute resolution, without losing the ability to go
  back and inspect the full-resolution data if it's ever needed.
- **MQTT**, the lightweight protocol common on newer sensors and IoT-style
  devices, is supported as a subscriber, with configurable topic and field
  mappings.

Every one of these speaks through one shared, consistent lifecycle
(connect, read, transform, publish, and a clean shutdown that doesn't
corrupt data mid-write) and one shared publishing path into the local log,
which means adding support for a new protocol in the future is additive
engineering work against a proven pattern, not a redesign.

---

## 4. The operator experience: the control plane in full

The web application that engineers and operators actually use covers real
operational depth, not a bare-bones config screen:

- **Equipment configuration** through protocol-aware forms, an engineer
  configuring a Modbus device sees register mapping fields with unit and
  scaling controls, not a raw text box expecting them to hand-write a
  configuration file.
- **Live status** across every gateway and every piece of connected
  equipment, including health, most recent reading, and connectivity state.
- **Historical data exploration**, with both real-time views (the last
  sixty seconds) and long-range historical queries (the last 24 hours at
  one-minute resolution, for instance) against the same underlying data.
- **Full alarm management**, the four-state lifecycle above, acknowledged
  and cleared through the same interface, with a complete history per
  alarm.
- **A dead-letter review queue**, any reading that failed validation is
  visible with its original value and the specific reason it was rejected,
  and an operator can approve it individually, approve a filtered batch at
  once, or discard it.
- **Escalation policy authoring**, entirely through the interface, no
  code, no configuration file, building out a named, ordered sequence of
  notification steps with configurable delays.
- **Role-based access control** across four tiers, view-only, operator
  (can acknowledge alarms and review rejected data), engineer (can
  configure equipment and destinations), and admin (full control including
  user and gateway management), so a large team can be given exactly the
  level of access appropriate to their role.
- **A complete audit trail** of every configuration change, by every user,
  timestamped and attributed, essential both for internal accountability
  and for the kind of compliance review (SOC 2-style change logs, access
  reviews) that industrial customers increasingly expect.
- **Gateway enrollment and approval**, a new gateway doesn't join the
  fleet silently; it registers as pending and requires explicit operator
  approval before it's issued credentials, closing an entire class of
  "unknown device joins the network" risk before it starts.

---

## 5. What's real and running today

This is implemented, exercised by automated tests, and running, not a
roadmap item described in the future tense:

- Direct connections to real equipment over Modbus TCP, Modbus RTU, MQTT,
  and OPC UA, with the protocol depth described in Section 3.
- Delivery to time-series/relational databases, a customer's own streaming
  infrastructure, generic webhook/HTTP endpoints, and Slack-or-webhook
  alert routing.
- The complete data-quality, fault-lifecycle, and escalation machinery
  described in Section 2.5.
- The complete operator web application described in Section 4.
- Automatic recovery from component failure, per-component resource
  isolation, and circuit-breaker protection against unreachable
  destinations, as described in Section 2.4.
- Gateway devices that only join the fleet after explicit human approval,
  authenticated with long-lived credentials from that point on; user
  accounts restricted by role.
- Automated test coverage spanning the gateway software itself, the
  server-side application, every protocol connection, every destination
  connection, and the web application's core operator workflows.
