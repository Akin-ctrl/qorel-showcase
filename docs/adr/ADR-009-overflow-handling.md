# ADR-009: Overflow Handling

**Status**: Accepted  
**Date**: 2026-01-29  
**Decision**: Tiered overflow with priority-based eviction

---

## Context

Edge gateways have limited disk space (typically 100-500GB). When sinks are offline for extended periods:
- The gateway-local Kafka-compatible broker fills up
- New data cannot be written
- Risk of data loss

We needed a strategy for handling disk exhaustion.

## Options Considered

### Option A: Block Producers
- Stop accepting writes when full
- Adapters block until space available

**Pros**: Simple, no data loss of existing data  
**Cons**: Current data lost, adapters may crash

### Option B: Oldest-First Eviction
- Delete oldest data to make room
- FIFO regardless of content

**Pros**: Simple, preserves recent data  
**Cons**: May delete critical alarms

### Option C: Priority-Based Eviction
- Assign priority to topics
- Evict lowest-priority first

**Pros**: Protects critical data  
**Cons**: More complex, priority tuning needed

### Option D: Tiered Overflow
- Multiple strategies applied in sequence
- Compress → Downsample → Evict → Block

**Pros**: Maximizes data preservation  
**Cons**: Most complex

## Decision

**Option D: Tiered overflow with priority-based eviction**

## Rationale

1. **Compress first**: Saves 50-70% space with minimal information loss
2. **Downsample before delete**: Keep aggregates, discard raw
3. **Priority eviction**: Alarms are never deleted before telemetry
4. **Block as last resort**: Better than crashing

## Tier Strategy

| Disk Usage | Action |
|------------|--------|
| 0-70% | Normal operation |
| 70-80% | **Alert**. Compress old broker segments where supported. |
| 80-90% | **Aggressive downsample**. Delete raw telemetry older than 1 hour, keep aggregates. |
| 90-95% | **Priority eviction**. Delete oldest data, lowest priority first. |
| 95%+ | **Block producers**. Critical alert. Stop accepting new data. |

## Topic Priority

| Priority | Topics | Eviction Order |
|----------|--------|----------------|
| Critical | `alarms.*`, `dlq.*` | Never evicted |
| High | `events.*`, `telemetry.1min` | Last to be evicted |
| Medium | `telemetry.1s` | After low priority |
| Low | `telemetry.raw` | First to be evicted |

**Key rule**: Alarms are NEVER evicted. If disk is 95% full and only alarms remain, block producers.

**Clarification (2026-07-08)**: "Never evicted" describes exemption from the tiered *disk-pressure* eviction above, it is not a promise of unbounded retention. `dlq.*` topics separately carry a fixed 7-day `retention.ms` (ADR-004 Mitigations), applied independently of disk usage, so a sustained bad-data loop still has a ceiling even though disk-pressure eviction will never be the thing that trims it. `alarms.*` has no such backstop today; alarm volume is expected to stay low enough that this hasn't been needed.

## Implementation

```python
# Gateway Runtime disk monitor (runs every 60s)
def check_disk_overflow():
    usage = get_disk_usage_percent()
    
    if usage < 70:
        return  # Normal
    
    if usage < 80:
        alert("Disk usage at 70%+, compressing")
        compress_old_segments()
        return
    
    if usage < 90:
        alert("Disk usage at 80%+, downsampling")
        delete_raw_older_than(hours=1)
        return
    
    if usage < 95:
        alert("Disk usage at 90%+, evicting low priority")
        evict_by_priority(target_percent=85)
        return
    
    # 95%+
    critical_alert("Disk full, blocking producers")
    block_kafka_producers()
```

## Overflow Event

When data is evicted, emit an event for audit. The shape below reflects
what's actually measurable given how eviction works (a `retention.ms`
config change on the topic, reclaimed asynchronously by the broker's log
cleaner) rather than the originally-documented shape, which implied
precise before/after knowledge this mechanism doesn't have:

```json
{
  "event_type": "buffer_overflow",
  "timestamp": "2026-01-29T10:00:00Z",
  "topic": "telemetry.raw",
  "action": "priority_evicting",
  "bytes_freed": 5368709120,
  "oldest_evicted": null,
  "newest_evicted": "2026-01-29T09:55:00Z"
}
```

- `bytes_freed`: a real disk-usage delta measured immediately before and
  after applying the retention change (`gateway_runtime/overflow.py`,
  `_used_bytes`/`_bytes_freed`). Since broker log-segment cleanup runs
  asynchronously, this is frequently `0` on the triggering event, the
  actual reclaim shows up as reduced disk usage on a later poll rather than
  synchronously with the config change. `0` is a legitimate "not yet
  reclaimed" reading, not a placeholder; only unreadable disk stats produce
  `null`.
- `newest_evicted`: the retention cutoff just applied (`now - retention_ms`)
 , the most recent timestamp any evicted message could have had.
- `oldest_evicted`: left `null`. The true oldest evicted timestamp isn't
  knowable without reading broker segment-file metadata directly, which
  this mechanism doesn't do, reporting a guessed value would be worse than
  reporting nothing.

This event is written to `dlq.overflow` (high priority, not evicted).

## Consequences

### Positive
- Maximizes data preservation
- Critical data (alarms) protected
- Clear escalation path
- Audit trail of evictions

### Negative
- Complex logic
- Raw data may be lost in overflow scenarios

### Mitigations
- UI shows overflow events
- Alerts at each tier threshold
- Recommend adequate disk sizing

## Disk Sizing Guidance

| Data Rate | Offline Duration | Recommended Disk |
|-----------|------------------|------------------|
| 1 MB/s | 1 day | 100 GB |
| 1 MB/s | 7 days | 700 GB |
| 10 MB/s | 1 day | 1 TB |

## Related Decisions
- [ADR-001: Edge Buffering](ADR-001-edge-buffering.md)
- [ADR-008: Failure Modes](ADR-008-failure-modes.md)
