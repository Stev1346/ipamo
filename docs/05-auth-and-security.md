# 05 – Auth & Security

> **Status:** Draft for approval · **Applies to:** v1
> **Related:** [02 – Architecture](02-architecture.md) · [03 – Data model](03-data-model.md) · [04 – API spec](04-api-spec.md) · [ADR-0004](decisions/0004-jwt-with-identity.md)

This document defines how Ipamo authenticates owners, keeps each owner's data private, validates input and handles secrets. Where it gives a number (a lifetime, limit or length), that number is the spec: code and tests use it exactly.

---

## 1. Threat model (short)

| Asset | Threat | Main defences |
| --- | --- | --- |
| Owner accounts | Password guessing, credential stuffing, stolen tokens | Strong password hashing, lockout, rate limits, short-lived access tokens, refresh rotation with reuse detection |
| Each owner's data | Another owner reading or changing it (IDOR) | Owner query filter, composite FKs, unguessable IDs, `404` for foreign records, isolation tests (§5) |
| Contacts' personal information | Leaking through logs, errors or other owners | No personal information in logs or errors, owner isolation (§9) |
| Secrets (signing key, DB password, invite code) | Committed to Git, leaked in logs | User secrets locally, Key Vault later, startup validation, GitHub secret scanning (§8) |
| The service | Abuse of public endpoints, oversized payloads | Invite-only registration, rate limiting, request size limits (§6, §7) |

**Out of scope for v1:** multi-factor authentication, account recovery by email, admin tooling, and protection against a fully compromised host.

## 2. Accounts and passwords

### 2.1 Identity setup

- `AddIdentityCore<ApplicationUser>()` with `IdentityUserContext<ApplicationUser, Guid>`. That gives users only: no roles, no cookie sign-in, and none of the default Identity UI or `MapIdentityApi` endpoints.
- The email address is the username, and `RequireUniqueEmail = true`.
- `ApplicationUser` lives in `Infrastructure/Identity/`. Application code reaches it only through `IIdentityService` (doc 02 §3).

### 2.2 Password policy

This follows current NIST SP 800-63B guidance: **length over composition rules.**

| Rule | Value |
| --- | --- |
| Minimum length | 12 characters |
| Maximum length | 128 characters (checked in the API, so huge inputs never reach the hasher) |
| Character-type rules | **None.** No "must have a digit or symbol" rules (`RequireDigit = false`, etc.) |
| Blocklist | Rejected if it's on an embedded list of the 10,000 most common passwords, or if it contains the user's email local part or display name. This is a custom `IPasswordValidator`. |
| Error | `400 auth.weak_password`, with a human-readable reason |

### 2.3 Password hashing

Ipamo uses ASP.NET Core Identity's default `PasswordHasher` (format v3): **PBKDF2 with HMAC-SHA512, 100,000 iterations, a 128-bit random salt and a 256-bit derived key**, stored with a version marker so hashes can be upgraded automatically when a user logs in.

- **Trade-off:** Argon2id is the stronger modern choice, but it needs a third-party package and its own tuning. Identity's hasher is maintained by Microsoft, well reviewed and good enough for this threat model. The version marker means the hashing algorithm can be swapped later without forcing password resets.
- Plain-text passwords never leave the request handler. They are never logged, stored or echoed back.

### 2.4 Login and lockout

| Setting | Value |
| --- | --- |
| Failed attempts before lockout | 5 |
| Lockout duration | 15 minutes |
| Response while locked out | The same `401 auth.invalid_credentials` as any other failure (doc 04) |
| Unknown email | A password hash check still runs against a fixed dummy hash, so response time doesn't reveal whether the account exists |
| On success | The failed-attempt count is reset |

**Trade-off:** a generic `401` doesn't tell a locked-out owner what happened. With only a few owners, they can ask. That's preferable to letting attackers discover which emails have accounts.

### 2.5 Registration (invite-only)

- The request must include an `inviteCode` that matches `Registration:InviteCode`. The comparison uses `CryptographicOperations.FixedTimeEquals`, so timing doesn't leak how close a guess was.
- **Fails closed:** if `Registration:InviteCode` is missing or empty, registration is **disabled**, and every attempt gets `403 auth.invalid_invite`.
- In one transaction, registration creates the user, the `owner_profiles` row and the default categories, then issues tokens.
- To rotate the invite code, change the configuration value. Existing accounts aren't affected.

## 3. Tokens

### 3.1 Access tokens (JWT)

| Aspect | Value |
| --- | --- |
| Format | JWT (JWS), signed with **HS256** using `Jwt:SigningKey` |
| Created with | `JsonWebTokenHandler` (`Microsoft.IdentityModel.JsonWebTokens`), the current Microsoft library |
| Validated by | `Microsoft.AspNetCore.Authentication.JwtBearer` |
| Lifetime | **15 minutes** (`Jwt:AccessTokenMinutes`) |
| Claims | `sub` (owner ID), `jti` (random ID), `iat`, `exp`, `iss`, `aud`. Deliberately nothing else: no email and no personal information. |
| Transport | `Authorization: Bearer …` header only. Never in query strings. |

**Validation parameters, all required:**
- `ValidateIssuer` against `Jwt:Issuer`
- `ValidateAudience` against `Jwt:Audience`
- `ValidateLifetime` and `RequireExpirationTime`
- `ValidateIssuerSigningKey` and `RequireSignedTokens`
- `ValidAlgorithms = [HS256]`. This blocks "alg: none" and algorithm-confusion attacks.
- `ClockSkew = 30 seconds`. The default is 5 minutes, which is too generous for 15-minute tokens.
- `MapInboundClaims = false`, so `sub` stays `sub` instead of being renamed to a long legacy claim URI.

**Signing key rules:** at least **64 random bytes**, base64-encoded. It's generated with `openssl rand -base64 64` (see README), and startup fails if the decoded key is shorter.

**Why HS256 (symmetric) and not RS256 (asymmetric)?** The same app both issues and validates tokens, so one shared secret is simplest and safe. Switch to RS256 (private key in Key Vault, public key published) if another service ever needs to validate tokens itself. See [ADR-0004](decisions/0004-jwt-with-identity.md).

**Revocation lag:** access tokens are not checked against the database on each request. A logged-out or password-changed session can therefore keep working for **up to 15 minutes**. That's the accepted cost of not hitting the database on every request.

### 3.2 Refresh tokens

| Aspect | Value |
| --- | --- |
| Format | **Opaque**, not a JWT: 64 bytes from `RandomNumberGenerator`, base64url-encoded |
| Stored as | The SHA-256 hash (`refresh_tokens.token_hash`). The raw value exists only in the response. |
| Lifetime | **14 days** from when each token is issued (`Jwt:RefreshTokenDays`) |
| Rotation | Every successful refresh revokes the token it used and issues a new one in the same `family_id`, linked via `replaced_by_id` |
| Reuse detection | If a **revoked** token from a family is presented again, the token was probably stolen. **Every token in that family is revoked**, and the request gets `401 auth.invalid_refresh_token`. The event is logged as a warning with the owner ID only. |
| Logout | Revokes the presented token's family |
| Password change | Revokes **all** of the owner's refresh tokens |

**Why a fast hash (SHA-256) and not PBKDF2?** Refresh tokens are 512 bits of randomness, so they can't be brute-forced the way human passwords can. Slow hashing only protects low-entropy secrets. A fast hash still means a database leak doesn't give out usable tokens.

**Known edge case:** two clients refreshing with the same token at the same moment will look like reuse, and the session is logged out. That's acceptable for an API-first v1. If it becomes a problem with the front end, add a short grace window (around 10 seconds) during which a just-rotated token returns the same successor.

**Housekeeping:** expired and revoked tokens are kept in v1 (the table grows slowly at this scale). A clean-up job is a post-v1 task.

### 3.3 When the React front end arrives (planned, not v1)

- The refresh token moves to an `HttpOnly; Secure; SameSite=Strict` cookie scoped to `/api/auth`. The access token is kept in memory only, never in `localStorage`.
- Because cookies are then involved, the refresh and logout endpoints get CSRF protection: `SameSite=Strict` plus a required custom header.
- The API contract in doc 04 stays the same for non-browser clients.

## 4. Authorization

- **Secure by default:** a global fallback policy, `RequireAuthenticatedUser()`. Only `register`, `login`, `refresh`, `logout` and `/health` have `[AllowAnonymous]`. In Development, the OpenAPI document and Swagger UI are also mapped as anonymous.
- There are no roles or permissions in v1. Being authenticated means you're an owner, and an owner can do anything to **their own** data.
- `ICurrentUser.OwnerId` (Application interface, implemented in Infrastructure) reads the `sub` claim. If it's missing or not a GUID, the request is rejected with `401`. It never falls back to a default.

## 5. Data isolation (BR-24)

Four layers, each enough to stop most bugs on its own:

| # | Layer | Mechanism |
| --- | --- | --- |
| 1 | **No owner input** | Request records have **no `ownerId` field**. Services set `OwnerId` from `ICurrentUser` when creating records, so there's nothing to over-post. Unknown JSON properties are rejected anyway (§6). |
| 2 | **Global query filter** | Every owned entity has `HasQueryFilter(e => e.OwnerId == CurrentOwnerId)` in `IpamoDbContext`. `owner_profiles` filters on `Id`. |
| 3 | **Composite FKs** | The database rejects links between records of different owners (doc 03 §2). |
| 4 | **Tests** | Every endpoint with an `{id}` route, and every body field that references an ID, is tested as a second owner, and must give `404` or the `*_not_found` error. Lists are tested to contain only the caller's records. See [06 – Testing strategy](06-testing-strategy.md). |

**Rules for working with the query filter:**
- The filter must reference a **property on the DbContext instance** (e.g. `CurrentOwnerId`, which reads `ICurrentUser`), not a value captured when the model is built. EF Core builds the model once and caches it. A captured value would freeze the *first* user's ID into every query. This is a classic bug, and a dedicated integration test covers it.
- `IgnoreQueryFilters()` is **banned** outside `Infrastructure/`. Inside Infrastructure it's allowed only with a comment explaining why, e.g. checking email uniqueness during registration.
- Raw SQL, such as the row lock in doc 02 §4, **always includes `owner_id = @owner`** as a parameter, because query filters don't apply to raw SQL.
- **Archived** records are not filtered globally. Each list query decides whether to include them.
- Another owner's record always gives **`404`, never `403`.** A `403` would confirm the record exists.

**Unguessable IDs:** UUID v7 values can't be enumerated. That's a bonus, not a control: isolation never relies on IDs being secret.

## 6. Input validation and request handling

| Area | Rule |
| --- | --- |
| **Shape validation** | DataAnnotations on request records (`[Required]`, `[MaxLength]`, `[Range]`, `[EmailAddress]`). `[ApiController]` returns `400 validation.failed` automatically. |
| **Business rules** | In the domain and services only (doc 02 §4). Never in attributes. |
| **Unknown JSON fields** | Rejected with `400` (`JsonUnmappedMemberHandling.Disallow`). This catches client bugs and over-posting attempts. |
| **Enums** | Accepted as strings only. Integers are rejected (`JsonStringEnumConverter` with `allowIntegerValues: false`). |
| **Strings** | Trimmed by the domain. Empty-after-trim strings are treated as missing where a field is required. |
| **Body size** | 256 KB maximum (`KestrelServerOptions.Limits.MaxRequestBodySize`). Larger requests get `413`. No v1 request comes close. |
| **Collections** | Explicit maximums, e.g. a rental has at most 100 items, and `pageSize` is at most 100 (doc 04). |
| **Dates** | Strict `YYYY-MM-DD` parsing. Range limits are in doc 04 (e.g. availability at most 366 days). |
| **SQL injection** | EF Core parameterises all LINQ. Raw SQL uses `FromSql`/`SqlQuery` with **interpolated parameters** only. `FromSqlRaw` built from string concatenation is banned. |
| **Errors** | Clients get ProblemDetails only. Stack traces and exception messages appear **only in Development**. In other environments a `500` returns a generic message plus `traceId`. |

## 7. Transport, headers and abuse protection

**HTTPS and headers**
- HTTPS redirection everywhere. HSTS outside Development (`max-age` of 1 year once deployed).
- A small middleware adds these headers to API responses: `X-Content-Type-Options: nosniff`, `Referrer-Policy: no-referrer`, `X-Frame-Options: DENY`, and `Content-Security-Policy: default-src 'none'; frame-ancestors 'none'`. Swagger UI paths are excluded in Development.

**CORS**
- Allowed origins come from `Cors:AllowedOrigins`, which is empty in v1, so no cross-origin access is possible.
- There is never a wildcard (`*`) origin. Credentials are allowed only when the cookie-based refresh token arrives (§3.3).

**Rate limiting** (built-in ASP.NET Core rate limiter)

| Policy | Applies to | Limit | Partitioned by |
| --- | --- | --- | --- |
| `auth` | `register`, `login`, `refresh` | 10 requests per minute (fixed window) | Client IP |
| `api` | All authenticated endpoints | 300 requests per minute (token bucket) | Owner ID |

Exceeding a limit gives `429 rate_limited` with a `Retry-After` header.

**Deployment note:** behind Azure's front end, the client IP arrives in `X-Forwarded-For`. `ForwardedHeaders` middleware must be configured with the known proxy, or every user would share one rate-limit bucket. This goes on the deployment checklist.

## 8. Secrets and configuration

### 8.1 Configuration keys

| Key | Secret? | Default / example | Validated at startup |
| --- | --- | --- | --- |
| `ConnectionStrings:Ipamo` | **Yes** | `Host=localhost;Port=5433;Database=ipamo;Username=ipamo;Password=…` | Present |
| `Jwt:SigningKey` | **Yes** | `openssl rand -base64 64` | Decodes to ≥ 64 bytes |
| `Jwt:Issuer` | No | `ipamo-api` | Present |
| `Jwt:Audience` | No | `ipamo-clients` | Present |
| `Jwt:AccessTokenMinutes` | No | `15` | 1–60 |
| `Jwt:RefreshTokenDays` | No | `14` | 1–90 |
| `Registration:InviteCode` | **Yes** | (any string ≥ 12 characters) | Empty means registration is disabled; when set, it must be ≥ 12 characters |
| `Cors:AllowedOrigins` | No | `[]` | Each entry is an absolute `https://` URL (`http://localhost` allowed in Development) |

Non-secret defaults go in `appsettings.json`. Secrets **never** go in any `appsettings*.json`. Each layer binds and validates its own Options classes with `ValidateDataAnnotations().ValidateOnStart()`, so a misconfigured app **refuses to start** rather than failing on the first request.

### 8.2 Where secrets live

| Environment | Store |
| --- | --- |
| Local development | .NET user secrets (`dotnet user-secrets`). Stored in `~/.microsoft/usersecrets/<id>/secrets.json`, outside the repo. |
| Local Docker Postgres | `.env` (in `.gitignore`). `.env.example` contains placeholders only. |
| Integration tests | Generated per test run (a random signing key and a throwaway container password). No real secrets needed. |
| GitHub Actions | None needed for v1, because tests generate their own |
| Azure (later) | App Service settings that reference **Azure Key Vault**, read through a managed identity |

### 8.3 Keeping secrets out of Git

- `.gitignore` covers `.env`, `*.user`, `appsettings.*.local.json` and `secrets.json`.
- Enable **GitHub secret scanning and push protection** on the repository. Both are free for public repos.
- If a secret is ever committed, **rotate it first**, then clean the history. Removing it from history alone isn't enough.

## 9. Logging and privacy

**Log:**
- Owner ID, request path, status code, duration and `traceId`.
- Auth events: login success or failure (with the owner ID if known, never the email), lockouts, refresh-token reuse, password changes.

**Never log:**
- Passwords, tokens, the `Authorization` header, or the invite code.
- Request or response bodies of `/api/auth/*` and `/api/profile/password`.
- Contact names, emails or phone numbers.

When a business error is logged, refer to entities by ID only.

Log level: `Information` by default, and `Warning` for auth anomalies. EF Core SQL logging is enabled in Development only, and **with parameter values hidden** (`EnableSensitiveDataLogging` is never turned on outside a local debugging session).

## 10. Dependencies and supply chain

- NuGet Audit is on (`<NuGetAudit>true</NuGetAudit>`, `<NuGetAuditMode>all</NuGetAuditMode>`), and vulnerability warnings are treated as errors in CI.
- Dependabot keeps NuGet packages, GitHub Actions and the `postgres` Docker image updated, opening weekly pull requests.
- Packages are added only with a stated reason in the pull request. Prefer Microsoft-maintained packages over third-party ones (doc 02 §5).
- The `postgres` image is pinned to a major version (`postgres:17`).

## 11. Security checklist (every pull request)

- [ ] New endpoints are authenticated by default; any `[AllowAnonymous]` is justified.
- [ ] No `ownerId` in request records; `OwnerId` is set from `ICurrentUser`.
- [ ] Isolation tests were added for new `{id}` routes and body references.
- [ ] No `IgnoreQueryFilters()` outside Infrastructure; raw SQL filters on `owner_id`.
- [ ] New string fields have `[MaxLength]` matching doc 03.
- [ ] No secrets, tokens or personal information in code, config, logs or test fixtures.
- [ ] Errors use ProblemDetails with a catalogued `code` (doc 04 §9).
- [ ] New packages are justified, and NuGet Audit is clean.

## 12. Known v1 limitations (accepted)

| Limitation | Why it's accepted | Planned fix |
| --- | --- | --- |
| No email password reset | No email infrastructure in v1 | With the reminders work (email provider) |
| Up to 15 minutes of revocation lag | Avoids a database lookup per request | Acceptable long term; a security-stamp check if ever needed |
| No MFA | Small, invite-only user base | Consider before opening to the public |
| Concurrent refreshes look like token reuse | Rare for API clients | Grace window (§3.2) |
| Single shared invite code | Fine for a small circle | Per-person invite tokens before any public launch |
