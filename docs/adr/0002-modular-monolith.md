# ADR-0002: Modular monolith, package-by-feature

- **Status:** Accepted
- **Date:** 2026-10-24

## Context

The team is small and the timeline short. Microservices would add networking, distributed transactions and deployment complexity. Money operations need strong consistency (one DB transaction).

## Decision

One Spring Boot application, organized by feature modules (`auth`, `account`, `ledger`, `payment`, …). Modules communicate through services only; each module owns its repositories.

## Alternatives considered

| Option | Pros | Cons |
|---|---|---|
| Microservices | Independent scaling | Distributed transactions for transfers, much more ops work |
| Package-by-layer (`controllers/`, `services/`) | Simple at first | Features scattered, unclear boundaries as the app grows |

## Consequences

Simple deployment and ACID transactions. Boundaries must be respected in code review; a module could later be extracted (e.g. clearing) if needed.
