# ADR-0013: Technology stack and versions

- **Status:** Accepted
- **Date:** 2026-10-24

## Context

The course prescribes Java, SQL and Angular. We want stable, supported versions for the whole project duration (until early 2027).

## Decision

| Component | Version | Note |
|---|---|---|
| Java | 21 LTS | Java 25 LTS acceptable if all members have it installed |
| Spring Boot | 4.0.x (latest patch at start) | Spring Framework 7 |
| PostgreSQL | 18 | Docker image `postgres:18` |
| Angular | 22.x | Current major; Angular Material same major |
| Node.js | Active LTS supported by Angular 22 | Check angular.dev version compatibility table |
| Build | Maven wrapper, npm | |
| Testing | JUnit 5, Mockito, Testcontainers, Playwright | |

Exact versions are pinned in build files at Sprint 1 start. Patch upgrades via Dependabot; major upgrades need a new ADR.

## Consequences

Modern, supported stack. Some tutorials target Spring Boot 3 — differences (e.g. Jackson 3, package changes) must be checked in the Spring Boot 4 migration notes when copying examples.
