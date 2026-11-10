# 06 – Testing Strategy

> **Status:** Draft for approval · **Applies to:** v1
> **Related:** [01 – Product scope](01-product-scope.md) (BR-xx) · [02 – Architecture](02-architecture.md) §2.2, §4 · [03 – Data model](03-data-model.md) §5 · [05 – Auth & security](05-auth-and-security.md)

This document defines what gets tested, at which level, with which tools, and lists the concrete test cases for every business rule. A feature isn't done until its cases here pass (§10).

---

## 1. Principles

1. **Test the rules, not the framework.** We don't test that ASP.NET Core routes requests or that EF Core saves a string. We test that *our* rules hold.
2. **Real PostgreSQL, never fakes.** Integration tests run against a real PostgreSQL 17 container. EF Core's in-memory provider and SQLite behave differently on dates, constraints, case-sensitivity and locking, and those are exactly the areas Ipamo's rules depend on.
3. **No mocking of `IIpamoDbContext`.** If a test needs a database, it's an integration test. Mocks are used only for true external boundaries, and v1 has none apart from time.
4. **Time is always controlled.** No test depends on the real date. Domain methods take `today` as a parameter, and integration tests use `FakeTimeProvider`.
5. **Every business rule is traceable.** Each test that covers a rule carries that rule's ID, so `dotnet test --filter "Rule=BR-02"` runs exactly that rule's tests.
6. **Tests go through the front door.** Integration tests arrange data by calling the **API**, not by inserting rows. This keeps them realistic and resilient to schema changes. Direct database access is reserved for constraint backstop tests (§6.3).

## 2. Test levels

| Level | Project | What's covered | Speed | Database |
| --- | --- | --- | --- | --- |
| **Unit** | `Ipamo.UnitTests` | Domain entities' rules and calculations; architecture rules A1–A4 | Milliseconds | None |
| **Integration** | `Ipamo.IntegrationTests` | Real HTTP requests through the full pipeline: auth, validation, services, availability queries, concurrency, isolation, DB constraints, migrations | Seconds | PostgreSQL in Docker (Testcontainers) |

There's no separate end-to-end or UI level in v1, because there's no UI. Integration tests *are* the end-to-end tests for an API.

**Rule of thumb for choosing a level:** if a rule can be decided from one entity's own data plus "today", write a **unit** test against the domain method. If it needs other rows (overlaps, uniqueness, other owners), write an **integration** test. Doc 02 §4 already assigns each BR to a layer, and the tables in §7 follow that assignment.

## 3. Tooling

| Tool | Purpose | Notes |
| --- | --- | --- |
| **xUnit v3** | Test framework | `[Fact]`, `[Theory]`, `[Trait]`, fixtures |
| xUnit `Assert` | Assertions | Built in; no extra assertion library. FluentAssertions now has a commercial licence, so we avoid it. |
| `Microsoft.AspNetCore.Mvc.Testing` | Hosts the real API in memory | `WebApplicationFactory<Program>` |
| `Testcontainers.PostgreSql` | Starts a throwaway `postgres:17` container per test run | Needs Docker Desktop running |
| `Respawn` | Wipes data between tests without rebuilding the schema | Keeps `__EFMigrationsHistory` |
| `Microsoft.Extensions.TimeProvider.Testing` | `FakeTimeProvider` | Controls "now" for the app and for JWT validation |
| `TngTech.ArchUnitNET` (+ its xUnit integration) | Architecture rules A1–A4 | doc 02 §2.2 |
| `coverlet.collector` + `ReportGenerator` | Coverage collection and an HTML report | §8 |

## 4. Unit tests (`Ipamo.UnitTests`)

**Scope:** everything under `Domain/`, plus `ArchitectureTests.cs`.

**Style:**
- One test class per entity or behaviour (e.g. `RentalCheckOutTests`, `RentalFeeTests`, `RentalItemReturnTests`).
- Domain methods receive `DateOnly today` (and any other inputs) as parameters, so tests just pass dates. There's no clock, DI or database.
- `[Theory]` with `[InlineData]` for boundary tables (dates, quantities, money).

**Layout:**

```text
tests/Ipamo.UnitTests/
├── ArchitectureTests.cs
├── Domain/
│   ├── Rentals/
│   │   ├── RentalCreateTests.cs
│   │   ├── RentalCheckOutTests.cs
│   │   ├── RentalReturnTests.cs
│   │   ├── RentalFeeTests.cs
│   │   ├── RentalEditTests.cs
│   │   └── RentalOverdueTests.cs
│   ├── Equipment/EquipmentTests.cs
│   └── Categories/CategoryTests.cs
└── TestData/                  # small builders, e.g. RentalBuilder
```

## 5. Integration tests (`Ipamo.IntegrationTests`)

### 5.1 Test host

```text
tests/Ipamo.IntegrationTests/
├── Infrastructure/
│   ├── IpamoApiFactory.cs        # WebApplicationFactory<Program> + Postgres container
│   ├── IntegrationTestBase.cs    # resets data, exposes helpers
│   ├── ApiClient.cs              # typed helpers: CreateOwnerAsync, CreateEquipmentAsync…
│   └── TestClock.cs              # wraps FakeTimeProvider in owner-local dates
├── Auth/
├── Isolation/
├── Equipment/
├── Contacts/
├── Categories/
├── Rentals/
│   ├── AvailabilityTests.cs
│   ├── BookingTests.cs
│   ├── CheckoutAndReturnTests.cs
│   ├── EditingTests.cs
│   ├── OverdueTests.cs
│   ├── SettlementTests.cs
│   └── ConcurrencyTests.cs
├── Database/                     # constraint backstops + migration checks
└── Platform/                     # auth-by-default, JSON strictness, headers, health
```

**`IpamoApiFactory`** (one per test run, shared through an xUnit collection fixture):
1. Starts `postgres:17` with Testcontainers and a random password.
2. Overrides configuration:
   - the connection string, pointing at the container
   - a random 64-byte `Jwt:SigningKey`
   - `Registration:InviteCode = "test-invite-code-0001"`
3. Replaces `TimeProvider` with a singleton `FakeTimeProvider`, set by default to **2026-12-01 at 12:00 in `America/Edmonton`** (19:00 UTC). The JWT bearer handler uses the same provider, so token expiry can be tested.
4. Applies all migrations once with `Database.MigrateAsync()`, which also tests that the migrations work.

**Between tests:** `IntegrationTestBase.InitializeAsync()` calls Respawn to delete all data and resets the fake clock to the default date.

**Parallelism:** integration tests share one database, so they run **sequentially** in a single xUnit collection. At v1 scale (about 150–250 tests), a full run should take under a minute. If it gets slow, switch to one database per test class.

### 5.2 Helpers

- `CreateOwnerAsync(name)` registers a new owner and returns an `HttpClient` already holding that owner's bearer token. Most tests start with two owners, **A** and **B**.
- `CreateEquipmentAsync`, `CreateContactAsync`, `ReserveAsync`, `CheckOutAsync` and `ReturnAsync` call the real endpoints and return the response record.
- `Clock.SetToday(new DateOnly(2026, 12, 7))` moves the fake clock to noon on that owner-local date.
- `AssertProblem(response, status, code, ruleId?)` checks the ProblemDetails shape, `code` and `ruleId` in one call.

### 5.3 Example

```csharp
[Fact]
[Trait("Rule", "BR-02")]
public async Task BR02_IndividualItem_ReturnDayIsBlocked_NextDayIsFree()
{
    var a = await Api.CreateOwnerAsync("A");
    var speaker = await a.CreateEquipmentAsync(individual: true);
    var contact = await a.CreateContactAsync();
    await a.ReserveAsync(contact, speaker, start: Dec(1), due: Dec(5));

    var clash = await a.TryReserveAsync(contact, speaker, start: Dec(5), due: Dec(7));
    var ok    = await a.TryReserveAsync(contact, speaker, start: Dec(6), due: Dec(8));

    await AssertProblem(clash, 409, "rental.unavailable", "BR-02");
    Assert.Equal(HttpStatusCode.Created, ok.StatusCode);
}
```

## 6. Naming and traceability

### 6.1 Test names

Format: `{RuleId}_{Subject}_{Scenario}_{ExpectedOutcome}`, or without a rule ID for non-BR tests.

Examples:
- `BR03_BulkItem_RequestExceedsRemainingQuantity_Returns409WithAvailable`
- `Auth_RefreshToken_ReusedAfterRotation_RevokesFamily`

### 6.2 Traits

| Trait | Values | Use |
| --- | --- | --- |
| `Rule` | `BR-01` … `BR-26` | `dotnet test --filter "Rule=BR-14"` |
| `Category` | `Isolation`, `Security`, `Concurrency`, `Database`, `Architecture` | `dotnet test --filter "Category=Isolation"` |

### 6.3 Database backstop tests

For each check constraint and unique index in doc 03, one test inserts an invalid row **directly** through a raw `NpgsqlConnection` (bypassing services) and expects a `PostgresException` with the right constraint name. This proves the database would catch a bug in the application code. These tests carry `Category=Database`.

## 7. Business-rule test cases

**Fixed test calendar:** unless stated otherwise, **today = 2026-12-01** (owner-local). Dates are written as `Dec n` (2026). The *Level* column shows **U** for unit or **I** for integration.

### Availability and booking

| Rule | # | Case | Expected | Level |
| --- | --- | --- | --- | --- |
| **BR-01** | 1 | Rental Dec 1–5 | `rentalDays = 5` | U |
| | 2 | `start = due` (Dec 3–3) | Allowed, `rentalDays = 1` | U |
| | 3 | Due before start (Dec 5–3) | Domain throws; API `422 rental.invalid_dates` | U + I |
| | 4 | Insert a row with `due_date < start_date` directly | Check constraint violation | I (DB) |
| **BR-02** | 1 | Speaker reserved Dec 1–5; book Dec 5–7 | `409 rental.unavailable` | I |
| | 2 | Same; book Dec 6–8 | `201` (next day after the return day is free) | I |
| | 3 | Reserved Dec 10–15; book Dec 5–10 / Dec 5–9 | `409` / `201` | I |
| | 4 | New booking fully contains an existing one (Dec 2–20 around Dec 10–15) | `409` | I |
| | 5 | The existing reservation is **cancelled**; book the same days | `201` | I |
| | 6 | Two parallel requests book the same speaker for the same days | Exactly one `201`, one `409` | I (Concurrency) |
| **BR-03** | 1 | 20 cables owned, 12 reserved Dec 6–8; request 8 / 9 for Dec 6–8 | `201` / `409` with `available: 8` | I |
| | 2 | 20 owned; A = 10 on Dec 6–7, B = 10 on Dec 8–9; request 10 for Dec 6–9 | `201` (per-day check, not a sum) | I |
| | 3 | Continuing case 2; request 1 more for Dec 7–8 | `409` | I |
| | 4 | Two parallel requests for 15 each (20 owned, same days) | One `201`, one `409` | I (Concurrency) |
| **BR-04** | 1 | Checked out, due Dec 5, not returned; today Dec 7; availability for Dec 7 | `quantityAvailable = 0` | I |
| | 2 | Same; availability for Dec 8 | Available (not blocked forever) | I |
| | 3 | Same; book the speaker starting Dec 7 | `409` | I |
| **BR-05** | 1 | Due Dec 10, returned Dec 3; availability for Dec 3 / Dec 4 | `0` / `1` | I |
| | 2 | Bulk: 20 out, 18 returned Dec 3; availability for Dec 4 | `20` (missing stock isn't auto-deducted; see BR-11) | I |
| **BR-06** | 1 | Start Nov 30 / start Dec 1 | `422 rental.start_in_past` / `201` | I |
| | 2 | Clock at 2026-12-02T05:00Z (still Dec 1 at 22:00 in Edmonton); start Dec 1 | `201` (uses owner-local today) | I |
| | 3 | Owner time zone `Asia/Tokyo`, same instant (already Dec 2 there); start Dec 1 | `422` | I |
| **BR-07** | 1 | `items` empty | `400 validation.failed` | I |
| | 2 | Same `equipmentId` twice | `422 rental.duplicate_item` | I |
| | 3 | Individual item with quantity 2 | `422 rental.invalid_quantity` | U + I |
| | 4 | Bulk item with quantity 0 | `400 validation.failed` | I |
| | 5 | Insert a duplicate `(rental_id, equipment_id)` directly | Unique violation | I (DB) |
| **BR-08** | 1 | Reserve archived equipment | `422 rental.equipment_archived` | I |
| | 2 | Reserve for an archived contact | `422 rental.contact_archived` | I |
| | 3 | Availability list | Archived equipment never appears | I |

### Checkout and return

| Rule | # | Case | Expected | Level |
| --- | --- | --- | --- | --- |
| **BR-09** | 1 | Check out a Reserved rental on its start date | Status `checkedOut`; `checkedOutOn` and line conditions set | U + I |
| | 2 | Check out a CheckedOut, Returned or Cancelled rental | Domain throws; `422 rental.not_reserved` | U + I |
| | 3 | Today is before the start date | `422 rental.checkout_too_early` | U + I |
| | 4 | `checkedOutOn` is after today, or before the start date | `422 rental.invalid_checkout_date` | U |
| | 5 | Request omits one line, or includes an unknown or duplicate line | `422 rental.checkout_incomplete` | I |
| **BR-10** | 1 | Return a line on a Reserved rental | `422 rental.not_checked_out` | I |
| | 2 | Return the same line twice | `422 rental.item_already_returned` | U + I |
| | 3 | `returnedOn` before `checkedOutOn`, or after today | `422 rental.invalid_return_date` | U |
| | 4 | Two parallel return requests for the same line | One `200`, the other `409 concurrency.conflict` or `422 item_already_returned` | I (Concurrency) |
| | 5 | Insert a row with `returned_on` set but `condition_in` null directly | Check constraint violation | I (DB) |
| **BR-11** | 1 | 20 out; return 18 | `quantityMissing = 2`; `quantity_owned` still 20 | U + I |
| | 2 | Return 21 of 20 | `422 rental.invalid_quantity_returned` | U |
| | 3 | Individual item returned with quantity 0 (lost) | Allowed; `quantityMissing = 1` | U |
| **BR-12** | 1 | Return 1 of 2 lines | Rental stays `checkedOut` | U |
| | 2 | Return the last line on Dec 4 | Rental `returned`, `completedOn = Dec 4` | U + I |
| | 3 | An Individual line returned `damaged` | `equipment.condition = damaged` (doc 03 §5.4) | I |

### Overdue

| Rule | # | Case | Expected | Level |
| --- | --- | --- | --- | --- |
| **BR-13** | 1 | Checked out, due Dec 5; today Dec 5 / Dec 6 | Not overdue / overdue | U + I |
| | 2 | Line returned, the other line outstanding past due | Rental overdue; only the outstanding line is overdue | U |
| | 3 | Reserved rental past its start date | `isPickupLate = true`, `isOverdue = false` | U + I |
| | 4 | `GET /api/rentals/overdue` | Lists only overdue rentals, oldest due first | I |
| **BR-14** | 1 | Speaker overdue (due Dec 5, today Dec 7); another contact reserved it Dec 8–9 | Conflict listed with that rental | I |
| | 2 | A reservation for a **different** item, or a cancelled one | Not listed | I |
| | 3 | Bulk: 5 overdue of 20; a reservation needs 16 on Dec 8 | Conflict listed (only 15 would be free) | I |

### Money

| Rule | # | Case | Expected | Level |
| --- | --- | --- | --- | --- |
| **BR-15** | 1 | Book at $45/day, then change the equipment's day rate to $50 | Existing line still $45; a new rental uses $50 | I |
| **BR-16** | 1 | $1.00 × 5 days × 20 | `lineFee = 100.00` | U |
| | 2 | $12.35 × 3 days × 7 | `259.35` exactly (decimal) | U |
| **BR-17** | 1 | Subtotal 310, adjustment −20 | `total = 290.00` | U |
| | 2 | Non-zero adjustment with no note | Domain throws; `422 rental.note_required` | U + I |
| | 3 | Returned 2 days late | `daysLate = 2`; total unchanged (no automatic late fee) | U + I |
| **BR-18** | 1 | Record a refund while the rental is CheckedOut | `422 rental.refund_not_allowed` | I |
| | 2 | Refund 120 on a 100 deposit / refund exactly 100 | `422 rental.invalid_refund` / `200` | I |
| | 3 | Insert a row with `deposit_refunded > deposit_amount` directly | Check constraint violation | I (DB) |
| **BR-19** | 1 | Create a rental (CAD), then change the profile currency to USD | The old rental stays `CAD`; a new rental is `USD` | I |

### Records and ownership

| Rule | # | Case | Expected | Level |
| --- | --- | --- | --- | --- |
| **BR-20** | 1 | `DELETE` on any resource route | `405 Method Not Allowed` | I |
| | 2 | Archive equipment | `archivedAt` set; still returned by `GET /{id}` and in rental history | I |
| **BR-21** | 1 | Archive equipment or a contact that's on a Reserved rental | `409 *.in_active_rental` with `rentalIds` | I |
| | 2 | Same after that rental is returned or cancelled | `200` | I |
| **BR-22** | 1 | 20 owned, 14 reserved for Dec 10; lower to 13 / 14 | `409` with `minimumQuantity: 14` / `200` | I |
| | 2 | 14 were out in the past (already returned) | Doesn't count toward the minimum | I |
| **BR-23** | 1 | Category "Audio" exists; create "audio " | `409 category.name_taken` | U (normalisation) + I |
| | 2 | Archived category "Décor"; create "Décor" | `409` | I |
| | 3 | Duplicate serial for the same owner / same serial for a different owner | `409 equipment.serial_taken` / `201` | I |
| | 4 | Several Individual items with no serial number | Allowed | I |
| **BR-24** | — | See §7.1 | | I (Isolation) |

### Editing rentals

| Rule | # | Case | Expected | Level |
| --- | --- | --- | --- | --- |
| **BR-25** | 1 | Extend a reservation by a day with no other bookings | `200` (doesn't conflict with itself) | I |
| | 2 | Edit the dates so they overlap another reservation of the same item | `409 rental.unavailable` | I |
| | 3 | Equipment rate changed since booking; edit adds a new line | The existing line keeps its old rate; the new line gets the current rate | I |
| | 4 | Edit leaves a line out | That line is removed; `itemCount` decreases | I |
| | 5 | Edit a CheckedOut rental | `422 rental.not_editable` | U + I |
| | 6 | Edit fails validation part-way (e.g. the second new line is unavailable) | `409`, and the rental is **unchanged** when re-fetched | I |
| | 7 | Edit moves the start into the past | `422 rental.start_in_past` | I |
| **BR-26** | 1 | Checked out Dec 1–5; extend to Dec 7 with no conflicts | `200`; `rentalDays = 7`; fees repriced | I |
| | 2 | Extend into another contact's reservation (Dec 6–8) | `409 rental.unavailable` | I |
| | 3 | The speaker line is already returned; extend for the remaining cable line only | The returned line is ignored in the conflict check → `200` | I |
| | 4 | New due date before `checkedOutOn` | `422 rental.invalid_dates` | U + I |
| | 5 | Shorten the due date to Dec 2 when today is Dec 3 | `200`, and the rental is now `isOverdue = true` | I |
| | 6 | Change the due date on a Reserved rental | `422 rental.not_checked_out` | I |

### 7.1 Isolation matrix (BR-24)

Setup: owner **A** creates one of every resource. Owner **B** then tries to read, change and reference each of them.

| Endpoint (as owner B, using A's ID) | Expected |
| --- | --- |
| `GET`, `PUT`, `archive`, `restore` on `/api/categories/{id}`, `/api/equipment/{id}`, `/api/contacts/{id}` | `404` |
| `GET`, `PUT`, `change-due-date`, `notes`, `settlement`, `checkout`, `returns`, `cancel` on `/api/rentals/{id}` | `404` |
| `POST /api/equipment` with A's `categoryId` | `422 equipment.category_not_found` |
| `POST /api/rentals` with A's `contactId` / A's `equipmentId` | `422 rental.contact_not_found` / `422 rental.equipment_not_found` |
| `POST /api/rentals/{B's id}/returns` with one of A's `rentalItemId` values | `422 rental.unknown_item` |
| Every list endpoint (`categories`, `equipment`, `contacts`, `rentals`, `active`, `overdue`, `equipment/available`) | Contains **none** of A's records |
| `GET /api/equipment/available` | A's bookings don't reduce B's availability (and the reverse) |
| `GET /api/rentals/overdue` | A's overdue rentals never appear in B's conflicts |

Two more isolation tests:
- **The captured-value trap (doc 05 §5):** A and B call `GET /api/equipment` alternately, five times each, against the same app instance. Each must only ever see their own items.
- **Composite FK backstop:** insert an `equipment` row for B that points at A's `category_id` directly. Expect an FK violation.

## 8. Other required tests

**Auth and security** (`Category=Security`)

| # | Case | Expected |
| --- | --- | --- |
| S1 | Register with a wrong invite code / with no invite code configured | `403 auth.invalid_invite` / `403` (registration disabled) |
| S2 | Register with an 11-character password / a common password / a password containing the email name | `400 auth.weak_password` |
| S3 | Register with an existing email | `409 auth.email_taken` |
| S4 | Log in with an unknown email, a wrong password, or a locked account | The same `401 auth.invalid_credentials` body for all three |
| S5 | 5 wrong passwords, then the right one; advance the clock 15 minutes and try again | `401`, then `200` |
| S6 | Refresh, then reuse the old refresh token | Second call `401`; the new token from the first refresh is also revoked |
| S7 | Logout, then refresh with that token | `401` |
| S8 | Change password | All refresh tokens revoked |
| S9 | Access token after the clock advances 16 minutes | `401` |
| S10 | Tampered signature / `alg: none` / wrong issuer / wrong audience | `401` |
| S11 | **Auth by default:** enumerate every endpoint in `EndpointDataSource` and call each without a token | `401`, except the documented anonymous endpoints |
| S12 | A request body with an unknown JSON property (e.g. `ownerId`) | `400` |
| S13 | A body larger than 256 KB | `413` |
| S14 | Security headers present on an API response | All four headers from doc 05 §7 |
| S15 | Hit an auth endpoint 11 times in a minute | The 11th gets `429` with `Retry-After` |

**Platform**
- `GET /health` returns `200` with no details in the body.
- `GET /openapi/v1.json` returns `200` in Development.
- **No missing migrations:** `context.Database.HasPendingModelChanges()` is `false`. This catches a model change committed without its migration.

**Architecture** (`Category=Architecture`)
- Rules A1–A4 from doc 02 §2.2, one test each.

## 9. Coverage

- **Target:** at least 80% line coverage across `Ipamo.Domain.*` and `Ipamo.Application.*` combined. Migrations, `Program.cs` and DI registration are excluded (`[ExcludeFromCodeCoverage]` or report filters).
- **Collect locally:**

  ```bash
  dotnet test --collect:"XPlat Code Coverage"
  dotnet reportgenerator -reports:"tests/**/coverage.cobertura.xml" -targetdir:coverage -reporttypes:Html
  open coverage/index.html
  ```

- **In CI:** coverage is reported on every run. The 80% threshold **fails the build** from the milestone where rentals are complete (see [07 – Build plan](07-build-plan.md)). Before that, the target is tracked but not enforced, so early milestones aren't blocked by placeholder code.
- Coverage is a floor, not the goal. A BR with no test named after it is a gap, even at 95% coverage.

## 10. Definition of done for tests (every feature)

- [ ] Every BR the feature touches has the cases listed in §7, named with its rule ID and tagged with the `Rule` trait.
- [ ] Every new `{id}` route and body reference is added to the isolation matrix (§7.1).
- [ ] Every new check constraint or unique index has a backstop test (§6.3).
- [ ] Error responses are checked for `status`, `code` and (where relevant) `ruleId`, not just the status code.
- [ ] No test depends on the real clock, test order or another test's data.
- [ ] `dotnet test` is green locally and in GitHub Actions.

## 11. Not tested (on purpose)

- Framework behaviour (routing, model binding, EF change tracking) beyond what our rules rely on.
- Trivial property getters and setters, and hand-written mappings that have no logic (they're covered indirectly by response assertions).
- Private methods directly. They're tested through the public method that uses them.
- Performance and load. At v1 scale correctness is the risk, not throughput; revisit when there's a front end.
