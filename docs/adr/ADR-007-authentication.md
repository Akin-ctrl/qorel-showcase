# ADR-007: Authentication Model

**Status**: Accepted  
**Date**: 2026-01-29 (amended 2026-07-09: gateway JWT now requires Ed25519
proof-of-possession, not `gateway_id` alone, see Option D; amended 2026-07-27:
gateway identity revocation model added, not yet implemented, see Gateway
Identity Revocation)  
**Decision**: enrollment-gated JWT for gateways, bound to an Ed25519
device-identity keypair; built-in users today with account lockout and
server-side logout invalidation; optional OAuth/OIDC later

---

## Context

Two types of identities need authentication:
1. **Gateways** - machines connecting to Control Plane
2. **Users** - humans accessing UI and API

Each has different requirements for credential lifecycle and management.

## Options Considered

### Gateway Authentication

#### Option A: API Keys
- Static keys per gateway
- Simple to implement

**Pros**: Simple  
**Cons**: No expiry, revocation requires key rotation, no identity claims

#### Option B: mTLS (Mutual TLS)
- Certificate-based authentication
- Gateway presents client certificate

**Pros**: Strong security, no shared secrets  
**Cons**: Certificate management complexity, rotation overhead

#### Option C: JWT Tokens
- Gateway receives JWT on registration
- Token includes gateway identity and permissions

**Pros**: Standard, contains claims, expirable, revocable  
**Cons**: Token must be stored securely

#### Option D: JWT + Ed25519 proof-of-possession (chosen, added 2026-07-09)
- Gateway generates an Ed25519 keypair on first use; the private key never
  leaves the device. The public key is bound to the `gateway_id` at
  enrollment (trust-on-first-enroll, not a CA-backed identity).
- Every `/token` and `/token/renew` request must include a fresh signature
  over `gateway_id|purpose|timestamp|nonce`, verified against the stored
  public key, with a clock-skew window and single-last-nonce reuse check.

**Pros**: Closes the actual gap in Option C as shipped (a JWT was mintable by
anyone who merely *knew* an approved `gateway_id`, which is not a secret,
it's visible in the UI, audit logs, and hostnames); no shared secret to leak
or rotate; no PKI/CA operational weight.  
**Cons**: TOFU, not CA-backed, a compromised enrollment token used before
the legitimate device enrolls can still bind the wrong key (mitigated by:
the *first* key bound to a `gateway_id` sticks, and re-enrollment with a
different key is rejected with `409` rather than silently re-keying).

### User Authentication

#### Option A: Built-in Only
- Username/password stored in Control Plane DB
- Session tokens

**Pros**: No external dependencies  
**Cons**: Another password to manage, no SSO

#### Option B: OAuth/OIDC Only
- Delegate to external identity provider
- No local user management

**Pros**: SSO, enterprise-ready  
**Cons**: Requires external IdP, complex for small deployments

#### Option C: Built-in + Optional OAuth
- Default: built-in username/password
- Optional: integrate with OAuth2/OIDC providers

**Pros**: Works out of box, scales to enterprise  
**Cons**: Two auth paths to maintain

## Decision

- **Gateways**: admin-managed enrollment token creates a pending gateway;
  operator approval gates long-lived JWT issuance; the gateway additionally
  proves possession of an Ed25519 private key (bound to the `gateway_id` at
  first enrollment, TOFU) on every token issue/renew, Option D above.
- **Users**: built-in authentication today; optional OAuth2/OIDC remains the
  enterprise direction. Account lockout (5 failed attempts / 15 minutes) and
  server-side logout invalidation (`token_valid_after` watermark) are shipped.

## Rationale

### Gateway JWT

1. **Long validity**: 1-year tokens reduce renewal frequency for remote gateways
2. **Auto-renew**: Gateway renews before expiry when connected
3. **Revocable**: Control Plane can invalidate tokens immediately
4. **Claims**: Token contains gateway ID, permissions, tenant (future)
5. **Bound to device identity**: knowledge of `gateway_id` alone was never
   meant to be sufficient, it isn't a secret. Requiring an Ed25519 signature
   closes that gap without full mTLS/PKI operational weight.

### User Auth

1. **Built-in for demo**: Works immediately, no setup required
2. **OAuth for enterprise**: Integrate with existing identity providers
3. **Progressive complexity**: Start simple, add SSO when needed

## Gateway Registration Flow

```
1. Operator creates a gateway enrollment token in the UI / control-plane API
   {
     "enrollment_id": "enroll-factory-north",
     "token": "qre_..."
   }
2. Gateway boots with CONTROL_PLANE_ENROLLMENT_TOKEN and its stable gateway ID,
   generating (or loading) its Ed25519 keypair, private key never leaves
   the device
3. Gateway calls POST /api/v1/gateways/enroll
   {
     "gateway_id": "gw-factory-north",
     "enrollment_token": "qre_...",
     "hostname": "gateway-01.local",
     "public_key": "<base64 Ed25519 public key>"
   }
   Control Plane stores the public key against the gateway (or rejects with
   409 if a different key is already on file for this gateway_id)
4. Control Plane records the gateway as pending
5. Operator approves the pending gateway
6. Gateway calls POST /api/v1/gateways/token, signing
   "gw-factory-north|token|<timestamp>|<nonce>" with its private key:
   {
     "gateway_id": "gw-factory-north",
     "timestamp": 1782000000,
     "nonce": "<random>",
     "signature": "<base64 Ed25519 signature>"
   }
   Control Plane verifies the signature against the stored public key,
   the timestamp is within a 120s clock-skew window, and the nonce doesn't
   match the immediately preceding request, before issuing a token.
7. Control Plane returns JWT:
   {
     "token": "eyJhbGc...",
     "expires_at": "2027-01-29T00:00:00Z",
     "gateway_id": "gw-factory-north"
   }
8. Gateway stores token securely
9. All subsequent API calls include: Authorization: Bearer <token>
```

## Token Renewal

The implemented refresh cadence is proactive and frequent rather than
timed against expiry, and it re-issues rather than renews in place:

```
While online, checked roughly every 24 hours:
1. If the in-memory token is more than ~1 day old, the gateway calls
   POST /api/v1/gateways/token again (the same enrollment-flow endpoint,
   not a dedicated /renew path) and receives a fresh ~365-day token.
2. Gateway replaces the old token in memory.

This is a rolling refresh, not an expiry-driven one: at the moment of
refresh the previous token typically still has ~364 days of validity left.
The dedicated POST /api/v1/gateways/token/renew endpoint exists
server-side but is not currently called by the gateway runtime.

If a refresh attempt fails:
- Gateway continues operating on its existing (still-valid) token
- Retries on the next ~24-hour check
- Gateway tracks consecutive refresh failures and reports the count in its
  regular heartbeat metrics (the same channel CPU/memory pressure already
  use), chosen because a refresh failure specifically doesn't block other
  authenticated calls, so the still-working heartbeat path can carry the
  signal even while refresh itself is failing
- Control Plane raises `alarm_type = "gateway.token_refresh_failing"` once
  consecutive failures exceed 3, and clears it the moment a refresh
  succeeds, implemented, closed remediation ledger item D3
```

This was a deliberate choice, not drift: daily refresh is simple and
self-healing (a missed refresh just retries the next day, with no risk since
the token isn't near expiry), and switching to a near-expiry-only renewal
would trade that resilience margin for fewer control-plane round-trips. Both
are reasonable; this ADR documents the cadence that is actually running.

## Consequences

### Positive
- Gateways work reliably with long-lived tokens
- Auto-renewal prevents expiry issues
- Built-in auth enables quick demos
- OAuth enables enterprise integration

### Negative
- Token storage on gateway must be secure
- Two auth systems to maintain

### Mitigations
- Store tokens with restricted file permissions
- Document secure token storage
- Clear separation between auth paths in code

## Gateway Identity Revocation

*Added 2026-07-27 (PRE_AI_ROADMAP item G4a). Not yet implemented, Stage 1.*

### Gap

There is no revocation primitive. `Gateway` has no `token_valid_after`
watermark, that exists only on `User`, so a JWT already issued to a gateway
stays valid until natural expiry with no way to cut it short. The only available
action is `DELETE /gateways/{gateway_id}`, a hard `db.delete(gateway)`, which
destroys the row *and* the TOFU public-key binding. The same device can then
re-enroll with the same site token and a fresh keypair.

Deletion is not revocation. It is amnesia. This is the gap an IEC 62443 review
raises first, and it undermines the site-token-only decision recorded in
PRE_AI_ROADMAP G4, site tokens are acceptable *because* TOFU binds the first
key, and hard delete throws that binding away.

### Decision

Three distinct operations rather than one overloaded flag:

| Operation | Effect | Risk |
|---|---|---|
| **Revoke** | Invalidate live tokens via `token_valid_after`; refuse new token issue/renew; refuse re-enrollment of the `gateway_id`. Row, audit history, and **public-key binding all preserved**. | Low, reversible |
| **Un-revoke** | Device resumes normal operation. | Low **by construction**, see below |
| **Re-key** | Clear `public_key` so a fresh TOFU bind may occur. For a compromised key or replaced hardware. | **High**, separate permission, loud audit event |

**Retaining the key binding on revoke is what makes reversibility safe.** Only
hardware still holding the matching private key can resume after an un-revoke.
Someone who merely knows the `gateway_id`, visible in the UI, audit logs, and
hostnames, gains nothing, because they cannot sign the proof-of-possession that
`/token` and `/token/renew` require.

Reversibility therefore adds no meaningful attack surface beyond what the
`gateways:manage` permission already grants, since that permission can approve
new gateways regardless.

**Re-key is deliberately separate from un-revoke.** Bundling them would mean
every routine un-revoke silently reopens the key-binding window, turning the
common, low-risk operation into the rare, high-risk one.

### `DELETE` semantics

`DELETE /gateways/{gateway_id}` is narrowed:

- **Never approved** → deletes as today. This is abandoned-enrollment cleanup and
  loses nothing of value.
- **Ever approved** → `409`, directing the operator to revoke instead.

Retirement becomes `revoke(reason="decommissioned")`. Anything that ever carried
real data keeps its audit trail and its key binding.

### Fields

`Gateway` gains `revoked_at`, `revoked_by`, `revoked_reason`, and
`token_valid_after`. Folded into the `0001_init.py` baseline per the pre-ship
squash rule.

## Related Decisions
- [ADR-006: Gateway Autonomy](ADR-006-gateway-autonomy.md)
- [ADR-015: Historian Placement and Intelligence Locality](ADR-015-historian-and-intelligence-placement.md)
