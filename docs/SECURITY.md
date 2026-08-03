# Security Model & Best Practices

**Security architecture and hardening notes for Qorel.**

> Current status: built-in user authentication, gateway JWTs, RBAC, audit
> logging, safer browser sessions, and secret-field handling exist today. OAuth,
> Vault-backed secret resolution, production TLS profiles, and deployment-time
> network hardening are still future production-packaging work.

---

## Table of Contents

1. [Security Principles](#security-principles)
2. [Authentication](#authentication)
3. [Authorization (RBAC)](#authorization-rbac)
4. [Network Security](#network-security)
5. [Data Encryption](#data-encryption)
6. [Secrets Management](#secrets-management)
7. [Audit & Compliance](#audit--compliance)
8. [Threat Model](#threat-model)
9. [Security Checklist](#security-checklist)
10. [Incident Response](#incident-response)

---

## Security Principles

### Defense in Depth

Multiple layers of security controls:
```
Physical Security → Network Security → Application Security → Data Security
```

### Least Privilege

Every component should have minimum required permissions:
- Adapters should only write to assigned Kafka-compatible topics
- Sinks should only read from specific topics
- Gateway Runtime cannot access Control Plane database directly
- UI users have role-based access

### Zero Trust

Never trust, always verify:
- Hardened deployments should encrypt network traffic, including internal links where practical
- All API calls authenticated
- All configuration changes audited
- No implicit trust between components

---

## Authentication

### User Authentication (Control Plane UI/API)

**Implemented method**: built-in user auth with JWT-backed sessions.

**Future enterprise path**: OAuth2/OIDC can be added later, but it is not part
of the current completed security baseline.

**Flow**:
```
1. User logs in via UI
   ↓
2. Control API validates credentials (or delegates to IdP)
   ↓
3. API issues JWT token (expires in 12 hours)
   ↓
4. UI includes token in Authorization header
   ↓
5. API validates token on every request
```

**JWT Structure**:
```json
{
  "sub": "alice@example.com",
  "scope": "user",
  "roles": ["Engineer"],
  "permissions": ["adapters:create", "sinks:update", "deployments:create"],
  "exp": 1704988800,
  "iat": 1704902400,
  "iss": "qorel-api"
}
```
Note: access is scoped by role/permission only. There is no per-gateway
access-control field on the token today, RBAC is flat and global (see
[Permission Matrix](#permission-matrix) below); per-gateway scoping is
not implemented.

**Token refresh**:
```bash
# UI automatically refreshes token before expiration
POST /api/v1/auth/refresh
Authorization: Bearer <current-token>

Response:
{
  "token": "<new-token>",
  "expires_at": "2025-01-12T00:00:00Z"
}
```

### Gateway Authentication (Machine-to-Machine)

**Current shipped method**: admin-created enrollment tokens let a gateway enroll
as `pending`; an operator approves it, then the control plane issues a
long-lived gateway JWT for config polling, heartbeat, and gateway-side actions.
Possession of the `gateway_id` alone is no longer sufficient to mint a token,
every `/token` and `/token/renew` request must additionally prove possession
of the gateway's private key (see "Device-identity keypair" below).

**Manual fallback**: operators can still create and approve a gateway record
directly in controlled local/dev environments before the gateway requests a
token. Doing so does not set a public key on the row; an admin must supply one
(via `PUT /api/v1/gateways/{gateway_id}` with `public_key`) before that
gateway can pass proof-of-possession, or the gateway must instead go through
the real `/enroll` flow so it self-registers its own key.

**Still planned, not shipped**: TPM-backed attestation and full mTLS/PKI are
reserved for a higher-assurance enterprise tier, deliberately not built as
part of the baseline device-identity fix (unnecessary weight for the
threat model this fixes).

#### Device-identity keypair (TOFU, bound to enrollment token)

On first use, the gateway generates an Ed25519 keypair
(`gateway_runtime/identity.py`) and persists the private key locally
(`/data/keys/gateway_identity.pem`, 0600 file / 0700 directory permissions,
never transmitted). The public key is sent once, at enrollment; the control
plane stores it against the gateway row and rejects any later enrollment
attempt for the same `gateway_id` that presents a *different* key (`409`),
closing the "leaked enrollment token used to hijack an already-claimed
gateway_id" gap. This is trust-on-first-enroll (TOFU), not a CA-backed
identity, the security property is "the same physical device that enrolled
is the one minting tokens," not "a third party has cryptographically vouched
for this device."

Every `/token` and `/token/renew` request must include a fresh Ed25519
signature over `f"{gateway_id}|{purpose}|{timestamp}|{nonce}"`, where
`purpose` is `"token"` or `"token_renew"` (binding a signature to one
endpoint so a captured `/token` signature can't be replayed against
`/token/renew`). The control plane verifies the signature against the stored
public key and additionally rejects the request if the timestamp is more
than 120 seconds from server time, or if the nonce matches the immediately
preceding request's nonce (`Gateway.last_token_nonce`), a lightweight replay
guard, not a full nonce-cache store.

#### Current shipped flow

1. Admin creates an enrollment token in the Gateways UI.
2. Gateway runtime starts with `CONTROL_PLANE_URL`,
   `CONTROL_PLANE_GATEWAY_ID`, and `CONTROL_PLANE_ENROLLMENT_TOKEN`, and
   generates (or loads) its Ed25519 keypair.
3. Gateway runtime calls `POST /api/v1/gateways/enroll` with its public key.
4. Control plane creates or updates the gateway as `pending`, storing the
   public key (or rejecting with `409` if a different key is already on file).
5. Operator reviews and approves the pending gateway in the UI.
6. Gateway runtime requests a gateway token, signing the request with its
   private key.
7. Gateway runtime polls config and posts heartbeat/health back to the control
   plane.

Self-registration is **not** enabled in the currently shipped implementation.
`POST /api/v1/gateways/register` is intentionally disabled today, so any doc or
example that assumes zero-touch self-registration should be treated as a
planned enrollment direction rather than current behavior.

#### Current enrollment bootstrap example

```bash
# 1. Admin creates an enrollment token in the Gateways UI.

# 2. Gateway runtime enrolls with that token, including its Ed25519 public key.
POST /api/v1/gateways/enroll
{
  "gateway_id": "gateway-rig-alpha-001",
  "enrollment_token": "qre_...",
  "hostname": "rig-alpha.local",
  "hardware_info": {
    "site": "Offshore Rig Alpha",
    "hardware": "Dell Edge Gateway 3200"
  },
  "public_key": "<base64 Ed25519 public key>"
}

# 3. Operator approves the pending gateway.
POST /api/v1/gateways/{gateway_id}/approve

# 4. Gateway runtime requests its machine token, signing
#    "gateway-rig-alpha-001|token|<timestamp>|<nonce>" with its private key.
POST /api/v1/gateways/token
{
  "gateway_id": "gateway-rig-alpha-001",
  "timestamp": 1782000000,
  "nonce": "<random>",
  "signature": "<base64 Ed25519 signature>"
}
```

#### Future stronger identity proof

```bash
# Gateway generates certificate on first boot
openssl req -new -x509 -days 3650 \
  -keyout /etc/qorel/gateway.key \
  -out /etc/qorel/gateway.crt \
  -subj "/CN=gateway-rig-alpha-001"

# Future signed enrollment / registration with Control API
curl -X POST https://api.qorel.example.com/api/v1/gateways/register \
  -H "Content-Type: application/json" \
  -d '{
    "gateway_id": "gateway-rig-alpha-001",
    "certificate": "<PEM-encoded-cert>",
    "metadata": {
      "location": "Offshore Rig Alpha",
      "hardware": "Dell Edge Gateway 3200"
    }
  }'

# Admin approves in UI
# Control API issues long-lived JWT (1 year expiration)
{
  "gateway_id": "gateway-rig-alpha-001",
  "token": "eyJhbGciOi...",
  "expires_at": "2026-01-10T00:00:00Z"
}
```

This full mTLS/PKI-issued-certificate model remains a *further* future step
beyond the Ed25519 TOFU keypair already shipped above, it adds a trusted CA
vouching for device identity, rather than trust-on-first-enroll.

#### Ongoing authentication
```bash
# Gateway includes token in every API call
GET /api/v1/gateways/gateway-rig-alpha-001/config/current
Authorization: Bearer <gateway-token>
```

#### Token rotation
```bash
# Renewal path exposed by the control plane, also requires a fresh signature,
# same as the initial token request, with purpose="token_renew".
POST /api/v1/gateways/token/renew
Authorization: Bearer <current-token>
{
  "timestamp": 1782000000,
  "nonce": "<random>",
  "signature": "<base64 Ed25519 signature over gateway_id|token_renew|timestamp|nonce>"
}
```

### Adapter Authentication to Kafka (Current)

The local dev stack and the current gateway runtime use plaintext Kafka connections with no authentication. `KafkaProducer` is initialized without SASL parameters.

### Production Hardening (Not Yet Implemented)

The following SASL/SCRAM-SHA-512 configuration is planned for production hardening but is not yet wired in the codebase.

**Planned method**: SASL/SCRAM-SHA-512

**Server-side configuration**:
```yaml
listeners=SASL_SSL://0.0.0.0:9093
security.inter.broker.protocol=SASL_SSL
sasl.mechanism.inter.broker.protocol=SCRAM-SHA-512
sasl.enabled.mechanisms=SCRAM-SHA-512
```

**Planned adapter credentials**, ⚠️ **not implemented.** `${secret:...}` is a
proposed reference syntax; the gateway runtime does not resolve it today, and a
config containing it will be passed through to the adapter verbatim. Shown to
record the intended design, not as a usable example.

```json
{
  "output": {
    "kafka_bootstrap": "kafka.example.com:9093",
    "security_protocol": "SASL_SSL",
    "sasl_mechanism": "SCRAM-SHA-512",
    "sasl_username": "${secret:kafka_adapter_user}   <- NOT RESOLVED TODAY",
    "sasl_password": "${secret:kafka_adapter_password} <- NOT RESOLVED TODAY"
  }
}
```

**Credential creation**:
```bash
kafka-configs --bootstrap-server kafka:9093 \
  --alter --add-config 'SCRAM-SHA-512=[password=<secure-password>]' \
  --entity-type users --entity-name adapter-user
```

---

## Authorization (RBAC)

### Roles & Permissions

| Role | Permissions | Use Case |
|------|-------------|----------|
| **Viewer** | Read dashboards, view configs | Business analysts, executives |
| **Operator** | Viewer + acknowledge alarms, view logs | Operations team, 24/7 monitoring |
| **Engineer** | Operator + create/edit adapters, sinks, deployments; test connections | Industrial engineers, integrators |
| **Admin** | Engineer + manage gateways, users, deploy configs | System administrators |
| **Copilot** | Read topology, suggest changes (requires approval) | AI assistant, not yet built; see [ADR-010](adr/ADR-010-copilot-mcp.md) |

### Permission Matrix

The real permission strings are `adapters:*`, `sinks:*`, `deployments:*`,
`validation:update`, and `configs:*` (see `core/security.py`), the
"pipeline" object was retired in favor of separately reusable adapters and
sinks composed into deployments, and the permission names below reflect that.

| Resource | Viewer | Operator | Engineer | Admin |
|----------|--------|----------|----------|-------|
| View dashboards | ✓ | ✓ | ✓ | ✓ |
| View configs | ✓ | ✓ | ✓ | ✓ |
| Acknowledge alarms | ✗ | ✓ | ✓ | ✓ |
| View logs | ✗ | ✓ | ✓ | ✓ |
| Create adapters / sinks | ✗ | ✗ | ✓ | ✓ |
| Create / edit deployments | ✗ | ✗ | ✓ | ✓ |
| Delete adapters / sinks / deployments | ✗ | ✗ | ✗ | ✓ |
| Deploy configs | ✗ | ✗ | ✗ | ✓ |
| Manage gateways | ✗ | ✗ | ✗ | ✓ |
| Manage users | ✗ | ✗ | ✗ | ✓ |

### Kafka Topic ACLs

**Per-adapter isolation**:
```bash
# Adapter can only write to its assigned topics
kafka-acls --bootstrap-server kafka:9093 \
  --add --allow-principal User:adapter-modbus-001 \
  --operation Write \
  --topic telemetry.raw \
  --topic events.raw

# Adapter cannot read (write-only)
kafka-acls --bootstrap-server kafka:9093 \
  --add --deny-principal User:adapter-modbus-001 \
  --operation Read \
  --topic '*'
```

**Sink permissions**:
```bash
# Sink can only read from clean topics
kafka-acls --bootstrap-server kafka:9093 \
  --add --allow-principal User:sink-timescaledb \
  --operation Read \
  --topic telemetry.clean \
  --topic events.clean \
  --group sink-timescaledb-group
```

### Implementation (Control API)

```python
# control_plane/api/auth.py
from functools import wraps
from jose import jwt
from fastapi import HTTPException, Depends
from fastapi.security import HTTPBearer

security = HTTPBearer()

def get_current_user(token: str = Depends(security)):
    try:
        payload = jwt.decode(token.credentials, JWT_SECRET, algorithms=["HS256"])
        return User(
            email=payload["sub"],
            roles=payload["roles"],
            permissions=payload["permissions"]
        )
    except jwt.JWTError:
        raise HTTPException(status_code=401, detail="Invalid token")

def require_permission(permission: str):
    def decorator(func):
        @wraps(func)
        async def wrapper(*args, user: User = Depends(get_current_user), **kwargs):
            if permission not in user.permissions:
                raise HTTPException(status_code=403, detail=f"Missing permission: {permission}")
            return await func(*args, user=user, **kwargs)
        return wrapper
    return decorator

# Usage in API endpoints
@app.post("/api/v1/deployments")
@require_permission("deployments:create")
async def create_deployment(deployment: DeploymentCreate, user: User = Depends(get_current_user)):
    # Create deployment logic
    audit_log(user.email, "deployment.create", deployment.id)
    return {"status": "created"}
```

---

## Network Security

### Network Segmentation

```
┌─────────────────────────────────────────────┐
│ DMZ (Public)                                │
│ - Load Balancer                             │
│ - UI (HTTPS only)                           │
└─────────────┬───────────────────────────────┘
              │ (Firewall)
┌─────────────▼───────────────────────────────┐
│ Control Plane Network                       │
│ - Control API (private IP)                  │
│ - PostgreSQL (no external access)           │
└─────────────┬───────────────────────────────┘
              │ (Firewall)
┌─────────────▼───────────────────────────────┐
│ Data Plane / Destination Network            │
│ - Customer Kafka-compatible sink target     │
│   (VPN/private peering only, if configured) │
│ - Schema Registry                           │
└─────────────┬───────────────────────────────┘
              │ (VPN)
┌─────────────▼───────────────────────────────┐
│ Edge Networks                               │
│ - Gateways (outbound only)                  │
│ - Protocol adapters (isolated)              │
└─────────────────────────────────────────────┘
```

### Firewall Rules

**DMZ → Control Plane**:
```
Allow: TCP 443 (HTTPS) → Control API
Block: All other inbound
```

**Control Plane → Data Plane / Destinations**:
```
Allow: TCP 9093 (Kafka-compatible protocol over TLS, if using an external sink target)
Allow: TCP 8081 (Schema Registry)
Allow: TCP 5432 (PostgreSQL, internal only)
```

**Edge → Cloud** (outbound only):
```
Allow: TCP 443 → control.qorel.cloud (Control API)
Allow: TCP 9093 → customer-owned Kafka-compatible sink target, if configured
Allow: UDP 123 → pool.ntp.org (NTP)
Block: All inbound (no SSH, no admin ports)
```

**Edge local management** (restricted):
```
Allow: TCP 22 (SSH) from 192.168.1.0/24 (management VLAN only)
Allow: TCP 9090 (Prometheus metrics) from monitoring server
```

### Docker Socket Security

The gateway runtime mounts the host Docker socket (`/var/run/docker.sock`) to
launch and control adapter containers. **The Docker socket grants the equivalent
of root on the host machine.** Any process that can write to it can escape the
container.

**Current dev-stack posture**: the socket is mounted directly into the
`gateway_runtime` container. This is acceptable for a trusted local dev
environment where Docker runs as your own user.

**Current production posture (as of 2026-07-10):** gateway_runtime (and
therefore its Docker socket access) runs on each edge host, not the
control-plane server, `deploy/prod/docker-compose.edge.yml`. That file
still mounts the raw socket directly, deliberately: edge devices are
single-tenant, which narrows the blast radius a shared multi-tenant host
would have had. Whether `docker-socket-proxy` is still worth adding on top of
that is an open question, not a committed Phase 2 target, revisit if a
concrete multi-tenant edge scenario shows up. The pattern below is
illustrative, not something currently wired into either compose file:

```yaml
# Illustrative only, not present in deploy/prod/docker-compose.edge.yml today.
socket-proxy:
  image: tecnativa/docker-socket-proxy:0.3.0
  volumes:
    - /var/run/docker.sock:/var/run/docker.sock
  environment:
    CONTAINERS: 1
    IMAGES: 1
    NETWORKS: 1
    POST: 1        # allow POST (create/start/stop)
    DELETE: 1      # allow DELETE (remove containers)
    # All other categories default to 0 (denied)
  networks:
    - socket-proxy-net

gateway_runtime:
  environment:
    DOCKER_HOST: tcp://socket-proxy:2375
  networks:
    - socket-proxy-net
    - app-net
  # No longer mounts /var/run/docker.sock directly
```

Until the socket proxy is in place, treat the gateway runtime container as
having implicit host-level access and restrict who can deploy or modify it.

### VPN Configuration (for Edge → Cloud)

**IPSec tunnel**:
```bash
# Edge gateway
ipsec up qorel-tunnel

# Traffic routed through tunnel:
# - Kafka replication (9093)
# - Control API (443)

# Public internet bypass:
# - NTP (no sensitive data)
```

**WireGuard alternative**:
```ini
# /etc/wireguard/wg0.conf
[Interface]
PrivateKey = <gateway-private-key>
Address = 10.0.1.2/24

[Peer]
PublicKey = <cloud-public-key>
Endpoint = vpn.qorel.cloud:51820
AllowedIPs = 10.0.0.0/16
PersistentKeepalive = 25
```

---

## Data Encryption

### In Transit

**Production target: encrypted network traffic**:

| Connection | Method | Key Strength |
|------------|--------|--------------|
| UI ↔ Control API | HTTPS/TLS 1.3 | RSA 2048 / ECDSA P-256 |
| Gateway ↔ Control API | HTTPS/TLS 1.3 | RSA 2048 |
| Local broker ↔ External Kafka-compatible sink destination | Kafka over TLS (SASL_SSL) | AES-256 |
| Adapter ↔ Local Kafka-compatible broker | TLS in hardened deployments; plaintext in local dev | AES-256 when TLS is enabled |

**TLS configuration** (Control API):
```nginx
# nginx.conf
ssl_protocols TLSv1.3;
ssl_ciphers 'ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384';
ssl_prefer_server_ciphers on;
ssl_session_cache shared:SSL:10m;
ssl_session_timeout 10m;
ssl_stapling on;
ssl_stapling_verify on;
```

**Kafka-compatible broker TLS** (production packaging target):
```properties
listeners=SSL://0.0.0.0:9093
ssl.keystore.location=/var/private/ssl/kafka.server.keystore.jks
ssl.keystore.password=${KEYSTORE_PASSWORD}
ssl.key.password=${KEY_PASSWORD}
ssl.truststore.location=/var/private/ssl/kafka.server.truststore.jks
ssl.truststore.password=${TRUSTSTORE_PASSWORD}
ssl.client.auth=required
ssl.enabled.protocols=TLSv1.3
ssl.cipher.suites=TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384
```

### At Rest

**Kafka logs** (disk encryption):
```bash
# Linux LUKS encryption
cryptsetup luksFormat /dev/sdb
cryptsetup luksOpen /dev/sdb kafka_data
mkfs.ext4 /dev/mapper/kafka_data
mount /dev/mapper/kafka_data /var/lib/kafka/data
```

**Database** (PostgreSQL):
```bash
# Transparent Data Encryption (TDE)
# Or cloud-managed encryption (AWS RDS encryption, Azure SQL TDE)
```

**Edge gateway** (local buffering):
```bash
# Full disk encryption (on sensitive deployments)
cryptsetup luksFormat /dev/mmcblk0p2
# Auto-unlock on boot with TPM
```

---

## Secrets Management

### Current Shipped Behavior

Secret fields in adapter and sink configs are handled safely by the control plane:
- Fields marked `secret: true` in the catalog are stored separately from config, never returned in GET responses, and cleared on re-fetch.
- The UI shows a "stored securely" state indicator for secrets that are set but not returned.
- Secret values at rest are Fernet-encrypted (`ConfigSecret.ciphertext`) before being persisted, keyed by `QOREL_CONFIG_SECRET_KEY`. Key rotation is supported: `MultiFernet` decrypts against the current and previous key during a rotation window, and each row records a `key_version`. This is application-layer encryption, not plaintext, database-level encryption (e.g. disk/volume encryption) remains a separate, additional operator responsibility for defense in depth.

### Production Hardening (Not Yet Implemented)

The following secret resolution paths are **planned for production hardening but are not wired in the current codebase**. Do not document these as shipped behavior.

#### HashiCorp Vault Integration

`${vault:path#key}` interpolation in adapter/sink configs is the intended production secret reference syntax. The gateway runtime does not currently resolve these references. No `hvac`-based secret resolver exists in the gateway runtime today.

Intended architecture (not yet implemented):
```
Gateway Runtime → Vault Agent → HashiCorp Vault
  ↓ (inject resolved secrets)
Adapter / sink config
```

When this is implemented, adapter configs would reference secrets as follows.
⚠️ **Copying this into a real config today will not work**, the placeholder is
stored and forwarded literally, not resolved:

```json
{
  "username": "${vault:secret/qorel/opcua/plc-01#username}   <- NOT RESOLVED TODAY",
  "password": "${vault:secret/qorel/opcua/plc-01#password} <- NOT RESOLVED TODAY"
}
```

#### Orchestrator Secrets

Production packaging is still pending. The intended direction is that packaged deployments inject sensitive values through the deployment environment (K8s Secrets, Docker secrets), while Qorel configs reference placeholders rather than embedding values directly.

### Future: Orchestrator Secrets

Production packaging is still pending, so Qorel does not yet document a
supported orchestrator-secret manifest. The intended direction is simple:
packaged deployments should inject sensitive values through the deployment
environment, while Qorel configs should reference secrets rather than
carry plaintext credentials.

---

## Audit & Compliance

### Audit Log

**Every security-relevant event logged**, matching the real `audit_events` columns below, `actor_username`, `action`, `resource_type`, `resource_public_id`, `ip_address`, `created_at`, plus a free-form `details` JSON column for event-specific fields. Example of a real recorded event (`app/routers/auth.py`, `record_audit_event`):

```json
{
  "actor_username": "alice",
  "action": "auth.login",
  "resource_type": "user",
  "resource_public_id": "alice",
  "details": {},
  "ip_address": "192.168.1.100",
  "created_at": "2025-01-10T14:30:00.123Z"
}
```

**Configuration changes**:
```json
{
  "actor_username": "bob",
  "action": "deployment.activated",
  "resource_type": "deployment",
  "resource_public_id": "offshore_well_12",
  "details": {},
  "ip_address": "10.0.1.50",
  "created_at": "2025-01-10T15:00:00Z"
}
```

**Failed authentication is not currently audited.** `POST /api/v1/auth/token` increments `User.failed_login_attempts` on a bad password but does not call `record_audit_event()` on that path, only successful logins are recorded. There is no `authentication.failure` row to query. If failed-login auditing becomes a requirement (e.g. for brute-force detection or a specific compliance control), it needs to be added rather than assumed present.

### Storage

**Current implementation: PostgreSQL only.**

```sql
CREATE TABLE audit_events (
  id SERIAL PRIMARY KEY,
  actor_username VARCHAR(128),
  action VARCHAR(128) NOT NULL,
  resource_type VARCHAR(64) NOT NULL,
  resource_public_id VARCHAR(128) NOT NULL,
  details JSONB DEFAULT '{}',
  ip_address VARCHAR(45),
  created_at TIMESTAMPTZ NOT NULL,
  INDEX idx_actor_username (actor_username),
  INDEX idx_action (action),
  INDEX idx_resource_type (resource_type),
  INDEX idx_resource_public_id (resource_public_id),
  INDEX idx_created_at (created_at)
);
```

Every control-plane mutation (adapters, sinks, deployments, alerting rules,
escalation policies, users, auth events) is recorded here via
`record_audit_event()`. This is the durability guarantee that actually
exists today: a relational row per event, queryable and indexed.

An earlier version of this document additionally described a second,
append-only write to a Kafka topic (`audit_trail`, infinite retention, 3x
replication) as a "dual storage" durability layer. **That write was never
implemented**, there is no Kafka producer anywhere in the audit path. It
has been removed from this document rather than left as an unbuilt
durability claim. If tamper-evident, append-only audit storage becomes a
genuine requirement (e.g. for a specific compliance certification), it
should be scoped as its own piece of work rather than assumed present.

### Compliance Reports

The queries below are written against the real `audit_events` schema above.

**GDPR**: User data access logs
```sql
SELECT created_at, action, resource_type, resource_public_id
FROM audit_events
WHERE actor_username = 'alice'
  AND created_at > NOW() - INTERVAL '90 days'
ORDER BY created_at DESC;
```

**SOC 2**: Config change audit trail
```sql
SELECT created_at, actor_username, action, details
FROM audit_events
WHERE resource_type IN ('deployment', 'gateway', 'adapter', 'sink')
  AND created_at BETWEEN '2025-01-01' AND '2025-12-31'
ORDER BY created_at;
```

**ISO 27001**: Access control review, as noted above, failed logins are not
audit-logged today, only counted on the `users` row. This reads that
counter directly rather than querying a nonexistent `authentication.failure`
audit event:
```sql
SELECT username, failed_login_attempts
FROM users
WHERE failed_login_attempts > 5;
```

---

## Threat Model

### Threats & Mitigations

| Threat | Impact | Mitigation |
|--------|--------|------------|
| **Unauthorized access to Control API** | Configuration tampering, data exposure | JWT auth, RBAC. Rate limiting and an IP allowlist are **not implemented**, see Production Hardening note below. |
| **MITM attack on edge → cloud** | Data interception, credential theft | TLS 1.3, certificate pinning |
| **Compromised adapter container** | Malicious data injection | Sandboxing, read-only filesystem, limited Kafka ACLs |
| **Kafka-compatible broker credential theft** | Unauthorized data access | SASL/SCRAM, credential rotation, Vault or orchestrator secret integration when production packaging supports it |
| **Unencrypted OPC UA session** | PLC data transmitted in cleartext on OT network | V1 limitation, security_mode=None only; adapter logs a WARNING on connect; upgrade path to Sign/SignAndEncrypt planned for V2 |
| **DDoS on Control API** | Service unavailability | **Not implemented today**, no rate limiting, throttling, or auto-scaling exists in the current codebase. CDN/edge protection is a deployment-environment decision outside Qorel's own code. |
| **Insider threat (malicious admin)** | Data deletion, config destruction | Audit logs, approval workflows, immutable backups |
| **Physical access to edge gateway** | Device tampering | Disk encryption, tamper-evident seals, secure boot |

**Production Hardening (Not Yet Implemented):** API rate limiting and an IP allowlist for the Control API are not wired into the current codebase, there is no rate-limiting middleware or IP-allowlist check anywhere in `control-plane/app`. Do not document these as shipped behavior until they exist.

### Attack Scenarios

**Scenario 1: Compromised Edge Gateway**

**Attack**:
```
Attacker gains physical access to edge gateway
Boots into recovery mode
Attempts to read Kafka data or credentials
```

**Mitigations**:
- Full disk encryption (LUKS)
- Secure boot (TPM-based)
- Credentials stored outside plaintext config once production secret resolution is implemented
- Tamper-evident physical seals
- Remote wipe capability

**Scenario 2: Stolen JWT Token**

**Attack**:
```
Attacker intercepts JWT token (XSS, browser vulnerability)
Uses token to access Control API
```

**Mitigations**:
- Short token expiration (12 hours)
- HttpOnly cookies (for web UI)
- Audit logging of login events (`auth.login`, `app/routers/auth.py`)
- Server-side logout invalidation, `POST /api/v1/auth/logout` sets
  `User.token_valid_after = now()` (in addition to clearing the HttpOnly
  cookie). `get_current_user()` rejects any token whose `iat` claim predates
  that watermark, so a bearer token captured before logout and replayed
  directly via the `Authorization` header (bypassing the cookie) stops
  working immediately on the next authenticated request, not just at natural
  expiration. This is a single watermark column, not a growing
  blocklist/`jti` store, logging out invalidates *every* token issued before
  that moment, including ones the user forgot they had, not just "this
  session."
  - Known limitation: JWT `iat` has whole-second precision. A fresh login
    issued in the same wall-clock second as a preceding logout is not
    retroactively invalidated by that logout. Real human re-logins always
    take longer than one second, so this has no practical security impact.

**Not implemented today:**
- IP address binding, no per-token/per-session IP binding exists, not even
  as an opt-in toggle.

---

## Security Checklist

### Pre-Deployment

- [ ] TLS certificates generated and installed
- [ ] Secrets kept out of plaintext config using the production-supported secret mechanism
- [ ] Firewall rules configured
- [ ] VPN tunnel established (for edge deployments)
- [ ] Kafka-compatible broker ACLs configured where authentication is enabled
- [ ] Database credentials rotated from defaults
- [ ] Admin accounts use strong passwords (or SSO)
- [ ] Audit logging enabled

### Post-Deployment

- [ ] Security scan completed (vulnerability assessment)
- [ ] Penetration testing performed
- [ ] Audit logs reviewed (no unauthorized access)
- [ ] Backups encrypted and tested
- [ ] Incident response plan documented
- [ ] Security training for operators completed

### Ongoing

- [ ] Monthly credential rotation
- [ ] Quarterly access review (remove unused accounts)
- [ ] Security patches applied within 7 days
- [ ] Audit logs reviewed weekly
- [ ] Backup restoration tested quarterly

---

## Incident Response

### Incident Response Plan

**Severity Levels**:

| Level | Definition | Example | Response Time |
|-------|------------|---------|---------------|
| **Critical** | Active breach, data exfiltration | Compromised admin account | < 1 hour |
| **High** | Potential breach, failed access attempts | 100 failed logins from unknown IP | < 4 hours |
| **Medium** | Security misconfiguration | Exposed API endpoint | < 24 hours |
| **Low** | Security best practice violation | Weak password detected | < 1 week |

### Response Procedure

**Step 1: Detection**
```
Automated alerts (failed logins, unusual access patterns)
Manual reports (user notices suspicious activity)
```

**Step 2: Containment**
```
# Revoke compromised credentials
vault token revoke <token-id>

# Block suspicious IP
iptables -A INPUT -s 203.0.113.45 -j DROP

# Disable compromised user, blocks all login attempts immediately.
# The user account is flagged disabled=True in the database; any subsequent
# login attempt returns HTTP 403 before the password is checked.
POST /api/v1/users/{username}/disable
Authorization: Bearer <admin-token>

# Re-enable after investigation and credential reset
POST /api/v1/users/{username}/enable
Authorization: Bearer <admin-token>
```

**Brute-force detection and automated lockout**: the control plane increments
`failed_login_attempts` on every failed password check and resets it to 0 on
successful login. After the 5th consecutive failed attempt, the account is
automatically locked (`User.locked_until = now() + 15 minutes`), further
login attempts return `423 Locked` immediately, without checking the
password, until the lockout window elapses. A successful login after the
window clears both `locked_until` and `failed_login_attempts`. An admin can
also clear an active lockout early via `POST /api/v1/users/{username}/enable`
(the same endpoint used to re-enable a disabled account). Query the audit log
or the users list (`GET /api/v1/users`) to surface accounts with elevated
failure counts or active lockouts.

**Step 3: Investigation**
```sql
-- Review audit logs
SELECT * FROM audit_log
WHERE user_email = 'alice@example.com'
  AND timestamp > NOW() - INTERVAL '24 hours'
ORDER BY timestamp;

-- Check for unauthorized config changes
SELECT * FROM audit_log
WHERE event_type LIKE '%.create' OR event_type LIKE '%.delete'
  AND user_email = 'alice@example.com';
```

**Step 4: Remediation**
```
- Rotate all affected credentials
- Apply security patches
- Update firewall rules
- Restore from backup if data compromised
```

**Step 5: Post-Incident**
```
- Document incident in report
- Update security procedures
- Conduct training
- Implement additional controls
```

### Emergency Contacts

```
Security Team Lead: security@example.com
On-Call Engineer: +1-555-ONCALL
Vendor Support: support@qorel.io
```

---

**End of Security Documentation**

For architecture details, see [ARCHITECTURE.md](ARCHITECTURE.md).
For deployment, see the deployment guide (not included in this showcase).
