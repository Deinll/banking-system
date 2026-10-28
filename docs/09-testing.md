# 09 — Testing & Quality

## 1. Strategy

Tests are written **in the same sprint and PR** as the feature — not in a separate phase. The QA/DevOps member owns the strategy, integration/e2e suites and CI gates; developers write unit tests for their code.

```
          ▲  E2E (Playwright)            ~15 scenarios  — critical user journeys
         ▲▲▲ Integration (MockMvc + Testcontainers)      — every endpoint, DB constraints, security
       ▲▲▲▲▲▲ Unit (JUnit/Mockito, Angular unit tests)    — business rules, validators, utils
```

| Level | Tool | Scope | Owner | Runs in CI |
|---|---|---|---|---|
| Backend unit | JUnit 5, Mockito, AssertJ | Services, validators (IBAN, PESEL, money), domain methods, interest math | Backend | Every PR |
| Backend integration | Spring Boot Test, MockMvc, **Testcontainers PostgreSQL 18** | Endpoints end-to-end through real DB, migrations, security, ledger invariants, concurrency | QA + Backend | Every PR |
| Frontend unit | Angular CLI test runner | Validators, pipes, services, key components | Frontend | Every PR |
| E2E | Playwright (Chromium + mobile viewport) | User journeys against docker compose stack | QA | PR to `main`, nightly on `develop` |
| Manual exploratory | Checklist per sprint | UX, responsiveness, accessibility (Lighthouse) | QA | Sprint review |
| Load (optional) | k6 | NFR-PERF-01 | QA | Manually, Sprint 4/5 |

## 2. Quality gates (CI)

A PR can be merged only if:

- [ ] Backend build and all tests pass; **JaCoCo line coverage ≥ 70%** for `*Service` classes (NFR-MNT-01).
- [ ] Frontend lint, unit tests and production build pass.
- [ ] No Checkstyle/Spotless or ESLint/Prettier violations.
- [ ] At least 1 approving review.

## 3. Test data

| Data | Source |
|---|---|
| Valid PESELs | `90051412343`, `85110356782`, `01222843219`; under-18: `10270111119` ([03 §8.4](03-domain-model.md#84-pesel)) |
| Valid IBANs | `PL45999000121000000000000001`, `PL18999000121000000000000002`; invalid checksum: `PL45999000121000000000000002`; foreign bank (valid checksum, other sort code): generate in test utils |
| Strong password | `Str0ng!Passw0rd` |
| Builders | `TestDataFactory` (backend) creates customers/accounts via services, so invariants hold |
| Clock | `MutableClock` bean in tests to move time (interest, lockout) |

## 4. Special test suites

### 4.1 Ledger invariant check (TC-38)

A reusable assertion `LedgerAssertions.assertBalanced()` runs **after every integration test class** (JUnit extension) and checks NFR-REL-02:

```sql
-- (a) total balance is zero
SELECT COALESCE(SUM(balance), 0) FROM accounts;                       -- = 0.00
-- (b) balance equals entries
SELECT a.id FROM accounts a
LEFT JOIN ledger_entries e ON e.account_id = a.id
GROUP BY a.id, a.balance
HAVING a.balance <> COALESCE(SUM(CASE e.direction WHEN 'CREDIT' THEN e.amount ELSE -e.amount END), 0);  -- no rows
-- (c) each transaction has one debit and one credit of equal amount
SELECT t.id FROM transactions t JOIN ledger_entries e ON e.transaction_id = t.id
GROUP BY t.id, t.amount
HAVING COUNT(*) <> 2 OR SUM(CASE e.direction WHEN 'CREDIT' THEN e.amount ELSE -e.amount END) <> 0
    OR MAX(e.amount) <> t.amount;                                          -- no rows
```

The same queries back the admin endpoint D15.

### 4.2 Concurrency (TC-30, TC-31)

- Account with 100.00; 50 threads each try to withdraw/transfer 10.00 simultaneously → exactly 10 succeed, 40 get `INSUFFICIENT_FUNDS`, final balance 0.00, invariants hold.
- Opposite transfers A→B and B→A in parallel (100 iterations) → no deadlock, invariants hold.
- 10 parallel requests with the same idempotency key → exactly 1 transaction.

### 4.3 Security (TC-40…43)

Parameterized test that calls every customer endpoint with a token of another customer (→ 404) and of an admin (→ 403), and every admin endpoint with a customer token (→ 403).

## 5. Test cases

`Level`: U = unit, I = integration, E = e2e. Each test method name references the TC ID, e.g. `TC23_internalTransfer_movesMoneyBetweenCustomers()`.

### 5.1 Authentication & registration

| ID | Story | Scenario | Expected | Level |
|---|---|---|---|---|
| TC-01 | US-01 | Register with valid data | 201; user ACTIVE, customer with 8-digit number, 1 personal account, balance 0.00, valid IBAN; audit events | I, E |
| TC-02 | US-01 | Email already registered (different case) | 409 `EMAIL_ALREADY_REGISTERED` | I |
| TC-03 | US-01 | PESEL with invalid check digit | 400, field `pesel` `INVALID_CHECKSUM` | U, I |
| TC-04 | US-01 | PESEL of a 17-year-old | 422 `CUSTOMER_UNDERAGE` | I |
| TC-05 | US-01 | Password without special character | 400 `WEAK_PASSWORD` | U, I |
| TC-06 | US-01 | Request contains `"role": "ADMIN"` | Ignored; user has role CUSTOMER | I |
| TC-07 | US-02 | Login with correct credentials | 200, JWT with correct claims, `lastLoginAt` updated | I, E |
| TC-08 | US-02 | Wrong password / unknown email | 401 `INVALID_CREDENTIALS` (identical body) | I |
| TC-09 | US-02 | 5 wrong passwords, then correct | 6th: 403 `USER_LOCKED` with `lockedUntil`; after clock +15 min: 200 | I |
| TC-10 | US-02 | Blocked user logs in | 403 `USER_BLOCKED` | I |
| TC-11 | US-03 | Logout | 204; UI redirects to login; protected route not accessible | I, E |
| TC-12 | US-04 | Expired token | 401 `UNAUTHENTICATED`; UI shows "session expired" | I, E |
| TC-13 | — | `/me` for customer and admin | Customer has `customer` object with masked PESEL; admin has `null` | I |

### 5.2 Accounts

| ID | Story | Scenario | Expected | Level |
|---|---|---|---|---|
| TC-14 | US-05 | View profile | Masked PESEL, all fields | I, E |
| TC-15 | US-07 | Open savings with active personal | 201; new IBAN unique and valid; name defaults to product name | I, E |
| TC-16 | US-07 | Open 3rd personal account | 422 `ACCOUNT_LIMIT_REACHED` | I |
| TC-17 | US-07 | Open savings when personal is blocked | 422 `PREREQUISITE_ACCOUNT_REQUIRED` | I |
| TC-18 | US-06/08 | List & details | Only own accounts; savings shows rate and accrued interest | I |
| TC-19 | US-09 | Rename to 51 chars / valid name | 400 / 200 | I |

### 5.3 Money movements

| ID | Story | Scenario | Expected | Level |
|---|---|---|---|---|
| TC-20 | US-10 | Own transfer personal → savings | 201, type `OWN_TRANSFER`, recipientName = own full name, default title "Own transfer" | I, E |
| TC-21 | US-10 | Own transfer savings → personal | 201 | I |
| TC-22 | US-10 | Same source and target | 422 `SAME_ACCOUNT_TRANSFER` | I |
| TC-23 | US-11 | Internal transfer to another customer | 201, type `INTERNAL_TRANSFER`; both histories updated; balances correct | I, E |
| TC-24 | US-11 | Amount greater than balance | 422 `INSUFFICIENT_FUNDS`; no transaction stored | I |
| TC-25 | US-11 | From savings to another customer | 422 `TRANSFER_POLICY_VIOLATION` | I |
| TC-26 | US-11 | Valid IBAN of another bank | 422 `EXTERNAL_TRANSFER_NOT_SUPPORTED` | I |
| TC-27 | US-11 | Valid Trio IBAN that does not exist | 422 `RECIPIENT_ACCOUNT_NOT_FOUND` | I |
| TC-28 | US-11 | Target account blocked | 422 `RECIPIENT_ACCOUNT_UNAVAILABLE` | I |
| TC-29 | US-11 | Missing recipient name / amount `10.001` / amount `0` | 400 with field errors | U, I |
| TC-30 | — | Concurrency (§4.2) | No negative balance, invariants hold | I |
| TC-31 | — | Idempotency: replay same key → 200 same body; same key different amount → 409 | | I |
| TC-32 | US-12 | History filters (direction, type, dates), pagination, `balanceAfter` correct | | I, E |
| TC-33 | US-13 | Transaction details own / foreign | 200 / 404 | I |
| TC-34 | US-14 | ATM deposit 200 | 201, `CASH_DEPOSIT`, channel ATM, BANK_CASH −200 | I, E |
| TC-35 | US-14/15 | ATM amount 15 or 25 000 | 422 `CASH_AMOUNT_INVALID` | I |
| TC-36 | US-15 | ATM withdrawal from savings | 422 `OPERATION_NOT_ALLOWED_FOR_ACCOUNT` | I |
| TC-37 | US-15 | ATM withdrawal more than balance | 422 `INSUFFICIENT_FUNDS` | I |
| TC-38 | — | Ledger invariants after every suite (§4.1) | Always true | I |

### 5.4 Security

| ID | Scenario | Expected | Level |
|---|---|---|---|
| TC-40 | Customer accesses another customer's account / history / transaction | 404 | I |
| TC-41 | Customer calls `/admin/**` | 403 `ACCESS_DENIED` | I |
| TC-42 | Tampered or expired JWT | 401 | I |
| TC-43 | Blocked user's token used after block | 401 within 30 s | I |

### 5.5 Interest

| ID | Scenario | Expected | Level |
|---|---|---|---|
| TC-50 | Accrual: balance 10 000.00, rate 4%, 1 day | accrued = 1.095890 | U |
| TC-51 | Accrual run twice for the same day | Second run skips (idempotent) | I |
| TC-52 | Capitalization after 31 days of TC-50: accrued 33.972590 | INTEREST 33.97, TAX 6.45 (19% of 33.97 = 6.4543 → 6.45), net +27.52, accrued reset to 0 | U, I |
| TC-53 | Blocked savings account | No accrual | I |

### 5.6 Admin & audit

| ID | Scenario | Expected | Level |
|---|---|---|---|
| TC-60 | Search customers by PESEL fragment / email / number | Matching results, paginated | I |
| TC-61 | Customer details | Full PESEL, accounts | I |
| TC-62 | Block user with reason; login | 403 `USER_BLOCKED`; audit with reason | I, E |
| TC-63 | Admin blocks themself | 422 `SELF_ACTION_FORBIDDEN` | I |
| TC-64 | Unlock locked user | ACTIVE, attempts = 0 | I |
| TC-65 | Block account; transfer to it | 422 `RECIPIENT_ACCOUNT_UNAVAILABLE` | I |
| TC-66 | Branch deposit 1234.56 | 201, channel BRANCH | I |
| TC-67 | Admin transaction details | 2 entries, initiatedBy | I |
| TC-68 | Demo jobs endpoint in `test` profile | 404 `FEATURE_DISABLED` | I |
| TC-69 | Audit events exist for: register, login ok/fail, lock, block, open account, transaction | All present with actor and target | I |

### 5.7 E2E journeys (Playwright)

1. Register → login → see personal account with IBAN.
2. ATM deposit → own transfer to new savings → history shows both.
3. Two customers: A transfers to B → B logs in and sees incoming transfer.
4. Failed transfer (insufficient funds) shows friendly message.
5. Lockout after 5 wrong passwords.
6. Admin blocks account → customer transfer to it fails.
7. Inactivity logout (shortened timer in e2e config).
8. Mobile viewport: dashboard and transfer usable.

## 6. Definition of Done

A story is **Done** when:

- [ ] All acceptance criteria are met and demoed.
- [ ] Code is merged to `develop` via PR with ≥ 1 approval and green CI.
- [ ] Unit and integration tests for the new behaviour exist and reference TC IDs; e2e scenario added if it is a user journey.
- [ ] API matches [05](05-api-contract.md); docs updated in the same PR if anything changed.
- [ ] Frontend handles loading/empty/error states; error codes have messages.
- [ ] No new lint warnings; no secrets or personal data committed.
- [ ] Ledger invariants still hold (CI).
- [ ] Issue closed by the PR (`Closes #nn`) and moved to Done on the board.

## 7. Bug handling

- Bugs are issues with type **Bug** (template in `.github/ISSUE_TEMPLATE/bug.yml`): steps, expected, actual, environment, `traceId`.
- Severity: **Critical** (money incorrect, security) → fix immediately, blocks release; **Major** → this sprint; **Minor** → backlog.
- Every fixed bug gets a regression test.
