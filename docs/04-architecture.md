# 04 — Architecture

## 1. System context

```mermaid
flowchart LR
    C([Customer]) -->|HTTPS| FE
    A([Admin]) -->|HTTPS| FE
    subgraph Trio Bank
        FE[Angular SPA<br/>static files served by Nginx]
        BE[Spring Boot API<br/>/api/v1]
        DB[(PostgreSQL 18)]
        FE -->|REST + JSON, JWT| BE
        BE -->|JDBC, Flyway| DB
        BE -.->|scheduled jobs| BE
    end
```

- **Frontend:** single-page application. Talks to the backend only through the REST API defined in [05](05-api-contract.md).
- **Backend:** stateless REST API (no server sessions). All business rules live here.
- **Database:** PostgreSQL. Schema owned by Flyway migrations in the backend.
- **Scheduled jobs:** run inside the backend (Spring `@Scheduled`) — interest accrual and capitalization.

Future external systems (Elixir clearing, BLIK, mail server) are integrated as backend modules behind interfaces; the frontend never talks to them directly.

## 2. Architectural style

**Modular monolith, package-by-feature** ([ADR-0002](adr/0002-modular-monolith.md)): one deployable backend, divided into feature modules with clear boundaries. This keeps deployment simple for a 3-person team while allowing a module to be extracted later.

```mermaid
flowchart TB
    subgraph web [API layer]
        AUTHc[auth] --- CUSc[customer] --- ACCc[account] --- PAYc[payment] --- ADMc[admin]
    end
    subgraph core [Domain core]
        LED[ledger<br/>LedgerService.post]
        PROD[product]
        INT[interest]
        AUD[audit]
    end
    PAYc --> LED
    INT --> LED
    ADMc --> LED
    ACCc --> PROD
    AUTHc --> AUD
    PAYc --> AUD
    ADMc --> AUD
    LED --> AUD
```

### 2.1 Backend modules

| Module | Responsibility | Depends on |
|---|---|---|
| `common` | Error handling, money/IBAN/PESEL utilities, clock, base entity, pagination | — |
| `audit` | Writing and querying audit events | common |
| `auth` | Registration, login, JWT, lockout, `/me`, admin bootstrap | customer, account, audit |
| `customer` | Customer profile | common |
| `product` | Account product catalog | common |
| `account` | Opening, listing, renaming accounts; IBAN generation; history queries | product, customer, ledger (read) |
| `ledger` | **Core:** posting transactions with double entry, locking, invariants | audit |
| `payment` | Transfers (own/internal), ATM operations, idempotency, type resolution | account, ledger |
| `interest` | Accrual and capitalization jobs | account, ledger |
| `admin` | Admin use cases: customers, blocking, branch cash, transactions, audit, jobs | all above |

Rule: a module uses another module only through its **service** (public API), never its repositories directly.

### 2.2 Layers inside a module

```
controller  →  service  →  repository  →  database
   ↑ DTOs        ↑ entities, domain logic
```

| Layer | Contains | Must not |
|---|---|---|
| Controller | REST endpoints, request validation (`@Valid`), mapping DTO ↔ service call | Contain business logic, touch repositories, return entities |
| Service | Use cases, business rules, `@Transactional` boundaries | Know about HTTP |
| Repository | Spring Data JPA interfaces, queries | Contain business logic |
| Entity / domain | JPA entities, enums, small domain methods (e.g. `account.block(reason)`) | Be serialized to JSON |

## 3. The ledger core

Every money movement in the system goes through **one method**:

```java
Transaction LedgerService.post(PostingCommand command)
```

```java
record PostingCommand(
    TransactionType type,
    Channel channel,
    UUID sourceAccountId,
    UUID targetAccountId,
    BigDecimal amount,
    String currency,
    String title,
    String senderName,
    String recipientName,
    UUID initiatedByUserId,     // null for SYSTEM
    String idempotencyKey,      // null for SYSTEM
    String requestHash
) {}
```

What `post()` does, in **one database transaction**:

1. Lock both accounts with `SELECT … FOR UPDATE`, **always in ascending order of account id** (prevents deadlocks when A→B and B→A run concurrently).
2. Check generic invariants: both accounts `ACTIVE`, same currency, amount > 0, source ≠ target, a customer source account has enough balance.
3. Decrease source balance, increase target balance.
4. Insert the `Transaction` (status `COMPLETED`, reference from sequence).
5. Insert two `LedgerEntry` rows (DEBIT on source, CREDIT on target) with `balanceAfter`.
6. Write audit event `TRANSACTION_POSTED`.

`post()` knows **nothing** about transfers, ATMs or interest. Operation-specific rules are checked by the caller before calling it.

### 3.1 Operations on top of the core (Strategy-like)

```mermaid
flowchart LR
    T[TransferService<br/>own / internal] --> L[LedgerService.post]
    ATM[AtmService<br/>deposit / withdraw] --> L
    BR[BranchCashService<br/>admin teller] --> L
    I[InterestService<br/>capitalize + tax] --> L
    F1[PhoneTransferService<br/>PKG-10]:::future --> T
    F2[BlikService<br/>PKG-11]:::future --> L
    F3[ElixirClearingService<br/>PKG-08]:::future --> L
    F4[StandingOrderJob<br/>PKG-07]:::future --> T
    classDef future stroke-dasharray: 5 5
```

| Operation | Resolves source / target | Specific rules | Type |
|---|---|---|---|
| Transfer | source = chosen own account; target = account by IBAN | BR-13…17, idempotency | `OWN_TRANSFER` / `INTERNAL_TRANSFER` |
| ATM deposit | `BANK_CASH` → own personal account | BR-19, BR-20 | `CASH_DEPOSIT` |
| ATM withdrawal | own personal account → `BANK_CASH` | BR-19, BR-20 | `CASH_WITHDRAWAL` |
| Branch cash | `BANK_CASH` ↔ any customer account | BR-19 (branch limits) | `CASH_DEPOSIT` / `CASH_WITHDRAWAL` |
| Interest | `BANK_INTEREST_EXPENSE` → savings; savings → `BANK_TAX_LIABILITY` | BR-21 | `INTEREST`, `TAX` |

**Adding a new payment method** = a new service that finds the two accounts and validates its own rules, then calls `post()`. No changes to the core, database constraints or history screens.

### 3.2 Transfer sequence

```mermaid
sequenceDiagram
    actor U as Customer
    participant FE as Angular
    participant API as TransferController
    participant S as TransferService
    participant L as LedgerService
    participant DB as PostgreSQL

    U->>FE: fill form, confirm
    FE->>API: POST /api/v1/transfers (Idempotency-Key)
    API->>S: transfer(userId, request)
    S->>DB: find existing by (user, idempotencyKey)
    alt key already used
        S-->>API: original transaction (200) or IDEMPOTENCY_KEY_REUSED (409)
    else new request
        S->>S: validate IBAN, ownership, policy, resolve type, default title
        S->>L: post(PostingCommand)
        L->>DB: SELECT accounts FOR UPDATE (ordered by id)
        L->>L: check status, currency, funds
        L->>DB: UPDATE balances, INSERT transaction + 2 entries + audit
        L-->>S: Transaction
        S-->>API: TransactionDto
        API-->>FE: 201 Created
    end
    FE-->>U: success screen with reference
```

### 3.3 Idempotency

- Client generates a UUID `Idempotency-Key` **when the confirmation screen opens** and reuses it for retries of the same confirmation.
- Backend looks up `(initiated_by_user_id, idempotency_key)`:
  - not found → execute;
  - found with same `requestHash` → return the stored transaction with **200**;
  - found with different hash → **409** `IDEMPOTENCY_KEY_REUSED`.
- Two simultaneous requests with the same key: the unique constraint makes one fail; the service catches the violation and returns the stored transaction.

## 4. Cross-cutting concerns

| Concern | Approach | Details |
|---|---|---|
| Authentication | Stateless JWT (Spring Security resource server style filter) | [08](08-security.md) |
| Authorization | Role checks on URL patterns + ownership checks in services | [08](08-security.md) |
| Validation | Bean Validation on DTOs (format), services for business rules | [06](06-backend-guide.md) |
| Errors | Global `@RestControllerAdvice` → RFC 9457 Problem Details | [05 §3](05-api-contract.md#3-error-format) |
| Time | Injected `java.time.Clock` (UTC); `ZoneId Europe/Warsaw` for business dates | Testable jobs |
| Transactions | `@Transactional` on service methods; ledger posting is a single transaction | [06](06-backend-guide.md) |
| Concurrency | Pessimistic lock for money, optimistic `@Version` for other updates | [ADR-0009](adr/0009-locking-and-idempotency.md) |
| Logging | SLF4J, JSON in demo env, `traceId` in MDC and in error responses | NFR-OBS-01 |
| API docs | springdoc-openapi, Swagger UI at `/swagger-ui.html` (local/demo only) | [05](05-api-contract.md) |
| Scheduling | `@Scheduled` with cron in `Europe/Warsaw`; jobs idempotent | BR-21 |
| Configuration | Spring profiles `local`, `test`, `demo`; secrets from env vars | [10](10-devops.md) |

## 5. Frontend architecture (summary)

Standalone Angular components, feature folders, services with signals for state, functional HTTP interceptors and guards. Details in [07](07-frontend-guide.md).

```
core/      auth, interceptors, guards, error mapping, session timer
shared/    UI components, pipes (money, iban), validators (IBAN, PESEL)
features/  auth, dashboard, accounts, payments, atm, history, profile, admin
```

## 6. Deployment view

```mermaid
flowchart LR
    subgraph Local [docker compose - local]
        N1[frontend :4200 ng serve<br/>or nginx :80] --> B1[backend :8080] --> P1[(postgres :5432)]
    end
    subgraph Demo [demo environment]
        N2[frontend container / static hosting] --> B2[backend container] --> P2[(managed PostgreSQL)]
    end
    GH[GitHub Actions] -->|build images, deploy from main| Demo
```

Hosting provider is decided in Sprint 1 ([10 §6](10-devops.md#6-environments)).

## 7. Quality attributes → design decisions

| Attribute | Decision |
|---|---|
| Correctness of money | Double-entry ledger, DB `CHECK` constraints, single posting method, invariant tests |
| Concurrency safety | Ordered pessimistic locks, idempotency keys |
| Security | BCrypt, JWT, 404 for foreign resources, audit log, lockout |
| Extensibility | Product catalog, reserved enum values, strategy services on the ledger core, additive schema evolution, API versioning |
| Testability | Injected clock, Testcontainers, layered services |
| Simplicity | Modular monolith, one repo, docker compose |
