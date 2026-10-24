# ADR-0007: Polish IBAN with fictional sort code

- **Status:** Accepted
- **Date:** 2026-10-24

## Context

The project simulates a Polish bank. Real account numbers have a well-defined format with check digits, which allows input validation and future interbank simulation.

## Decision

- Account numbers are Polish IBANs: `PL` + 2 check digits (ISO 13616 mod 97) + 8-digit sort code + 16-digit account number.
- Trio Bank sort code: `99900012` (bank 999, branch 0001, valid check digit) — fictional, configurable.
- Account numbers come from a DB sequence (customers from `1000000000000001`, system accounts `9000…`).
- Stored without spaces, displayed in groups of 4.
- Recipient IBAN is validated (format + checksum) on frontend and backend; a different sort code means "another bank" (rejected until PKG-08).

## Consequences

Realistic UX and validation; external transfer simulation can route by sort code. Sequential numbers are predictable — acceptable because access is protected by ownership checks, not by secrecy of IBANs (as in real banks).
