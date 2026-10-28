# ADR-0009: Pessimistic locking and idempotency keys for money operations

- **Status:** Accepted
- **Date:** 2026-10-24

## Context

Two concurrent withdrawals could both read the same balance and overdraw the account. A double click or network retry could execute a transfer twice.

## Decision

- **Locking:** `LedgerService.post()` locks both accounts with `SELECT … FOR UPDATE`, always in ascending `id` order to avoid deadlocks. Other entity updates use optimistic locking (`@Version`).
- **Idempotency:** money endpoints require an `Idempotency-Key` header. The key is stored on the transaction with a SHA-256 hash of the request; unique `(initiated_by_user_id, idempotency_key)`. Replay with the same payload returns the original result (200); different payload → 409.
- **Defence in depth:** DB `CHECK (balance >= 0)` for customer accounts.

## Alternatives considered

| Option | Pros | Cons |
|---|---|---|
| Optimistic locking only for balances | No blocking | Many retries under contention; harder to reason about |
| Separate idempotency table | Works for non-transaction operations | Extra table; not needed in MVP |

## Consequences

Correct under concurrency (verified by TC-30/31). Frontend must generate and reuse keys correctly.
