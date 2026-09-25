# Authentication Service — Exhaustive API & Contract Reference

**Source of truth:** `Resolve_Documentation/General/{SRS.md, Features_Phases.md, Schemas_High_Level.md}` · `Resolve_Documentation/tasks.md` (Task 4) · `Resolve_Documentation/kafka.md`
**Service nature:** Synchronous REST only. **No Kafka producer, no Kafka consumer** — confirmed explicitly in `kafka.md`: *"RBAC, Auth, Organization, User, Case (as a consumer) — none of them touch Kafka at all. They're purely synchronous."*
**Owned database table:** none. `users.password_hash` physically lives in User & Team Service's database; this service never connects to it directly (ownership decision below). Authentication's only persisted state is in Redis.
**Depends on:** Task 3 (User & Team Service) — needs the `users` table/service to exist and expose its internal credential endpoints.
**Depended on by:** every other service, indirectly — each validates the JWT this service issues, locally, without calling back here per request.

---

## 1. Ownership Decision Recap (governs several contracts below)

Per Task 4's own flagged ambiguity ("agree in Task 0 whether Auth owns that one column exclusively or calls User Service for it") and Task 0's rule of "one table, one owning service, no exceptions":

- **User & Team Service owns the entire `users` table**, including `password_hash`.
- **Authentication Service has zero direct database access to `users`.** It calls two internal, service-to-service-only endpoints on User & Team Service instead:
  - `POST /internal/v1/users/{id}/verify-credentials`
  - `PATCH /internal/v1/users/{id}/password-hash`
- Authentication also needs an email/tenant → user-id lookup before it can call `verify-credentials`. User & Team Service exposes `GET /internal/v1/users/lookup` for that purpose (see §14.1) — it takes an already-resolved `organizationId`, **not** a slug. Slug resolution is Organization Service's job (§14.0): Authentication resolves `organizationSlug` → `organizationId` via `GET /internal/v1/organizations/by-slug/{slug}` **before** calling User & Team, so tenant-slug resolution and suspended-tenant rejection live in the one service that owns `organizations`, not duplicated inside User & Team.

This keeps every contract below internally consistent with the rest of the documented architecture.

---

## 2. Inherited Platform Conventions (apply to every endpoint below)

| Convention | Rule |
|---|---|
| Base path | `/api/v1` for public endpoints |
| Auth header | `Authorization: Bearer {jwt}` — **not required** only on `POST /auth/login`, `POST /auth/refresh`, `POST /auth/password-reset/request`, `POST /auth/password-reset/confirm`, and `GET /.well-known/jwks.json` |
| Tenant scoping | Never accepted as a body/query field for authorization purposes; always derived server-side |
| Idempotency | State-changing POSTs that aren't naturally idempotent accept `Idempotency-Key` (see per-endpoint notes — login/refresh are naturally safe to retry and don't require it) |
| Error shape | `{ "error": { "code", "message", "details", "requestId", "traceId" } }` |
| Rate limiting | `POST /auth/login` and `POST /auth/password-reset/request` are the two highest-value brute-force targets in the whole platform — apply Redis-backed rate limiting here first (FR-RED-02), even before it's rolled out elsewhere. **Mandatory, not optional** — `Schemas/01_Authentication_Service_Schema.md` §13 item 7 states rate limiting "is mandatory for login and reset endpoints" (this contract previously downgraded it to "recommended... flag as a team decision" in §15's key table; that was inconsistent with the schema doc and has been corrected). Keys: `resolve:auth:ratelimit:login:email:{normalizedEmail}:{window}` and `resolve:auth:ratelimit:login:ip:{ip}:{window}`, both checked independently (see §15). **Gap:** no equivalent key pattern is documented yet for `/auth/password-reset/request` specifically — the schema doc mandates rate limiting there too, but no key name has been assigned. The built Authentication Service currently only rate-limits `/auth/login`; `/auth/password-reset/request` has no rate limiting yet — this is a known, currently-unaddressed gap between the schema doc's requirement and the implementation. |

---

## 3. JWT Claim Contract

This is the single most important contract in the document — every other service's authorization logic depends on this shape being stable.

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

| Claim | Type | Description |
|---|---|---|
| `sub` | UUID | The user's id — matches `identity`/User Service's `users.id`. |
| `tenantId` | UUID | FR-TEN-04 — present and validated on every authenticated request by every service. |
| `email` | string | Convenience claim; not authoritative (a service needing the current email should still read from User Service if freshness matters). |
| `tokenId` | UUID | Unique per issued token. Used as the revocation key: `session:revoked:{tokenId}` in Redis. Every verifying service must check this key even after signature verification succeeds — FR-IAM-06 requires rejecting a token that is "valid, non-expired, **non-revoked**." |
| `type` | `"access"` \| `"refresh"` | Refresh tokens are opaque Redis-backed values, not JWTs, in this design (see §5) — this claim exists for forward-compatibility if that ever changes; access tokens are always `"access"`. |
| `iat` / `exp` | Unix timestamp | Standard JWT claims. |
| `iss` | string | Fixed issuer string, lets a service confirm it isn't accepting a token forged by a different signer. |
| `kid` | string | Key ID, matches the key published at `/.well-known/jwks.json` — enables key rotation without breaking already-issued tokens. |

**Deliberately excluded:** roles and permissions. RBAC Service is the single source of truth for authorization and is queried (with local caching) at authorization time — embedding permissions in the token would let a revoked permission remain effective until the token naturally expires, which is a real security gap this design avoids.

**Verification algorithm every other service implements** (documented here since Authentication is the contract owner, even though the code lives elsewhere):
```text
1. Fetch/cache JWKS from GET /.well-known/jwks.json (refresh on kid cache-miss or on a schedule)
2. Verify signature using the key matching the token's `kid`
3. Verify `exp` has not passed
4. Verify `iss` matches the expected issuer
5. Check Redis: GET session:revoked:{tokenId} — if present, reject regardless of signature validity
6. Extract `sub`, `tenantId` for use in the request context
```

---

## 4. Full Endpoint Index (A–Z by path)

| Method | Path | Phase | Requirement | Auth Required |
|---|---|---|---|---|
| `GET` | `/.well-known/jwks.json` | 2 (Must) | supports FR-IAM-02/06 | No |
| `POST` | `/api/v1/auth/login` | 2 (Must) | FR-IAM-02 | No |
| `POST` | `/api/v1/auth/logout` | 2 (Must) | FR-IAM-04 | Yes |
| `POST` | `/api/v1/auth/mfa/disable` | 4 (Could) | FR-IAM-10 | Yes |
| `POST` | `/api/v1/auth/mfa/enroll` | 4 (Could) | FR-IAM-10 | Yes |
| `POST` | `/api/v1/auth/mfa/verify` | 4 (Could) | FR-IAM-10 | No* (see §9) |
| `POST` | `/api/v1/auth/password-reset/confirm` | 3 (Should) | FR-IAM-07 | No |
| `POST` | `/api/v1/auth/password-reset/request` | 3 (Should) | FR-IAM-07 | No |
| `POST` | `/api/v1/auth/refresh` | 2 (Must, per SRS §8 initial API surface)** | FR-IAM-03 | No (refresh token in body) |
| `GET` | `/api/v1/auth/token-settings` | 4 (Could) | FR-IAM-09 | Yes (Org Admin) |
| `PATCH` | `/api/v1/auth/token-settings` | 4 (Could) | FR-IAM-09 | Yes (Org Admin) |

\* MFA verify is unauthenticated by design — it's step 2 of login, before a full session exists; it's protected instead by a short-lived, single-use MFA challenge token (see §9).

\*\* Note on `/auth/refresh`'s phase: `SRS.md` §8 ("Initial API Surface") lists it alongside login/logout as part of the initial contract, while `Features_Phases.md` §5.1 tags the underlying requirement FR-IAM-03 as Should-have (Phase 3). Build the endpoint's contract now (Task 0/4), but it's acceptable for the full rotation/revocation robustness described in §5 to land in Phase 3 rather than Phase 2 if time is tight — flag this explicitly rather than silently deciding either way.

Not listed above but part of this service's integration surface: two **internal** endpoints it *calls* on User & Team Service (§14) and one **internal** endpoint it *exposes* for other services to check a token's revocation status without needing direct Redis access (§13).

---

## 5. `POST /api/v1/auth/login`

**Requirement:** FR-IAM-02 — *"authenticate users via email/username and password, issuing a signed JWT access token on success; invalid login attempts shall be rejected without issuing a token."*

### Request
```json
{
  "email": "jane.doe@acmebank.com",
  "password": "TempPass!2026",
  "organizationSlug": "acme-bank"
}
```

| Field | Type | Required | Notes |
|---|---|---|---|
| `email` | string | Yes | — |
| `password` | string | Yes | Plaintext, sent over TLS only. |
| `organizationSlug` | string | Yes | Disambiguates which tenant's `users` row to check, since email is unique **per tenant**, not globally (`Schemas_High_Level.md` `users` table: `UNIQUE(organization_id, email)`). This field is required precisely because there is no other way to resolve the tenant before authentication succeeds. |

### Response `200 OK`
```json
{
  "accessToken": "eyJhbGciOiJSUzI1NiIsImtpZCI6IjIwMjYtMDktLWtleS0xIn0...",
  "refreshToken": "8b2c1e4a-6f3d-4b2a-8c1d-abcdef123456",
  "tokenType": "Bearer",
  "expiresIn": 3600
}
```

| Field | Type | Notes |
|---|---|---|
| `accessToken` | string (JWT) | Shape per §3. |
| `refreshToken` | string (opaque UUID) | Not a JWT — a random value stored server-side in Redis (§5's implementation note below), so it can be revoked/rotated without needing a JWT blocklist for every refresh token ever issued. |
| `tokenType` | `"Bearer"` | Fixed. |
| `expiresIn` | integer (seconds) | Access token lifetime; default platform-wide unless FR-IAM-09 (Phase 4) tenant override is active. |

### Response `401 Unauthorized`
```json
{
  "error": {
    "code": "AUTH_INVALID_CREDENTIALS",
    "message": "Email or password is incorrect.",
    "requestId": "req_9f2a1c"
  }
}
```
Intentionally identical whether the email doesn't exist, the tenant slug is wrong, or the password is wrong — never reveal which part failed (prevents user enumeration).

### Response `403 Forbidden`
```json
{
  "error": {
    "code": "AUTH_ACCOUNT_DISABLED",
    "message": "This account has been disabled. Contact your organization administrator.",
    "requestId": "req_9f2a1c"
  }
}
```
Distinct from invalid-credentials because the credentials *were* correct — this is a legitimate, different failure mode a disabled user should be told about, per `users.status IN ('INACTIVE','SUSPENDED')` in the schema (User & Team Service's canonical enum — see `Contracts/User_Team_Service.md` §5). Either non-`ACTIVE` value maps to this same `403`. **This requires credentials to have actually been checked first — see the resolved Step 3 below; this response can only be reached via a `valid: true` result from `verify-credentials`, never as a shortcut based on the earlier lookup's status.**

### Response `202 Accepted` (MFA-enrolled account — see §9)
```json
{
  "mfaRequired": true,
  "mfaChallengeToken": "c9d8e7f6-...",
  "expiresIn": 300
}
```
**Not yet implementable — see Step 4 below.** The flag this response depends on doesn't exist in any documented contract yet.

### Implementation Notes
- **Step 1:** resolve the tenant — call `GET /internal/v1/organizations/by-slug/{organizationSlug}` on Organization Service (§14.0) → returns `{ id, status }` or `404`. If `status = SUSPENDED`, reject the login immediately (generic `AUTH_INVALID_CREDENTIALS`, same anti-enumeration principle as below — a suspended tenant should not be distinguishable from a bad password).
- **Step 2:** call `GET /internal/v1/users/lookup?organizationId={id}&email={email}` on User & Team Service (§14.1) → returns `{ userId, status }` or `404`. A `404` here maps to the same generic `AUTH_INVALID_CREDENTIALS` — never a distinguishable error (anti-enumeration).
- **Step 3 (resolved — previously ambiguous):** once a user is found by lookup, **always** call `POST /internal/v1/users/{userId}/verify-credentials` (§14.2), regardless of the lookup's `status` field. Do not gate the call to `verify-credentials` on lookup's `status = ACTIVE`, even though that reading was previously implied here — doing so would make the `403 Forbidden` response above unreachable, since that response's own description requires credentials to have been checked first. Instead: use `verify-credentials`' own returned `status` field (not the earlier lookup's) as the single source of truth for ACTIVE vs. disabled. This is the only reading under which both response descriptions in this section are simultaneously true, and it's what the built Authentication Service implements (see `src/services/authService.js`'s inline rationale).
- **Step 4 — MFA branch, NOT YET DEFINED:** on `valid: true` with `status = ACTIVE`, this service should check whether the user has MFA enrolled and return the `202` challenge response instead of tokens if so. **How that flag is determined has never been decided** — candidate approaches, none chosen: (a) User & Team Service's `verify-credentials`/`lookup` response gains an `mfaEnabled` boolean field; (b) RBAC Service or Authentication itself owns a small `user_mfa_enrollment` lookup; (c) the flag lives in Redis, written whenever `/auth/mfa/enroll/confirm` (§9, Phase 4) succeeds, and is checked here. **The built Authentication Service does not implement this branch at all** — it always falls through to Step 5 — because MFA is Phase 4 (Could-have) and guessing at an undocumented cross-service data source would mean inventing behavior no other service has agreed to.
- **Step 5:** on `valid: true` with `status = ACTIVE` (and, once Step 4 is resolved, no MFA challenge required), issue `accessToken` (signed JWT, §3 shape) and `refreshToken` (random UUID, stored as `resolve:auth:refresh:{refreshTokenId}` → `{ userId, tenantId, issuedAt, expiresAt }` in Redis — see §15 for the naming convention).
- Apply rate limiting here (§2) before Step 1, so a brute-force attempt against a nonexistent email/tenant still consumes rate-limit budget rather than skipping straight to a cheap `404`-style short-circuit (which would itself leak information via timing).

---

## 6. `POST /api/v1/auth/refresh`

**Requirement:** FR-IAM-03 — *"issue a refresh token and expose `POST /auth/refresh` to obtain a new access token without re-entering credentials."*

### Request
```json
{ "refreshToken": "8b2c1e4a-6f3d-4b2a-8c1d-abcdef123456" }
```

### Response `200 OK`
```json
{
  "accessToken": "eyJhbGciOiJSUzI1NiIs...",
  "refreshToken": "a1b2c3d4-9e8f-4a2b-8c1d-fedcba987654",
  "tokenType": "Bearer",
  "expiresIn": 3600
}
```
**Rotation:** the old `refreshToken` is invalidated in Redis and a new one issued on every use (refresh-token rotation). This means a stolen-and-replayed old refresh token, used after the legitimate client already rotated it, is detectable — treat that as a signal to revoke the entire session family, not just reject the single request.

### Response `401 Unauthorized`
```json
{
  "error": {
    "code": "AUTH_INVALID_REFRESH_TOKEN",
    "message": "Refresh token is invalid, expired, or already used.",
    "requestId": "req_9f2a1c"
  }
}
```

---

## 7. `POST /api/v1/auth/logout`

**Requirement:** FR-IAM-04 — *"provide `POST /auth/logout`, invalidating the current session/refresh token."*
Implementation backs FR-IAM-08 (Redis-backed session revocation, Phase 3) even though the endpoint itself ships in Phase 2.

### Request
Empty body. `Authorization: Bearer {accessToken}` header carries the token to revoke. Optionally:
```json
{ "refreshToken": "8b2c1e4a-6f3d-4b2a-8c1d-abcdef123456" }
```
to also invalidate the paired refresh token in the same call (recommended default behavior — logging out should end the whole session, not just the access token).

### Response `204 No Content`
No body. **Side effects:**
```text
SET session:revoked:{tokenId} = "1"   (TTL = remaining access-token life)
DEL refresh:{refreshTokenId}          (if a refreshToken was provided)
```

### Response `401 Unauthorized`
Standard shape, `code: "AUTH_INVALID_TOKEN"` — logging out with an already-invalid token is still a client error, not silently a no-op `204`, so the client knows its stored token was already bad.

---

## 8. `POST /api/v1/auth/password-reset/request` and `/confirm`

**Requirement:** FR-IAM-07 — *"support a password reset/change workflow."* (Phase 3, Should-have)

### `POST /api/v1/auth/password-reset/request`
**Request**
```json
{ "email": "jane.doe@acmebank.com", "organizationSlug": "acme-bank" }
```
**Response `202 Accepted`** (always — never reveals whether the email exists, same anti-enumeration principle as login):
```json
{ "message": "If an account with that email exists, a reset link has been sent." }
```
**Side effect:** generates a single-use reset token, stores `resolve:auth:password-reset:{resetToken}` → `{ userId, expiresAt }` in Redis (TTL ~30 min).

> ⚠️ **Known gap, not yet resolved:** this service has no email-sending capability and no Kafka producer per `kafka.md` §32 ("RBAC, Authentication, Organization, User & Team — no Kafka involvement at all") — so token generation/storage works, but nothing currently *delivers* the reset link to the user. Delivery must happen via a direct synchronous call to a notification/email capability, but Notification Service's documented design (`tasks.md` Task 10) is consumer-only — it reacts to Kafka events, it doesn't expose a synchronous "send transactional email" endpoint anywhere in this doc set. **The built Authentication Service implements this exact gap as a deliberately inert, logged stub** (see its `src/services/passwordResetService.js`) rather than guessing at an endpoint shape for a service it has no contract for. This needs a real decision with whoever owns Notification Service before password reset is genuinely usable end-to-end — it is not blocking Auth's own tests, but it is blocking the feature actually working for a real user.

### `POST /api/v1/auth/password-reset/confirm`
**Request**
```json
{ "resetToken": "f4e3d2c1-...", "newPassword": "NewPass!2027" }
```
**Response `204 No Content`.**
**Internal call:** `PATCH /internal/v1/users/{userId}/password-hash` on User & Team Service (§14.2), using the `userId` resolved from `resolve:auth:password-reset:{resetToken}` in Redis. The reset token is deleted from Redis only *after* that call succeeds — if User & Team Service is briefly unavailable, the token stays valid for a retry rather than being burned on a failed attempt. The reset token is single-use once the password-hash update succeeds.

> ⚠️ **Missing from this contract, but required by `Schemas/01_Authentication_Service_Schema.md` §11–12:** a successful password reset "should invalidate all active refresh sessions for the user" (§11), and §12's edge-case table repeats this as "Password reset succeeds → Invalidate existing sessions." This contract never states that requirement, and **the built Authentication Service does not currently implement it** — confirming a reset only updates the password hash and consumes the reset token; it does not revoke the user's other active refresh sessions or already-issued access tokens. Schema §11 offers two implementation options (a per-user session index, or a per-user session-version counter checked at refresh time) and recommends the session-index approach as simplest. This is a real, currently-open gap between documented requirement and implementation, not a resolved decision.

**Response `400 Bad Request`**
```json
{ "error": { "code": "AUTH_RESET_TOKEN_INVALID", "message": "Reset token is invalid or has expired." } }
```

---

## 9. MFA Endpoints (`FR-IAM-10`, Phase 4 — Could-have)

### `POST /api/v1/auth/mfa/enroll`
**Auth:** required (user enrolling their own account).
**Request:** empty body.
**Response `200 OK`**
```json
{
  "secret": "JBSWY3DPEHPK3PXP",
  "otpauthUrl": "otpauth://totp/Resolve:jane.doe@acmebank.com?secret=JBSWY3DPEHPK3PXP&issuer=Resolve"
}
```
Client renders `otpauthUrl` as a QR code for an authenticator app. Enrollment isn't finalized until confirmed:

**Follow-up: `POST /api/v1/auth/mfa/enroll/confirm`**
```json
{ "code": "482913" }
```
→ `204` on success, flips the account's MFA-enrolled flag on.

### `POST /api/v1/auth/mfa/verify` — step 2 of login
**Auth:** none (see §5's `202` response) — protected by the short-lived challenge token instead.
**Request**
```json
{ "mfaChallengeToken": "c9d8e7f6-...", "code": "482913" }
```
**Response `200 OK`** — same shape as `POST /auth/login`'s success response (issues `accessToken` + `refreshToken`).
**Response `401 Unauthorized`**
```json
{ "error": { "code": "AUTH_MFA_INVALID_CODE", "message": "Invalid or expired code." } }
```
**Redis-backed:** `mfa:otp:{userId}` per `Schemas_High_Level.md` §4.1 — hashed 6-digit code, ~5 min TTL. The challenge token itself (`c9d8e7f6-...`) is a separate short-lived Redis entry binding to `userId`, distinct from the OTP code itself, so a leaked challenge token alone (without the code) is useless.

### `POST /api/v1/auth/mfa/disable`
**Auth:** required. **Request:** `{ "currentPassword": "..." }` (re-confirm identity before disabling a security feature). **Response `204`.**

---

## 10. Token TTL Configuration (`FR-IAM-09`, Phase 4 — Could-have)

### `GET /api/v1/auth/token-settings`
**Auth:** Org Admin (`ROLE_MANAGE` or a dedicated `AUTH_SETTINGS_MANAGE` permission — confirm with RBAC's permission catalog in Task 0). **This endpoint is currently blocked**, not just undecided: no RBAC Service contract exists at all yet (see `Contracts/RBAC_Service.md`, a gap-flag stub), so there's no permission catalog to confirm against. The built Authentication Service has not implemented this endpoint for exactly this reason, rather than guessing at a permission name for a service with no frozen contract.
**Response `200 OK`**
```json
{
  "organizationId": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
  "accessTokenTtlSeconds": 3600,
  "refreshTokenTtlSeconds": 2592000
}
```

### `PATCH /api/v1/auth/token-settings`
**Request**
```json
{ "accessTokenTtlSeconds": 1800 }
```
**Response `200 OK`** — updated settings object. Applies to tokens issued *after* the change; does not retroactively shorten already-issued tokens (that's what `session:revoked:*` and logout are for, if immediate effect is ever needed).

---

## 11. `GET /.well-known/jwks.json`

**Auth:** none — must be publicly fetchable by every other service.
**Response `200 OK`**
```json
{
  "keys": [
    {
      "kty": "RSA",
      "kid": "2026-09-key-1",
      "use": "sig",
      "alg": "RS256",
      "n": "0vx7agoebGcQSuuPiLJXZptN9nndrQmbXEps2aiAFbWhM78LhWx4cbbfAAtVT86zwu1RK7aPFFxuhDR1L6tSoc_BJECPebWKRXjBZCiFV4n3oknjhMstn64tZ_2W-5JsGY4Hc5n9yBXArwl93lqt7_RN5w6Cf0h4QyQ5v-65YGjQR0_FDW2QvzqY368QQMicAtaSqzs8KJZgnYb9c7d0zgdAZHzu6qMQvRL5hajrn1n91CbOpbISD08qNLyrdkt-bFTWhAI4vMQFh6WeZu0fM4lFd2NcRwr3XPksINHaQ-G_xBniIqbw0Ls1jF44-csFCur-kEgU8awapJzKnqDKgw",
      "e": "AQAB"
    }
  ]
}
```
Standard JWKS format (RFC 7517). Every service fetches and caches this; a `kid` cache-miss triggers a re-fetch (supports rotation without downtime). Old keys stay listed (with `use: "sig"`) for as long as any token signed with them could still be valid, then are dropped.

---

## 12. Error Code Catalog

| Code | HTTP Status | Meaning |
|---|---|---|
| `AUTH_INVALID_CREDENTIALS` | 401 | Wrong email/password/tenant combination (deliberately non-specific). |
| `AUTH_ACCOUNT_DISABLED` | 403 | Credentials correct, account `status = DISABLED`. |
| `AUTH_INVALID_TOKEN` | 401 | Access token missing, malformed, expired, or revoked. |
| `AUTH_INVALID_REFRESH_TOKEN` | 401 | Refresh token invalid, expired, or already rotated/used. |
| `AUTH_RESET_TOKEN_INVALID` | 400 | Password-reset token invalid or expired. |
| `AUTH_MFA_REQUIRED` | 202 (not an error, informational) | Returned as `mfaRequired: true`, not a 4xx — the request is progressing correctly. |
| `AUTH_MFA_INVALID_CODE` | 401 | Wrong or expired OTP code. |
| `AUTH_MFA_CHALLENGE_EXPIRED` | 401 | The `mfaChallengeToken` itself expired (>5 min) — must restart login. |
| `AUTH_RATE_LIMITED` | 429 | Too many login/reset attempts; `Retry-After` header included. |
| `AUTH_VALIDATION_ERROR` | 400 | Malformed request body (missing field, bad format). |
| `DEPENDENCY_UNAVAILABLE` | 503 | **Missing from this catalog until now.** `Schemas/01_Authentication_Service_Schema.md` §12's edge-case table requires "User Service unavailable during login → Return controlled `503`, not invalid credentials" and the same for Organization Service — but no corresponding error code existed anywhere in this document. Added here to close that gap. This must never collapse into `AUTH_INVALID_CREDENTIALS`; a dependency outage is not the same failure mode as a wrong password, and conflating them would make the anti-enumeration property accidentally hide real outages from monitoring. |

---

## 13. Internal Endpoint Exposed by This Service

### `GET /internal/v1/auth/sessions/{tokenId}/status`
**Purpose:** lets a service that, for some reason, cannot do local JWT+Redis verification itself (e.g., a non-JVM tool, or a debugging/ops script) check revocation status without direct Redis access. **Not** intended to be the primary verification path for the 9 core services — those verify locally per §3's algorithm.
**Response `200 OK`**
```json
{ "tokenId": "8b2c1e4a-...", "revoked": false }
```

---

## 14. Outbound Contracts — Endpoints This Service Calls

These are owned/documented by Organization Service and User & Team Service respectively, listed here because Authentication's own behavior is meaningless without them.

### 14.0 `GET /internal/v1/organizations/by-slug/{slug}` — Organization Service
**Response `200 OK`**
```json
{ "id": "3fa85f64-5717-4562-b3fc-2c963f66afa6", "name": "Acme Bank", "slug": "acme-bank", "status": "ACTIVE" }
```
**Response `404`** if no organization has that slug — Authentication maps this to the same generic `AUTH_INVALID_CREDENTIALS` shown to the client (§5), never a distinguishable 404. A `200` with `status: "SUSPENDED"` is treated the same way (login rejected, generic message) — see §5's Step 1.

This is the **only** place `organizationSlug` gets resolved. Neither Authentication nor User & Team Service resolves a slug itself past this point — everything downstream uses the resolved `organizationId`.

### 14.1 `GET /internal/v1/users/lookup?organizationId={id}&email={email}` — User & Team Service
**Response `200 OK`**
```json
{ "userId": "9e1071ea-...", "status": "ACTIVE" }
```
**Response `404`** if no matching user in that tenant — Authentication maps this to the same generic `AUTH_INVALID_CREDENTIALS` shown to the client (§5), never a distinguishable 404.

### 14.2 `POST /internal/v1/users/{id}/verify-credentials` — User & Team Service
**Request**
```json
{ "plaintextPassword": "TempPass!2026" }
```
**Response `200 OK`**
```json
{ "valid": true, "status": "ACTIVE", "organizationId": "3fa85f64-5717-4562-b3fc-2c963f66afa6" }
```
`valid: false` (not an error status) lets Authentication decide the exact client-facing response itself. **`passwordHash` is never returned by this or any other User & Team endpoint, public or internal** — this is the one supported way Authentication verifies a password, and it never sees the hash itself.

### 14.3 `PATCH /internal/v1/users/{id}/password-hash` — User & Team Service
**Request**
```json
{ "newPlaintextPassword": "NewPass!2027" }
```
**Response `204 No Content`** — User & Team Service hashes internally; Authentication never computes or sees a hash.

---

## 15. Redis Keys Owned by This Service

**Naming resolved:** these keys use the hierarchical `resolve:auth:{domain}:{...}` form, matching `Schemas/01_Authentication_Service_Schema.md` §4 exactly (this table previously used shorter names like `refresh:{id}`; that inconsistency between the two "frozen" documents has been reconciled in favor of the hierarchical form, which the built Authentication Service actually implements).

| Key Pattern | Value | TTL | Used By |
|---|---|---|---|
| `resolve:auth:session:revoked:{tokenId}` | `"1"` | Remaining access-token life | Logout (§7); checked by every service during JWT verification (§3) |
| `resolve:auth:refresh:{refreshTokenId}` | JSON `{ userId, tenantId, issuedAt, expiresAt }` | Refresh-token lifetime | Login (§5), Refresh (§6) |
| `resolve:auth:mfa:otp:{userId}` | Hashed 6-digit code | ~5 min | MFA verify (§9) — matches `Schemas_High_Level.md` §4.1 exactly |
| `resolve:auth:mfa:challenge:{mfaChallengeToken}` | `{ userId }` | ~5 min | Binds a login's `202` response to the follow-up `/mfa/verify` call |
| `resolve:auth:password-reset:{resetToken}` | `{ userId, expiresAt }` | ~30 min | Password reset confirm (§8) |
| `resolve:auth:ratelimit:login:email:{normalizedEmail}:{window}` / `resolve:auth:ratelimit:login:ip:{ip}:{window}` | Integer counter | Matches window | Brute-force protection on `/auth/login`, checked independently by identity and by source. **Mandatory per Schema §13 item 7**, not merely recommended (this row previously said otherwise — corrected). No equivalent key exists yet for `/auth/password-reset/request`, which the same schema requirement also covers — that's an open gap, not a resolved omission. |

---

*Every endpoint above traces to an FR-IAM-* requirement (or is explicitly marked as a reasonable, flagged addition beyond the documented SRS scope — MFA challenge-token mechanics, JWKS publishing, and internal lookup/verification endpoints exist because FR-IAM-02/06/10 can't be implemented correctly without them, even though they aren't spelled out as separate line items in the SRS itself).*
