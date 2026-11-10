# ADR-0003: EF Core with migrations; DbContext behind `IIpamoDbContext`, no repositories

- **Status:** Accepted
- **Date:** 2026-11-09
- **Deciders:** Stephen Okanlawon
- **Related docs:** [02 – Architecture](../02-architecture.md) §3, §4 · [03 – Data model](../03-data-model.md) §7

## Context

Ipamo needs:
- data access to PostgreSQL;
- versioned schema changes;
- a way to keep business logic testable and the code ready for a later split into Clean Architecture projects (doc 02 §10).

Learning modern EF Core is an explicit goal of the project. A few operations, such as row locks and per-day availability, need PostgreSQL-specific SQL.

## Options considered

### Option A – EF Core, services use the concrete `IpamoDbContext`
- **Pros:** simplest; full EF Core features.
- **Cons:** Application would depend on Infrastructure, which breaks architecture rule A2 and blocks a clean project split.

### Option B – EF Core behind an `IIpamoDbContext` interface (in Application), no repositories
- **Pros:**
  - Application depends on an abstraction it owns, and keeps LINQ, `Include`, `AsNoTracking` and async queries.
  - Infrastructure owns the mapping, migrations and provider details.
  - It's the stance taken by the most widely used .NET Clean Architecture templates.
- **Cons:** Application references EF Core's abstractions, so it isn't "pure". That's accepted deliberately.

### Option C – EF Core behind repositories (`IRentalRepository`, …)
- **Pros:** Application has no EF Core reference at all.
- **Cons:**
  - Duplicates what `DbSet<T>` and `DbContext` already provide (repository and unit of work).
  - Either hides useful EF features or leaks them through `IQueryable`.
  - More code to write and test, for a swap of ORM that's very unlikely.

### Option D – Dapper (micro-ORM) + DbUp migrations
- **Pros:** fast, explicit SQL; very little magic.
- **Cons:**
  - Hand-written mapping and change tracking, more boilerplate, and migrations as raw scripts.
  - Misses the EF Core learning goal.

## Decision

We will use **EF Core 10 with the Npgsql provider and code-first migrations** (Option B):
- Application services depend on **`IIpamoDbContext`**. `IpamoDbContext` in Infrastructure implements it.
- There's **no repository layer**.
- Database-specific SQL lives in Infrastructure behind small interfaces (e.g. `IEquipmentLocks`), written with `FromSql`/`SqlQuery` and interpolated parameters only.
- Mapping uses the Fluent API in `Infrastructure/Data/Configurations/`, with no attributes on entities and snake_case naming.
- Migrations are created per milestone, reviewed before commit, and never edited after merge.

## Consequences

- **Positive:**
  - Expressive queries, safe parameterisation, versioned schema, and global query filters for isolation.
  - The Clean Architecture split stays mechanical.
- **Negative:**
  - EF Core's change tracking and query translation have to be learned and occasionally fought (e.g. complex availability queries may be clearer in SQL).
  - Application isn't ORM-agnostic.
- **Follow-ups:** an integration test asserts `HasPendingModelChanges()` is false, which catches forgotten migrations.

## Revisit when

- Profiling shows EF Core is the bottleneck on a hot path. Then use Dapper or raw SQL for *that path only*, behind an interface.
- The app genuinely has to support a second database provider.
