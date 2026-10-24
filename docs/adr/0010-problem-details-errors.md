# ADR-0010: RFC 9457 Problem Details with stable error codes

- **Status:** Accepted
- **Date:** 2026-10-24

## Context

The frontend must show precise, user-friendly messages and map field errors to form controls. Ad-hoc error bodies differ per endpoint.

## Decision

All errors use `application/problem+json` (RFC 9457, Spring `ProblemDetail`) extended with `code` (stable machine code), `traceId`, `timestamp` and `errors[]` (field, code, message). The catalog of codes is in [05 §4](../05-api-contract.md#4-error-codes). The frontend maps codes to messages in one table.

## Consequences

Consistent handling across the app; messages can be translated without backend changes. Adding a code requires updating 05 and the frontend table.
