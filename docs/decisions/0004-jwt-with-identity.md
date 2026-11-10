# ADR-0004: ASP.NET Core Identity + self-issued JWT + rotating refresh tokens

- **Status:** Accepted
- **Date:** 2026-11-09
- **Deciders:** Stephen Okanlawon
- **Related docs:** [05 – Auth & security](../05-auth-and-security.md) · [04 – API spec](../04-api-spec.md) §2 · [03 – Data model](../03-data-model.md) §3.1

## Context

Ipamo is an API-first, multi-user app for a small, invite-only group of owners. It needs:
- secure password storage;
- protection against brute force;
- stateless request authentication that a future React front end and an Azure Function can use;
- sessions that last days without long-lived access tokens.

The requirements call for **JWT**, and learning how JWT authentication works end to end is a goal of the project.

## Options considered

### Option A – Hand-rolled user table, password hashing and tokens
- **Pros:** maximum learning.
- **Cons:** rolling your own password hashing and lockout is a well-known source of vulnerabilities. Rejected.

### Option B – ASP.NET Core Identity with `MapIdentityApi` (built-in endpoints)
- **Pros:** almost no code; Microsoft maintained.
- **Cons:**
  - Issues Identity's own **opaque bearer tokens, not JWTs**, so it doesn't meet the requirement.
  - Fixed endpoint shapes that don't match our API spec or error format.
  - No invite-code hook.
  - Hides the parts we want to learn.

### Option C – External identity provider (Microsoft Entra External ID, Auth0, Clerk…)
- **Pros:** MFA, password reset and social login out of the box, with security offloaded.
- **Cons:**
  - An external dependency and account, possible cost, and more complex local development and testing.
  - Little JWT-issuing learning.
  - Overkill for under 10 invited users.

### Option D – Identity core (users, hashing, lockout) + self-issued JWT + opaque rotating refresh tokens
- **Pros:**
  - Proven password hashing, lockout and user store, without the UI or `MapIdentityApi`.
  - Full control of the token format, claims, lifetimes and API shape.
  - Standard JWT validation via `JwtBearer`.
  - Teaches the whole flow.
- **Cons:**
  - We own the token-issuing and refresh-rotation code, and it must be tested thoroughly (doc 06 S6–S11).
  - No password reset by email or MFA in v1.

### Option E – Cookie authentication
- **Pros:** simplest for a same-site browser app.
- **Cons:** awkward for non-browser clients (Azure Function, tests, Swagger), and needs CSRF handling from day one. It will partly return for the *refresh token only* when the front end arrives (doc 05 §3.3).

## Decision

We will use **Option D**:
- `AddIdentityCore<ApplicationUser>()` with `IdentityUserContext<ApplicationUser, Guid>` (no roles), in Infrastructure.
- **Access tokens:** JWTs signed with **HS256**, a 64-byte key, **15-minute** lifetime, claims `sub`/`jti`/`iat`/`exp`/`iss`/`aud` only, and strict validation (pinned algorithm, 30-second clock skew).
- **Refresh tokens:** opaque 64-byte random values, stored **SHA-256 hashed**, **14-day** lifetime, **rotated on every use**, with **family revocation on reuse**.
- Registration requires an invite code and fails closed.

## Consequences

- **Positive:**
  - Standard, inspectable tokens that any client can use.
  - Short exposure if an access token leaks, and theft detection for refresh tokens.
  - No external dependencies.
- **Negative:**
  - Up to 15 minutes of lag before a logout or password change takes effect.
  - Concurrent refreshes with the same token look like reuse (doc 05 §3.2).
  - We maintain security-critical code, which is mitigated by the test suite and the PR security checklist.
- **Follow-ups:**
  - The cookie-based refresh token and CSRF protection when the React front end ships.
  - Email password reset with the reminders work.
  - MFA before any public launch.

## Revisit when

- Another service must validate tokens independently. Then switch to **RS256** with the key in Azure Key Vault.
- The user base opens to the public, or MFA or social login is required. Then reconsider an external identity provider (Option C).
