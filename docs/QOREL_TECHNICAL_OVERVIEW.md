# Qorel: Technical & Market Reference

A full picture of Qorel, what it does, how it does it,
what it can already do today, and where it's headed. Real numbers and real scenarios instead,
pulled directly from the system's own design documentation.

Last updated: 2026-07-08

---

## 1. The vision, stated plainly

There is a specific, enormous, and still largely unsolved problem sitting
underneath every factory, oil rig, power substation, water plant, telecom
tower, and solar mini-grid on Earth: the equipment that actually runs the
physical world, PLCs, sensors, SCADA systems, was built to talk to a
control-room screen a few meters away, not to a database, a cloud platform,
or an AI agent. Getting that data out reliably, continuously, and without
losing any of it during the outages that are the *normal* condition at most
real industrial sites, is a problem the software industry has been chipping
away at for twenty years with tools that were mostly designed before cloud
computing, before modern data streaming, and certainly before anyone
expected an AI agent to be a consumer of this data.

Qorel's bet is that this problem deserves to be solved with the same
seriousness and the same underlying technology the rest of the data
industry already uses for reliability at scale, not a bespoke,
purpose-built buffer bolted onto legacy SCADA software, but a real,
general-purpose streaming data backbone, running at the very edge, on the
gateway itself, so that no single point of failure, not the network, not
the cloud, not even a hard power cut, can cause data loss. Everything else
in this document is either a direct consequence of that one bet, or a
capability built on top of it.

---

## 2. What a Qorel deployment actually looks like

Picture an offshore oil platform fifty kilometers from shore, where the
only connectivity is a satellite or 4G link that goes down roughly a third
of the time. A PLC on the platform monitors pressure and temperature on a
pump skid over Modbus TCP. This is exactly the kind of site Qorel is built
for, and walking through it concretely shows how the pieces fit together.

An engineer configures, through Qorel's web application, a gateway for this
site, an adapter pointed at the PLC's Modbus registers (pressure on one
register, temperature on another, each tagged with its physical unit and
scaling), and a destination, in this case, a TimescaleDB instance for
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

## 3. The engineering underneath: why this works the way it does

### 3.1 A real, replayable log: not a purpose-built buffer

Most tools in this space solve "what if the network is down" with some
version of a retry queue: write locally, retry sending, delete once
delivered. That's a reasonable starting point, but it has a structural
ceiling; it's built for one consumer and one delivery attempt. The moment
a site needs a real-time dashboard *and* a historical database *and* an
alert pipeline all consuming the same stream independently, a simple retry
queue starts needing workarounds bolted onto it.

Qorel instead runs a genuine distributed streaming log on every gateway, 
Kafka-compatible, specifically an implementation called Redpanda, chosen
because it keeps the same protocol and client guarantees as the broader
Kafka ecosystem while being lighter to run on constrained edge hardware.
Concretely, this buys real properties a retry queue doesn't have:

- Every reading lands in an ordered, append-only record that isn't removed
  the moment one consumer reads it; it persists until it's explicitly aged
  out, which means multiple independent systems can each read the same
  data at their own pace without interfering with each other.
- Any consumer can rewind and replay from an earlier point in time. If a
  downstream system needs to reprocess a window of history, say, a
  correction to how a value gets interpreted; it replays rather than
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
for that segment is active near-term work (Section 8).

### 3.2 Full edge autonomy

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

### 3.3 Protecting data when local storage runs out

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

### 3.4 Fault isolation, described in terms a controls engineer already trusts

- **Every connection to a piece of equipment, and every connection to a
  destination system, runs in its own isolated, resource-limited process.**
  A misbehaving or crashed connection cannot consume resources that starve
  the others on the same gateway, the direct software analogue of a
  bulkhead.
- **A crashed component restarts automatically**, with a bounded retry
  count rather than an infinite crash loop.
- **Connections to unreachable destinations use a circuit-breaker pattern**
, after a run of consecutive failures, the gateway stops actively
  hammering the dead connection for a cooldown period, then cautiously
  probes it again before resuming full traffic. This is functionally
  identical to how a control system avoids continuing to drive into a
  stalled actuator, the same "detect, back off, carefully retry" logic,
  applied to a network connection instead of a motor.

### 3.5 Data quality and the alarm lifecycle, walked through concretely

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
operator acknowledges it in the web application; that acknowledgement is
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

## 4. Protocol depth: this is not a shallow integration layer

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

## 5. The operator experience: the control plane in full

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

## 6. What's real and running today

This is implemented, exercised by automated tests, and running, not a
roadmap item described in the future tense:

- Direct connections to real equipment over Modbus TCP, Modbus RTU, MQTT,
  and OPC UA, with the protocol depth described in Section 4.
- Delivery to time-series/relational databases, a customer's own streaming
  infrastructure, generic webhook/HTTP endpoints, and Slack-or-webhook
  alert routing.
- The complete data-quality, fault-lifecycle, and escalation machinery
  described in Section 3.5.
- The complete operator web application described in Section 5.
- Automatic recovery from component failure, per-component resource
  isolation, and circuit-breaker protection against unreachable
  destinations, as described in Section 3.4.
- Gateway devices that only join the fleet after explicit human approval,
  authenticated with long-lived credentials from that point on; user
  accounts restricted by role.
- Automated test coverage spanning the gateway software itself, the
  server-side application, every protocol connection, every destination
  connection, and the web application's core operator workflows.

---

## 7. How this compares to the established players

This is a real, existing market with mature incumbents, **Ignition**
(Inductive Automation), **Kepware/ThingWorx** (PTC), **HighByte
Intelligence Hub**, **Litmus Edge**, and **AVEVA** among them, some with
15-20 years in the market and installed bases in the thousands of sites.
Being credible about this comparison matters more than being flattering
about it:

**Where the established players are ahead, honestly:** protocol breadth
(some support well over a hundred equipment types), production-grade
security hardening refined over many customer deployments, and real
field-proven references. Two of them, HighByte and Litmus, have also
already begun shipping AI/agent-oriented features, so "AI-native industrial
platform" isn't a clean white space anymore.

**Where Qorel's underlying approach is genuinely different, not just
another implementation of the same idea:** essentially every established
tool buffers data at the edge with a purpose-built mechanism designed for
exactly one job, Ignition's store-and-forward, Kepware's local historian
archive. Qorel instead uses a real, general-purpose streaming log as that
buffer, which is what makes the replay-and-multiple-independent-consumers
property in Section 3.1 possible in the first place, a structural
capability most single-purpose buffers were never built to offer. The
honest tradeoff, as already noted, is a heavier resource footprint than a
purpose-built queue.

**Where the real opportunity is:** not "better technology" as an abstract
claim, but market fit. The established players are priced, packaged, and
sold for large enterprise accounts with dedicated systems-integrator
relationships. That leaves real room in the mid-market, operators who need
the same underlying reliability guarantees without the budget or in-house
technical staff an enterprise contract assumes.

---

## 8. The market opportunity, and it's bigger than one vertical

### 8.1 Nigeria and West Africa: the anchor market

Rather than a head-on global launch against the players in Section 7, the
initial focus is the Nigeria / West Africa industrial mid-market. Two facts
make this a genuinely strong starting point, not a consolation choice:

Nigeria's industrial automation market is valued at roughly $169 million as
of 2025, projected to reach $356 million by 2035 at a 7.75% compound annual
growth rate. Nigeria's national oil company, NNPC, is actively executing
SCADA modernization under national digital-transformation programs.
Broader African industrial IoT, condition monitoring and predictive
maintenance across mining, oil & gas, and manufacturing, is growing 14-18%
annually, with the connected-sensors segment alone estimated at $450-550
million in 2026.

More importantly: the region's actual infrastructure reality is precisely
the scenario the architecture in Section 3 was built for, at national
scale rather than as an edge case. Roughly 70% of Nigeria's telecom network
downtime traces directly to power failures rather than network issues
alone, the national grid suffered 12 full collapses in 2024, and telecom
operators logged over 19,000 fiber-cable cuts in just the first eight
months of 2025. A system engineered to survive extended, ungraceful
outages without losing data isn't a nice-to-have in that environment; it's
close to a baseline requirement, because that failure condition is the
routine case, not the exception. Most competing tools in Section 7 were
engineered against far more reliable Western grids and networks and simply
were never stress-tested against this reality the way Qorel's core design
already has been.

This is a real, underserved segment rather than an empty field, AVEVA
already operates in the region through a named distributor, actively
building training and partner infrastructure there. The opening is that the
established players present are priced and packaged for the largest
national accounts, leaving mid-tier operators, same reliability needs,
without the budget or in-house integration staff an enterprise contract
assumes, genuinely underserved.

### 8.2 The opportunity is bigger than factories

The underlying pattern Qorel fits isn't "industrial factories" specifically
; it's "any site where unreliable power and connectivity is the normal
operating condition, and there's equipment worth monitoring reliably
through that." Several adjacent verticals fit that description as well as
a factory floor does, in some cases better, and, notably, several of them
require zero new engineering because they already run on protocols Qorel
already speaks:

- **Telecom tower and base-station monitoring.** Remote cell towers across
  Africa run on diesel generators and battery/UPS systems precisely because
  grid power doesn't reach them reliably, and the standard way to monitor
  that equipment, fuel level, generator run hours, battery voltage, fault
  state, already runs over Modbus. Nigeria's telecom tower business in
  particular runs on a shared-infrastructure ("towerco") model: companies
  like IHS Towers own roughly 37,000 towers across seven countries, leased
  out to multiple competing carriers, MTN, Airtel, Orange, and others, who
  each rent space on the same physical infrastructure. A single towerco
  managing tens of thousands of remote sites is an outstanding anchor-customer
  profile: one company, one fleet, enormous scale, and real money riding on
  generator uptime and fuel patterns across every site, which plugs
  directly into the fleet-wide analysis capability described in Section 9,
  at a scale far beyond any single factory.
- **Water and wastewater utilities.** Pump stations and treatment plants
  spread across a wide geographic area are a textbook remote-monitoring
  problem, and the protocols in active use, Modbus, MQTT, are ones Qorel
  already speaks. Zero new adapter engineering required here either.
- **Solar mini-grids and renewable energy monitoring.** Nigeria is a
  recognized leader in rural solar mini-grid deployment, for the exact same
  unreliable-grid-power reason driving this whole market thesis, and solar
  inverters overwhelmingly report their telemetry over Modbus. This
  vertical is almost recursive in its fit: mini-grids exist specifically to
  solve unreliable power, and they themselves need the same kind of
  resilient remote monitoring Qorel is built around.
- **Power substations.** Ordinary power metering runs over Modbus (already
  covered), though full substation automation runs on DNP3 and IEC 61850, 
  genuinely different protocols. A real fit, and one worth pursuing, but
  one that costs real (well-understood) adapter engineering rather than
  being available today.
- **Precision agriculture.** Real and already showing measurable results, 
  documented pilots report 15-25% yield increases for cassava farmers in
  Nigeria and 50% disease-prevention improvements for maize in Rwanda using
  IoT sensing. This vertical typically runs on LoRaWAN connectivity rather
  than Modbus/MQTT/OPC UA, worth noting that LoRa support is already an
  identified (if not yet built) item on the adapter roadmap.

None of these are empty fields, telecom tower monitoring in particular is
already a mature product category with dedicated vendors, but the
opportunity shape is consistent across every one of them: established
players serving the largest accounts, real room underneath for a platform
built around the reliability guarantees these specific environments
actually need.

---

## 9. Where this is headed: the intelligence layer

### 9.1 Learning across a single customer's own fleet

A customer with many sites will naturally want their system to notice
patterns across the whole fleet, not just within one gateway, "this type
of equipment has had the same fault repeat across a dozen sites this
quarter." This is more within reach than it might sound: raw,
moment-to-moment readings never have to leave a given site, but the
already-computed rolling summaries every gateway produces get sent up to
the one shared place a single customer's whole fleet is managed from. Once
that's centralized, spotting a pattern repeated across many sites is a
straightforward grouped analysis over data already in one place, no
distributed model training, no new infrastructure required. A single
telecom towerco with tens of thousands of sites, or a mini-grid operator
managing hundreds of installations, gets real fleet-wide intelligence from
this alone.

What that captures is descriptive: it tells you a pattern *has* repeated,
after the fact. The next, genuinely valuable step is predictive, "this
pattern usually precedes a failure within two weeks", which requires
knowing whether equipment actually failed afterward and when. That's a
deliberate future data-capture step: recording the real outcome when an
operator closes out a fault, building a labeled history to learn from over
time. Real, valuable, and clearly sequenced as future work rather than
something to build before the foundation is solid.

Closely related and worth building toward directly: turning "this station
was down for six hours" into "that cost us this much", attaching an
estimated cost figure to downtime by asset or site and joining it against
the fault history already being tracked. A near-certain early customer
question ("why is this costing us money, and it always seems to trace back
to specific sites") that this closes cleanly once in place.

### 9.2 A bigger idea: intelligence shared across separate organizations

There's a genuinely larger idea worth having on the map: not one customer
learning across their own sites, but *multiple separate companies*, 
competitors, even, jointly learning from each other's operational patterns
without exposing their own raw data to one another. This isn't
speculative. It's a real, production-proven technique called federated
learning, each participant trains locally and shares only the learned
patterns, never the underlying data, already deployed by wind turbine
manufacturers Vestas and Siemens Gamesa across geographically distributed
wind farms, and built into industrial IoT platforms from Bosch, Siemens,
and Schneider Electric, with published results in some deployments showing
downtime reductions above 50%.

And there's already a working precedent for exactly this, in exactly the
region this plan targets: an association of more than thirty African
mini-grid operators across twelve countries, direct competitors among them
, has been running a shared, anonymized benchmarking program since 2019.
The demand for this kind of cross-organization intelligence in this market
is not hypothetical; it already exists and is already operating. Building
toward it is a genuinely bigger undertaking than single-customer fleet
learning; it needs a neutral aggregation point that sits outside any one
customer's own system, but it's a real, proven-valuable direction with
concrete regional demand already visible.

### 9.3 The AI reasoning layer

Beyond analysis, the longer-term direction is exposing Qorel's data and
capabilities as callable tools for an AI agent, a customer's own AI
system, or an optional built-in assistant, to diagnose why a piece of
equipment looks unhealthy, spot patterns across an entire fleet, and
suggest configuration changes. This is deliberately sequenced after the
fleet-management and reliability foundation above, on the reasoning that an
AI layer built on top of a data foundation that hasn't been proven reliable
in the field isn't trustworthy, no matter how capable the AI itself sounds.
And critically: no standing autonomous agent that acts on its own
initiative. Every consequential action, changing a configuration,
restarting a component, adjusting a threshold, requires an explicit human
approval, every time, with a full audit trail of who approved what. The
ambition here is real and the architecture for it is already designed; it's
being built last on purpose, because its credibility depends entirely on
everything underneath it already holding up.

---

## 10. The roadmap, honestly

Framed the way it should be, a sequenced plan:

**Near-term**, making the system field-ready rather than just demo-ready:
verified survival of sudden, ungraceful power loss (not just network loss)
given how routinely the target market experiences exactly that; a tuned
deployment profile for lower-cost gateway hardware; configurable control
over how much data crosses an expensive or metered connection versus
staying local until a cheaper link is available; real-world validation
against physical equipment at an actual site; additional protocol support
driven by confirmed customer need.

**Mid-term**, once a single site is proven solid: safely managing and
updating many gateways across many sites at once, with staged,
health-checked rollouts and automatic rollback if something goes wrong;
stronger cryptographic device identity; a genuine relationship model
between pieces of equipment (not just a flat list of sensors, but "this
actuator drives that mechanism"), unlocking real fleet-wide analysis.

**Longer-term:** the fleet intelligence and AI reasoning layer described in
Section 9, including the genuinely larger cross-organization federated
intelligence direction, sequenced deliberately last because it depends on
everything before it being solid, not because it's an afterthought.

The reason to be direct about exactly what's built versus what's ahead
isn't caution for its own sake; it's that a team that can state precisely
what is and isn't done, and why, is a far more credible partner for the
work still ahead than one that oversells a demo. That precision is a real
asset going into any serious conversation, and it's worth carrying that
same tone into the pitch built from this document.

---

