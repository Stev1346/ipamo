# Ìpamọ́ (Ipamo)

**Keep track of your rental gear: what you own, who has it, and when it's coming back.**

*Ìpamọ́* is Yoruba for "safekeeping." Ipamo is a small, multi-user web API that lets equipment owners track their gear, the people who rent it and every rental, from reservation to return. Each owner's data is completely private to them.

It's built for people who rent out A/V, DJ, lighting, staging or event décor on the side and are tired of keeping it all in a spreadsheet.

> **Status:** pre-v1. Being built milestone by milestone. See [docs/07-build-plan.md](docs/07-build-plan.md) for progress.

---

## What it does (v1)

- **Equipment.** Track individual items (a mixer with a serial number) or bulk stock (20 XLR cables) with condition, replacement value, day rate and owner-defined categories.
- **Contacts.** Keep a record of the people who rent from you. Renters don't need an account.
- **Rentals.** Reserve one or many items for a date range, check them out, and return them line by line, recording condition and any missing quantity.
- **Availability.** Ask "what's free from Dec 6 to Dec 8?" and get back items and available quantities.
- **Overdue tracking.** See everything past its due date that hasn't come back, plus upcoming reservations that conflict with it.
- **Built-in safeguards.** No double bookings, no double returns, no impossible dates, and no access to another owner's data.

**Not in v1:** payments, photo uploads, email reminders, shared inventory. See [docs/01-product-scope.md](docs/01-product-scope.md).

**Planned later:** photo-to-form AI (snap a photo and the item's fields are pre-filled), photos in Azure Blob Storage, reminders via Azure Functions, and a React + TypeScript front end.

---

## Tech stack

| Concern | Choice |
| --- | --- |
| Runtime | .NET 10 (LTS) |
| API | ASP.NET Core Web API (controllers) |
| Data access | Entity Framework Core + migrations, Npgsql provider |
| Database | PostgreSQL 17 (Docker locally, Azure Database for PostgreSQL later) |
| Auth | ASP.NET Core Identity (user store and password hashing) + JWT access tokens + rotating refresh tokens |
| API docs | OpenAPI document + Swagger UI (Development only) |
| Tests | xUnit, with integration tests against a real PostgreSQL instance in Docker |
| Source control / CI | GitHub, GitHub Actions |
| Hosting (later) | Azure App Service |

The reasons behind these choices are in [docs/decisions/](docs/decisions/).

---

## Running locally on macOS

### 1. Prerequisites

Follow [docs/08-dev-setup-mac.md](docs/08-dev-setup-mac.md) once. In short, you need:

- the .NET 10 SDK
- Docker Desktop (running)
- Git
- VS Code with the C# Dev Kit extension

Check them:

```bash
dotnet --version        # 10.x
docker --version
docker compose version
```

### 2. Clone and restore

```bash
git clone https://github.com/<your-username>/ipamo.git
cd ipamo
dotnet tool restore     # installs the repo's pinned dotnet-ef tool
dotnet restore
```

### 3. Start PostgreSQL

```bash
cp .env.example .env    # then set POSTGRES_PASSWORD in .env
docker compose up -d
docker compose ps       # the "db" service should show "healthy"
```

PostgreSQL is mapped to **port 5433** on your Mac, not the default 5432. That way it won't clash with a Postgres.app or Homebrew install you might already have.

### 4. Set local secrets

Secrets never go in `appsettings.json` or Git. For local development they live in .NET **user secrets**, which are stored in your home folder outside the repo.

```bash
cd src/Ipamo.Api

dotnet user-secrets set "ConnectionStrings:Ipamo" \
  "Host=localhost;Port=5433;Database=ipamo;Username=ipamo;Password=<same as .env>"

dotnet user-secrets set "Jwt:SigningKey" "$(openssl rand -base64 64)"

dotnet user-secrets set "Registration:InviteCode" "<pick-an-invite-code>"

cd ../..
```

The full list of configuration keys is in [docs/05-auth-and-security.md](docs/05-auth-and-security.md).

### 5. Create the database schema

```bash
dotnet ef database update --project src/Ipamo.Api
```

### 6. Run the API

```bash
dotnet run --project src/Ipamo.Api --launch-profile https
```

- API: `https://localhost:7180`
- Swagger UI: `https://localhost:7180/swagger`
- Health check: `https://localhost:7180/health`

If your browser warns about the certificate, trust the .NET development certificate once:

```bash
dotnet dev-certs https --trust
```

### 7. Run the tests

Docker must be running, because the integration tests start their own throwaway PostgreSQL container.

```bash
dotnet test
```

### Stopping and resetting

```bash
docker compose down        # stop Postgres, keep the data
docker compose down -v     # stop Postgres and DELETE all local data
```

---

## Repository layout

```text
ipamo/
├── src/
│   └── Ipamo.Api/                 # the Web API (single project; see docs/02-architecture.md)
├── tests/
│   ├── Ipamo.UnitTests/           # fast tests of business rules, no database
│   └── Ipamo.IntegrationTests/    # HTTP-level tests against real PostgreSQL
├── docs/                          # product, architecture and build docs (start here)
│   └── decisions/                 # Architecture Decision Records (ADRs)
├── docker-compose.yml             # local PostgreSQL
├── .env.example                   # template for local Docker settings (copy to .env)
├── .config/dotnet-tools.json      # pinned local tools (dotnet-ef, reportgenerator)
├── Ipamo.slnx                     # solution file
├── CLAUDE.md                      # working instructions for Claude Code
└── README.md
```

---

## Documentation

| Doc | What it covers |
| --- | --- |
| [01 – Product scope](docs/01-product-scope.md) | Users, problems, v1 features, out of scope, success criteria |
| [02 – Architecture](docs/02-architecture.md) | Project structure, layers, key decisions and trade-offs |
| [03 – Data model](docs/03-data-model.md) | Entities, fields, relationships, constraints, indexes |
| [04 – API spec](docs/04-api-spec.md) | Every endpoint, with request and response shapes and errors |
| [05 – Auth & security](docs/05-auth-and-security.md) | JWT, passwords, data isolation, validation, secrets |
| [06 – Testing strategy](docs/06-testing-strategy.md) | Unit vs. integration tests, business-rule test cases |
| [07 – Build plan](docs/07-build-plan.md) | Ordered milestones with definitions of done |
| [08 – Dev setup (Mac)](docs/08-dev-setup-mac.md) | Tools to install and how to verify them |
| [Decisions](docs/decisions/) | ADRs for the main technical choices |

---

## How this project is built

Ipamo is a learning project and a portfolio piece. The owner writes the major code by hand, with Claude Code acting as a senior developer who guides and reviews but doesn't write the app. The docs are written and approved first, features are built one milestone at a time, and GitHub is the source of truth. See [CLAUDE.md](CLAUDE.md).

---

## License

To be decided.
