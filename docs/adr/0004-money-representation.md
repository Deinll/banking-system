# ADR-0004: Money representation

- **Status:** Accepted
- **Date:** 2026-10-24

## Context

Binary floating point cannot represent 0.10 exactly; errors accumulate. JavaScript `number` has the same problem and JSON numbers may be parsed as floats.

## Decision

| Layer | Representation |
|---|---|
| Java | `BigDecimal`, scale 2 (accruals scale 6), rounding `HALF_EVEN` |
| PostgreSQL | `NUMERIC(19,2)` (accruals `NUMERIC(19,6)`) |
| JSON | string with 2 decimals, e.g. `"1250.00"`, currency in a separate field |
| TypeScript | `string`; display via `Intl.NumberFormat`; no arithmetic on `number` |

Every amount carries its currency (PLN only in MVP).

## Consequences

No rounding drift; multi-currency can be added later. Frontend needs small helpers for input normalization and totals.
