# ADR-0008: Stateless JWT authentication, no refresh token in MVP

- **Status:** Accepted
- **Date:** 2026-10-24

## Context

SPA + REST API. We want simple, stateless authentication that works with Angular interceptors, while keeping sessions short like real banks.

## Decision

- Login returns an HS256 JWT valid 30 minutes, sent as `Authorization: Bearer`.
- Token stored in `sessionStorage`; frontend logs out after 10 minutes of inactivity.
- No refresh token in MVP; on expiry the user logs in again.
- A cached user-status check (≤ 30 s) makes admin blocks effective immediately.

## Alternatives considered

| Option | Pros | Cons |
|---|---|---|
| Server sessions + cookie | Easy revocation | CSRF handling, sticky state |
| Access + refresh tokens | Better UX, revocation | More work — planned as PKG-02 |

## Consequences

Simple to implement and test. Logout is client-side only (token valid until expiry) — documented limitation in [08 §2.4](../08-security.md#24-known-limitations).
