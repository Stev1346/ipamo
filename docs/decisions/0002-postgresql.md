# ADR-0002: PostgreSQL as the database

- **Status:** Accepted
- **Date:** 2026-11-09
- **Deciders:** Stephen Okanlawon
- **Related docs:** [02 – Architecture](../02-architecture.md) §4, [03 – Data model](../03-data-model.md)

## Context

Ipamo needs a relational database that:
- runs locally on a Mac (including Apple Silicon) in Docker, and in CI on GitHub's Ubuntu runners;
- has managed hosting on Azure, the planned deployment target;
- supports the features the business rules depend on: real `date` columns, check constraints, partial unique indexes, row locks (`SELECT … FOR UPDATE`), and set-based date queries;
- has first-class EF Core support;
- costs nothing to develop with.

Stephen's professional background is SQL Server with WebForms, and his personal projects use Supabase, which is PostgreSQL.

## Options considered

### Option A – PostgreSQL 17
- **Pros:**
  - Free and open source.
  - Official multi-arch Docker images that run natively on Apple Silicon.
  - Excellent Npgsql/EF Core provider.
  - Rich constraints and partial indexes, `generate_series`, and `FOR UPDATE`.
  - Managed hosting via **Azure Database for PostgreSQL – Flexible Server**.
  - The same engine as Supabase, so the skills transfer both ways.
- **Cons:**
  - SQL dialect differences from SQL Server: case-sensitive text comparison by default, `snake_case` conventions, and different date functions.
  - Slightly less "default" in the Microsoft ecosystem.

### Option B – SQL Server (Developer edition / Azure SQL)
- **Pros:** familiar from work; the most "Microsoft-native" option, with first-class Azure SQL and great tooling.
- **Cons:**
  - The official Linux container images are x64-only at the time of writing, so on Apple Silicon they run under emulation (slower, occasionally flaky).
  - Azure SQL costs more than a small PostgreSQL Flexible Server for a hobby workload.
  - Less new learning, since a goal of this project is to broaden skills.

### Option C – SQLite
- **Pros:** zero setup; a single file.
- **Cons:**
  - Weak concurrency model and no `FOR UPDATE` row locks.
  - Loose typing (dates stored as text).
  - Not suitable for a multi-user hosted API.
  - Tests against it wouldn't reflect production.

## Decision

We will use **PostgreSQL 17**: locally in Docker (`postgres:17`, port 5433), in integration tests via Testcontainers, and on **Azure Database for PostgreSQL – Flexible Server** when deployed.

## Consequences

- **Positive:**
  - Native performance on the Mac.
  - All the business-rule backstops (check constraints, partial unique indexes, composite FKs) and the row-lock concurrency strategy are available.
  - The skills apply directly to Stephen's Supabase projects.
- **Negative:**
  - Stephen learns PostgreSQL-specific behaviour (case sensitivity, `xmin`, `timestamptz`).
  - Raw SQL in `Infrastructure/` is PostgreSQL-specific, which is why it sits behind interfaces (e.g. `IEquipmentLocks`).
- **Follow-ups:** pin the image's major version, and let Dependabot propose minor updates.

## Revisit when

- An employer or client requires SQL Server, *and* the app must run there. The EF Core model is mostly portable; the raw SQL in Infrastructure would need rewriting.
- Azure PostgreSQL pricing or availability changes materially.
