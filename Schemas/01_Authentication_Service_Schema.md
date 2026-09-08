# Resolve — Authentication Service: Exhaustive Data & Persistence Schema

**Document type:** Service-level database/schema design  
**Service:** Authentication Service  
**Status:** Proposed implementation specification, reconciled against the current Resolve SRS, high-level schema, service contracts, task plan, and Kafka contract  
**Primary responsibility:** Authentication, access-token issuance, refresh-token/session lifecycle, password reset, MFA, JWT key publication  
**PostgreSQL ownership:** **None**  
**Persistent state owned:** **Redis only**  
**Kafka:** None  
**External dependency:** User & Team Service for user lookup/credential verification/password-hash updates; Organization Service should be used for authoritative tenant/slug resolution.

---

## 1. Purpose

The Authentication Service is responsible for proving identity and maintaining authentication/session state. It must **not** become the owner of users, teams, roles, or permissions.

The current project contracts intentionally define:

- `users` as owned by User & Team Service.
- `users.password_hash` as physically stored in User & Team Service but modified only through an internal contract invoked by Authentication.
- Authentication state as Redis-backed.
- Access tokens as signed JWTs.
- Refresh tokens as opaque, server-side Redis records.
- Roles and permissions as **excluded from JWT claims** because RBAC remains the authorization source of truth.

This document converts those contracts into an implementable persistence model, lifecycle rules, indexes/key design, security controls, and failure handling.

---

## 2. Ownership Boundary

### 2.1 Data owned by Authentication

| Data | Store | Owner |
|---|---|---|
| Access-token signing keys/private keys | Secret manager / protected runtime storage | Authentication |
| Published public keys/JWKS | Derived from signing keys | Authentication |
| Refresh sessions | Redis | Authentication |
| Revoked access-token IDs | Redis | Authentication |
| MFA OTP challenge state | Redis | Authentication |
| MFA login challenge state | Redis | Authentication |
| Password-reset tokens | Redis | Authentication |
| Login/reset rate-limit counters | Redis | Authentication |
| Token TTL configuration, if Phase 4 is implemented | Configuration store / service configuration | Authentication |

### 2.2 Data explicitly not owned by Authentication

| Data | Owner |
|---|---|
| `users` profile row | User & Team Service |
| `users.email` | User & Team Service |
| `users.organization_id` | User & Team Service |
| `users.status` | User & Team Service |
| `users.password_hash` physical column | User & Team Service |
| Teams and memberships | User & Team Service |
| Roles and permissions | RBAC Service |
| Organization lifecycle | Organization Service |

### 2.3 Important architectural rule

Authentication must **never connect directly to the User Service PostgreSQL database**.

The supported path is:

```text
Client
  |
  | login(email, password, organizationSlug)
  v
Authentication Service
  |
  | resolve organizationSlug
  v
Organization Service
  |
  | active organization
  v
Authentication Service
  |
  | lookup user
  v
User & Team Service
  |
  | verify credentials
  v
Authentication Service
  |
  | create refresh session + issue JWT
  v
Client
```

For password reset:

```text
Client
  |
  v
Auth
  |-- Redis reset token
  |
  |-- internal password update
  v
User & Team Service
```

---

## 3. Storage Architecture

```text
Authentication Service
│
├── Redis
│   ├── refresh sessions
│   ├── revoked access tokens
│   ├── MFA OTPs
│   ├── MFA login challenges
│   ├── password reset tokens
│   └── rate-limit counters
│
├── Signing-key storage
│   └── RSA private/public key material
│
└── No PostgreSQL tables
```

Redis is a state store here, not a relational system of record. Authentication state is deliberately short-lived and reconstructable/revocable.

---

## 4. Redis Namespace Convention

All keys should be namespaced by service:

```text
resolve:auth:{key}
```

The existing contract examples use shorter names such as `refresh:{id}`. The recommended production implementation is:

```text
resolve:auth:refresh:{refreshTokenId}
resolve:auth:session:revoked:{tokenId}
resolve:auth:mfa:otp:{userId}
resolve:auth:mfa:challenge:{challengeToken}
resolve:auth:password-reset:{resetToken}
resolve:auth:ratelimit:login:email:{normalizedEmail}:{window}
resolve:auth:ratelimit:login:ip:{ip}:{window}
```

If the team freezes the shorter names from the contract, keep them consistent across every service and deployment.

---

# 5. Redis Data Schemas

## 5.1 Refresh Session

### Key

```text
resolve:auth:refresh:{refreshTokenId}
```

### Value

```json
{
  "userId": "9e1071ea-9a3b-4a2f-9e2e-1a2b3c4d5e6f",
  "organizationId": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
  "issuedAt": "2026-09-08T17:00:00Z",
  "expiresAt": "2026-10-08T17:00:00Z",
  "tokenFamilyId": "0a8e...",
  "status": "ACTIVE",
  "rotatedFrom": null,
  "createdByIp": "optional",
  "userAgentHash": "optional"
}
```

### Required fields

| Field | Type | Required | Purpose |
|---|---|---:|---|
| `userId` | UUID | Yes | Authenticated identity |
| `organizationId` | UUID | Yes | Tenant binding |
| `issuedAt` | timestamp | Yes | Session creation |
| `expiresAt` | timestamp | Yes | Absolute expiration |
| `status` | enum | Yes | `ACTIVE`, `REVOKED`, `ROTATED` |

### Recommended security fields

`tokenFamilyId`, `rotatedFrom`, device/user-agent metadata, and last-used timestamp are recommended if refresh-token rotation is implemented.

### TTL

Set Redis TTL equal to the refresh-token expiration period.

### Invariants

1. A refresh token is never reusable after successful rotation.
2. A token belonging to one tenant cannot mint a token for another tenant.
3. Missing Redis record means invalid refresh token.
4. Expired Redis record means invalid refresh token.
5. Revoked/rotated record means invalid refresh token.
6. Refresh-token identifiers must be cryptographically random.
7. Never log the raw refresh token.

---

## 5.2 Revoked Access Token

### Key

```text
resolve:auth:session:revoked:{tokenId}
```

### Value

```text
1
```

### TTL

```text
access_token_expiry - current_time
```

There is no reason to retain a revocation marker after the JWT could no longer be accepted.

### Invariant

A JWT with a valid signature and valid `exp` is still rejected if this key exists.

---

## 5.3 MFA OTP

### Key

```text
resolve:auth:mfa:otp:{userId}
```

### Value

```json
{
  "otpHash": "...",
  "attempts": 0,
  "createdAt": "2026-09-08T17:00:00Z"
}
```

### TTL

Approximately 5 minutes.

### Security requirements

- Never store the plaintext OTP.
- Compare using a safe verification mechanism.
- Limit verification attempts.
- Delete after successful verification.
- Delete after maximum failed attempts.
- A new OTP invalidates the previous OTP.

---

## 5.4 MFA Login Challenge

### Key

```text
resolve:auth:mfa:challenge:{challengeToken}
```

### Value

```json
{
  "userId": "9e1071ea-9a3b-4a2f-9e2e-1a2b3c4d5e6f",
  "organizationId": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
  "createdAt": "2026-09-08T17:00:00Z"
}
```

### TTL

Approximately 5 minutes.

### Invariants

- Single-use.
- Bound to exactly one user and tenant.
- Cannot be used after expiration.
- Successful MFA verification deletes it.

---

## 5.5 Password Reset Token

### Key

```text
resolve:auth:password-reset:{resetToken}
```

### Value

```json
{
  "userId": "9e1071ea-9a3b-4a2f-9e2e-1a2b3c4d5e6f",
  "organizationId": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
  "expiresAt": "2026-09-08T17:30:00Z"
}
```

### TTL

Approximately 30 minutes.

### Invariants

1. Single-use.
2. Delete immediately after successful password update.
3. Never return whether a reset token belongs to a real account.
4. Never log the raw token.
5. Password reset should revoke existing sessions after successful password change.

---

## 5.6 Login Rate Limit

### Keys

```text
resolve:auth:ratelimit:login:email:{normalizedEmail}:{window}
resolve:auth:ratelimit:login:ip:{ip}:{window}
```

### Value

Integer counter.

### Requirements

- Rate-limit by both identity and source.
- Return HTTP `429`.
- Include `Retry-After`.
- Do not allow an attacker to bypass the limit by changing casing of email.
- Normalize email before rate-limit lookup.
- Avoid exposing whether an email exists.

---

# 6. JWT Schema

Authentication issues an access JWT with this canonical claim set:

```json
{
  "sub": "9e1071ea-9a3b-4a2f-9e2e-1a2b3c4d5e6f",
  "tenantId": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
  "email": "jane.doe@acmebank.com",
  "tokenId": "8b2c1e4a-6f3d-4b2a-8c1d-abcdef123456",
  "type": "access",
  "iat": 1893456000,
  "exp": 1893459600,
  "iss": "resolve-authentication-service",
  "kid": "2026-09-key-1"
}
```

## 6.1 Claim rules

| Claim | Required | Authority |
|---|---:|---|
| `sub` | Yes | User Service user ID |
| `tenantId` | Yes | Organization/User relationship |
| `email` | Yes | Convenience snapshot |
| `tokenId` | Yes | Authentication |
| `type` | Yes | Authentication |
| `iat` | Yes | Authentication |
| `exp` | Yes | Authentication |
| `iss` | Yes | Fixed service identifier |
| `kid` | Yes | Signing-key identifier |

### Deliberately absent

```text
roles
permissions
teamIds
```

These must not be embedded as authoritative authorization data.

---

# 7. JWT Verification Contract

Every protected service must perform:

```text
1. Parse Bearer token.
2. Read kid.
3. Load cached JWKS key.
4. On kid miss, refresh JWKS.
5. Verify RS256 signature.
6. Validate issuer.
7. Validate exp.
8. Validate required sub.
9. Validate required tenantId.
10. Check access-token type.
11. Check Redis revocation marker.
12. Put userId + tenantId into request security context.
13. Continue to RBAC authorization.
```

### Tenant invariant

The tenant in the JWT is never accepted from a client body/query parameter as the security context.

---

# 8. Signing-Key Storage

Private keys must not be stored in PostgreSQL or Redis.

Recommended:

```text
AWS Secrets Manager / KMS
or
Kubernetes Secret backed by KMS
or
Docker secret for local development
```

### Key metadata

| Field | Example |
|---|---|
| `kid` | `2026-09-key-1` |
| algorithm | `RS256` |
| key type | RSA |
| usage | `sig` |
| status | `ACTIVE` / `RETIRING` |
| createdAt | timestamp |
| retireAfter | timestamp |

### Rotation

```text
Generate new key
      ↓
Publish new public key
      ↓
Start signing new JWTs with new kid
      ↓
Keep old public key published
      ↓
Wait until all old JWTs expire
      ↓
Remove old public key
```

Never remove an old key while a valid token signed with it can still exist.

---

# 9. Endpoint-to-Persistence Mapping

| Endpoint | Reads | Writes |
|---|---|---|
| `POST /auth/login` | Organization Service, User Service, Redis rate limits | Refresh session |
| `POST /auth/refresh` | Redis refresh session | New refresh session; old session rotated/revoked |
| `POST /auth/logout` | JWT context | Revocation marker; refresh session invalidation |
| `POST /auth/password-reset/request` | Organization/User Service | Reset token |
| `POST /auth/password-reset/confirm` | Reset token | User password via User Service; session invalidation |
| `POST /auth/mfa/enroll` | User identity | MFA secret state |
| `POST /auth/mfa/enroll/confirm` | MFA state | MFA activation state |
| `POST /auth/mfa/verify` | Challenge + OTP | Session |
| `POST /auth/mfa/disable` | Current user | MFA disable state |
| `GET /.well-known/jwks.json` | Signing-key public material | None |
| `GET /internal/v1/auth/sessions/{tokenId}/status` | Revocation marker | None |

---

# 10. Authentication State Machines

## 10.1 Access session

```text
ISSUED
  |
  +--> ACTIVE
  |
  +--> REVOKED
  |
  +--> EXPIRED
```

## 10.2 Refresh token

```text
ACTIVE
  |
  +--> ROTATED
  |
  +--> REVOKED
  |
  +--> EXPIRED
```

A rotated refresh token must never become active again.

## 10.3 MFA login

```text
PASSWORD_VALID
      |
      v
MFA_REQUIRED
      |
      +--> CHALLENGE_EXPIRED
      |
      +--> INVALID_ATTEMPTS
      |
      v
MFA_VERIFIED
      |
      v
SESSION_ISSUED
```

---

# 11. Password-Change Session Policy

A successful password change/reset should invalidate all active refresh sessions for the user.

Because the current Redis design is keyed by refresh-token ID, production implementation should maintain one of:

### Option A — User session index

```text
resolve:auth:user-sessions:{userId}
```

Containing refresh-token IDs.

### Option B — Token-family revocation

Maintain a user-level/session-family version:

```text
resolve:auth:session-version:{userId}
```

and include the version in issued sessions.

**Recommendation:** Option A is simplest for this capstone; keep it bounded and clean expired members periodically.

---

# 12. Failure and Edge Cases

| Scenario | Required behavior |
|---|---|
| Wrong password | `401 AUTH_INVALID_CREDENTIALS` |
| Unknown email | Same response as wrong password |
| Unknown organization slug | Same generic login failure |
| Suspended organization | Login rejected |
| Disabled/inactive user | `403 AUTH_ACCOUNT_DISABLED` only after credentials are proven correct |
| Expired access JWT | `401` |
| Revoked JWT | `401` |
| Unknown `kid` | Refresh JWKS; reject if still unknown |
| Expired refresh token | `401` |
| Reused rotated refresh token | `401`; consider revoking entire token family |
| Password reset token reused | `400` |
| MFA challenge expired | `401 AUTH_MFA_CHALLENGE_EXPIRED` |
| MFA OTP wrong | `401 AUTH_MFA_INVALID_CODE` |
| MFA brute force | Lock/delete challenge after max attempts |
| Redis unavailable | Fail closed for revocation/session operations; never silently accept revoked tokens |
| User Service unavailable during login | Return controlled `503`, not invalid credentials |
| Organization Service unavailable | Return controlled `503` |
| Duplicate reset request | Still return `202` |
| Password reset succeeds | Invalidate existing sessions |
| Token settings changed | Apply only to newly issued tokens |

---

# 13. Security Requirements

1. TLS for all client and internal service communication.
2. Passwords are never persisted in plaintext.
3. Refresh tokens are never logged.
4. Reset tokens are never logged.
5. MFA OTPs are never logged.
6. Authentication errors must avoid user enumeration.
7. Rate limiting is mandatory for login and reset endpoints.
8. JWT private keys never enter application logs.
9. Redis access must be authenticated.
10. Internal endpoints require service authentication.
11. Authentication fails closed where token revocation cannot be verified.
12. Password reset invalidates sessions.
13. Refresh-token reuse detection should invalidate the affected token family.
14. Secrets must come from environment/secret management, not source control.

---

# 14. Contract Dependencies

### User & Team Service

```text
GET /internal/v1/users/lookup
POST /internal/v1/users/{id}/verify-credentials
PATCH /internal/v1/users/{id}/password-hash
```

### Organization Service

Recommended authoritative dependency:

```text
GET /internal/v1/organizations/by-slug/{slug}
GET /internal/v1/organizations/{id}
```

This removes the current hidden coupling where User & Team Service would otherwise resolve an organization slug it does not own.

---

# 15. Known Contract Inconsistency and Final Decision

The source documents have one important architectural inconsistency:

- The high-level schema says `users.password_hash` exists in `users`.
- Task 4 says User & Team owns the table.
- Authentication must manage credentials.
- Direct shared-database ownership is explicitly undesirable.

### Final decision for this service

**User & Team Service remains the physical table owner and sole database writer. Authentication accesses credential operations only through internal APIs.**

This preserves the one-table/one-owner rule and avoids shared database connections.

---

# 16. Traceability

| Requirement | Covered by |
|---|---|
| FR-IAM-01 | User lookup + credential contract |
| FR-IAM-02 | Login + JWT |
| FR-IAM-03 | Refresh sessions |
| FR-IAM-04 | Revocation/session invalidation |
| FR-IAM-05 | User Service password hashing endpoint |
| FR-IAM-06 | JWT/JWKS/revocation verification |
| FR-IAM-07 | Password reset |
| FR-IAM-09 | Token settings |
| FR-IAM-10 | MFA |
| FR-RED-02 | Login/reset rate limits |
| FR-TEN-04 | `tenantId` JWT claim |
| NFR-SEC-02 | Tenant binding and fail-closed verification |
| BR-11 | Idempotent reset/refresh semantics |
| BR-13 | No mutation of audit history; Auth does not own audit |

---

# 17. Implementation Checklist

- [ ] Redis connection and namespace configured.
- [ ] Refresh-session repository implemented.
- [ ] Revocation repository implemented.
- [ ] MFA state repository implemented.
- [ ] Password-reset repository implemented.
- [ ] Login rate limiter implemented.
- [ ] RSA key generation/loading implemented.
- [ ] JWKS endpoint implemented.
- [ ] JWT issuer implemented.
- [ ] JWT verification contract published.
- [ ] Refresh-token rotation implemented.
- [ ] Refresh-token replay detection implemented.
- [ ] Password reset invalidates sessions.
- [ ] User Service internal credential contract secured.
- [ ] Organization Service slug resolution integrated.
- [ ] Redis outage behavior tested.
- [ ] Authentication enumeration tests added.
- [ ] Expired/revoked/rotated token tests added.
- [ ] MFA abuse tests added.
