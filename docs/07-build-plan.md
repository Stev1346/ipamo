# 07 – Build Plan

> **Status:** Draft for approval · **Applies to:** v1
> **Related:** every doc. Each milestone lists exactly which docs and sections to read.

v1 is built in **14 small milestones, one feature at a time.** Each milestone ends in a merged pull request with green CI, and none starts until the previous one is merged.

---

## 1. Time budget (honest estimate)

| | |
| --- | --- |
| Availability | 4–5 hours a week |
| Estimated effort | **≈ 55 hours**, including learning time for framework concepts that are new to you |
| Realistic duration | **≈ 12 weeks** at 4.5 hours a week |
| Your target | ~10 weeks |

The scope we agreed (26 business rules, refresh-token auth, rental editing, overdue conflicts) is about two weeks larger than the target. There are two honest options:

- **A. Keep the full scope and plan for 12 weeks.** *Recommended.* Cutting rules from a portfolio piece weakens its main selling point, which is correctness.
- **B. Hit 10 weeks by moving M12** (overdue conflicts BR-14, settlement BR-17/18, guards BR-21/22, ≈ 4.5 h) **and part of M13 into v1.1.** The app is still usable for real rentals without them.

The estimates assume you write the code with Claude Code guiding and reviewing. If a milestone runs over, finish it properly rather than starting the next one; the following week's plan just moves along by one.

## 2. Milestone overview

| # | Milestone | Main rules / features | Est. hours | Week |
| --- | --- | --- | --- | --- |
| M0 | Repository and tooling | CI, Docker, solution skeleton, `/health` | 3 | 1 |
| M1 | Database foundation | EF Core, DbContext, test host, architecture tests | 4 | 1–2 |
| M2 | API conventions | ProblemDetails, JSON strictness, security headers, Swagger | 3 | 2 |
| M3 | Auth 1: register and login | Identity, JWT, invite code, lockout, auth by default | 5 | 3 |
| M4 | Auth 2: sessions and profile | Refresh rotation, logout, profile, password change | 4 | 4 |
| M5 | Categories and isolation | Owner query filter, composite keys, BR-23, BR-24 | 3 | 5 |
| M6 | Equipment | Individual/Bulk, paging, archive, BR-20, BR-23 | 5 | 5–6 |
| M7 | Contacts | CRUD, archive (reinforcement milestone) | 2.5 | 6 |
| M8 | Rentals 1: reserve | Rental model, create/read, BR-01, 06, 07, 08, 15, 16, 19 | 4 | 7 |
| M9 | Rentals 2: availability and locking | BR-02, BR-03, `/available`, concurrency | 5 | 8 |
| M10 | Rentals 3: checkout, return, cancel | BR-04, 05, 09–12 | 5 | 9 |
| M11 | Rentals 4: editing | BR-25, BR-26, notes | 4 | 10 |
| M12 | Overdue, settlement and guards | BR-13, 14, 17, 18, 21, 22; coverage gate on | 4.5 | 11 |
| M13 | v1 hardening and release | Security audit, README check, `v1.0.0` tag | 3 | 12 |

## 3. How every milestone runs

1. **Branch:** `mNN-short-name` (e.g. `m03-auth-login`) from an up-to-date `main`.
2. **Read:** Claude Code reads **only** the docs and sections listed for the milestone, plus `CLAUDE.md`.
3. **Teach, then build, in checkpoints:** for each step, Claude explains the concept and the plan (files, placement, commands). You write the code. Claude reviews it before the next step. Domain rules are written **test first**.
4. **Checkpoint at the end of each step:** it builds with no warnings, the step's tests pass, and you can explain what you just wrote.
5. **Pull request:** a description that names the BR IDs covered, plus the two checklists: doc 05 §11 (security) and doc 06 §10 (tests).
6. **Merge** when CI is green, then tick the milestone in §5.

**Commits:** small, with present-tense messages, e.g. `Add equipment archive endpoint (BR-20)`.

**Docs stay in step:** if building reveals a doc is wrong, update the doc **in the same pull request**. The docs remain the source of truth.

---

## 4. Milestones

### M0 – Repository and tooling (≈ 3 h)

**Goal:** an empty but production-shaped repo that builds and tests in CI.

**Read:** README · doc 02 §2 · doc 05 §8.3, §10 · doc 08

**Learn:** the SDK-style `.csproj`, `Directory.Build.props`, `global.json`, `.slnx` solutions, local dotnet tools, and how GitHub Actions runs .NET.

**Steps**
1. Create the GitHub repo `ipamo`. Add `.gitignore` (`dotnet new gitignore`, plus `.env`), `.editorconfig`, and the `docs/` folder with all approved docs.
2. Add `global.json` pinning the .NET 10 SDK, and `Directory.Build.props` with:
   - `Nullable` enabled
   - `TreatWarningsAsErrors` on
   - `AnalysisLevel` set to `latest-recommended`
   - `NuGetAudit` set to `all`
3. Create `Ipamo.slnx`, `src/Ipamo.Api` (`dotnet new webapi --use-controllers`), `tests/Ipamo.UnitTests` and `tests/Ipamo.IntegrationTests`. Set `RootNamespace` to `Ipamo`, and create the empty `Domain/`, `Application/`, `Infrastructure/` and `Api/` folders. Delete the template's WeatherForecast code.
4. Set the `https` launch profile to `https://localhost:7180`. Map `GET /health` (no database check yet).
5. Add `docker-compose.yml` (`postgres:17`, port `5433:5432`, a healthcheck, a named volume) and `.env.example`.
6. `dotnet new tool-manifest`, then `dotnet tool install dotnet-ef` and `dotnet tool install dotnet-reportgenerator-globaltool`. Both are local tools pinned in `.config/dotnet-tools.json`.
7. Add `.github/workflows/ci.yml` (checkout → setup .NET from `global.json` → restore → build → test) and `.github/dependabot.yml` (NuGet, GitHub Actions, Docker). Enable secret scanning and push protection on the repo.

**Definition of done**
- [ ] `dotnet build` has 0 warnings, and `dotnet test` is green (placeholder test).
- [ ] `docker compose up -d` gives a healthy Postgres container.
- [ ] `dotnet run` → `https://localhost:7180/health` returns `Healthy`.
- [ ] CI is green on `main`.

---

### M1 – Database foundation (≈ 4 h)

**Goal:** the API talks to PostgreSQL through EF Core, and the integration test host works.

**Read:** doc 02 §2.2, §3 ("The DbContext interface") · doc 03 §2, §7 · doc 06 §3, §5.1

**Learn:** dependency injection and service lifetimes (scoped DbContext), the Options pattern with startup validation, EF Core's DbContext and migrations workflow, `TimeProvider`, `WebApplicationFactory`, Testcontainers.

**Steps**
1. Add the packages: EF Core, Npgsql, `EFCore.NamingConventions`, and the EF Core health check.
2. Create `IIpamoDbContext` (Application/Abstractions) and `IpamoDbContext` (Infrastructure/Data). Add the snake_case naming convention.
3. Create `Infrastructure/DependencyInjection.cs` (`AddInfrastructure`) with a validated `ConnectionStrings:Ipamo` and `TimeProvider.System`. Register the audit-timestamp `SaveChanges` interceptor (it has no entities to act on yet).
4. Add the DB check to `/health`. Set user secrets as described in the README.
5. Test host: `IpamoApiFactory` (Postgres container, config overrides, `FakeTimeProvider`, migrations on start), `IntegrationTestBase` (Respawn), and a sequential collection.
6. Tests:
   - `/health` returns `200` against a real database
   - `HasPendingModelChanges()` is false
   - architecture rules A1–A4 (they pass trivially now, and protect every later milestone)

**Definition of done**
- [ ] The app fails to start if the connection string is missing, with a clear error.
- [ ] Integration tests start their own Postgres container, pass, and leave nothing running.
- [ ] Architecture tests A1–A4 are in place and green.
- [ ] CI is green, so Testcontainers works on GitHub's Ubuntu runners.

---

### M2 – API conventions (≈ 3 h)

**Goal:** every response, success or error, has the shape the spec promises.

**Read:** doc 02 §5, §6 · doc 04 §1 · doc 05 §6, §7 (headers)

**Learn:** the middleware pipeline and why its order matters, `IExceptionHandler`, ProblemDetails, System.Text.Json options, the built-in OpenAPI document plus Swagger UI.

**Steps**
1. Exceptions: `BusinessRuleException` (Domain/Common), `NotFoundException` and `ConflictException` (Application/Common). Add `Api/Errors/ProblemDetailsExceptionHandler`, which maps them to `status`, `code`, `ruleId` and `traceId`. Make `[ApiController]` validation responses use `code: validation.failed`.
2. JSON: camelCase, string enums with no integers, and `UnmappedMemberHandling.Disallow`. Set a 256 KB request body limit.
3. Security-headers middleware. Use the pipeline order from doc 02 §6.
4. `AddOpenApi()` plus Swagger UI, in Development only.
5. Tests: S12, S13, S14, the ProblemDetails shape for a thrown `NotFoundException` (via a temporary test-only endpoint, or deferred to M5), and `GET /openapi/v1.json` → `200`.

**Definition of done**
- [ ] No response leaks a stack trace outside Development.
- [ ] Swagger UI loads at `/swagger` in Development.
- [ ] Tests S12–S14 are green.

---

### M3 – Auth 1: register and login (≈ 5 h)

**Goal:** owners can register with an invite code and log in. Every endpoint requires a token by default.

**Read:** doc 03 §3.1 (users), §3.2 (`owner_profiles`) · doc 04 §2 (register, login), §3 (GET profile) · doc 05 §2, §3.1, §4, §7 (rate limiting) · ADR-0004

**Learn:** ASP.NET Core Identity's building blocks (`UserManager`, password hashers and validators), how a JWT is built, signed and validated, authentication vs. authorization, fallback policies, the rate limiter.

**Steps**
1. `ApplicationUser`, `IdentityUserContext`, and the `OwnerProfile` domain entity. Migration: `AddIdentityAndOwnerProfiles`. Read it before committing.
2. `IIdentityService` and `ITokenService` (Application), with their Infrastructure implementations.
3. Password policy, including the blocklist validator (embed the list as a resource).
4. `AuthService.RegisterAsync`: check the invite code (fixed-time, fails closed), create the user and profile in one transaction, issue an access token. Refresh tokens come in M4, so return a placeholder or omit them, and note this in the PR.
5. `LoginAsync`: lockout, dummy-hash timing, generic `401`.
6. `AddAuthentication().AddJwtBearer(...)` with **every** validation parameter from doc 05 §3.1, plus `ICurrentUser`, the fallback authorization policy and the `auth` rate-limit policy.
7. `GET /api/profile`.
8. Tests: S1–S5, S9, S10, S11 (auth by default), S15.

**Definition of done**
- [ ] Registering in Swagger, clicking Authorize and calling `GET /api/profile` works end to end.
- [ ] S11 passes, so no endpoint is reachable without a token unless documented.
- [ ] Startup fails with a signing key shorter than 64 bytes.

---

### M4 – Auth 2: sessions and profile (≈ 4 h)

**Goal:** long sessions without long-lived access tokens, plus profile management.

**Read:** doc 03 §3.1 (`refresh_tokens`) · doc 04 §2 (refresh, logout), §3 · doc 05 §3.2

**Learn:** opaque tokens vs. JWTs, hashing for storage, token rotation and reuse detection, time zones in .NET (`TimeZoneInfo`, IANA IDs).

**Steps**
1. `RefreshToken` entity and migration `AddRefreshTokens`.
2. Issue a refresh token at register and login. `POST /api/auth/refresh` with rotation and family revocation on reuse. `POST /api/auth/logout`.
3. `PUT /api/profile` (validate the time zone), and `POST /api/profile/password` (revokes all refresh tokens).
4. `IOwnerClock` (Application) and `OwnerClock` (Infrastructure). Its unit test covers the Edmonton and Tokyo cases from BR-06 #2 and #3 at the clock level.
5. Tests: S6, S7, S8, plus profile tests (invalid time zone → `422`).

**Definition of done**
- [ ] The full session cycle works: login → refresh → refresh → logout → refresh fails.
- [ ] No raw refresh token is stored in the database (check with `psql`).

---

### M5 – Categories and isolation (≈ 3 h)

**Goal:** the first owned resource, and the isolation machinery every later feature reuses.

**Read:** doc 03 §2 ("Ownership backstop"), §3.2 (`categories`) · doc 04 §4 · doc 05 §5 · doc 06 §7 (BR-23 #1–2), §7.1

**Learn:** EF Core global query filters (and the captured-value trap), alternate keys and composite FKs, idempotent endpoints.

**Steps**
1. `OwnedEntity` base (Domain/Common) and the `Category` entity with name normalisation. Configuration: alternate key `(owner_id, id)`, unique `(owner_id, normalized_name)`, query filter. Migration: `AddCategories`.
2. Seed the default categories during registration, in the same transaction.
3. `CategoryService` and `CategoriesController`: list, create, rename, archive, restore.
4. Tests:
   - BR-23 #1 and #2
   - isolation rows for categories
   - the **captured-value trap** test
   - DB backstop for the unique index

**Definition of done**
- [ ] A new owner sees exactly 8 default categories, and another owner sees none of them.
- [ ] Every category route returns `404` to a second owner.

---

### M6 – Equipment (≈ 5 h)

**Goal:** full equipment management for both Individual and Bulk items.

**Read:** doc 03 §3.2 (`equipment`) · doc 04 §1.2, §5 (excluding `/available`) · doc 06 §7 (BR-20, BR-23 #3–4)

**Learn:** rich domain entities (constructors and methods that enforce invariants), check constraints in EF configuration, composite FK mapping, building a reusable paging/search/sort helper, partial (filtered) unique indexes.

**Steps**
1. `Equipment` entity, with the `TrackingType` and `ItemCondition` enums. Its invariants are the tracking-type and quantity rules from doc 03. Write the unit tests first.
2. Configuration: check constraints, composite FK to categories, partial unique index on the serial number. Migration: `AddEquipment`.
3. A paging helper in `Application/Common/Paging` (validated `page`, `pageSize` and `sort`, plus the envelope).
4. Equipment endpoints: list with filters, get, create, update, archive and restore. `trackingType` can't be updated. The BR-21 and BR-22 guards are **added in M12**, once rentals exist.
5. Tests: BR-20 #1–2, BR-23 #3–4, isolation rows, DB backstops for each check constraint, and paging and sort validation.

**Definition of done**
- [ ] Bulk and Individual items can both be created, and invalid combinations get the documented `422` codes.
- [ ] A `DELETE` returns `405`.

---

### M7 – Contacts (≈ 2.5 h)

**Goal:** contacts with archiving. A short milestone that reinforces the M5/M6 patterns without help.

**Read:** doc 03 §3.2 (`contacts`), §6 · doc 04 §6

**Learn:** applying a pattern on your own. Claude reviews rather than explains, unless you ask.

**Steps:** the `Contact` entity, its configuration and the migration `AddContacts`, then the service, the controller, and search across name, email, phone and organization.

**Tests:** isolation rows, archive and restore, search.

**Definition of done**
- [ ] Contacts work end to end, and no contact details appear in the logs during the test run.

---

### M8 – Rentals 1: reserve (≈ 4 h)

**Goal:** create and read reservations. **Without** availability checks (that's M9), so the model can be learned first.

**Read:** doc 01 §4, §5 (BR-01, 06, 07, 08, 15, 16, 19) · doc 03 §3.3, §5.3 · doc 04 §7.1, §7.2 (`GET` list and by ID), §7.3 (`POST`) · doc 06 §7 (those BRs)

**Learn:** aggregates (a `Rental` controls its `RentalItem`s), one-to-many configuration, calculated properties versus stored ones, decimal arithmetic, mapping to nested responses.

**Steps**
1. `Rental` and `RentalItem` entities with `Rental.Reserve(...)`. Domain unit tests first: BR-01, BR-07 and BR-16.
2. Configuration: check constraints, status consistency, composite FKs, the `xmin` token. Migration: `AddRentals`.
3. `RentalService.CreateAsync`: start not in the past (BR-06), active contact and equipment (BR-08), rate snapshot (BR-15), currency snapshot (BR-19).
4. `GET /api/rentals/{id}` and `GET /api/rentals`, with filters and paging.
5. Tests: those BRs' cases from doc 06 §7, isolation rows, and DB backstops.

**Definition of done**
- [ ] A reservation can be created and read back with the correct `rentalDays`, `lineFee`, `subtotal` and `total`.
- [ ] The PR states clearly that double booking is **not yet prevented** (M9).

---

### M9 – Rentals 2: availability and locking (≈ 5 h)

**Goal:** no double bookings, even under concurrent requests.

**Read:** doc 01 BR-02, BR-03 · doc 02 §4 (concurrency) · doc 03 §5.1, §5.2 · doc 04 §5 (`/available`), §7.3 (`409 rental.unavailable`) · doc 06 §7 (BR-02, BR-03)

**Learn:** database transactions in EF Core, `SELECT … FOR UPDATE` row locks, raw parameterised SQL (`FromSql`/`SqlQuery`), `generate_series`, and how to write a deterministic concurrency test.

**Steps**
1. `IEquipmentLocks` (Application) and the Npgsql implementation (Infrastructure). IDs are locked in sorted order, and the query filters on `owner_id`.
2. `AvailabilityService`: per-day occupancy and availability, following doc 03 §5.1–5.2 exactly (with the overdue `max(due, today)` rule ready for M10).
3. Wire it into `CreateAsync` inside a transaction: lock → check → insert → commit. Return `409` with the `unavailable[]` array.
4. `GET /api/equipment/available`.
5. Tests: BR-02 #1–6 and BR-03 #1–4, including **both concurrency tests**, and the availability isolation row.

**Definition of done**
- [ ] The concurrency tests pass 20 times in a row (`for i in {1..20}; do dotnet test --filter "Category=Concurrency" || break; done`).
- [ ] You can explain why the lock order is sorted.

---

### M10 – Rentals 3: checkout, return and cancel (≈ 5 h)

**Goal:** the full lifecycle, from reservation to return.

**Read:** doc 01 §4, BR-04, 05, 09–12 · doc 03 §5.1, §5.4 · doc 04 §7.4 · doc 06 §7 (those BRs)

**Learn:** state transitions as domain methods, optimistic concurrency with `xmin`, mapping `DbUpdateConcurrencyException` to `409`.

**Steps**
1. Domain: `CheckOut`, `ReturnItems` and `Cancel`, test first (BR-09 to BR-12).
2. Endpoints: `checkout`, `returns` and `cancel`. Update an Individual item's condition when it's returned.
3. Make the occupancy rules for returned and overdue lines live (BR-04, BR-05).
4. Tests: BR-04, BR-05 and BR-09 to BR-12 cases, the double-return concurrency test, and isolation rows.

**Definition of done**
- [ ] A rental can go Reserved → CheckedOut → (partial) → Returned entirely through Swagger.
- [ ] An early return makes the next day available, as shown by `/available`.

---

### M11 – Rentals 4: editing (≈ 4 h)

**Goal:** change reservations, and extend rentals that are already out.

**Read:** doc 01 BR-25, BR-26 · doc 04 §7.3 (`PUT`, `change-due-date`, `notes`) · doc 06 §7 (BR-25, BR-26)

**Learn:** all-or-nothing updates, merging child collections in EF Core (matching by key, adding, removing), reusing the availability check while excluding the rental's own lines.

**Steps**
1. Domain: `EditReservation(...)` and `ChangeDueDate(...)`, test first.
2. `AvailabilityService` gains an "exclude rental ID" option.
3. Endpoints: `PUT /api/rentals/{id}`, `change-due-date` and `notes`, all using the M9 lock-and-check transaction.
4. Tests: BR-25 #1–7 and BR-26 #1–6, including #6 for BR-25 (the rental is unchanged after a failed edit).

**Definition of done**
- [ ] Extending a rental into someone else's booking is refused, and extending into free days reprices the rental.

---

### M12 – Overdue, settlement and guards (≈ 4.5 h)

**Goal:** the remaining rules, and turning on the coverage gate.

**Read:** doc 01 BR-13, 14, 17, 18, 21, 22 · doc 03 §5.2 (quantity guard), §5.4 · doc 04 §5 (archive, `PUT` errors), §6 (archive), §7.2 (`active`, `overdue`), §7.3 (`settlement`) · doc 06 §7, §9

**Learn:** read-only query projections (`AsNoTracking`, `Select` into response records), expressing "today" consistently across queries.

**Steps**
1. `GET /api/rentals/active` and `GET /api/rentals/overdue`, with conflict detection (BR-13, BR-14).
2. `PUT /api/rentals/{id}/settlement` (BR-17, BR-18).
3. Guards: archiving equipment or a contact on an active rental (BR-21), and lowering a Bulk quantity below what's committed (BR-22, inside the lock transaction).
4. Turn on the CI coverage threshold (80% for Domain and Application combined).
5. Tests: the remaining BR cases and isolation rows for `active` and `overdue`.

**Definition of done**
- [ ] Every BR-01 to BR-26 has at least one test tagged with its rule ID (check with `dotnet test --list-tests`).
- [ ] CI fails if coverage drops below 80%.

---

### M13 – v1 hardening and release (≈ 3 h)

**Goal:** confidence that v1 meets the success criteria in doc 01 §8.

**Read:** doc 01 §8 · doc 05 §11, §12 · doc 06 §10

**Steps**
1. Go through the doc 05 §11 checklist across the whole codebase, and fix anything it finds.
2. Compare the OpenAPI document with doc 04, endpoint by endpoint. Fix whichever one is wrong.
3. Fresh-clone test: follow the README word for word in a new folder, and fix any step that doesn't work.
4. Review dependencies for vulnerabilities (`dotnet list package --vulnerable`).
5. Update the README status, then tag `v1.0.0` and publish a GitHub release with notes.
6. Start the **two-week real-use period** (doc 01 §8, "Practical"). Log any issues as GitHub Issues labelled `v1.1`.

**Definition of done**
- [ ] Every box in doc 01 §8 is ticked, except the two-week real-use period, which starts now.
- [ ] `v1.0.0` is tagged on `main`.

---

## 5. Progress

| Milestone | Status | PR | Notes |
| --- | --- | --- | --- |
| M0 | Not started | | |
| M1 | Not started | | |
| M2 | Not started | | |
| M3 | Not started | | |
| M4 | Not started | | |
| M5 | Not started | | |
| M6 | Not started | | |
| M7 | Not started | | |
| M8 | Not started | | |
| M9 | Not started | | |
| M10 | Not started | | |
| M11 | Not started | | |
| M12 | Not started | | |
| M13 | Not started | | |

## 6. After v1 (not scheduled)

These are listed in the likely order. Each one gets its own docs update before it's built.

1. **v1.1:** Azure deployment (App Service, Azure Database for PostgreSQL, Key Vault, forwarded headers), plus fixes from the real-use period.
2. **Project split** (doc 02 §10), when the reminders function needs to share code.
3. **Reminders** via an Azure Function, plus email password reset.
4. **React + TypeScript front end** (cookie-based refresh token; doc 05 §3.3).
5. **Photos** in Azure Blob Storage.
6. **Photo-to-form AI.**
7. **Pre-launch items:** per-person invite tokens, contact anonymisation, MFA.
