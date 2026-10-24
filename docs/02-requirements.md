# 02 — Requirements

This document defines **what** the system must do. **How** data looks is in [03 Domain model](03-domain-model.md); **how** it is exposed is in [05 API contract](05-api-contract.md).

Priority uses MoSCoW: **M** = must (MVP), **S** = should (MVP if time allows), **C** = could (extension package).

---

## 1. Functional requirements

### 1.1 Authentication — `FR-AUTH`

| ID | Requirement | Priority | Rules |
|---|---|---|---|
| FR-AUTH-01 | A guest can register as a customer by providing email, password, first name, last name, PESEL, phone number and residential address. | M | BR-01 … BR-06 |
| FR-AUTH-02 | A user can log in with email and password and receives an access token (JWT). | M | BR-23, BR-24 |
| FR-AUTH-03 | A user can log out. The client discards the token; the event is audited. | M | — |
| FR-AUTH-04 | After 5 consecutive failed login attempts the user is locked for 15 minutes. | M | BR-23 |
| FR-AUTH-05 | The session ends when the token expires (30 min) or after 10 minutes of inactivity in the UI; the user is warned 1 minute before. | M | NFR-SEC-02 |
| FR-AUTH-06 | A logged-in user can retrieve information about themself (identity, role, and customer profile if any). | M | — |

### 1.2 Customer — `FR-CUS`

| ID | Requirement | Priority | Rules |
|---|---|---|---|
| FR-CUS-01 | A customer can view their profile: customer number, name, masked PESEL, date of birth, contact data, address. | M | NFR-SEC-06 |
| FR-CUS-02 | A customer can update phone number and address and change password. | C (PKG-04) | — |

### 1.3 Accounts — `FR-ACC`

| ID | Requirement | Priority | Rules |
|---|---|---|---|
| FR-ACC-01 | On registration the system automatically opens a personal account for the new customer. | M | BR-06 |
| FR-ACC-02 | A customer can open a new account by choosing a product from the catalog and optionally giving it a name. | M | BR-07, BR-08, BR-28 |
| FR-ACC-03 | A customer can list their accounts with IBAN, name, type, balance and status. | M | — |
| FR-ACC-04 | A customer can view details of one of their accounts (incl. interest rate and accrued interest for savings). | M | — |
| FR-ACC-05 | A customer can rename their account. | S | BR-28 |
| FR-ACC-06 | Any logged-in customer can view the catalog of available account products. | M | — |
| FR-ACC-07 | A customer can close an account with zero balance. | C (PKG-05) | BR-22 |

### 1.4 Transactions — `FR-TRX`

| ID | Requirement | Priority | Rules |
|---|---|---|---|
| FR-TRX-01 | A customer can transfer money between their own accounts. | M | BR-09 … BR-18 |
| FR-TRX-02 | A customer can transfer money from a personal account to an account of another Trio Bank customer, identified by IBAN. | M | BR-09 … BR-18 |
| FR-TRX-03 | A customer can browse the history of an account, newest first, filtered by date range, direction (incoming/outgoing) and type, with pagination; each item shows the balance after the operation. | M | — |
| FR-TRX-04 | A customer can view details of a transaction in which one of their accounts took part. | M | — |
| FR-TRX-05 | A customer can deposit cash to a personal account via the simulated ATM. | M | BR-19, BR-20 |
| FR-TRX-06 | A customer can withdraw cash from a personal account via the simulated ATM. | M | BR-11, BR-19, BR-20 |

### 1.5 Interest — `FR-INT`

| ID | Requirement | Priority | Rules |
|---|---|---|---|
| FR-INT-01 | Every day the system accrues interest on savings accounts based on the end-of-day balance. | M | BR-21 |
| FR-INT-02 | On the first day of each month the system capitalizes interest accrued in the previous month and withholds 19% tax. | M | BR-21 |
| FR-INT-03 | The customer sees the interest rate and the interest accrued so far in the current month. | M | — |

### 1.6 Administration — `FR-ADM`

| ID | Requirement | Priority | Rules |
|---|---|---|---|
| FR-ADM-01 | An admin can search customers by name, email, customer number or PESEL. | M | — |
| FR-ADM-02 | An admin can view a customer's profile, user status and accounts. | M | — |
| FR-ADM-03 | An admin can block and unblock a user (with a reason). A blocked user cannot log in. | M | BR-24, BR-25 |
| FR-ADM-04 | An admin can unlock a user locked after failed logins. | M | BR-23 |
| FR-ADM-05 | An admin can block and unblock an account (with a reason). A blocked account cannot send or receive money. | M | BR-12 |
| FR-ADM-06 | An admin can browse all transactions with filters and view their ledger entries. | M | — |
| FR-ADM-07 | An admin can perform teller cash deposits and withdrawals on any active customer account (branch channel). | M | BR-11, BR-19 |
| FR-ADM-08 | An admin can browse the audit log with filters. | S | — |
| FR-ADM-09 | In the demo environment an admin can manually run interest accrual for a given date and capitalization for a given month. | S | BR-21 |
| FR-ADM-10 | An admin can view system account balances and the result of the ledger integrity check. | S | NFR-REL-02 |

### 1.7 System — `FR-SYS`

| ID | Requirement | Priority |
|---|---|---|
| FR-SYS-01 | The system records security and financial events in an append-only audit log (see [03 §5.8](03-domain-model.md#58-auditevent)). | M |
| FR-SYS-02 | On first start the system creates an admin user from configuration if no admin exists. | M |
| FR-SYS-03 | System accounts and account products are created by database migrations. | M |

---

## 2. Business rules

Business rules are enforced **on the backend** (always) and mirrored on the frontend where it improves UX. Error codes are defined in [05 §4](05-api-contract.md#4-error-codes).

### 2.1 Customers and users

| ID | Rule | Error code |
|---|---|---|
| BR-01 | Email is unique (case-insensitive) and stored in lower case. Max 254 characters, valid email format. | `EMAIL_ALREADY_REGISTERED` |
| BR-02 | Password: 12–128 characters, at least one lowercase letter, one uppercase letter, one digit and one special character. Must not contain the email local part. | `VALIDATION_ERROR` (`WEAK_PASSWORD`) |
| BR-03 | PESEL must have 11 digits, a valid check digit and encode a valid date. Unique across customers. Date of birth is **derived from PESEL** (not entered separately). | `VALIDATION_ERROR` (`INVALID_CHECKSUM`), `PESEL_ALREADY_REGISTERED` |
| BR-04 | A customer must be at least 18 years old on the day of registration. | `CUSTOMER_UNDERAGE` |
| BR-05 | One user per email; one customer per user; one customer per PESEL. | — |
| BR-06 | Registration atomically creates: user (`ACTIVE`, role `CUSTOMER`), customer (with generated customer number) and one `PERSONAL_STANDARD` account. If any step fails, nothing is created. | — |

### 2.2 Accounts

| ID | Rule | Error code |
|---|---|---|
| BR-07 | A customer may hold at most `maxAccountsPerCustomer` non-closed accounts of a given product (`PERSONAL_STANDARD`: 2, `SAVINGS_STANDARD`: 5). | `ACCOUNT_LIMIT_REACHED` |
| BR-08 | Opening a product with a prerequisite (savings requires personal) is only possible if the customer has an active account of the required type. | `PREREQUISITE_ACCOUNT_REQUIRED` |
| BR-22 | Accounts are never deleted. They can be closed only with a zero balance (PKG-05). | — |
| BR-28 | Account name: 1–50 characters, trimmed. Default: the product name (set by backend). | `VALIDATION_ERROR` |

### 2.3 Money movements

| ID | Rule | Error code |
|---|---|---|
| BR-09 | Target IBAN must be a valid Polish IBAN: `PL` + 26 digits, mod-97 checksum = 1. Spaces are allowed in input and removed. | `VALIDATION_ERROR` (`INVALID_CHECKSUM`) |
| BR-10 | Amount: > 0, at most 2 decimal places, at most 1 000 000.00 per operation. | `VALIDATION_ERROR` |
| BR-11 | A customer account balance can never become negative. | `INSUFFICIENT_FUNDS` |
| BR-12 | Source and target accounts must be `ACTIVE`. A blocked or closed account can neither send nor receive. | `SOURCE_ACCOUNT_UNAVAILABLE`, `RECIPIENT_ACCOUNT_UNAVAILABLE` |
| BR-13 | Source and target must be different accounts. | `SAME_ACCOUNT_TRANSFER` |
| BR-14 | Outgoing transfers from an account whose product policy is `OWN_ACCOUNTS_ONLY` (savings) are allowed only to the same customer's accounts. | `TRANSFER_POLICY_VIOLATION` |
| BR-15 | The backend determines the transaction type: target owned by the same customer → `OWN_TRANSFER`; another Trio Bank customer → `INTERNAL_TRANSFER`. A valid IBAN with a different bank sort code → rejected (until PKG-08). A valid Trio Bank IBAN that does not exist → rejected. | `EXTERNAL_TRANSFER_NOT_SUPPORTED`, `RECIPIENT_ACCOUNT_NOT_FOUND` |
| BR-16 | Title: 1–140 characters after trimming. If omitted or blank, the **backend** sets the default for the transaction type (see [03 §7](03-domain-model.md#7-defaults-policy)). | `VALIDATION_ERROR` |
| BR-17 | Recipient name is required for `INTERNAL_TRANSFER` (1–140 characters). For `OWN_TRANSFER` it is ignored and set by the backend to the customer's full name. It is **not** verified against the account holder (as in Polish banks before Verification of Payee). | `VALIDATION_ERROR` |
| BR-18 | Every money operation request carries an `Idempotency-Key` header (UUID). The same key from the same user with the same payload returns the original transaction (HTTP 200) instead of executing again. The same key with a different payload is rejected. | `IDEMPOTENCY_KEY_MISSING`, `IDEMPOTENCY_KEY_REUSED` |
| BR-19 | Cash limits per operation — ATM: deposit 10.00–20 000.00, withdrawal 10.00–5 000.00, **multiples of 10 PLN** (banknotes). Branch (admin): 0.01–100 000.00, any amount. | `CASH_AMOUNT_INVALID` |
| BR-20 | ATM operations are allowed only on `PERSONAL` accounts. | `OPERATION_NOT_ALLOWED_FOR_ACCOUNT` |
| BR-26 | All amounts are in PLN in the MVP. A request with another currency is rejected. | `CURRENCY_NOT_SUPPORTED` |
| BR-27 | Timestamps are stored in UTC. "Today", "end of day" and "month" are evaluated in `Europe/Warsaw`. | — |

### 2.4 Interest

| ID | Rule |
|---|---|
| BR-21 | **Accrual:** daily, for each `ACTIVE` savings account: `balance × annualRate / 100 / 365`, kept with 6 decimal places in `accruedInterest`. The job for day D runs after midnight of D+1 (Warsaw time) and is idempotent per account and day. **Capitalization:** on the 1st day of each month: gross = accrued interest rounded HALF_EVEN to 0.01; if gross > 0, post `INTEREST` (BANK_INTEREST_EXPENSE → account) and `TAX` = 19% of gross rounded HALF_EVEN to 0.01 (account → BANK_TAX_LIABILITY); the rounding remainder is discarded and accrued interest reset to 0. Accounts not `ACTIVE` do not accrue. *(Simplified model for education.)* |

### 2.5 Security

| ID | Rule | Error code |
|---|---|---|
| BR-23 | 5 consecutive failed logins → user `LOCKED` for 15 minutes (`lockedUntil`). A successful login resets the counter. A locked user is unlocked automatically after the time or by an admin. Failed attempts for unknown emails return the same error as a wrong password. | `INVALID_CREDENTIALS`, `USER_LOCKED` |
| BR-24 | Only `ACTIVE` users can log in. `BLOCKED` users get an error; their accounts are unchanged. | `USER_BLOCKED` |
| BR-25 | An admin cannot block themself. Admins have no customer profile and no accounts. | `SELF_ACTION_FORBIDDEN` |

---

## 3. Non-functional requirements

| ID | Category | Requirement | Verified by |
|---|---|---|---|
| NFR-SEC-01 | Security | Passwords hashed with BCrypt, cost 12. Never logged or returned. | Code review, unit test |
| NFR-SEC-02 | Security | Stateless JWT, HS256, lifetime 30 min, secret ≥ 256 bits from environment. | Integration test |
| NFR-SEC-03 | Security | A customer can access only their own resources; foreign resources return **404** (existence is not revealed). | Integration tests TC-40… |
| NFR-SEC-04 | Security | No secrets, passwords, tokens or full PESEL in logs. | Code review, log check |
| NFR-SEC-05 | Security | Deployed environment uses HTTPS only; CORS limited to the frontend origin. | Manual check on deploy |
| NFR-SEC-06 | Privacy | PESEL is masked (`*******1234`) in all customer-facing responses; full value only in admin customer details. | Integration test |
| NFR-REL-01 | Reliability | A money operation is atomic: all balance changes, ledger entries and the transaction are committed together or not at all. | Integration test |
| NFR-REL-02 | Reliability | **Ledger invariants** hold at all times: (a) sum of balances of all accounts = 0; (b) each account balance = sum of its credits − sum of its debits; (c) every transaction has exactly one debit and one credit of equal amount. | Automated test after every integration test suite; admin check endpoint |
| NFR-CON-01 | Concurrency | 50 parallel withdrawals/transfers from the same account never make the balance negative and never lose updates. | Concurrency test TC-30 |
| NFR-CON-02 | Concurrency | Concurrent requests with the same idempotency key create at most one transaction. | TC-31 |
| NFR-PERF-01 | Performance | With 20 concurrent users on the demo environment: p95 response time < 500 ms for reads, < 1 s for money operations. | Simple load test (k6 or JMeter), PKG optional |
| NFR-USE-01 | Usability | UI usable from 360 px width (mobile) up to desktop. | Manual test, e2e on mobile viewport |
| NFR-USE-02 | Usability | Every API error code has a user-friendly message in the UI; forms show field errors inline. | e2e tests |
| NFR-USE-03 | Accessibility | Basic WCAG 2.1 AA: labels on all inputs, keyboard navigation, sufficient contrast. | Lighthouse ≥ 90 accessibility |
| NFR-MNT-01 | Maintainability | Backend service layer line coverage ≥ 70% (JaCoCo); CI fails below. | CI |
| NFR-MNT-02 | Maintainability | Lint/format checks pass (Checkstyle or Spotless for Java, ESLint + Prettier for Angular). | CI |
| NFR-MNT-03 | Maintainability | Database schema changes only via Flyway migrations; applied migrations are never edited. | Code review |
| NFR-PORT-01 | Portability | `docker compose up` starts database, backend and frontend locally. | README check by every member |
| NFR-AUD-01 | Auditability | Audit events are append-only (no update/delete through the application). | Code review |
| NFR-OBS-01 | Observability | Health endpoint (`/actuator/health`) and structured logs with request trace ID. | Manual check |

---

## 4. User stories

Format: *As a … I want … so that …*, followed by acceptance criteria (Given/When/Then style). Each story becomes a GitHub issue of type **Story** ([11 Process](11-process.md)). Implementation tasks are sub-issues of the story.

### Epic E2 — Authentication

**US-01 Register** · FR-AUTH-01, FR-ACC-01 · BR-01…06 · Sprint 1
> As a guest, I want to register with my personal data, so that I become a customer of the bank and get my first account.

- [ ] Given valid data, when I submit the form, then my user, customer profile and a personal account with a valid IBAN and zero balance are created, and I see my customer number and IBAN.
- [ ] Given an email or PESEL that is already registered, then I see a clear error on the corresponding field.
- [ ] Given a PESEL with an invalid check digit, then the form shows an error before submitting (frontend) and the API rejects it (backend).
- [ ] Given a PESEL of a person under 18, then registration is rejected with `CUSTOMER_UNDERAGE`.
- [ ] Given a weak password, then I see which password rule is not met.
- [ ] Event `USER_REGISTERED` and `ACCOUNT_OPENED` are written to the audit log.

**US-02 Log in** · FR-AUTH-02, FR-AUTH-04 · BR-23, BR-24 · Sprint 1
> As a user, I want to log in with email and password, so that I can access my accounts securely.

- [ ] Given correct credentials of an active user, I am logged in and redirected to the dashboard (customer) or admin panel (admin).
- [ ] Given wrong credentials, I see "Invalid email or password" (the same message for unknown email).
- [ ] Given 5 consecutive failed attempts, the user is locked for 15 minutes and I see when I can try again.
- [ ] Given a blocked user, I see that my access is blocked and should contact the bank.
- [ ] Every success/failure is audited (`LOGIN_SUCCEEDED`, `LOGIN_FAILED`, `USER_LOCKED`).

**US-03 Log out** · FR-AUTH-03 · Sprint 1
> As a user, I want to log out, so that nobody else can use my session on this device.

- [ ] When I click "Log out", the token is removed from the client, I am redirected to the login page and protected pages are no longer accessible.
- [ ] The event `LOGOUT` is audited.

**US-04 Session expiry** · FR-AUTH-05 · Sprint 1
> As a customer, I want my session to end automatically when I am inactive, so that my money is safe if I forget to log out.

- [ ] After 9 minutes of inactivity I see a warning with a countdown and a "Stay logged in" button.
- [ ] After 10 minutes of inactivity, or when the token expires, I am logged out and see a message explaining why.
- [ ] Any API response `401` logs me out and redirects to the login page.

### Epic E3 — Customer & Accounts

**US-05 View profile** · FR-CUS-01 · Sprint 2
> As a customer, I want to view my profile, so that I can check the data the bank has about me.

- [ ] I see customer number, full name, masked PESEL, date of birth, email, phone and address.

**US-06 View my accounts** · FR-ACC-03 · Sprint 2
> As a customer, I want to see all my accounts and their balances on the dashboard, so that I know how much money I have.

- [ ] Each account shows name, type, formatted IBAN, balance with currency and status.
- [ ] The dashboard shows the total balance of all active accounts.
- [ ] Blocked accounts are visibly marked.

**US-07 Open a new account** · FR-ACC-02, FR-ACC-06 · BR-07, BR-08 · Sprint 2
> As a customer, I want to open an additional personal or savings account, so that I can organize my money.

- [ ] I see the product catalog with name, type, interest rate and main rules.
- [ ] When I choose a product and confirm, a new account with a new unique IBAN and zero balance appears in my list.
- [ ] If I already have the maximum number of accounts of this product, I see `ACCOUNT_LIMIT_REACHED`.
- [ ] I cannot open a savings account without an active personal account.

**US-08 Account details** · FR-ACC-04 · Sprint 2
> As a customer, I want to see the details of an account, so that I can copy its IBAN and see its parameters.

- [ ] I see name, IBAN (with a copy button), product, status, balance, opening date; for savings: interest rate and interest accrued this month.

**US-09 Rename an account** · FR-ACC-05 · BR-28 · Sprint 2
> As a customer, I want to give my accounts names, so that I can distinguish them easily.

- [ ] I can change the name to 1–50 characters; the new name is shown everywhere.

### Epic E4/E5/E6 — Money movements

**US-10 Transfer between my accounts** · FR-TRX-01 · BR-09…18 · Sprint 3
> As a customer, I want to move money between my own accounts, so that I can save or spend it as needed.

- [ ] I choose source and target from lists of my active accounts; recipient name is not required.
- [ ] I see a confirmation screen (from, to, amount, title) before sending.
- [ ] After confirmation both balances change immediately and the operation appears in both histories with type `OWN_TRANSFER`.
- [ ] Transfers from a savings account to my personal account are allowed.
- [ ] Double-clicking "Confirm" never creates two transfers.

**US-11 Transfer to another customer** · FR-TRX-02 · BR-09…18 · Sprint 3
> As a customer, I want to send money to another Trio Bank customer by IBAN, so that I can pay them.

- [ ] I enter recipient name, IBAN, amount and an optional title; the IBAN is validated while typing.
- [ ] Without a title, the transfer gets the default title "Funds transfer".
- [ ] I cannot send more than my balance (`INSUFFICIENT_FUNDS`).
- [ ] I cannot send from a savings account to someone else (`TRANSFER_POLICY_VIOLATION`).
- [ ] A non-existing Trio Bank IBAN gives `RECIPIENT_ACCOUNT_NOT_FOUND`; another bank's IBAN gives `EXTERNAL_TRANSFER_NOT_SUPPORTED`.
- [ ] After success I see a summary with the transaction reference; the recipient sees the incoming transfer in their history.

**US-12 Transaction history** · FR-TRX-03 · Sprint 2 (basic) / Sprint 3 (filters)
> As a customer, I want to browse the history of my account, so that I can track my spending and income.

- [ ] Items are sorted newest first, 20 per page, with date, counterparty, title, signed amount (+/−) and balance after.
- [ ] I can filter by date range, direction (incoming/outgoing) and type.

**US-13 Transaction details** · FR-TRX-04 · Sprint 3
> As a customer, I want to open a single transaction, so that I can see all its details (e.g. to confirm a payment).

- [ ] I see reference, type, status, date, amount, title, sender name and IBAN, recipient name and IBAN.
- [ ] I cannot open a transaction that does not involve my accounts (404).

**US-14 ATM deposit** · FR-TRX-05 · BR-19, BR-20 · Sprint 2
> As a customer, I want to deposit cash at an ATM, so that I have money on my account.

- [ ] I choose a personal account and an amount (multiple of 10, 10–20 000 PLN); the balance increases immediately.
- [ ] History shows `CASH_DEPOSIT` with title "ATM cash deposit".

**US-15 ATM withdrawal** · FR-TRX-06 · BR-11, BR-19, BR-20 · Sprint 2
> As a customer, I want to withdraw cash from an ATM, so that I can pay in cash.

- [ ] Amount must be a multiple of 10, 10–5 000 PLN and not exceed the balance.
- [ ] Savings accounts are not available for ATM operations.

### Epic E7 — Interest

**US-16 Earn interest on savings** · FR-INT-01…03 · BR-21 · Sprint 4
> As a customer, I want my savings account to earn interest, so that my money grows.

- [ ] Interest is accrued daily and shown on the account details as "Interest accrued this month".
- [ ] On the 1st day of the month I see two history items: `INTEREST` (gross, credit) and `TAX` (19%, debit).
- [ ] Accounts with zero balance or not active earn nothing.

### Epic E8 — Administration

**US-17 Find customers** · FR-ADM-01, FR-ADM-02 · Sprint 4
> As an admin, I want to search customers and see their details, so that I can help them and supervise accounts.

- [ ] I can search by name, email, customer number or PESEL; results are paginated.
- [ ] Customer details show profile (full PESEL), user status, failed login count and all accounts.

**US-18 Block, unblock and unlock users** · FR-ADM-03, FR-ADM-04 · BR-23…25 · Sprint 4
> As an admin, I want to block or unlock a user, so that I can react to suspicious activity or help a locked-out customer.

- [ ] Blocking requires a reason; a blocked user cannot log in.
- [ ] Unlocking a `LOCKED` user resets failed attempts.
- [ ] I cannot block myself. All actions are audited with the reason.

**US-19 Block and unblock accounts** · FR-ADM-05 · BR-12 · Sprint 4
> As an admin, I want to block an account, so that no money can leave or enter it during an investigation.

- [ ] Blocking requires a reason; transfers from/to a blocked account are rejected.

**US-20 Teller cash operations** · FR-ADM-07 · BR-19 · Sprint 4
> As an admin acting as a teller, I want to deposit or withdraw cash for a customer, so that customers can use the branch.

- [ ] I enter the account IBAN and amount (any amount 0.01–100 000); the operation has channel `BRANCH`.

**US-21 Browse transactions** · FR-ADM-06 · Sprint 4
> As an admin, I want to browse all transactions, so that I can investigate issues.

- [ ] Filters: date range, type, status, account IBAN, reference. Details show both ledger entries.

**US-22 Audit log** · FR-ADM-08, FR-SYS-01 · Sprint 4
> As an admin, I want to see the audit log, so that I know who did what and when.

- [ ] Filters: date range, action, actor email, target. Read-only.

**US-23 Run interest jobs (demo)** · FR-ADM-09 · Sprint 4
> As an admin preparing a demo, I want to run interest accrual and capitalization manually, so that I can show interest without waiting a month.

- [ ] Available only in `local` and `demo` profiles; running twice for the same date has no additional effect.

**US-24 Ledger integrity** · FR-ADM-10 · Sprint 4
> As an admin, I want to see system account balances and an integrity check, so that I can trust the books are balanced.

- [ ] I see balances of all system accounts and the result of invariants (a)–(c) from NFR-REL-02.

---

## 5. Traceability matrix

| Requirement | User story | Sprint | Test cases ([09](09-testing.md)) |
|---|---|---|---|
| FR-AUTH-01 | US-01 | 1 | TC-01 … TC-06 |
| FR-AUTH-02, 04 | US-02 | 1 | TC-07 … TC-10 |
| FR-AUTH-03 | US-03 | 1 | TC-11 |
| FR-AUTH-05 | US-04 | 1 | TC-12 |
| FR-AUTH-06 | US-01, US-02 | 1 | TC-13 |
| FR-CUS-01 | US-05 | 2 | TC-14 |
| FR-ACC-01 | US-01 | 1 | TC-01 |
| FR-ACC-02, 06 | US-07 | 2 | TC-15 … TC-17 |
| FR-ACC-03, 04 | US-06, US-08 | 2 | TC-18 |
| FR-ACC-05 | US-09 | 2 | TC-19 |
| FR-TRX-01 | US-10 | 3 | TC-20 … TC-22 |
| FR-TRX-02 | US-11 | 3 | TC-23 … TC-29 |
| FR-TRX-03, 04 | US-12, US-13 | 2–3 | TC-32, TC-33 |
| FR-TRX-05, 06 | US-14, US-15 | 2 | TC-34 … TC-37 |
| FR-INT-01…03 | US-16 | 4 | TC-50 … TC-53 |
| FR-ADM-01…10 | US-17 … US-24 | 4 | TC-60 … TC-68 |
| FR-SYS-01 | US-22 | 1–4 | TC-69 |
| NFR-REL-02, NFR-CON-01/02 | — | 2–3 | TC-30, TC-31, TC-38 |
| NFR-SEC-03 | — | 2–4 | TC-40 … TC-43 |
