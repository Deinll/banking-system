# ADR-0011: Business defaults owned by the backend

- **Status:** Accepted
- **Date:** 2026-10-24

## Context

Some fields are optional with defaults (transfer title, account name, country code). Defaults could be applied by the frontend or by the backend. Some transactions are not created by the frontend at all (interest job, admin teller, future mobile app), and the default title depends on the transaction type that only the backend resolves.

## Decision

- Business defaults are applied by the **backend**. The frontend omits the field (or sends `null`) and may show the default only as a placeholder.
- All defaults and their source are listed in [03 §7](../03-domain-model.md#7-defaults-policy).
- UI-only defaults (preselected account, default date filter) belong to the frontend.

## Consequences

One source of truth, consistent data regardless of client. The frontend placeholder text must match the backend constant (checked in review; could later be served by an endpoint).
