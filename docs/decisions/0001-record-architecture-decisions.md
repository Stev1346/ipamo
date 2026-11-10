# ADR-0001: Record architecture decisions as ADRs

- **Status:** Accepted
- **Date:** 2026-11-09
- **Deciders:** Stephen Okanlawon
- **Related docs:** [02 – Architecture](../02-architecture.md), [CLAUDE.md](../../CLAUDE.md)

## Context

Ipamo is built by one developer, with an AI partner (Claude Code) that starts each session with no memory of earlier ones. Decisions made in chat or in someone's head get lost. Then they get re-argued, or quietly reversed by a later change. The repo is also a portfolio piece, and reviewers value seeing *why* choices were made.

## Options considered

### Option A – No formal record
- **Pros:** zero overhead.
- **Cons:** reasoning is lost. Claude Code can't know past trade-offs, and decisions drift.

### Option B – Decisions inside the main docs only
- **Pros:** one place to read.
- **Cons:** the main docs describe the *current* design. They don't preserve history or the alternatives that were rejected, and they get long.

### Option C – Lightweight ADRs in `docs/decisions/`
- **Pros:** each decision is short and dated, with its alternatives and consequences. They're easy for an AI or a reviewer to find and read, and they're immutable once accepted.
- **Cons:** a small amount of writing per significant decision.

## Decision

We will record each **significant, hard-to-reverse** technical decision as an ADR using [0000-template.md](0000-template.md). "Significant" means a database, framework, library family, security mechanism, project structure, or anything that changes the boundary rules in doc 02.

## Consequences

- **Positive:** Claude Code and future reviewers can see why things are the way they are, and changes become deliberate.
- **Negative:** each significant decision costs 15–30 minutes of writing.
- **Follow-ups:** `CLAUDE.md` points to this folder. Adding a NuGet package that introduces a new *kind* of dependency (e.g. a message bus) requires an ADR.

## Revisit when

ADRs are routinely skipped or go stale, which would be a sign the threshold for "significant" is wrong.
