# ADR-0006: Account product catalog

- **Status:** Accepted
- **Date:** 2026-10-24

## Context

Personal and savings accounts differ in rules (interest, allowed transfers, ATM, max count, prerequisites). More account types may come (currency, youth, business). Hard-coding `if (type == SAVINGS)` everywhere does not scale.

## Decision

Account rules live in table `account_products`. Each customer account references a product. Services read rules from the product (`outgoingTransferPolicy`, `atmAllowed`, `maxAccountsPerCustomer`, `prerequisiteType`, `interestRate`). Products are seeded by Flyway.

## Consequences

New product = new row (plus migration). Rule changes are data changes. Product changes affect existing accounts unless rules are copied to the account — accepted for MVP (rates may change for all accounts, as in real banks with notice).
