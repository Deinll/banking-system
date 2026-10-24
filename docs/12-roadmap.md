# 12 — Roadmap & Sprint Plan

The plan is built so that **the MVP is finished early** and the remaining time is filled with **extension packages** chosen by priority. Because the final amount of time is not known yet, the plan has three scenarios and a "cut list".

## 1. Timeline (aligned with the Gantt chart)

| Phase | Dates | Output |
|---|---|---|
| Project description | 12.10 – 16.10.2026 | Vision document ✅ |
| Requirements & system design | 19.10 – 25.10.2026 | This documentation v0.1 |
| **Sprint 1** — Foundation & authentication | 26.10 – 08.11 | `v0.1.0` |
| **Sprint 2** — Accounts, ledger core, ATM, history | 09.11 – 22.11 | `v0.2.0` |
| **Sprint 3** — Transfers | 23.11 – 06.12 | `v0.3.0` |
| **Sprint 4** — Interest, admin, audit, first deploy | 07.12 – 20.12 | `v0.4.0` = **MVP feature-complete** |
| **Sprint 5** — Stabilization & release (holidays, reduced capacity) | 21.12 – 10.01.2027 | `v1.0.0` deployed = "gotowy projekt" |
| Buffer: polish, extension packages or catch-up | 11.01 – ~22.01 | Presentation ready |
| Final assessment ("zaliczenie") | ~25.01.2027 | Demo |

**Capacity assumption:** each member ~8–10 h/week → ~20 h per person per sprint (~2.5 working days). Re-estimate after Sprint 1 using actual velocity (story points done).

## 2. MVP sprint plan

### Sprint 1 — Foundation & authentication (26.10 – 08.11)

**Goal:** a user can register (getting a customer number and a personal account with a real-format IBAN) and log in; CI is green on every PR.

| Backend | Frontend | QA / DevOps |
|---|---|---|
| Spring Boot skeleton, error handling, Clock, Jackson money config | Angular skeleton, Material, routing, shells, proxy | Repo, branches, protection, templates, Project board |
| Migrations V001–V004 (users, customers, products, accounts, system accounts, audit) | Core: AuthStore, interceptors, guards, error messages table | `backend-ci`, `frontend-ci`, Dependabot |
| IBAN generator/validator, PESEL validator | Shared validators (IBAN, PESEL, password, money), pipes | Docker Compose, Dockerfiles, README |
| US-01 register (incl. auto account), US-02 login + lockout, US-03 logout, `/me` | Login, register stepper, register success, session timer (US-04) | Testcontainers base, TC-01…13 |
| JWT security config, admin bootstrap | Mocks for A1–A4 | Hosting decision ADR |

Stories: US-01, US-02, US-03, US-04.

### Sprint 2 — Accounts, ledger core, ATM, history (09.11 – 22.11, 11.11 holiday)

**Goal:** a customer sees their accounts, opens a savings account, deposits/withdraws cash at the ATM, and sees history with balance after each operation. Ledger invariants are tested.

| Backend | Frontend | QA / DevOps |
|---|---|---|
| V005 transactions + ledger entries | Dashboard (accounts, total, last operations) | Ledger invariant JUnit extension (TC-38) |
| `LedgerService.post` with locking + invariants | Open account page, account details + rename | Integration tests TC-14…19, TC-34…37 |
| Products (P1), accounts C1–C4 | ATM page with confirmation | Concurrency test for withdrawals (TC-30) |
| ATM service T3/T4 with idempotency | History tab (basic list + paginator) | First e2e journeys (1, 2) |
| History C5 (basic) | Profile page | Demo seed data |

Stories: US-05, US-06, US-07, US-08, US-09, US-12 (basic), US-14, US-15.

### Sprint 3 — Transfers (23.11 – 06.12)

**Goal:** customers can send money to their own accounts and to each other, safely and idempotently.

| Backend | Frontend | QA / DevOps |
|---|---|---|
| TransferService: validation order, type resolution, policy, default titles | Transfer page: own/other toggle, IBAN input, confirmation, result | TC-20…33, TC-40…43 |
| T1 `/transfers`, T2 `/transactions/{id}` | Transaction details page | Concurrency A↔B and idempotency (TC-30, TC-31) |
| History filters | History filters | e2e journeys 3, 4 |
| Performance check of history query (index) | Error messages for all transfer codes | Staging/demo deploy pipeline draft |

Stories: US-10, US-11, US-12 (filters), US-13.

### Sprint 4 — Interest, admin, audit, first deploy (07.12 – 20.12)

**Goal:** savings earn interest; admin can supervise; the app runs on the demo environment.

| Backend | Frontend | QA / DevOps |
|---|---|---|
| Interest accrual & capitalization jobs + demo endpoints D13/D14 | Admin shell; customers search & details with block/unlock dialogs | TC-50…53, TC-60…69 |
| Admin endpoints D1–D12, D15 | Admin transactions, branch cash, audit log, ledger page | `deploy.yml`, first deploy to demo |
| Audit queries | Savings interest display | e2e journeys 5–8 |
| Security hardening (status check cache, headers) | Accessibility & mobile pass | Lighthouse check |

Stories: US-16 … US-24.

### Sprint 5 — Stabilization & release (21.12 – 10.01)

**Goal:** `v1.0.0` deployed; no critical bugs; documentation and demo ready.

- Bug fixing (critical/major first), regression run of all e2e journeys.
- Final deploy, smoke tests, demo accounts.
- Docs updated to reflect reality; PDF generated; sprint reports.
- Demo script (§5) rehearsed.
- If time remains: one **S** package from Tier 1 (§4).

## 3. Scenarios

| Scenario | Trigger | Plan |
|---|---|---|
| **A — As planned** | Velocity matches plan | MVP by 20.12, release 10.01, buffer for 1–2 Tier-1 packages |
| **B — More time** (MVP done early, or deadline later) | Sprint 4 goals met before 20.12, or extra weeks available | Each extra 2-week sprint ≈ one **M** package or two **S** packages. Recommended order in §4.4 |
| **C — Less time / slippage** | A sprint misses its goal | Apply the cut list (§3.1) from the top; never cut tests of money operations |

### 3.1 Cut list (drop in this order if needed)

1. US-23/US-24 admin demo jobs & ledger page → run jobs via scheduler only, check invariants in tests only
2. US-22 audit log **UI** (events are still recorded)
3. Interest tax (post gross interest only)
4. US-09 account rename
5. US-21 admin transaction search (keep details by reference only)
6. History filters (keep pagination)
7. Branch cash operations (ATM only)

Never cut: registration/login, accounts, ledger core, transfers, ATM, history, ownership security, invariant tests, deploy.

## 4. Extension packages

Each package is designed against the existing model ([03 §10](03-domain-model.md#10-how-the-model-extends-examples)) — no breaking changes.
**Size:** S ≈ 3–4 person-days, M ≈ 6–8, L ≈ 10–15 (whole team, incl. tests).

### 4.1 Tier 1 — quick wins (S)

| ID | Package | What it adds (realistic behaviour) | Model / API changes | Size |
|---|---|---|---|---|
| PKG-01 | **Transfer limits** | Customer sets daily and per-transfer limit within product max; own transfers not counted; "day" in Warsaw time | Limit columns (already reserved), `PATCH /accounts/{id}/limits`, new error `DAILY_LIMIT_EXCEEDED` | S |
| PKG-04 | **Profile & password** | Edit phone/address, change password (requires current password), audit | `PATCH /me`, `POST /me/password` | S |
| PKG-05 | **Close account** | Close with zero balance (or transfer remaining balance first); cannot close last personal account if others exist | `POST /accounts/{id}/close`, status `CLOSED`, `closedAt` | S |
| PKG-14 | **PDF statements** | Monthly statement (opening balance, operations, closing balance) and transfer confirmation PDF ("potwierdzenie przelewu") | `GET /accounts/{id}/statements/{yyyy-MM}.pdf`, `GET /transactions/{id}/confirmation.pdf` (OpenPDF / Flying Saucer) | S–M |
| PKG-15 | **Polish UI** | Language switch EN/PL | Angular i18n; no backend change | S |

### 4.2 Tier 2 — impressive, one sprint (M)

| ID | Package | What it adds | Model / API changes | Size |
|---|---|---|---|---|
| PKG-02 | **Refresh tokens & session management** | Short access token (5–10 min) + rotating refresh token in HttpOnly cookie; "log out from all devices"; server-side revocation | Table `refresh_tokens`; `/auth/refresh`; CSRF protection for cookie | M |
| PKG-03 | **Email verification & notifications** | Account `PENDING_VERIFICATION` until link clicked; emails for login from new device, incoming transfer | Table `verification_tokens`; Mailpit in docker compose for local mail | M |
| PKG-06 | **Fees & tariff** | Monthly account fee (waived if monthly inflow ≥ X), fee for 2nd personal account | `FEE` type, `BANK_FEE_INCOME`, `monthly_fee` on product, monthly job | M |
| PKG-07 | **Scheduled & standing orders** | Transfer with future date; standing order (every month on day N, until date); cancel before execution; insufficient funds → `REJECTED` with retry next day | Table `standing_orders`; `scheduledFor`; statuses `PENDING`/`CANCELLED`/`REJECTED`; daily job | M |
| PKG-08 | **External transfers via simulated Elixir** | Transfers to other banks are accepted as `PENDING`, collected into **sessions** (e.g. 9:30, 13:30, 16:00 on business days), settled through a `BANK_CLEARING` account; a simulated second bank ("Partner Bank", other sort code) returns incoming transfers; rejected transfers are refunded | Tables `clearing_sessions`, `clearing_items`; types `EXTERNAL_TRANSFER_OUT/IN`; `BANK_CLEARING` system account; admin session view | M–L |

### 4.3 Tier 3 — "wow" features (L)

| ID | Package | What it adds | Model / API changes | Size |
|---|---|---|---|---|
| PKG-09 | **2FA / transaction confirmation** | TOTP (Google Authenticator) at login; confirm transfers above a threshold with a code, like mobile authorization | `users.totp_secret` (encrypted), `/auth/2fa/*`, transfer `PENDING_AUTHORIZATION` step | M–L |
| PKG-10 | **Phone transfers** ("BLIK na telefon"-like) | Customer registers phone alias → account; others send money to a phone number | Table `phone_aliases`; `POST /transfers/phone` resolves alias then uses TransferService | M |
| PKG-11 | **BLIK codes** | Customer generates a 6-digit code valid 2 minutes; a simulated shop/ATM page uses the code; customer confirms in app; payment executed | Tables `blik_codes`, `blik_requests`; types `BLIK_PAYMENT`/`BLIK_WITHDRAWAL`; `MERCHANT` system/merchant accounts; polling or SSE for confirmation | L |
| PKG-12 | **Virtual debit cards** | Issue card for a personal account (number with Luhn check, masked display, expiry, CVV shown once), freeze/unfreeze, simulated card payment | Table `cards`; `CARD_PAYMENT`; card limits | L |
| PKG-13 | **Multi-currency accounts** | EUR/USD accounts, exchange between own accounts with a rate table and spread | New products, `FX_EXCHANGE`, `exchange_rates` table, system accounts per currency | L |
| PKG-16 | **Real-time notifications** | In-app notification centre, live balance update | Server-Sent Events or WebSocket, `notifications` table | M |

### 4.4 Recommended order if there is extra time

1. **PKG-01 Limits** — completes realistic transfer rules, small.
2. **PKG-14 PDF confirmations/statements** — very visible in a demo.
3. **PKG-07 Standing orders** — introduces scheduling and `PENDING`, impressive and moderate effort.
4. **PKG-08 Elixir simulation** — the most "real bank" feature; showcases the ledger design.
5. **PKG-02 Refresh tokens** — security maturity.
6. **PKG-11 BLIK** or **PKG-10 phone transfers** — the "wow" finale.

| Extra time available | Suggested packages |
|---|---|
| 1 week | PKG-01 + PKG-14 (confirmation PDF only) |
| 2 weeks (1 sprint) | PKG-01, PKG-14, PKG-05 |
| 4 weeks (2 sprints) | + PKG-07, PKG-08 |
| 6+ weeks | + PKG-02, PKG-11 or PKG-10 |

## 5. Demo script (for the final presentation)

1. Register a new customer live (show PESEL and IBAN validation).
2. Log in, show dashboard with the auto-opened personal account.
3. ATM deposit 1 000 zł → open savings account → own transfer 500 zł.
4. Log in as a second customer in another browser; first customer sends 150 zł; second sees it instantly.
5. Show a failed transfer (insufficient funds) and the friendly error.
6. Admin: run interest accrual/capitalization for a past month → show INTEREST and TAX entries.
7. Admin: block an account → transfer to it fails; show audit log with reason.
8. Admin: ledger integrity page — total of all balances is 0.00.
9. Show GitHub: board, a story with sub-issues, PR with review and green CI, release notes.

## 6. Milestones & tags

| Tag | Date | Content |
|---|---|---|
| `v0.1.0` | 08.11 | Registration, login |
| `v0.2.0` | 22.11 | Accounts, ATM, history |
| `v0.3.0` | 06.12 | Transfers |
| `v0.4.0` | 20.12 | Interest, admin, audit, deployed |
| `v1.0.0` | 10.01 | Stable release |
| `v1.x.0` | after | Extension packages |
