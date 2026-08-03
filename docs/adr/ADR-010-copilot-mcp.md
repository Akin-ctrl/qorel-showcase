# ADR-010: Copilot Tools-First Approach

**Status**: Accepted, deferred until core production-readiness gates are closed
**Date**: 2026-01-29  
**Last Updated**: 2026-05-30
**Decision**: Expose MCP-compatible tools, make built-in Copilot optional

---

## Context

AI assistants can help operators:
- Diagnose issues
- Suggest configurations
- Analyze DLQ patterns
- Generate adapter templates

However, customers may already have their own AI agents, RAG systems, or ML platforms. Multiple standalone agents create confusion.

Current implementation note: this ADR records the intended AI/tooling direction,
not a shipped subsystem. Qorel should not resume copilot/MCP
implementation until gateway onboarding, physical-device verification,
topology, and failure-path verification are stronger.

## Options Considered

### Option A: Built-in Copilot Only
- Chat UI in Qorel dashboard
- Proprietary integration

**Pros**: Seamless experience for Qorel-only users  
**Cons**: Doesn't integrate with customer's other AI systems, agent sprawl

### Option B: API Only (No AI)
- REST API for all operations
- Customers build their own AI layer

**Pros**: Maximum flexibility  
**Cons**: Poor UX, every customer reinvents the wheel

### Option C: Tools-First (MCP)
- Expose capabilities as MCP-compatible tools
- External agents call Qorel tools
- Optional built-in Copilot uses same tools

**Pros**: Integrates with any agent, no agent sprawl, consistent capabilities  
**Cons**: MCP is relatively new standard

## Decision

**Option C: Tools-first with MCP**

Qorel should expose MCP-compatible tools. Built-in Copilot remains an
optional convenience layer that would use the same tools internally.

## Rationale

1. **No agent sprawl**: Customer has one orchestrating agent, many tools
2. **Integration**: Qorel becomes a capability, not a silo
3. **Flexibility**: Works with any agent framework (LangChain, AutoGen, custom)
4. **Consistency**: Built-in and external agents have identical capabilities
5. **Future-proof**: MCP is emerging standard from Anthropic

## Architecture

```
[Customer's Orchestrating Agent]
         ↓
    [MCP Protocol]
         ↓
   ┌─────┴─────┬──────────┬──────────┐
   ↓           ↓          ↓          ↓
[Qorel] [Data     [RAG       [ML
 Tools]       Platform]  System]   Model]
```

## Tool Categories

### Read/Query Tools (Always Allowed)

| Tool | Purpose |
|------|---------|
| `list_gateways` | List all gateways with status |
| `get_gateway_details` | Full details for one gateway |
| `get_adapter_status` | Adapter health and metrics |
| `get_sink_status` | Sink health and lag |
| `get_logs` | Query logs with filters |
| `get_metrics` | Query metrics |
| `get_alarms` | List alarms |
| `get_dlq_messages` | Query DLQ |
| `get_config` | Read config for any component |
| `get_schemas` | List schemas |

### Analysis Tools (Read-Only)

| Tool | Purpose |
|------|---------|
| `diagnose_gateway` | Analyze health, suggest fixes |
| `analyze_dlq` | Pattern analysis on failed messages |
| `explain_alarm` | Context and actions for alarm |
| `compare_configs` | Diff two versions |

### Action Tools (Require Confirmation)

| Tool | Purpose |
|------|---------|
| `suggest_config_change` | Returns diff, does not apply |
| `apply_config_change` | Apply with approval token |
| `restart_component` | Restart adapter/sink |
| `approve_dlq_messages` | Reprocess DLQ messages |
| `acknowledge_alarm` | Ack an alarm |
| `generate_adapter_template` | Scaffold new adapter code |
| `generate_sink_template` | Scaffold new sink code |

## Safety Boundaries

### Hard Blocks (Never Allowed)

- Delete gateway
- Delete sink data
- Modify production without review
- Access credentials directly
- Execute arbitrary code
- Disable authentication
- Self-approve changes

### Soft Blocks (Require Human Confirmation)

- Deploy new adapter/sink
- Change validation rules
- Update gateway version
- Modify alarm thresholds
- Restart components

### Audit Trail

Every tool invocation logged:

```json
{
  "timestamp": "2026-01-29T10:00:00Z",
  "user": "john@example.com",
  "tool": "apply_config_change",
  "target": "adapter-modbus-01",
  "input": {...},
  "output": {...},
  "approved_by": "john@example.com"
}
```

## Built-in Copilot

For users who only use Qorel:
- Chat UI in dashboard
- Calls the **same MCP tools** internally
- Can be disabled in settings
- Same safety boundaries apply

## MCP Server Deployment

```yaml
# docker-compose.yml
services:
  mcp-server:
    image: qorel/mcp-server:1.0.0
    ports:
      - "3000:3000"
    environment:
      CONTROL_PLANE_URL: http://control-plane:8000
      AUTH_TOKEN: ${MCP_AUTH_TOKEN}
```

## Consequences

### Positive
- Integrates with customer's AI ecosystem
- No agent sprawl
- Consistent capabilities internal/external
- Clear safety boundaries

### Negative
- MCP is emerging (may evolve)
- Must maintain MCP server component

### Mitigations
- Abstract tool definitions from protocol
- Built-in Copilot as fallback
- Version MCP server independently

## Related Decisions
- [ADR-007: Authentication](ADR-007-authentication.md)
