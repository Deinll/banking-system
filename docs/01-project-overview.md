# 01 — Project Overview

## 1. Vision

Trio Bank is a web application that simulates a modern Polish retail bank. Customers can open bank accounts, check balances, move money and review their transaction history in a simple and secure way.

The project is educational, but it deliberately follows practices used by real banks — double-entry bookkeeping, account products, Polish IBAN numbers with checksums, PESEL validation, interest capitalization, audit logging — so that the system is realistic and can grow feature by feature without redesign.

## 2. Value proposition

| For | Trio Bank provides |
|---|---|
| Customers | One place to manage their money: accounts, balances, transfers, history |
| | Fast access to the current balance and every past operation |
| | Clear, intuitive interface that works on desktop and mobile |
| | Security of personal data and money |
| Bank staff (admin) | Overview of customers and accounts, ability to block suspicious activity, teller cash operations, full audit trail |
| The team | A realistic, extensible codebase that demonstrates professional engineering practices |

## 3. Goals

| ID | Goal | Measured by |
|---|---|---|
| G-1 | Deliver a working MVP of a bank (accounts + transfers) | All MVP user stories pass acceptance criteria |
| G-2 | Financial correctness | Ledger invariant tests always green (see [09](09-testing.md)) |
| G-3 | Professional engineering process | Every change traceable: issue → branch → PR → review → CI → merge |
| G-4 | Extensibility | Extension packages ([12](12-roadmap.md)) can be added without breaking API or schema |
| G-5 | Deployable product | Application runs with one command locally and is deployed to a public demo environment |

## 4. Scope

### 4.1 In scope (MVP)

- Registration of a customer with realistic personal data (PESEL, address) and login with email + password
- Automatic opening of a personal account on registration
- Opening additional personal and savings accounts from the product catalog
- Polish IBAN account numbers with checksum validation
- Transfers between own accounts and to other Trio Bank customers
- Simulated ATM deposits and withdrawals
- Transaction history with filters and balance after each operation
- Savings interest: daily accrual, monthly capitalization, 19% tax withholding
- Admin panel: customers, accounts, blocking, teller cash operations, transactions, audit log
- Security: password policy, login lockout, JWT, role-based access, audit log

The full list of requirements is in [02 Requirements](02-requirements.md).

### 4.2 Out of scope (MVP)

These are **planned** as extension packages — the model is designed so they can be added later (see [12 Roadmap](12-roadmap.md)):

- Customer-configurable transfer limits (PKG-01)
- Refresh tokens, email verification, 2FA (PKG-02, PKG-03, PKG-09)
- Profile editing, account closing, fees (PKG-04, PKG-05, PKG-06)
- Scheduled and standing orders (PKG-07)
- Transfers to other banks via simulated Elixir sessions (PKG-08)
- Phone transfers and BLIK (PKG-10, PKG-11)
- Cards, multi-currency accounts, PDF statements, notifications (PKG-12 … PKG-16)

### 4.3 Explicitly never in scope

- Handling real money or connecting to real payment systems
- Storing real personal data — **the system must only ever contain fictional data** (fake names, generated PESEL numbers)
- Legal/regulatory compliance (KYC/AML, PSD2, GDPR audits) beyond demonstrating good practices

## 5. MVP definition

The MVP is the smallest version in which a customer can **register, get an account, put money in and send it to someone else**, and an admin can supervise it.

| Area | MVP feature | Requirement |
|---|---|---|
| Auth | Register, log in, log out, lockout after 5 failed attempts | FR-AUTH-01…05 |
| Customer | View own profile | FR-CUS-01 |
| Accounts | Auto personal account, open personal/savings, list, details, rename | FR-ACC-01…06 |
| Money | Own transfer, internal transfer, ATM deposit/withdrawal | FR-TRX-01, 02, 05, 06 |
| History | History per account with filters, transaction details | FR-TRX-03, 04 |
| Interest | Daily accrual, monthly capitalization with tax | FR-INT-01…03 |
| Admin | Customers, blocking, unlocking, teller cash, transactions, audit log | FR-ADM-01…09 |
| Audit | Security and financial events recorded | FR-SYS-01 |

## 6. Stakeholders and roles

### 6.1 System roles

| Role | Description |
|---|---|
| **Guest** | Not logged in. Can register and log in. |
| **Customer** | Client of the bank. Manages own accounts and money. |
| **Admin** | Bank employee. Supervises customers and accounts, performs teller operations. Has no bank accounts. |

### 6.2 Team

| Member | Primary role | Responsibilities |
|---|---|---|
| Hryhorii Martynkian | Backend developer | Spring Boot API, database, business logic, backend unit tests |
| Marko Zhenchuk | Frontend developer | Angular application, UX, frontend unit tests |
| Kateryna Valentyuk | QA / DevOps engineer | CI/CD, Docker, deployment, integration & e2e tests, test strategy, release |

Team members: **Marko Zhenchuk, Hryhorii Martynkian, Kateryna Valentyuk** — fill in the table above.

Shared responsibilities: requirements, API contract, code review, documentation.

## 7. Technology stack

| Layer | Technology | Version policy | Why |
|---|---|---|---|
| Backend language | Java | **21 LTS** | Long-term support, widest tooling support |
| Backend framework | Spring Boot | **4.0.x** (latest patch at project start) | Current stable generation (Spring Framework 7) |
| Persistence | Spring Data JPA (Hibernate), Flyway | Managed by Spring Boot | Standard, versioned migrations |
| Database | PostgreSQL | **18** | Current stable major version |
| API docs | springdoc-openapi | Version compatible with Boot 4 | Generated OpenAPI/Swagger UI |
| Build | Maven (wrapper) | Managed by wrapper | Reproducible builds |
| Frontend | Angular + Angular Material | **22.x** | Current stable major; Material for consistent UI |
| Frontend runtime | Node.js | Active LTS supported by Angular 22 | Required by Angular CLI |
| E2E tests | Playwright | Latest stable | Cross-browser, reliable |
| Containers | Docker, Docker Compose | Latest stable | One-command local environment |
| CI/CD | GitHub Actions | — | Integrated with the repository |
| Project management | GitHub Projects + Issues | — | Jira-like workflow linked to code ([ADR-0012](adr/0012-github-projects-for-planning.md)) |

Exact versions are pinned in `pom.xml`, `package.json` and `docker-compose.yml`. Upgrades of major versions require an ADR. See [ADR-0013](adr/0013-technology-stack.md).

## 8. Constraints and assumptions

- Team of 3, part-time, university schedule; timeline in [12 Roadmap](12-roadmap.md).
- Single currency (PLN) in the MVP; currency is nevertheless stored on every amount.
- Single bank instance; transfers to other banks are rejected until PKG-08.
- All times are stored in UTC; business days are defined in the `Europe/Warsaw` time zone.
- The UI is in English in the MVP; all texts are kept in one place to allow a Polish translation (PKG-15).

## 9. Project status

Planning and requirements analysis. Next milestone: start of Sprint 1 on 2026-10-26.
