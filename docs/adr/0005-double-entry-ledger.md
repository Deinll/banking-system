# ADR-0005: Double-entry ledger with system accounts

- **Status:** Accepted
- **Date:** 2026-10-24

## Context

The bank must support many kinds of money movement (transfers, cash, interest, tax, later fees, BLIK, external transfers) and we must be able to prove that no money was created or lost.

## Decision

- Every money movement is a `Transaction` between a **source** and a **target** account, recorded with exactly two `LedgerEntry` rows (DEBIT and CREDIT) storing `balanceAfter`.
- Money entering or leaving the customer world goes through **system accounts** (`BANK_CASH`, `BANK_INTEREST_EXPENSE`, `BANK_TAX_LIABILITY`, …). System accounts may have negative balances; customer accounts may not.
- All postings go through one method, `LedgerService.post()`. Operation-specific services (transfer, ATM, interest, future BLIK…) resolve accounts and validate rules, then call it.
- Account balance is stored on the account (fast reads) and is verifiable from the entries.

Invariants: sum of all balances = 0; balance = credits − debits; each transaction balanced.

## Alternatives considered

| Option | Pros | Cons |
|---|---|---|
| Only a `balance` column + log table with from/to | Simpler | "Deposit" has no source; harder to verify; history needs special cases |
| Balance computed only from entries (no stored balance) | Single truth | Slow reads, complex locking |

## Consequences

Strong correctness guarantees and a uniform history model. New payment types need no schema change in the core. Slightly more rows per transaction.
