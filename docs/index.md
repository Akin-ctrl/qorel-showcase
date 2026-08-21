# Qorel

**PLCs and SCADA were built to talk to a control-room screen a few metres away,
not to a database, a cloud platform, or an AI agent.**

Getting that data out reliably, continuously, and without losing any of it
during the outages that are the *normal* condition at most real industrial
sites, is a problem the software industry has been chipping away at for twenty
years, mostly with tools designed before modern data streaming existed.

Qorel's bet is that this deserves the same technology the rest of the data
industry already trusts for reliability at scale: not a bespoke buffer bolted
onto legacy SCADA software, but a real, general-purpose streaming backbone,
running at the very edge, on the gateway itself, so that no single point of
failure, not the network, not the cloud, not a hard power cut, causes data loss.

Everything else here is either a consequence of that bet, or built on top of it.

---

## Start here

<div class="grid cards" markdown>

-   **[Technical and market reference](QOREL_TECHNICAL_OVERVIEW.md)**

    The full picture: what it does, what runs today, and where it is headed.
    Start here if you are reading one document.

-   **[Product tour](product-tour.md)**

    The operator interface, in screenshots.

-   **[Architecture](ARCHITECTURE.md)**

    How the gateway, control plane, adapters and sinks fit together.

-   **[Decisions](adr/README.md)**

    Fifteen architecture decision records, with the reasoning and the
    alternatives that were rejected.

</div>

---

## What runs today

Reusable adapter, sink and deployment objects. Protocol-aware configuration for
Modbus TCP, Modbus RTU, MQTT and OPC UA. TimescaleDB, Kafka-compatible, HTTP and
alert-routing sinks. A gateway runtime with local stream processing, validation,
aggregation, health and overflow control. Gateway enrollment with Ed25519 device
identity and operator approval. An always-on edge historian that backs operator
history independently of any sink. Production Compose packaging for both the
control plane and the edge gateway.

## What does not

Qorel is not a finished product. The documents here are deliberately explicit
about the difference between what ships and what is designed but unbuilt, and
the [decision records](adr/README.md) say so individually. The AI and MCP tooling
layer described in [ADR-010](adr/ADR-010-copilot-mcp.md) is a recorded direction,
not a shipped subsystem.

## About this site

This is a public showcase. The source is private while the project is under
active development. Everything here is copyright Akingbade Emmanuel Adebowale,
all rights reserved; Qorel itself is licensed under the Business Source
License 1.1.
