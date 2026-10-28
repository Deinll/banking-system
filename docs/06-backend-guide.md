# 06 — Backend Guide

Rules for the Spring Boot application in `/backend`. Architecture is described in [04](04-architecture.md); data in [03](03-domain-model.md); API in [05](05-api-contract.md).

## 1. Project setup

| Item | Value |
|---|---|
| Group / artifact | `com.triobank` / `trio-bank-api` |
| Java | 21 |
| Spring Boot | 4.0.x |
| Build | Maven wrapper (`./mvnw`) |
| Starters | web, data-jpa, security, validation, actuator, flyway (+ `flyway-database-postgresql`), postgresql driver |
| Other | springdoc-openapi (Boot 4 compatible), jjwt or Spring Security OAuth2 resource server (JWT), MapStruct (optional), Testcontainers, JaCoCo |

## 2. Package structure

```
com.triobank
├── TrioBankApplication.java
├── common
│   ├── config          # Jackson (BigDecimal as string), Clock, OpenAPI, CORS
│   ├── error           # ApiException, ErrorCode, GlobalExceptionHandler
│   ├── money           # Money utils, rounding constants
│   ├── iban            # IbanGenerator, IbanValidator
│   ├── pesel           # PeselValidator, PeselParser
│   ├── persistence     # BaseEntity (id, createdAt, updatedAt, version)
│   └── web             # PageResponse, TraceIdFilter, @CurrentUser
├── audit
├── auth
├── customer
├── product
├── account
├── ledger
├── payment
├── interest
└── admin
```

Inside each feature module:

```
account
├── AccountController.java
├── AccountService.java
├── AccountRepository.java
├── Account.java                 # entity
├── AccountStatus.java           # enum
└── dto
    ├── AccountDto.java          # record
    ├── OpenAccountRequest.java
    └── AccountMapper.java
```

Small modules stay flat; split into sub-packages (`web`, `domain`, `persistence`) only if a module grows beyond ~15 classes.

## 3. Coding rules

### 3.1 General

- Constructor injection only (`private final` fields, Lombok `@RequiredArgsConstructor` allowed).
- DTOs are Java `record`s. **Entities are never returned from controllers.**
- No business logic in controllers or repositories.
- `Optional` only as a return type, never as a field or parameter.
- No `System.out`; use SLF4J. Never log passwords, tokens, full PESEL, full request bodies of `/auth/*`.
- Use `Clock` (injected bean) instead of `Instant.now()` / `LocalDate.now()` — makes time testable.
- Constants for business values (`BusinessRules.ATM_MAX_WITHDRAWAL`) — no magic numbers.

### 3.2 Money

```java
public final class MoneyConstants {
    public static final int SCALE = 2;
    public static final int ACCRUAL_SCALE = 6;
    public static final RoundingMode ROUNDING = RoundingMode.HALF_EVEN;
}
```

- Always `BigDecimal`, created from `String` (`new BigDecimal("10.00")`), never from `double`.
- Compare with `compareTo`, not `equals` (`10.0` vs `10.00`).
- JPA columns: `@Column(precision = 19, scale = 2)`.
- Jackson: serialize `BigDecimal` as string with 2 decimals (custom serializer for money DTO fields, or `@JsonFormat(shape = STRING)`).

### 3.3 Entities

```java
@MappedSuperclass
@EntityListeners(AuditingEntityListener.class)
public abstract class BaseEntity {
    @Id private UUID id = UUID.randomUUID();
    @CreatedDate @Column(nullable = false, updatable = false) private Instant createdAt;
    @LastModifiedDate @Column(nullable = false) private Instant updatedAt;
    @Version private long version;
}
```

- `@Enumerated(EnumType.STRING)` for all enums.
- Associations `LAZY` by default; fetch explicitly with `JOIN FETCH` or entity graphs where needed.
- `equals/hashCode` based on `id` only.
- Domain methods for state changes, e.g. `account.block(reason)`, which throw `ApiException(INVALID_STATE_TRANSITION)` on invalid transitions.
- Immutable entities (`Transaction`, `LedgerEntry`, `AuditEvent`): no setters, `@Immutable` (Hibernate), created through constructors/factory methods.

### 3.4 Validation

| Level | Where | Example |
|---|---|---|
| Format | DTO annotations: `@NotBlank`, `@Size`, `@Email`, `@Pattern`, custom `@ValidIban`, `@ValidPesel`, `@ValidMoney` | IBAN checksum, max 2 decimals |
| Business | Service methods → `ApiException(ErrorCode)` | Insufficient funds, limits, ownership |
| Last line of defence | DB constraints (`CHECK`, `UNIQUE`, `NOT NULL`, FK) | `balance >= 0` |

Custom validators live in `common` and are reused by both DTO validation and services.

### 3.5 Errors

```java
public enum ErrorCode {
    VALIDATION_ERROR(HttpStatus.BAD_REQUEST, "Validation failed"),
    INSUFFICIENT_FUNDS(HttpStatus.UNPROCESSABLE_ENTITY, "Insufficient funds"),
    // … one constant per code in 05 §4
    ;
    final HttpStatus status; final String title;
}

public class ApiException extends RuntimeException {
    private final ErrorCode code;
    private final Map<String, Object> properties; // e.g. lockedUntil
}
```

`GlobalExceptionHandler` maps `ApiException`, `MethodArgumentNotValidException`, `HttpMessageNotReadableException`, `OptimisticLockingFailureException` (→ `CONCURRENT_MODIFICATION`), `DataIntegrityViolationException` (→ mapped by constraint name), `AccessDeniedException`, and `Exception` (→ `INTERNAL_ERROR`, logged with stack trace) to `ProblemDetail` as in [05 §3](05-api-contract.md#3-error-format).

**Not found vs not owned:** repositories for customer-facing reads query by `id AND customer_id`, so a foreign resource is simply "not found" (404).

### 3.6 Transactions

- `@Transactional` on **service** methods (public use cases), never on controllers or repositories.
- Read-only use cases: `@Transactional(readOnly = true)`.
- `LedgerService.post()` runs in the caller's transaction (`Propagation.REQUIRED`); `TransferService.transfer()` is the transactional boundary.
- Account locking: repository method

```java
@Lock(LockModeType.PESSIMISTIC_WRITE)
@Query("select a from Account a where a.id in :ids order by a.id")
List<Account> findAllForUpdate(@Param("ids") Collection<UUID> ids);
```

- Interest jobs process each account in **its own transaction** (one failing account does not roll back others).

### 3.7 Current user

A `@CurrentUser AuthenticatedUser user` argument resolver provides `userId`, `role` and `customerId` (null for admins) from the JWT. Services receive IDs, not security objects.

## 4. Database & migrations

- Flyway location `classpath:db/migration`; demo data in `classpath:db/demo` (enabled only in `local`/`demo` profiles).
- File naming: `V{3-digit number}__{snake_case_description}.sql`, e.g. `V001__create_users_and_customers.sql`. Numbers are assigned in PR order; on conflict, the later PR renumbers.
- **Never edit a migration that is merged into `develop`.** Fix with a new migration.
- `spring.jpa.hibernate.ddl-auto=validate` — Hibernate checks entities against the schema; it never changes it.
- Constraint names are explicit (`uk_users_email`, `ck_accounts_balance_non_negative`) so errors can be mapped.

Initial migration plan:

| Migration | Content | Sprint |
|---|---|---|
| V001 | `users`, `customers`, sequences `customer_number_seq` | 1 |
| V002 | `account_products` + seed | 1 |
| V003 | `accounts`, `account_number_seq`, system accounts seed | 1 |
| V004 | `audit_events` | 1 |
| V005 | `transactions`, `ledger_entries`, `transaction_reference_seq` | 2 |

## 5. Configuration

`application.yml` (shared) + `application-{profile}.yml`. All secrets via environment variables:

```yaml
spring:
  datasource:
    url: ${DB_URL:jdbc:postgresql://localhost:5432/triobank}
    username: ${DB_USER:triobank}
    password: ${DB_PASSWORD:triobank}
  jpa:
    hibernate.ddl-auto: validate
    open-in-view: false
bank:
  sort-code: "99900012"
  currency: PLN
  time-zone: Europe/Warsaw
security:
  jwt:
    secret: ${JWT_SECRET}            # required, ≥ 32 bytes
    ttl: PT30M
  lockout:
    max-attempts: 5
    duration: PT15M
  admin-bootstrap:
    email: ${ADMIN_EMAIL:}
    password: ${ADMIN_PASSWORD:}
cors:
  allowed-origins: ${CORS_ALLOWED_ORIGINS:http://localhost:4200}
features:
  demo-jobs-endpoint: false          # true in local/demo
```

## 6. Scheduled jobs

| Job | Cron (Europe/Warsaw) | Action |
|---|---|---|
| `InterestAccrualJob` | `0 5 0 * * *` | Accrue interest for yesterday for all active savings accounts (skip if `lastAccrualDate >= day`) |
| `InterestCapitalizationJob` | `0 30 0 1 * *` | Capitalize previous month (runs after accrual for the last day) |

Both jobs call the same services as the demo endpoints D13/D14. Enable scheduling with `@EnableScheduling`; disabled in `test` profile.

## 7. API documentation

- Annotate controllers with `@Tag`, operations with `@Operation`, error responses with `@ApiResponse`.
- Swagger UI enabled in `local` and `demo`, disabled otherwise.
- The QA/DevOps member can export `/v3/api-docs` to `docs/openapi.json` in CI as an artifact.

## 8. Checklist for a new endpoint

1. Endpoint and DTOs exist in [05](05-api-contract.md) (or a docs PR is opened first).
2. Request DTO with validation annotations.
3. Service method with business rules, `@Transactional`, audit event if relevant.
4. Controller method, correct status code, `Location` for 201.
5. Error codes mapped; ownership enforced.
6. Unit tests for the service, integration test (MockMvc + Testcontainers) for the endpoint.
7. OpenAPI annotations.
