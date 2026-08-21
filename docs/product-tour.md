# Product tour

The operator interface, as it runs today. These are screenshots of the real
application, not mockups.

## Overview

Fleet health, throughput and active alarms at a glance.

![Overview](screenshots/overview.jpeg)

## Gateways

Enrollment, approval, revocation and per-gateway health.

![Gateways](screenshots/gateways.jpeg)

## Topology

What is connected to what, across the fleet.

![Topology](screenshots/topology.jpeg)

## Adapters

Protocol adapters and the assets they are bound to.

![Adapters](screenshots/adapters.jpeg)

## Creating an adapter

Protocol-aware forms: registers, units and scaling are first-class, not free text.

![Creating an adapter](screenshots/create-adapter.jpeg)

## Sinks

Where clean data goes: TimescaleDB, Kafka-compatible, HTTP, alert routing.

![Sinks](screenshots/sinks.jpeg)

## Creating a sink

Secrets are stored separately and never returned by the API.

![Creating a sink](screenshots/create-sink.jpeg)

## Alarms

Lifecycle from raised through acknowledged to cleared.

![Alarms](screenshots/alarms.jpeg)

## Alerting rules

Conditions evaluated in-process, routed by escalation policy.

![Alerting rules](screenshots/alerting-rules.jpeg)
