# 02 – Architecture

> **Status:** Draft for approval · **Applies to:** v1
> **Related:** [01 – Product scope](01-product-scope.md) · [03 – Data model](03-data-model.md) · [05 – Auth & security](05-auth-and-security.md) · [ADRs](decisions/)

This document describes how Ipamo is put together and why. The guiding principle is **the simplest structure that keeps the business rules correct, testable and easy to find, while making a later move to Clean Architecture a mechanical step rather than a rewrite.** Every pattern here has to earn its place. §9 lists what was deliberately left out, and §10 shows how the split into separate projects works when the time comes.

---

## 1. System context

```text
 v1                                        later
 ──                                        ─────
 Swagger UI / curl / tests                 React + TS front end
        │                                  Azure Function (reminders)
        ▼                                         │
 ┌───────────────────────────┐                    │
 │   Ipamo.Api               │◄───────────────────┘
 │   ASP.NET Core Web API    │──── later ───► Azure Blob Storage (photos)
 │   (.NET 10)               │──── later ───► AI vision provider
 └─────────────┬─────────────┘
               │ EF Core (Npgsql)
               ▼
        PostgreSQL 17
```

There is one deployable app and one database. Everything in the right-hand column is future work; §8 says where each piece plugs in.

## 2. Approach: Clean Architecture layers inside one project

v1 ships as **one project** (`Ipamo.Api`), but its code is already organised into the four Clean Architecture layers. Each layer lives in its own top-level folder, with a namespace that matches the separate project it would become:

| Folder | Namespace (= future project) | Contains |
| --- | --- | --- |
| `Domain/` | `Ipamo.Domain` | Entities, enums, business rules that need no database |
| `Application/` | `Ipamo.Application` | Use-case services, request and response records, mappings, and the **interfaces** for everything external |
| `Infrastructure/` | `Ipamo.Infrastructure` | EF Core DbContext and migrations, Identity, JWT, row locking, clock: the **implementations** of those interfaces |
| `Api/` | `Ipamo.Api` | Controllers, exception-to-ProblemDetails mapping, auth and rate-limit setup |

Dependencies point **inward only**:

```text
   Api ──► Application ──► Domain
    │           ▲
    ▼           │ implements interfaces
 Infrastructure ┘──► Domain
```

The folders alone don't stop someone from breaking a boundary, so **architecture tests do** (§2.2). If those tests pass, splitting into four projects later is a matter of moving folders and adding project references, with no logic changes (§10).

**Why not four projects from day one?** A multi-project solution makes the compiler enforce the boundaries, but it costs extra `.csproj` files, reference management and slower builds. That ceremony doesn't pay for itself for one developer and roughly 10 entities. The architecture tests give the same guarantee with none of that overhead. See [ADR-0005](decisions/0005-project-structure.md).

**Trade-off accepted:** a feature now spans several folders (e.g. `Application/Rentals/` and `Api/Controllers/RentalsController.cs`) instead of one. That's the price of a seamless split later.

### 2.1 Folder layout

```text
ipamo/
├── Ipamo.slnx
├── src/
│   └── Ipamo.Api/
│       ├── Ipamo.Api.csproj           # <RootNamespace>Ipamo</RootNamespace>, so folders map to Ipamo.Domain etc.
│       ├── Program.cs                 # calls AddApplication(), AddInfrastructure(), AddApi(), then the pipeline
│       ├── appsettings.json           # non-secret defaults only
│       │
│       ├── Domain/                    # namespace Ipamo.Domain.*
│       │   ├── Common/                # OwnedEntity base, BusinessRuleException
│       │   ├── Owners/OwnerProfile.cs
│       │   ├── Categories/Category.cs
│       │   ├── Equipment/             # Equipment.cs, TrackingType.cs, ItemCondition.cs
│       │   ├── Contacts/Contact.cs
│       │   └── Rentals/               # Rental.cs, RentalItem.cs, RentalStatus.cs
│       │
│       ├── Application/               # namespace Ipamo.Application.*
│       │   ├── Abstractions/          # IIpamoDbContext, ICurrentUser, IOwnerClock, IEquipmentLocks,
│       │   │                          #   IIdentityService, ITokenService
│       │   ├── Common/                # NotFoundException, ConflictException, paging types
│       │   ├── Auth/                  # AuthService + requests/responses
│       │   ├── Profile/
│       │   ├── Categories/
│       │   ├── Equipment/
│       │   ├── Contacts/
│       │   ├── Rentals/               # RentalService, AvailabilityService, requests, responses, mappings
│       │   └── DependencyInjection.cs # AddApplication()
│       │
│       ├── Infrastructure/            # namespace Ipamo.Infrastructure.*
│       │   ├── Data/                  # IpamoDbContext, Configurations/, Migrations/, EquipmentLocks.cs
│       │   ├── Identity/              # ApplicationUser, RefreshToken, IdentityService, JwtTokenService
│       │   ├── Time/                  # OwnerClock (TimeProvider + owner time zone)
│       │   ├── Security/              # CurrentUser (reads the authenticated user's claims)
│       │   └── DependencyInjection.cs # AddInfrastructure(configuration)
│       │
│       └── Api/                       # namespace Ipamo.Api.*
│           ├── Controllers/           # one controller per feature
│           ├── Errors/                # IExceptionHandler → ProblemDetails
│           └── DependencyInjection.cs # AddApi(): controllers, auth, CORS, rate limiting, OpenAPI
└── tests/
    ├── Ipamo.UnitTests/               # Domain rules + architecture tests; no database
    └── Ipamo.IntegrationTests/        # real HTTP + real PostgreSQL (Testcontainers)
```

### 2.2 Boundary rules (enforced by architecture tests)

`tests/Ipamo.UnitTests/ArchitectureTests.cs` uses **ArchUnitNET** to check these on every `dotnet test`:

| # | Rule | Why |
| --- | --- | --- |
| A1 | `Ipamo.Domain` depends on no other Ipamo layer, and not on EF Core, ASP.NET Core, Identity or Npgsql. | Domain stays pure C# and unit-testable. |
| A2 | `Ipamo.Application` doesn't depend on `Ipamo.Infrastructure`, `Ipamo.Api`, ASP.NET Core, Identity or Npgsql. | Use cases don't know how they're hosted or stored. |
| A3 | `Ipamo.Infrastructure` doesn't depend on `Ipamo.Api`. | Infrastructure is plugged in by the host, not the other way round. |
| A4 | Controllers depend on `Ipamo.Application` only, never on `Ipamo.Infrastructure` or `IpamoDbContext`. | Controllers stay thin and replaceable. |

**Conventions that keep the split mechanical** (checked in code review):
- Types used by another layer are `public`. Use `internal` only within one layer, because `internal` breaks when layers become separate assemblies.
- Domain entities carry **no attributes** (`[Key]`, `[MaxLength]`…). All mapping lives in `Infrastructure/Data/Configurations/`.
- Each layer registers its own services in its `DependencyInjection.cs`. `Program.cs` only calls the three extension methods, so they move with their layers.
- No static access to `HttpContext`, configuration or the clock from Domain or Application. Those always come through an interface.

## 3. Layer responsibilities

| Layer | Does | Never does |
| --- | --- | --- |
| **Api (controllers)** | Binds the request, calls one Application service method, returns a status code and body. Usually under 10 lines per action. | Business logic, DbContext access, try/catch for flow control |
| **Application (services)** | Orchestrates a use case: loads the current owner's data through `IIpamoDbContext`, calls domain methods, takes locks, saves, maps to response records. | Builds HTTP responses; returns entities |
| **Domain (entities)** | Enforces rules that depend only on its own data (e.g. `rental.CheckOut(today, conditions)` throws if the rental isn't Reserved). | Queries, logging, HTTP, persistence attributes |
| **Infrastructure** | Implements the Application interfaces: EF Core mapping, the owner query filter, PostgreSQL row locks, Identity, JWT issuing, the owner-local clock. | Business decisions |

### The DbContext interface (a deliberate, pragmatic choice)

Application services depend on **`IIpamoDbContext`**, which is defined in `Application/Abstractions/`, not on the concrete `IpamoDbContext`. The interface exposes the `DbSet<T>` properties, `SaveChangesAsync` and the `Database` facade (for transactions).

This means Application references the **EF Core abstractions** package, so services can use LINQ, `Include`, `AsNoTracking` and async queries. They never see Npgsql, migrations or configuration classes. This is the "pragmatic Clean Architecture" stance used by the widely adopted .NET templates. The stricter alternative (repositories that hide EF Core completely) is rejected in §9.

**PostgreSQL-specific operations** sit behind their own small interfaces, so Application never contains provider SQL. Example: `IEquipmentLocks.LockAsync(ownerId, equipmentIds)` runs `SELECT … FOR UPDATE` (§4).

**Why no repository layer?** EF Core's DbContext already *is* a unit of work, and each `DbSet<T>` already *is* a repository. Wrapping them either hides useful features or leaks them anyway. Tests don't need wrappers: domain rules are unit-tested without a database, and services are integration-tested against real PostgreSQL. See [ADR-0003](decisions/0003-ef-core.md).

### Identity stays out of the Domain

ASP.NET Core Identity's user type (`ApplicationUser : IdentityUser<Guid>`) lives in `Infrastructure/Identity/`. The Domain only knows an **`OwnerId` (Guid)**, plus an `OwnerProfile` entity for settings (display name, time zone, currency) that shares the same ID. Application talks to Identity only through `IIdentityService` and `ITokenService`. Swapping Identity for an external provider such as Entra ID later would then touch Infrastructure only.

## 4. Where each business rule lives

Each rule is enforced in the **lowest layer that has the information**, with the database as a backstop wherever it can express the rule.

| Where | Rules | Tested by |
| --- | --- | --- |
| **Domain entities** | BR-01, BR-07, BR-09, BR-10, BR-11, BR-12, BR-16, BR-17, BR-18; BR-25 and BR-26 state checks | Unit tests |
| **Application services** | BR-02, BR-03, BR-04, BR-05, BR-06, BR-08, BR-13, BR-14, BR-15, BR-21, BR-22, BR-23; BR-25 and BR-26 availability checks | Integration tests |
| **Infrastructure: query filter** | BR-24 (isolation) | An integration test per endpoint |
| **Infrastructure: DB constraints** (backstop) | BR-01 dates, BR-07 quantities, BR-23 uniqueness, positive money | Integration tests that bypass the service |

Rule IDs are defined in [01 – Product scope](01-product-scope.md). The exact constraints are in [03 – Data model](03-data-model.md).

### Preventing double bookings under concurrency

Availability is "check, then insert." Without care, two requests for the last speaker on the same weekend could both pass the check. Ipamo uses **row locks inside a transaction** to stop that:

1. Begin a transaction (`IIpamoDbContext.Database.BeginTransactionAsync`).
2. `IEquipmentLocks.LockAsync(...)` locks the equipment rows involved, in ID order so requests can't deadlock each other: `SELECT … FROM equipment WHERE id = ANY(@ids) AND owner_id = @owner ORDER BY id FOR UPDATE`.
3. Run the availability check (BR-02 and BR-03) against committed data.
4. Insert or update, then commit. Any competing request was waiting at step 2, and its check now sees this booking.

The same pattern is used for create, edit (BR-25), due-date changes (BR-26) and quantity decreases (BR-22).

**Alternative considered:** a PostgreSQL exclusion constraint on `daterange`. It handles Individual items elegantly, but it can't express Bulk quantity limits. An item's occupied range also depends on its return state. Row locks cover both cases with one mechanism.

### Preventing double returns and lost updates

`Rental` and `RentalItem` use PostgreSQL's `xmin` system column as an **optimistic concurrency token**. This is configured in Infrastructure, so the Domain doesn't know about it. If two requests return the same item at once, the second save throws `DbUpdateConcurrencyException`, which becomes `409 Conflict`. The domain check ("already returned", BR-10) catches the non-concurrent case.

## 5. Cross-cutting decisions

| Concern | Decision | Trade-off accepted |
| --- | --- | --- |
| **API style** | Controllers with `[ApiController]` | More ceremony than Minimal APIs, but clear conventions, widely used in industry, and a natural step from your WebForms/MVC background. See [ADR-0005](decisions/0005-project-structure.md). |
| **Route prefix** | `/api/...`, with no version segment | Versioning is easy to add later; v1 has one client. |
| **Errors** | Every error is RFC 9457 `ProblemDetails` with a stable `code` (e.g. `rental.unavailable`) and, where relevant, `ruleId` (e.g. `BR-02`) | Clients get one predictable shape. |
| **Signalling failures** | Domain throws `BusinessRuleException`. Application throws `NotFoundException` and `ConflictException`. One `IExceptionHandler` in `Api/Errors/` maps them all. | Exceptions for expected failures cost a little performance that doesn't matter at this scale. In return they're simpler to learn and keep controllers clean. A `Result<T>` type is the alternative if this ever becomes awkward. |
| **Input validation** | DataAnnotations on Application request records, which `[ApiController]` turns into automatic `400`s. Business rules stay in Domain and Application. | FluentValidation is more expressive but adds a dependency. DataAnnotations is part of .NET itself (not ASP.NET Core), so it doesn't break rule A2. |
| **Mapping** | Hand-written mapping methods in Application | More typing than AutoMapper (now commercially licensed), but explicit and debuggable, and fields never leak by accident. |
| **IDs** | `Guid` version 7 (`Guid.CreateVersion7()`), stored as PostgreSQL `uuid` | Larger than `int`, but IDs can't be guessed or enumerated (which supports BR-24), and v7 is time-ordered so indexes stay efficient. |
| **Dates and money** | `DateOnly` → `date`; `decimal` → `numeric(10,2)`; instants as UTC `DateTimeOffset` → `timestamptz` | None. These are the correct types. |
| **"Today"** | `IOwnerClock` (Application) is implemented in Infrastructure using .NET's `TimeProvider` and the owner's time zone. Code never calls `DateTime.Now`. | One extra abstraction, which makes every date rule testable with `FakeTimeProvider`. |
| **DB naming** | snake_case tables and columns via `EFCore.NamingConventions` (Infrastructure only) | One small dependency; makes raw SQL and `psql` pleasant. |
| **Configuration** | Strongly typed Options classes, validated at startup (`ValidateOnStart`), and registered by the layer that uses them | The app refuses to start with a missing secret instead of failing on the first request. |
| **Logging** | Built-in `ILogger` with structured messages. No passwords, tokens or full contact details. | Serilog or OpenTelemetry can be added at deployment time. |
| **Health** | `/health` with an EF Core database check (Microsoft package) | None. |
| **CORS** | Allowed origins from config; none in v1 | Ready for the React app with no code change. |

## 6. Request pipeline

The order in `Program.cs` matters:

```text
Request
  → Exception handler       (any thrown exception → ProblemDetails)
  → HTTPS redirection / HSTS (HSTS outside Development only)
  → CORS                    (only if origins are configured)
  → Rate limiter            (tight limits on /api/auth/*)
  → Authentication          (validates the JWT, builds the user principal)
  → Authorization           (auth required by default; [AllowAnonymous] is the exception)
  → Controller → Application service → IIpamoDbContext
Response
```

**Secure by default:** a global authorization policy requires an authenticated user everywhere. Only register, login, refresh and health opt out with `[AllowAnonymous]`, so a new endpoint is protected even if someone forgets an attribute.

## 7. Data isolation (summary)

BR-24 is enforced in three layers. Full detail is in [05 – Auth & security](05-auth-and-security.md).

1. **Every owned entity has `OwnerId`.** Application services set it from `ICurrentUser` on create. It is never accepted from the request body.
2. **An EF Core global query filter** (`OwnerId == currentUserId`) is defined in Infrastructure and applies to every owned entity, so a forgotten `Where` can't leak data. Archived records are *not* filtered globally, because history still needs them.
3. **Integration tests** call every ID-based endpoint as a second owner and expect `404`.

## 8. Extension points (designed for, not built)

| Future feature | What v1 does now | What gets added later |
| --- | --- | --- |
| **Photos (Azure Blob)** | Nothing in the schema yet. | An `EquipmentPhoto` entity (Domain), `IPhotoStorage` (Application), a Blob implementation (Infrastructure), and endpoints under `/api/equipment/{id}/photos`. No existing columns change. |
| **Photo-to-form AI** | `CreateEquipmentRequest` is the single input shape for creating equipment, and categories are per owner. | `IEquipmentSuggestionService` (Application) with a provider implementation (Infrastructure). `POST /api/equipment/suggestions` returns a *draft* `CreateEquipmentRequest` for the owner to edit and submit. The AI never writes to the database. |
| **Reminders (Azure Function)** | "Due soon / overdue" logic lives in Application query methods, not in controllers. | Once the split in §10 is done, the Function can reference `Ipamo.Application` and `Ipamo.Infrastructure` directly. This is a concrete payoff of the layering. Alternatively, it calls a protected endpoint. |
| **React front end** | ProblemDetails errors, a consistent paging envelope, refresh tokens, CORS from config. | CORS origins configured. No API changes expected. |
| **Azure hosting** | Everything that varies is in configuration, secrets are out of the repo, and there's a health endpoint. | App Service, Azure Database for PostgreSQL, Key Vault references for secrets. |

## 9. Deliberately not used (and when to reconsider)

| Pattern / tool | Why not now | Reconsider when |
| --- | --- | --- |
| Separate projects per layer | The layers already exist as folders guarded by architecture tests; extra projects add ceremony now | A second host (Azure Function) needs to share Application/Infrastructure, or boundary violations keep slipping through. Then follow §10. |
| Repositories hiding EF Core | DbContext already provides unit of work and repositories; wrappers leak or cripple EF | We genuinely need to swap EF Core (unlikely) |
| MediatR / CQRS | Indirection without a payoff yet; MediatR is now commercially licensed | Cross-cutting behaviours per request multiply |
| AutoMapper | Hidden mapping bugs; commercial licence | Never, for this project |
| Generic base services and controllers | Every feature's rules differ; generics hide them | Never |
| Domain events / message bus | Nothing reacts asynchronously in v1 | Reminders or photo processing need it |
| Caching | Small per-owner data; correctness first | Measured slowness |
| Microservices | One developer, one database | Never, for this project |

## 10. Splitting into separate projects later

Because of §2, the split is a **mechanical refactor** of roughly an evening's work, with no logic changes. It becomes its own milestone when it's needed.

1. On a branch, create `src/Ipamo.Domain`, `src/Ipamo.Application` and `src/Ipamo.Infrastructure` as class libraries targeting the same .NET version, and add them to `Ipamo.slnx`.
2. **Move** each folder's contents into the matching project, dropping the folder prefix (e.g. `Domain/Rentals/Rental.cs` → `src/Ipamo.Domain/Rentals/Rental.cs`). Namespaces already match, so no code edits are needed. Move the `Api/` contents to the root of `Ipamo.Api` and set its `RootNamespace` back to `Ipamo.Api`.
3. Add project references that match the arrows in §2: Application → Domain; Infrastructure → Application, Domain; Api → Application, Infrastructure.
4. Move NuGet packages to the projects that use them: EF Core abstractions → Application; Npgsql, naming conventions, Identity EF stores, JWT → Infrastructure; the rest stays in Api.
5. In `AddInfrastructure`, set `MigrationsAssembly("Ipamo.Infrastructure")`. Existing migrations move with the folder. The database's migration history stores only migration IDs, not namespaces, so **no new migration and no data change** is needed. Run `dotnet ef migrations list --project src/Ipamo.Infrastructure --startup-project src/Ipamo.Api` to confirm.
6. Point the architecture tests at the four assemblies instead of one.
7. Done when `dotnet build` has no warnings, `dotnet test` is green (architecture tests included), and the app starts against an existing database without pending migrations.

## 11. Decision records

| ADR | Decision |
| --- | --- |
| [0000](decisions/0000-template.md) | ADR template |
| [0001](decisions/0001-record-architecture-decisions.md) | Record architecture decisions as ADRs |
| [0002](decisions/0002-postgresql.md) | PostgreSQL as the database |
| [0003](decisions/0003-ef-core.md) | EF Core with migrations; DbContext behind `IIpamoDbContext`, no repositories |
| [0004](decisions/0004-jwt-with-identity.md) | ASP.NET Core Identity + self-issued JWT + rotating refresh tokens |
| [0005](decisions/0005-project-structure.md) | Clean Architecture layers as folders in one project, guarded by architecture tests; controllers |
