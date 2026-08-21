# Data Flow Documentation

**Detailed data flow examples across different industrial scenarios.**

---

## Table of Contents

1. [Core Data Flow Pattern](#core-data-flow-pattern)
2. [Example 1: Offshore Pump Skid (Modbus → Local Stream → Optional Cloud Sink)](#example-1-offshore-pump-skid-modbus--local-stream--optional-cloud-sink)
3. [Example 2: Smart Factory (Modbus PLC → TimescaleDB)](#example-2-smart-factory-modbus-plc--timescaledb)
4. [Example 3: OPC UA → Real-time Dashboard](#example-3-opc-ua--real-time-dashboard)
5. [Example 4: Network Outage Recovery](#example-4-network-outage-recovery)
6. [Example 5: Alarm Lifecycle](#example-5-alarm-lifecycle)
7. [Message Format Examples](#message-format-examples)

---

## Core Data Flow Pattern

**Every data flow follows this pattern**:

```
Physical Device
    ↓ (Protocol: Modbus/OPC UA/MQTT/etc.)
Protocol Adapter (Container)
    ↓ (Normalized message)
Local Kafka-compatible stream (edge buffering)
    ↓ (Gateway-local consumption)
    ├─→ Quality Validator → telemetry.clean
    ├─→ Aggregator → telemetry.1s, telemetry.1min
    ├─→ Sink Services → Databases, customer-owned Kafka-compatible systems, Cloud APIs
    └─→ Alert Router → Slack, generic webhook (PagerDuty/SMS/Email not yet implemented)
```

---

## Example 1: Offshore Pump Skid (Modbus → Local Stream → Optional Cloud Sink)

### Scenario
- **Location**: Offshore oil rig, 50km from shore
- **Connectivity**: Intermittent 4G/satellite (down 30% of the time)
- **Device**: PLC exposing pressure and temperature over Modbus TCP or RTU
- **Requirements**: durable local buffering, replay after outage, critical alarm routing

### Configuration

**Step 1: UI Configuration**

Operator configures adapters and sinks, then composes a deployment for the target gateway:
```
Gateway: gateway-offshore-01
Adapter: modbus_offshore_12
  Protocol: Modbus TCP
  Device: PLC at 192.168.10.42:502
  Parameters:
    - Register 40001: Pressure (PSI)
    - Register 40003: Temperature (Celsius)
Sink: timescaledb_offshore_primary
Validation: enabled
```

**Step 2: Active deployment config (sent to gateway)**

The control plane sends one gateway-level config containing the selected
adapters, sinks, validation rules, event rules, and aggregate rules. Secret
values shown below are redacted; the control plane stores write-only secret
fields and resolves them for the gateway runtime.

```json
{
  "gateway_id": "gateway-offshore-01",
  "deployment_id": "deployment-offshore-pump-skid",
  "adapters": [
    {
      "adapter_id": "modbus_offshore_12",
      "adapter_type": "modbus_tcp",
      "config": {
        "host": "192.168.10.42",
        "port": 502,
        "unit_id": 1,
        "asset_id": "offshore_well_12",
        "kafka_bootstrap": "kafka:9092",
        "topic": "telemetry.raw",
        "registers": {
          "40001": {
            "param": "pressure",
            "unit": "psi",
            "type": "float32",
            "word_order": "big",
            "byte_order": "big",
            "scale": 1.0
          },
          "40003": {
            "param": "temperature",
            "unit": "celsius",
            "type": "float32",
            "word_order": "big",
            "byte_order": "big",
            "scale": 0.1
          }
        },
        "poll_interval_ms": 1000
      }
    }
  ],
  "sinks": [
    {
      "sink_id": "timescaledb_offshore_primary",
      "sink_type": "timescaledb",
      "config": {
        "topic": "telemetry.clean",
        "group_id": "qr-sink-timescaledb-offshore",
        "table": "telemetry_clean",
        "db_dsn": "<resolved write-only DSN>"
      }
    }
  ],
  "validation": {
    "enabled": true
  },
  "events": {},
  "aggregates": {},
  "version": "2026-06-02T12:00:00Z"
}
```

### Data Flow (Normal Operation)

**Time: 12:01:03.123 - Physical measurement**
```
Sensor: Pressure = 185.4 PSI, Temperature = 34.2°C
PLC exposes values in configured holding registers
```

**Time: 12:01:03.987 - Modbus read**
```
Adapter container: Reads configured register batch
  Register 40001 → 185.4 (float32)
  Register 40003 → 342 (int16, scale 0.1) = 34.2°C
```

**Time: 12:01:04.001 - Normalization**

One telemetry message batches every reading from the same poll cycle, 
the adapter does not publish one message per parameter (see
`adapters/adapter_base/telemetry.avsc` and `modbus_tcp_adapter.py`'s
`transform()`):
```python
# Adapter code
telemetry_message = {
    "asset_id": "offshore_well_12",
    "gateway_time": "2025-01-10T12:01:04.001Z",
    "topology": None,
    "readings": [
        {"parameter": "pressure", "value": 185.4, "unit": "psi", "quality": "GOOD", "device_time": "2025-01-10T12:01:03.123Z"},
        {"parameter": "temperature", "value": 34.2, "unit": "celsius", "quality": "GOOD", "device_time": "2025-01-10T12:01:03.123Z"}
    ],
    "validation_reason": None,
    "dlq_resolution": None
}
```

**Time: 12:01:04.015 - Local stream write**
```
Topic: telemetry.raw
Partition: 0 (keyed by asset_id)
Offset: 1523847
Message: [JSON above]
Acks: all (written to disk before acknowledge)
```

**Time: 12:01:04.050 - Local validation**
```python
# Gateway-local validator
input: telemetry.raw
validation:
  ✓ 0 < pressure < 500 PSI
  ✓ -50 < temperature < 200 °C
  ✓ timestamps valid
  ✓ no duplicates
quality: GOOD
output: telemetry.clean
```

**Time: 12:01:05.000 - Aggregation**

Every field below is always present on the wire (Avro record, not a
sparse/partial summary), see the full shape under
[Aggregated Telemetry](#aggregated-telemetry-1-minute-window):
```python
# 1-second window aggregator
window: [12:01:04.000 → 12:01:05.000]
samples: 1 (only this reading in this second)
output to telemetry.1s:
{
  "asset_id": "offshore_well_12",
  "parameter": "pressure",
  "unit": "psi",
  "classification": "TELEMETRY_AGGREGATE",
  "window_start": "2025-01-10T12:01:04Z",
  "window_end": "2025-01-10T12:01:05Z",
  "aggregates": {"avg": 185.4, "min": 185.4, "max": 185.4, "stddev": 0.0, "count": 1, "p50": 185.4, "p95": 185.4, "p99": 185.4},
  "quality_summary": {"good_samples": 1, "suspect_samples": 0, "uncertain_samples": 0, "bad_samples": 0, "pct_good": 100.0},
  "metadata": {"aggregation_version": "1.0.0", "resolution": "1s", "source_topic": "telemetry.clean", "gateway_id": "gateway-offshore-01"}
}
```

**Time: 12:01:04.300 - Sink writes**
```
Sink: TimescaleDB
Table: telemetry_clean
SQL: INSERT INTO telemetry_clean (gateway_time, device_time, asset_id, parameter, value, unit, quality, payload)
     VALUES ('2025-01-10T12:01:04.001Z', '2025-01-10T12:01:03.123Z', 'offshore_well_12', 'pressure', 185.4, 'psi', 'GOOD', '{...}')

Sink: Kafka-compatible system (optional, customer-owned cluster)
Target: customer.example.com:9092
Behavior: Drain from local telemetry.clean when WAN is available

Sink: Cloud ML API
HTTP POST to https://ml.example.com/predict
Body: {"timestamp": "...", "pressure": 185.4, "temperature": 34.2}
```

### Data Flow (Network Outage)

**Time: 14:30:00 - Network fails**
```
4G link down
External sinks unavailable
Local stream broker: Continues accepting writes (local disk)
```

**Time: 14:30:01 → 17:00:00 - 2.5 hours offline**
```
Adapter: Still reading sensor every 1s
Local stream broker: Buffering locally
  Topics: telemetry.raw, events.raw
  Disk usage: 42% → 58% (16GB accumulated)
  Status: ✓ No data loss
```

**Time: 17:00:00 - Network restored**
```
Sink connectivity restored
Backlog drain begins from the local stream
Catch-up speed: ~1000 msg/sec
```

**Time: 17:02:30 - Fully synced**
```
All buffered data drained from the local stream to configured sinks
External sinks catch up:
  - TimescaleDB: Writes 9,000 rows (backfill)
  - HTTP/webhook destinations: Catch up according to retry settings
  - Customer Kafka-compatible system: Receives delayed replay
  - ML API: Skips (real-time only)
```

**Result**: no expected data loss under the designed outage path; historical analysis remains intact if local storage is preserved

---

## Example 2: Smart Factory (Modbus PLC → TimescaleDB)

### Scenario
- **Location**: Factory floor, stable LAN
- **Device**: Modbus TCP PLC (assembly line controller)
- **Requirements**: Real-time dashboard, historical analytics

### Configuration

```json
{
  "gateway_id": "gateway-factory-01",
  "deployment_id": "deployment-assembly-line",
  "adapters": [
    {
      "adapter_id": "modbus_assembly_line_01",
      "adapter_type": "modbus_tcp",
      "config": {
        "host": "192.168.10.50",
        "port": 502,
        "unit_id": 1,
        "asset_id": "assembly_line_plc_01",
        "kafka_bootstrap": "kafka:9092",
        "topic": "telemetry.raw",
        "events_topic": "events.raw",
        "holding_registers": {
          "40001": {"param": "conveyor_speed", "unit": "m/min", "type": "uint16"},
          "40002": {"param": "motor_current", "unit": "amps", "type": "uint16", "scale": 0.1},
          "40003": {"param": "cycle_count", "unit": "count", "type": "uint32", "word_order": "big", "byte_order": "big"}
        },
        "coils": {
          "00001": {"param": "motor_running", "type": "bool"}
        },
        "poll_interval_ms": 500
      }
    }
  ],
  "sinks": [
    {
      "sink_id": "timescaledb_factory_primary",
      "sink_type": "timescaledb",
      "config": {
        "topic": "telemetry.clean",
        "group_id": "qr-sink-timescaledb-factory",
        "table": "telemetry_clean",
        "db_dsn": "<resolved write-only DSN>"
      }
    },
    {
      "sink_id": "events_factory_primary",
      "sink_type": "timescaledb",
      "config": {
        "topic": "events.clean",
        "group_id": "qr-sink-timescaledb-events",
        "table": "events_clean",
        "db_dsn": "<resolved write-only DSN>"
      }
    }
  ],
  "validation": {"enabled": true},
  "events": {"enabled": true},
  "aggregates": {"enabled": true}
}
```

### Data Flow

**Every 500ms**:

```
Modbus TCP request to 192.168.10.50:502
Read holding registers 40001-40003
Read coil 00001

Response:
  40001 = 120 → conveyor_speed = 120 m/min
  40002 = 153 → motor_current = 15.3 amps
  40003 = 48291 → cycle_count = 48291
  00001 = 1 → motor_running = true
```

**Adapter publishes 2 messages**, one telemetry message batching every
reading from this poll cycle, plus one event only when a monitored
coil/state actually changed:

**Message 1 (Telemetry, all readings from this cycle in one message)**:
```json
{
  "asset_id": "assembly_line_plc_01",
  "gateway_time": "2025-01-10T08:30:00.123Z",
  "topology": null,
  "readings": [
    {"parameter": "conveyor_speed", "value": 120, "unit": "m/min", "quality": "GOOD", "device_time": null},
    {"parameter": "motor_current", "value": 15.3, "unit": "amps", "quality": "GOOD", "device_time": null}
  ],
  "validation_reason": null,
  "dlq_resolution": null
}
```

**Message 2 (Event - only when state changes)**:
```json
{
  "asset_id": "assembly_line_plc_01",
  "event_type": "motor_state_change",
  "classification": "EVENT",
  "previous_state": {"motor_running": false},
  "new_state": {"motor_running": true},
  "timestamps": {
    "device_time": null,
    "gateway_time": "2025-01-10T08:30:00.123Z"
  },
  "metadata": {
    "adapter_id": "modbus_assembly_line_01",
    "deployment_id": "deployment-assembly-line"
  }
}
```

**Local stream → TimescaleDB**:
```sql
-- Sink writes telemetry. TimescaleDB promotion is best-effort; regular
-- PostgreSQL tables still work if create_hypertable is unavailable.
INSERT INTO telemetry_clean (gateway_time, asset_id, parameter, value, unit, quality, payload)
VALUES 
  ('2025-01-10T08:30:00.123Z', 'assembly_line_plc_01', 'conveyor_speed', 120, 'm/min', 'GOOD', '{...}'),
  ('2025-01-10T08:30:00.123Z', 'assembly_line_plc_01', 'motor_current', 15.3, 'amps', 'GOOD', '{...}');

INSERT INTO events_clean (gateway_time, asset_id, event_type, classification, previous_state, new_state, metadata, payload)
VALUES ('2025-01-10T08:30:00.123Z', 'assembly_line_plc_01', 'motor_state_change', 'EVENT', '{"motor_running": false}', '{"motor_running": true}', '{}', '{...}');
```

**Dashboard queries**:
```sql
-- Real-time (last 60 seconds)
SELECT gateway_time, value
FROM telemetry_clean
WHERE asset_id = 'assembly_line_plc_01' 
  AND parameter = 'conveyor_speed'
  AND gateway_time > NOW() - INTERVAL '60 seconds'
ORDER BY gateway_time DESC;

-- Historical (last 24 hours, 1-minute averages)
SELECT time_bucket('1 minute', gateway_time) AS bucket,
       AVG(value) as avg_speed
FROM telemetry_clean
WHERE asset_id = 'assembly_line_plc_01'
  AND parameter = 'conveyor_speed'
  AND gateway_time > NOW() - INTERVAL '24 hours'
GROUP BY bucket
ORDER BY bucket;
```

---

## Example 3: OPC UA → Real-time Dashboard

### Scenario
- **Device**: OPC UA server (SCADA system)
- **Pattern**: Subscription-based (push, not poll)

### Configuration

```json
{
  "adapter_id": "opcua_reactor_01",
  "adapter_type": "opcua",
  "config": {
    "endpoint": "opc.tcp://scada.example.com:4840",
    "security_mode": "SignAndEncrypt",
    "security_policy": "Basic256Sha256",
    "username": "<stored write-only username>",
    "password": "<stored write-only password>",
    "asset_id": "reactor_01",
    "kafka_bootstrap": "kafka:9092",
    "topic": "telemetry.raw",
    "monitored_items": [
      {
        "node_id": "ns=2;s=Reactor.Temperature",
        "param": "reactor_temperature",
        "unit": "celsius",
        "sampling_interval_ms": 100
      },
      {
        "node_id": "ns=2;s=Reactor.Pressure",
        "param": "reactor_pressure",
        "unit": "bar",
        "sampling_interval_ms": 100
      }
    ]
  }
}
```

### Data Flow

**Subscription-based (push from OPC UA server)**:

```
OPC UA Server: Value changed (reactor_temperature = 342.7°C)
  ↓ (push notification)
Adapter: Receives data change notification
  ↓
Publishes to the local Kafka-compatible stream immediately (no polling delay)
```

**High-frequency data (100ms sampling)**:
```
10:00:00.000 → temp=342.5
10:00:00.100 → temp=342.6
10:00:00.200 → temp=342.7
10:00:00.300 → temp=342.8
...
(10 readings/second)
```

**Aggregation saves bandwidth**:
```
telemetry.raw: 10 msg/sec per parameter = 20 msg/sec total
telemetry.1s: 2 msg/sec (aggregated)
telemetry.1min: 0.033 msg/sec (aggregated)

Dashboard consumes telemetry.1s (sufficient resolution)
```

---

## Example 4: Network Outage Recovery

### Timeline

**Day 1, 10:00 - Normal operation**
```
Edge gateway online
Optional cloud sink lag: 50ms
Disk usage: 20GB / 100GB
```

**Day 1, 14:30 - Network failure**
```
Satellite link down (storm)
Kafka-compatible/HTTP sinks retrying external connections...
Local stream broker: Buffering locally
Adapters: Continue operating normally
```

**Day 1, 14:30 → Day 3, 08:00 (41.5 hours offline)**
```
Total messages buffered: 149,200 (1 msg/sec × 41.5 hours)
Disk usage: 20GB → 52GB (+32GB)
Topics buffered:
  - telemetry.raw: 120,000 messages
  - events.raw: 28,000 messages
  - alarms.raw: 1,200 messages

Note: recent gateway-runtime logs are transported to the control plane via
the heartbeat payload, not a Kafka topic; there is no logs.* topic to
buffer. Log visibility during an outage is limited to whatever the
gateway's local recent-log ring buffer still holds when connectivity
returns.
```

**Day 3, 08:00 - Network restored**
```
External sink connectivity re-established
Replay begins:
  Offset lag: 149,400 messages
  Throughput: 2,000 msg/sec
  ETA: ~75 seconds to catch up
```

**Day 3, 08:01:15 - Fully synced**
```
All data drained from the local stream to external destinations
Sinks processing backlog:
  - TimescaleDB: Bulk insert 120,000 rows (30 seconds)
  - Customer Kafka-compatible system: Receives 120,000 delayed records
  - HTTP/webhook destinations: Catch up according to their retry and backpressure settings
  - Alerting: Processes 1,200 alarms (checks if still active)
```

**Result**:
- ✓ Buffered replay without expected data loss when local storage remains healthy
- ✓ Historical continuity maintained
- ✓ Alarms processed in order
- ✓ Automatic recovery, no operator intervention

---

## Example 5: Alarm Lifecycle

### Scenario: Overpressure Alarm

**Time: 15:30:00 - Threshold crossed**
```
Sensor reading: pressure = 205 PSI (threshold = 200 PSI)

Adapter evaluates alarm condition (configured in pipeline):
{
  "alarm_rules": [
    {
      "parameter": "pressure",
      "condition": "value > 200",
      "severity": "CRITICAL",
      "type": "overpressure"
    }
  ]
}

Publishes alarm to alarms.raw, full record, not a partial patch:
{
  "alarm_id": "alarm-uuid-1234",
  "asset_id": "offshore_well_12",
  "type": "overpressure",
  "severity": "CRITICAL",
  "state": "ACTIVE",
  "classification": "ALARM",
  "raised_at": "2025-01-10T15:30:00.123Z",
  "acked_at": null,
  "acked_by": null,
  "cleared_at": null,
  "suppressed_at": null,
  "suppressed_by": null,
  "value": 205,
  "threshold": 200,
  "unit": "psi",
  "message": "Pressure exceeded critical threshold",
  "escalation_policy_id": null,
  "metadata": {"adapter_id": "validator", "deployment_id": "gateway-offshore-01"}
}
```

**Time: 15:30:00.500 - Alert routing**
```
alert_router sink consumes alarms.raw. If the alarm carries a bound
escalation_policy_id, it resolves and walks that policy's steps
(immediate steps deliver right away; escalate steps are scheduled after
delay_minutes and only fire if the alarm is still ACTIVE at that time,
checked via GET /api/v1/alarms/{alarm_id}/gateway-view). Otherwise it
falls back to a single configured Slack channel or generic webhook.

Note: there is still no severity-routing table, no PagerDuty/SMS channel
type, and no digest batching, only slack and webhook channel types exist.
This example's alarm_rules config has no escalation_policy_id bound, so it
takes the static webhook/Slack fallback path, not the policy-walk path.
```

**Time: 15:32:00 - Operator acknowledges**
```
Operator Alice clicks "Acknowledge" in UI

UI sends:
POST /api/v1/alarms/alarm-uuid-1234/acknowledge
{
  "acked_by": "alice@example.com"
}

Control plane updates the alarms row directly (state, acked_at, acked_by)
and returns it. This is a database update only, the control plane does
not publish anything back to the gateway's local Kafka-compatible stream.
There is no channel for the control plane to push state changes back down
to a gateway; the gateway only ever finds out about an ACK if it later
polls GET /api/v1/alarms/{alarm_id}/gateway-view (this is exactly what
alert_router does before firing a delayed escalate step, so it can skip
escalating an alarm an operator already acknowledged).

Note: escalation-policy step-walking (an escalate step scheduled after
delay_minutes, checked against current alarm state before firing) is
implemented (ADR-013/ADR-014, sink_alert_router). External incident-status
synchronization (e.g. auto-resolving a PagerDuty incident) is not; there
is no PagerDuty channel type at all yet, only slack and webhook.
```

**Time: 15:45:00 - Pressure returns to normal**
```
Sensor reading: pressure = 198 PSI (below threshold)

Adapter evaluates: condition no longer true

Gateway-local validator publishes the full alarm record to alarms.raw
(not a partial patch, every alarm field is present, most unchanged from
the ACTIVE record):
{
  "alarm_id": "alarm-uuid-1234",
  "asset_id": "offshore_well_12",
  "type": "overpressure",
  "severity": "CRITICAL",
  "state": "CLEARED",
  "classification": "ALARM",
  "raised_at": "2025-01-10T15:30:00.123Z",
  "acked_at": null,
  "acked_by": null,
  "cleared_at": "2025-01-10T15:45:00Z",
  "suppressed_at": null,
  "suppressed_by": null,
  "value": 198,
  "threshold": 200,
  "unit": "psi",
  "message": "pressure returned to normal range",
  "escalation_policy_id": null,
  "metadata": {"adapter_id": "validator", "deployment_id": "gateway-offshore-01"}
}

Note: this CLEARED record was published from the gateway's own local
validator state, not mirrored from the operator's ACK in the step above, 
the gateway and control plane track alarm lifecycle independently and can
disagree about acked_at/acked_by until the gateway next polls
gateway-view. `duration_seconds` is not on this wire message at all; the
control plane computes and stores it when persisting the row.

alert_router does notify on this CLEARED transition when there's no bound
escalation policy, its static webhook fallback does not filter by alarm
state, so every alarm record on alarms.raw produces a notification. It
uses the same generic message template shown above rather than a distinct
"all clear" format.
```

**Alarm stored in database**, the real table is `alarms` (not
`alarms_history`), and the wire message's `type` field is stored as
`alarm_type` (`app/db/models.py`):
```sql
INSERT INTO alarms (
  alarm_id, gateway_id, asset_id, alarm_type, severity, state,
  raised_at, acked_at, acked_by, cleared_at, duration_seconds
) VALUES (
  'alarm-uuid-1234', 'gateway-offshore-01', 'offshore_well_12', 'overpressure', 'CRITICAL', 'CLEARED',
  '2025-01-10T15:30:00Z', '2025-01-10T15:32:00Z', 'alice@example.com',
  '2025-01-10T15:45:00Z', 900
);
```

---

## Message Format Examples

### Telemetry Message (Complete)

This is the real Avro wire shape (`adapters/adapter_base/telemetry.avsc`).
One message batches every reading from a single poll cycle; there is no
per-parameter message, no `clock_skew_ms`, and no `schema_version` field on
the envelope (schema evolution is handled by the Schema Registry, not a
field in the payload). `topology` is an optional ISA-95 hierarchy reference,
null unless populated for that asset.

```json
{
  "asset_id": "offshore_well_12",
  "gateway_time": "2025-01-10T12:01:04.001Z",
  "topology": {
    "enterprise": null,
    "site": "offshore-rig-alpha",
    "area": "wellhead",
    "line": null,
    "cell": null,
    "equipment": "pump_skid_12"
  },
  "readings": [
    {
      "parameter": "pressure",
      "value": 185.4,
      "unit": "psi",
      "quality": "GOOD",
      "device_time": "2025-01-10T12:01:03.123Z"
    }
  ],
  "validation_reason": null,
  "dlq_resolution": null
}
```

`validation_reason` and `dlq_resolution` are populated by the gateway-local
validator, not the adapter, never on the raw wire message an adapter first
publishes. `validation_reason` is set whenever the validator attaches a
quality reason to a reading (e.g. `missing_device_time` for `UNCERTAIN`, or
a range/rate/gap breach reason for `BAD`). `dlq_resolution` only appears on
DLQ reprocess-preview payloads shown to an operator reviewing the DLQ
(`mode: "operator_reprocess_preview"`) or on undecodable messages that need
manual intervention (`mode: "manual_decode_required"`).

### Event Message (State Change)

Matches `schemas/event.avsc`. The metadata field is `deployment_id`; it was
originally named `pipeline_id` from before the platform's UI/API terminology
moved from "pipeline" to "deployment"; the schema and every producer were
renamed to match. `event_validator.py` still accepts an incoming `pipeline_id`
key as a fallback for messages produced by an out-of-date adapter build
during a rolling upgrade, but no current producer emits it.

```json
{
  "asset_id": "assembly_line_plc_01",
  "event_type": "motor_state_change",
  "classification": "EVENT",
  "previous_state": {
    "motor_running": false,
    "speed": 0
  },
  "new_state": {
    "motor_running": true,
    "speed": 120
  },
  "timestamps": {
    "device_time": "2025-01-10T08:30:00.123Z",
    "gateway_time": "2025-01-10T08:30:00.150Z"
  },
  "metadata": {
    "adapter_id": "adapter_modbus_tcp_002",
    "deployment_id": "deployment-assembly-line"
  }
}
```

### Alarm Message (All States)

Matches `schemas/alarm.avsc`. Avro records are not partial/patch documents, 
every state transition (raise, acknowledge, clear) publishes a **complete**
alarm record with every field present (nulled where not applicable), not a
delta containing only the changed keys. There is no `duration_seconds` or
`final_value` field on the wire message itself, `duration_seconds` is
computed and stored by the control plane when it persists the row
(`app/routers/alarms.py`), not published by the gateway.

**ACTIVE**:
```json
{
  "alarm_id": "alarm-uuid-1234",
  "asset_id": "reactor_01",
  "type": "overpressure",
  "severity": "CRITICAL",
  "state": "ACTIVE",
  "classification": "ALARM",
  "raised_at": "2025-01-10T15:30:00.123Z",
  "acked_at": null,
  "acked_by": null,
  "cleared_at": null,
  "suppressed_at": null,
  "suppressed_by": null,
  "value": 205,
  "threshold": 200,
  "unit": "psi",
  "message": "Pressure exceeded critical threshold",
  "escalation_policy_id": null,
  "metadata": {
    "adapter_id": "validator",
    "deployment_id": "gateway-rig-alpha"
  }
}
```

**ACKNOWLEDGED**, same full record, `state`/`acked_at`/`acked_by` updated:
```json
{
  "alarm_id": "alarm-uuid-1234",
  "asset_id": "reactor_01",
  "type": "overpressure",
  "severity": "CRITICAL",
  "state": "ACKNOWLEDGED",
  "classification": "ALARM",
  "raised_at": "2025-01-10T15:30:00.123Z",
  "acked_at": "2025-01-10T15:32:00Z",
  "acked_by": "alice@example.com",
  "cleared_at": null,
  "suppressed_at": null,
  "suppressed_by": null,
  "value": 205,
  "threshold": 200,
  "unit": "psi",
  "message": "Pressure exceeded critical threshold",
  "escalation_policy_id": null,
  "metadata": {
    "adapter_id": "validator",
    "deployment_id": "gateway-rig-alpha"
  }
}
```

**CLEARED**, same full record, `state`/`cleared_at`/`value` updated:
```json
{
  "alarm_id": "alarm-uuid-1234",
  "asset_id": "reactor_01",
  "type": "overpressure",
  "severity": "CRITICAL",
  "state": "CLEARED",
  "classification": "ALARM",
  "raised_at": "2025-01-10T15:30:00.123Z",
  "acked_at": "2025-01-10T15:32:00Z",
  "acked_by": "alice@example.com",
  "cleared_at": "2025-01-10T15:45:00Z",
  "suppressed_at": null,
  "suppressed_by": null,
  "value": 198,
  "threshold": 200,
  "unit": "psi",
  "message": "Pressure returned to normal range",
  "escalation_policy_id": null,
  "metadata": {
    "adapter_id": "validator",
    "deployment_id": "gateway-rig-alpha"
  }
}
```

### Log Entry (Recent Runtime Logs)

This is **not** a Kafka message; there is no `logs.*` topic. Gateway
runtime logs are captured in a bounded in-memory ring buffer
(`gateway_runtime/logging_utils.py`, `RecentLogBufferHandler`) and the most
recent entries are transported to the control plane inside the gateway's
periodic HTTP heartbeat payload, from which the Logs page reads them. The
real per-entry shape (`_payload_for_record`) is flatter than a structured
event message and has no `classification`, `log_level`, or `details` field:

```json
{
  "timestamp": "2025-01-10T12:05:00.456Z",
  "level": "WARNING",
  "logger": "adapters.modbus_tcp",
  "component": "modbus_tcp",
  "message": "Modbus read timeout, retrying...",
  "gateway_id": "gateway-rig-alpha"
}
```

`exception` is added only when the log record carries exception info
(a formatted traceback string). There is no dedicated field for structured
details like host/port/retry count; that context has to be in the free-text
`message` itself.

### Aggregated Telemetry (1-minute window)

Matches `schemas/telemetry_aggregate.avsc` (`gateway_runtime/aggregator.py`).
`quality_summary` has a fourth sample bucket, `uncertain_samples`, alongside
good/suspect/bad; `metadata` also carries `resolution` (`"1s"`/`"1min"`) and
`gateway_id`:

```json
{
  "asset_id": "reactor_01",
  "parameter": "temperature",
  "unit": "celsius",
  "classification": "TELEMETRY_AGGREGATE",
  "window_start": "2025-01-10T10:00:00Z",
  "window_end": "2025-01-10T10:01:00Z",
  "aggregates": {
    "avg": 342.5,
    "min": 341.8,
    "max": 343.2,
    "stddev": 0.41,
    "count": 600,
    "p50": 342.4,
    "p95": 343.0,
    "p99": 343.1
  },
  "quality_summary": {
    "good_samples": 598,
    "suspect_samples": 1,
    "uncertain_samples": 1,
    "bad_samples": 0,
    "pct_good": 99.67
  },
  "metadata": {
    "aggregation_version": "1.0.0",
    "resolution": "1min",
    "source_topic": "telemetry.clean",
    "gateway_id": "gateway-rig-alpha"
  }
}
```

Note `source_topic` here is `telemetry.clean` (the validated stream the
aggregator actually consumes), not `telemetry.raw`.

---

**End of Data Flow Documentation**

For architecture details, see [ARCHITECTURE.md](ARCHITECTURE.md).
For deployment, see DEPLOYMENT.md.
