# ADR-0005: Clean Architecture layers as folders in one project, guarded by architecture tests; controllers

- **Status:** Accepted
- **Date:** 2026-11-09
- **Deciders:** Stephen Okanlawon
- **Related docs:** [02 – Architecture](../02-architecture.md) §2, §3, §10 · [06 – Testing strategy](../06-testing-strategy.md) §8

## Context

Ipamo is a one-developer API with about 10 entities and 26 business rules. It must:
- keep business rules isolated and unit-testable;
- be easy to navigate for a developer new to modern ASP.NET Core;
- **allow a seamless move to full Clean Architecture (separate projects) later.** The likely trigger is an Azure Function needing to share Application and Infrastructure code.

Separately, we need to choose between **controllers** and **Minimal APIs** for the HTTP layer.

## Options considered: structure

### Option A – Clean Architecture as four projects from day one (Domain, Application, Infrastructure, Api)
- **Pros:** the compiler enforces boundaries; the industry-recognised shape.
- **Cons:** more `.csproj` files, reference and package management, and slower builds. The ceremony outweighs the benefit at this size and pace (4–5 hours a week).

### Option B – Single project, feature folders, no layer separation
- **Pros:** simplest; each feature in one folder.
- **Cons:** nothing stops domain code from depending on EF Core or ASP.NET Core, so a later split becomes a real refactor, not a move.

### Option C – Single project with the four layers as top-level folders and matching namespaces, enforced by architecture tests
- **Pros:**
  - The same dependency rules as Option A, enforced by ArchUnitNET tests instead of the compiler.
  - One project to build and run.
  - Splitting later is mechanical: move folders, add references, set the migrations assembly (doc 02 §10).
- **Cons:**
  - Boundary violations are caught by tests rather than by compilation.
  - A feature spans several folders (Application + Api) rather than one.

## Options considered: HTTP layer

### Controllers (`[ApiController]`)
- **Pros:**
  - Clear conventions (automatic `400`s, binding sources, attribute routing, filters).
  - Very common in enterprise .NET jobs.
  - A natural step from WebForms/MVC.
- **Cons:** more ceremony than Minimal APIs.

### Minimal APIs
- **Pros:** less code; Microsoft's direction for new small services, with built-in validation in recent .NET versions.
- **Cons:** conventions have to be assembled by hand (endpoint groups, filters), and there's less structure for someone learning the framework.

## Decision

We will use **Option C** with **controllers**:
- `src/Ipamo.Api/` has `RootNamespace` set to `Ipamo`, with top-level folders `Domain/`, `Application/`, `Infrastructure/` and `Api/`, whose namespaces are `Ipamo.Domain.*`, etc.
- Architecture tests **A1–A4** (doc 02 §2.2) run with every `dotnet test`.
- The conventions that keep the split mechanical (cross-layer types are public, no attributes on entities, per-layer `DependencyInjection.cs`) are enforced in code review and listed in `CLAUDE.md`.
- Controllers stay thin and call Application services only.

## Consequences

- **Positive:**
  - Clean Architecture discipline without multi-project overhead.
  - A documented, low-risk path to separate projects.
  - The structure is recognisable to reviewers.
- **Negative:**
  - Folder discipline relies on tests and review.
  - The `Api/` folder inside the `Ipamo.Api` project looks slightly odd until the split happens.
- **Follow-ups:** the split runbook in doc 02 §10 becomes its own milestone when it's triggered.

## Revisit when

- **Split into projects** when a second host (e.g. the reminders Azure Function) needs Application/Infrastructure, when boundary violations keep slipping past tests, or when build times or team size grow.
- **Minimal APIs:** reconsider if controllers become a burden, or for small standalone services (such as the Function's HTTP triggers). This decision doesn't stop the two styles coexisting.
