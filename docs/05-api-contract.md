# 05 — API Contract

The contract between frontend and backend. **Frontend develops against this document** (with mocks) before the backend is ready; **backend implements exactly this**.

- After an endpoint is implemented, the generated OpenAPI spec (`/v3/api-docs`, Swagger UI `/swagger-ui.html`) must match this document.
- Any change here needs approval from both developers ([README §3](README.md#3-documentation-rules)).
- Data types and validation of fields come from [03 Domain model](03-domain-model.md).

---

## 1. General conventions

| Topic | Rule |
|---|---|
| Base URL | `/api/v1` |
| Versioning | Breaking changes → `/api/v2`. Non-breaking (new optional fields, new endpoints, new enum values) stay in v1. |
| Format | `application/json; charset=UTF-8`; errors: `application/problem+json` |
| Naming | Resource names plural, kebab-case (`/account-products`); JSON fields camelCase |
| IDs | UUID strings |
| Money | **string** with exactly 2 decimals: `"1250.00"`. Currency in a separate field. Requests accept `"1250"`, `"1250.5"`, `"1250.50"`; more than 2 decimals → validation error. |
| Dates | `"2026-10-24"`; timestamps ISO-8601 UTC `"2026-10-24T13:02:11Z"` |
| Enums | `UPPER_SNAKE_CASE` strings. Clients must tolerate unknown values. |
| Nulls | Optional fields are always present in responses, with `null` when empty (simpler TypeScript types). In requests, optional fields may be omitted or `null`. |
| IBAN | Responses: without spaces. Requests: spaces allowed, normalized by backend. Frontend formats for display. |
| Pagination | Query `page` (0-based, default 0), `size` (default 20, max 100). Response `PageResponse<T>` (§7.1). |
| Created resources | `201 Created` + `Location` header + body. |
| Idempotent money operations | Header `Idempotency-Key: <uuid>` required on `POST /transfers`, `/atm/*`, `/admin/branch/*`. Replay → `200 OK` with original body. |
| Tracing | Every response has header `X-Trace-Id`; the same value is in error bodies and logs. |

### 1.1 Headers

| Header | Direction | When |
|---|---|---|
| `Authorization: Bearer <jwt>` | request | All endpoints except `POST /auth/register`, `POST /auth/login` |
| `Idempotency-Key: <uuid>` | request | Money operations |
| `Content-Type: application/json` | request | Requests with body |
| `X-Trace-Id` | response | Always |
| `Location` | response | 201 responses |

## 2. HTTP status codes

| Code | Meaning in this API |
|---|---|
| 200 | OK (also idempotent replay) |
| 201 | Created |
| 204 | No content (logout) |
| 400 | Malformed JSON, field validation failed, missing idempotency key |
| 401 | Not authenticated / invalid or expired token / invalid credentials |
| 403 | Authenticated but forbidden (wrong role, user blocked/locked at login) |
| 404 | Resource not found **or not owned by the caller** (NFR-SEC-03) |
| 409 | Conflict: duplicate email/PESEL, idempotency key reuse, concurrent modification, invalid state transition |
| 422 | Request is valid but violates a business rule (insufficient funds, policy, limits…) |
| 500 | Unexpected error (never exposes stack traces) |

## 3. Error format

RFC 9457 **Problem Details** with extension fields. Spring's `ProblemDetail` is used.

```json
{
  "type": "https://triobank.example/errors/insufficient-funds",
  "title": "Insufficient funds",
  "status": 422,
  "detail": "The account balance is lower than the requested amount.",
  "instance": "/api/v1/transfers",
  "code": "INSUFFICIENT_FUNDS",
  "traceId": "4f1c2a9be0d34c11",
  "timestamp": "2026-10-24T13:02:11Z",
  "errors": []
}
```

Validation error (400):

```json
{
  "type": "https://triobank.example/errors/validation-error",
  "title": "Validation failed",
  "status": 400,
  "detail": "One or more fields are invalid.",
  "instance": "/api/v1/auth/register",
  "code": "VALIDATION_ERROR",
  "traceId": "9a0b77c1f2e84d02",
  "timestamp": "2026-10-24T13:05:40Z",
  "errors": [
    { "field": "pesel", "code": "INVALID_CHECKSUM", "message": "PESEL check digit is invalid." },
    { "field": "address.postalCode", "code": "INVALID_FORMAT", "message": "Postal code must match NN-NNN." }
  ]
}
```

**Frontend rule:** show messages based on `code` (and `errors[].code`), using its own translation table. `detail`/`message` are for developers and fallback only.

## 4. Error codes

### 4.1 Top-level codes (`code`)

| Code | HTTP | When |
|---|---|---|
| `VALIDATION_ERROR` | 400 | Field validation failed (see `errors[]`) |
| `MALFORMED_REQUEST` | 400 | Invalid JSON, wrong types |
| `IDEMPOTENCY_KEY_MISSING` | 400 | Missing/invalid `Idempotency-Key` |
| `UNAUTHENTICATED` | 401 | No/invalid/expired token |
| `INVALID_CREDENTIALS` | 401 | Wrong email or password |
| `USER_LOCKED` | 403 | Locked after failed logins; body contains `lockedUntil` |
| `USER_BLOCKED` | 403 | Blocked by admin |
| `ACCESS_DENIED` | 403 | Wrong role |
| `RESOURCE_NOT_FOUND` | 404 | Generic: account, transaction, customer, product not found or not owned |
| `EMAIL_ALREADY_REGISTERED` | 409 | BR-01 |
| `PESEL_ALREADY_REGISTERED` | 409 | BR-03 |
| `IDEMPOTENCY_KEY_REUSED` | 409 | BR-18 |
| `CONCURRENT_MODIFICATION` | 409 | Optimistic lock failure — retry |
| `INVALID_STATE_TRANSITION` | 409 | E.g. blocking an already blocked account |
| `CUSTOMER_UNDERAGE` | 422 | BR-04 |
| `PRODUCT_NOT_AVAILABLE` | 422 | Product inactive |
| `ACCOUNT_LIMIT_REACHED` | 422 | BR-07 |
| `PREREQUISITE_ACCOUNT_REQUIRED` | 422 | BR-08 |
| `INSUFFICIENT_FUNDS` | 422 | BR-11 |
| `SOURCE_ACCOUNT_UNAVAILABLE` | 422 | BR-12, source not active |
| `RECIPIENT_ACCOUNT_UNAVAILABLE` | 422 | BR-12, target not active |
| `RECIPIENT_ACCOUNT_NOT_FOUND` | 422 | BR-15 |
| `EXTERNAL_TRANSFER_NOT_SUPPORTED` | 422 | BR-15 |
| `SAME_ACCOUNT_TRANSFER` | 422 | BR-13 |
| `TRANSFER_POLICY_VIOLATION` | 422 | BR-14 |
| `CURRENCY_NOT_SUPPORTED` | 422 | BR-26 |
| `CASH_AMOUNT_INVALID` | 422 | BR-19 |
| `OPERATION_NOT_ALLOWED_FOR_ACCOUNT` | 422 | BR-20 |
| `SELF_ACTION_FORBIDDEN` | 422 | BR-25 |
| `FEATURE_DISABLED` | 404 | Demo-only endpoint called in another profile |
| `INTERNAL_ERROR` | 500 | Unexpected |

### 4.2 Field error codes (`errors[].code`)

`REQUIRED`, `INVALID_FORMAT`, `TOO_SHORT`, `TOO_LONG`, `OUT_OF_RANGE`, `INVALID_CHECKSUM`, `WEAK_PASSWORD`, `TOO_MANY_DECIMALS`, `INVALID_DATE`.

---

## 5. Endpoint overview

| # | Method | Path | Role | Description | Story |
|---|---|---|---|---|---|
| A1 | POST | `/auth/register` | public | Register customer | US-01 |
| A2 | POST | `/auth/login` | public | Log in, get JWT | US-02 |
| A3 | POST | `/auth/logout` | any | Audit logout | US-03 |
| A4 | GET | `/me` | any | Current user + customer profile | US-05 |
| P1 | GET | `/account-products` | CUSTOMER | Product catalog | US-07 |
| C1 | GET | `/accounts` | CUSTOMER | My accounts | US-06 |
| C2 | POST | `/accounts` | CUSTOMER | Open account | US-07 |
| C3 | GET | `/accounts/{accountId}` | CUSTOMER | Account details | US-08 |
| C4 | PATCH | `/accounts/{accountId}` | CUSTOMER | Rename | US-09 |
| C5 | GET | `/accounts/{accountId}/history` | CUSTOMER | Account history | US-12 |
| T1 | POST | `/transfers` | CUSTOMER | Own / internal transfer | US-10, US-11 |
| T2 | GET | `/transactions/{transactionId}` | CUSTOMER | Transaction details | US-13 |
| T3 | POST | `/atm/deposits` | CUSTOMER | ATM deposit | US-14 |
| T4 | POST | `/atm/withdrawals` | CUSTOMER | ATM withdrawal | US-15 |
| D1 | GET | `/admin/customers` | ADMIN | Search customers | US-17 |
| D2 | GET | `/admin/customers/{customerId}` | ADMIN | Customer details | US-17 |
| D3 | POST | `/admin/users/{userId}/block` | ADMIN | Block user | US-18 |
| D4 | POST | `/admin/users/{userId}/unblock` | ADMIN | Unblock user | US-18 |
| D5 | POST | `/admin/users/{userId}/unlock` | ADMIN | Unlock locked user | US-18 |
| D6 | POST | `/admin/accounts/{accountId}/block` | ADMIN | Block account | US-19 |
| D7 | POST | `/admin/accounts/{accountId}/unblock` | ADMIN | Unblock account | US-19 |
| D8 | POST | `/admin/branch/deposits` | ADMIN | Teller deposit | US-20 |
| D9 | POST | `/admin/branch/withdrawals` | ADMIN | Teller withdrawal | US-20 |
| D10 | GET | `/admin/transactions` | ADMIN | Search transactions | US-21 |
| D11 | GET | `/admin/transactions/{transactionId}` | ADMIN | Transaction with entries | US-21 |
| D12 | GET | `/admin/audit-events` | ADMIN | Audit log | US-22 |
| D13 | POST | `/admin/jobs/interest-accrual` | ADMIN | Run accrual (demo) | US-23 |
| D14 | POST | `/admin/jobs/interest-capitalization` | ADMIN | Run capitalization (demo) | US-23 |
| D15 | GET | `/admin/ledger/integrity` | ADMIN | System accounts + invariants | US-24 |

---

## 6. Endpoint details

### A1 `POST /auth/register`

Request `RegisterRequest`:

```json
{
  "email": "anna.nowak@example.com",
  "password": "Str0ng!Passw0rd",
  "firstName": "Anna",
  "lastName": "Nowak",
  "pesel": "90051412343",
  "phoneNumber": "+48501234567",
  "address": {
    "street": "Krakowska",
    "buildingNumber": "12A",
    "apartmentNumber": "4",
    "postalCode": "45-018",
    "city": "Opole",
    "countryCode": null
  }
}
```

Response `201 RegistrationResponse` (Location: `/api/v1/me`):

```json
{
  "userId": "0b6f2a3e-8f44-4f0e-9a51-2c1e5d7b9a10",
  "customerNumber": "10000001",
  "email": "anna.nowak@example.com",
  "account": {
    "id": "7d1e4c22-5b3a-4a8e-b0f1-93c2d4e5f601",
    "iban": "PL45999000121000000000000001",
    "name": "Trio Personal Account",
    "type": "PERSONAL",
    "productCode": "PERSONAL_STANDARD",
    "productName": "Trio Personal Account",
    "currency": "PLN",
    "balance": "0.00",
    "status": "ACTIVE",
    "interestRate": "0.0000",
    "accruedInterest": null,
    "openedAt": "2026-10-24T13:02:11Z"
  }
}
```

Errors: `VALIDATION_ERROR`, `EMAIL_ALREADY_REGISTERED`, `PESEL_ALREADY_REGISTERED`, `CUSTOMER_UNDERAGE`.
After registration the frontend redirects to login (no auto-login in MVP).

### A2 `POST /auth/login`

Request: `{ "email": "anna.nowak@example.com", "password": "Str0ng!Passw0rd" }`

Response `200 LoginResponse`:

```json
{
  "accessToken": "eyJhbGciOiJIUzI1NiJ9...",
  "tokenType": "Bearer",
  "expiresIn": 1800,
  "expiresAt": "2026-10-24T13:32:11Z",
  "user": { "id": "0b6f…", "email": "anna.nowak@example.com", "role": "CUSTOMER", "status": "ACTIVE", "lastLoginAt": "2026-10-23T18:10:00Z", "customer": null }
}
```

`user.customer` is `null` here to keep login light; full profile via A4. `lastLoginAt` is the **previous** login (shown as "Last login" like in real banks).

Errors: `INVALID_CREDENTIALS` (401), `USER_LOCKED` (403, body has extra field `"lockedUntil": "2026-10-24T13:17:11Z"`), `USER_BLOCKED` (403).

### A3 `POST /auth/logout`

No body. Response `204`. Audits `LOGOUT`. The token remains technically valid until expiry (stateless, see [08 §2.4](08-security.md#24-known-limitations)).

### A4 `GET /me`

Response `200 CurrentUserDto` — for customers `customer` contains `CustomerProfileDto` (PESEL masked); for admins `customer` is `null`.

### P1 `GET /account-products`

Response `200 AccountProductDto[]` (active products only).

```json
[
  {
    "id": "c0a8…", "code": "SAVINGS_STANDARD", "name": "Trio Savings Account",
    "description": "Savings account with daily accrual and monthly capitalization.",
    "type": "SAVINGS", "currency": "PLN", "interestRate": "4.0000",
    "outgoingTransferPolicy": "OWN_ACCOUNTS_ONLY", "atmAllowed": false,
    "maxAccountsPerCustomer": 5, "prerequisiteType": "PERSONAL"
  }
]
```

### C1 `GET /accounts`

Response `200 AccountDto[]` — customer's accounts, sorted: personal first, then by `openedAt`. Includes blocked accounts.

### C2 `POST /accounts`

Request `OpenAccountRequest`: `{ "productCode": "SAVINGS_STANDARD", "name": "Holiday savings" }` (`name` optional).
Response `201 AccountDto`. Errors: `RESOURCE_NOT_FOUND` (product), `PRODUCT_NOT_AVAILABLE`, `ACCOUNT_LIMIT_REACHED`, `PREREQUISITE_ACCOUNT_REQUIRED`, `VALIDATION_ERROR`.

### C3 `GET /accounts/{accountId}`

Response `200 AccountDto`. Errors: `RESOURCE_NOT_FOUND`.

### C4 `PATCH /accounts/{accountId}`

Request `RenameAccountRequest`: `{ "name": "Daily expenses" }`. Response `200 AccountDto`.

### C5 `GET /accounts/{accountId}/history`

Query parameters:

| Param | Type | Default | Notes |
|---|---|---|---|
| `from` | date | — | Inclusive, Warsaw date |
| `to` | date | — | Inclusive, Warsaw date; `from ≤ to` |
| `direction` | `INCOMING` \| `OUTGOING` | both | INCOMING = CREDIT, OUTGOING = DEBIT |
| `type` | TransactionType (repeatable) | all | `?type=OWN_TRANSFER&type=INTERNAL_TRANSFER` |
| `page`, `size` | int | 0, 20 | |

Response `200 PageResponse<AccountHistoryItemDto>`:

```json
{
  "content": [
    {
      "entryId": "e1…", "transactionId": "t1…", "reference": "TB202610240000000123",
      "type": "INTERNAL_TRANSFER", "status": "COMPLETED", "channel": "WEB",
      "direction": "OUTGOING", "amount": "150.00", "currency": "PLN",
      "balanceAfter": "850.00", "title": "Dinner", "counterpartyName": "Jan Kowalski",
      "counterpartyIban": "PL18999000121000000000000002",
      "createdAt": "2026-10-24T13:02:11Z", "bookedAt": "2026-10-24T13:02:11Z"
    }
  ],
  "page": 0, "size": 20, "totalElements": 1, "totalPages": 1
}
```

For operations with system accounts (cash, interest, tax) `counterpartyName` = `"Trio Bank"` and `counterpartyIban` = `null`.

### T1 `POST /transfers`

Headers: `Idempotency-Key`. Request `TransferRequest`:

```json
{
  "sourceAccountId": "7d1e4c22-5b3a-4a8e-b0f1-93c2d4e5f601",
  "targetIban": "PL18 9990 0012 1000 0000 0000 0002",
  "recipientName": "Jan Kowalski",
  "title": null,
  "amount": "150.00",
  "currency": null
}
```

Response `201 TransactionDto` (or `200` on replay):

```json
{
  "id": "t1…",
  "reference": "TB202610240000000123",
  "type": "INTERNAL_TRANSFER",
  "status": "COMPLETED",
  "channel": "WEB",
  "amount": "150.00",
  "currency": "PLN",
  "title": "Funds transfer",
  "sender":    { "name": "Anna Nowak",   "iban": "PL45999000121000000000000001", "ownAccount": true },
  "recipient": { "name": "Jan Kowalski", "iban": "PL18999000121000000000000002", "ownAccount": false },
  "createdAt": "2026-10-24T13:02:11Z",
  "bookedAt": "2026-10-24T13:02:11Z"
}
```

Errors: `VALIDATION_ERROR`, `IDEMPOTENCY_KEY_MISSING`, `IDEMPOTENCY_KEY_REUSED`, `RESOURCE_NOT_FOUND` (source not owned), `SOURCE_ACCOUNT_UNAVAILABLE`, `RECIPIENT_ACCOUNT_UNAVAILABLE`, `RECIPIENT_ACCOUNT_NOT_FOUND`, `EXTERNAL_TRANSFER_NOT_SUPPORTED`, `SAME_ACCOUNT_TRANSFER`, `TRANSFER_POLICY_VIOLATION`, `INSUFFICIENT_FUNDS`, `CURRENCY_NOT_SUPPORTED`.

Validation order (backend; the first failing rule is returned):
1. Field validation (format, IBAN checksum, amount) → 400
2. Source account exists and belongs to caller → 404
3. Currency supported → 422
4. Target IBAN bank code is Trio Bank → `EXTERNAL_TRANSFER_NOT_SUPPORTED`
5. Target account exists → `RECIPIENT_ACCOUNT_NOT_FOUND`
6. Source ≠ target → `SAME_ACCOUNT_TRANSFER`
7. Resolve type (own/internal); recipient name required for internal → 400
8. Source policy (`OWN_ACCOUNTS_ONLY`) → `TRANSFER_POLICY_VIOLATION`
9. In ledger: statuses → `*_UNAVAILABLE`; balance → `INSUFFICIENT_FUNDS`

### T2 `GET /transactions/{transactionId}`

Response `200 TransactionDto`. Only if the source or target account belongs to the caller; otherwise `404`.

### T3 `POST /atm/deposits` · T4 `POST /atm/withdrawals`

Headers: `Idempotency-Key`. Request `CashOperationRequest`: `{ "accountId": "7d1e…", "amount": "200.00", "currency": null }`.
Response `201 TransactionDto`. Errors: `VALIDATION_ERROR`, `RESOURCE_NOT_FOUND`, `CASH_AMOUNT_INVALID`, `OPERATION_NOT_ALLOWED_FOR_ACCOUNT`, `SOURCE_ACCOUNT_UNAVAILABLE`/`RECIPIENT_ACCOUNT_UNAVAILABLE`, `INSUFFICIENT_FUNDS` (withdrawal), idempotency errors.

### D1 `GET /admin/customers`

Query: `query` (matches first/last name, email, customer number, PESEL; case-insensitive, min 2 chars), `userStatus`, `page`, `size`.
Response `200 PageResponse<AdminCustomerSummaryDto>`.

### D2 `GET /admin/customers/{customerId}`

Response `200 AdminCustomerDetailsDto` (full PESEL, user status, accounts).

### D3–D7 status changes

Request (block only) `StatusChangeRequest`: `{ "reason": "Suspicious activity reported by customer" }` (5–255 chars). Unblock/unlock: optional `reason`.
Response `200 AdminUserDto` / `AdminAccountDto`.
Errors: `RESOURCE_NOT_FOUND`, `INVALID_STATE_TRANSITION`, `SELF_ACTION_FORBIDDEN`.

### D8 `POST /admin/branch/deposits` · D9 `POST /admin/branch/withdrawals`

Headers: `Idempotency-Key`. Request `BranchCashRequest`: `{ "iban": "PL45…", "amount": "1000.00", "title": null, "currency": null }`.
Response `201 AdminTransactionDto`.

### D10 `GET /admin/transactions`

Query: `from`, `to`, `type` (repeatable), `status`, `channel`, `iban`, `reference`, `page`, `size`. Response `200 PageResponse<AdminTransactionDto>` (without `entries` in list — `entries: []`).

### D11 `GET /admin/transactions/{transactionId}`

Response `200 AdminTransactionDto` with both ledger entries.

### D12 `GET /admin/audit-events`

Query: `from`, `to`, `action` (repeatable), `actorEmail`, `targetId`, `page`, `size`. Response `200 PageResponse<AuditEventDto>`, newest first.

### D13 `POST /admin/jobs/interest-accrual?date=2026-10-23` · D14 `POST /admin/jobs/interest-capitalization?month=2026-10`

Only in profiles `local`, `demo`; elsewhere `404 FEATURE_DISABLED`. Response `200 JobRunResponse`.

### D15 `GET /admin/ledger/integrity`

Response `200 LedgerIntegrityDto`.

---

## 7. DTO catalog

Written in TypeScript notation (frontend copies these into `core/api/models`). Backend implements them as Java `record`s with the same names. `Money = string`, `Uuid = string`, `IsoDate = string`, `IsoDateTime = string`.

### 7.1 Common

```ts
type Money = string;        // "1250.00"
type Uuid = string;
type IsoDate = string;      // "2026-10-24"
type IsoDateTime = string;  // "2026-10-24T13:02:11Z"

interface PageResponse<T> {
  content: T[];
  page: number;
  size: number;
  totalElements: number;
  totalPages: number;
}

interface ProblemDetail {
  type: string; title: string; status: number; detail: string; instance: string;
  code: string; traceId: string; timestamp: IsoDateTime;
  errors: FieldError[];
  lockedUntil?: IsoDateTime;           // only for USER_LOCKED
}
interface FieldError { field: string; code: string; message: string; }
```

### 7.2 Enums

```ts
type Role = 'CUSTOMER' | 'ADMIN';
type UserStatus = 'PENDING_VERIFICATION' | 'ACTIVE' | 'LOCKED' | 'BLOCKED' | 'CLOSED';
type AccountType = 'PERSONAL' | 'SAVINGS';
type AccountStatus = 'ACTIVE' | 'BLOCKED' | 'CLOSED';
type OutgoingTransferPolicy = 'ANY' | 'OWN_ACCOUNTS_ONLY';
type TransactionType = 'OWN_TRANSFER' | 'INTERNAL_TRANSFER' | 'CASH_DEPOSIT' | 'CASH_WITHDRAWAL'
                     | 'INTEREST' | 'TAX' | string;          // tolerate future values
type TransactionStatus = 'PENDING' | 'COMPLETED' | 'REJECTED' | 'CANCELLED';
type Channel = 'WEB' | 'ATM' | 'BRANCH' | 'SYSTEM' | string;
type Direction = 'INCOMING' | 'OUTGOING';
type EntryDirection = 'DEBIT' | 'CREDIT';
type SystemAccountCode = 'BANK_CASH' | 'BANK_INTEREST_EXPENSE' | 'BANK_TAX_LIABILITY' | 'BANK_FEE_INCOME';
```

### 7.3 Auth & customer

```ts
interface AddressDto {
  street: string; buildingNumber: string; apartmentNumber: string | null;
  postalCode: string; city: string; countryCode: string | null;   // response: always set
}

interface RegisterRequest {
  email: string; password: string; firstName: string; lastName: string;
  pesel: string; phoneNumber: string; address: AddressDto;
}

interface RegistrationResponse { userId: Uuid; customerNumber: string; email: string; account: AccountDto; }

interface LoginRequest { email: string; password: string; }

interface LoginResponse {
  accessToken: string; tokenType: 'Bearer'; expiresIn: number; expiresAt: IsoDateTime;
  user: CurrentUserDto;
}

interface CurrentUserDto {
  id: Uuid; email: string; role: Role; status: UserStatus;
  lastLoginAt: IsoDateTime | null;
  customer: CustomerProfileDto | null;
}

interface CustomerProfileDto {
  id: Uuid; customerNumber: string; firstName: string; lastName: string;
  peselMasked: string;               // "*******2343"
  dateOfBirth: IsoDate; email: string; phoneNumber: string;
  address: AddressDto; createdAt: IsoDateTime;
}
```

### 7.4 Products & accounts

```ts
interface AccountProductDto {
  id: Uuid; code: string; name: string; description: string | null;
  type: AccountType; currency: string; interestRate: string;
  outgoingTransferPolicy: OutgoingTransferPolicy; atmAllowed: boolean;
  maxAccountsPerCustomer: number; prerequisiteType: AccountType | null;
}

interface AccountDto {
  id: Uuid; iban: string; name: string; type: AccountType;
  productCode: string; productName: string; currency: string;
  balance: Money; status: AccountStatus;
  interestRate: string; accruedInterest: string | null;   // null for non-savings; 6 decimals
  openedAt: IsoDateTime;
}

interface OpenAccountRequest { productCode: string; name?: string | null; }
interface RenameAccountRequest { name: string; }

interface AccountHistoryItemDto {
  entryId: Uuid; transactionId: Uuid; reference: string;
  type: TransactionType; status: TransactionStatus; channel: Channel;
  direction: Direction; amount: Money; currency: string; balanceAfter: Money;
  title: string; counterpartyName: string; counterpartyIban: string | null;
  createdAt: IsoDateTime; bookedAt: IsoDateTime | null;
}
```

### 7.5 Payments & transactions

```ts
interface TransferRequest {
  sourceAccountId: Uuid; targetIban: string;
  recipientName?: string | null; title?: string | null;
  amount: Money; currency?: string | null;
}

interface CashOperationRequest { accountId: Uuid; amount: Money; currency?: string | null; }

interface PartyDto { name: string; iban: string | null; ownAccount: boolean; }

interface TransactionDto {
  id: Uuid; reference: string; type: TransactionType; status: TransactionStatus;
  channel: Channel; amount: Money; currency: string; title: string;
  sender: PartyDto; recipient: PartyDto;
  createdAt: IsoDateTime; bookedAt: IsoDateTime | null;
}
```

### 7.6 Admin

```ts
interface AdminCustomerSummaryDto {
  customerId: Uuid; userId: Uuid; customerNumber: string;
  firstName: string; lastName: string; email: string;
  userStatus: UserStatus; accountsCount: number; createdAt: IsoDateTime;
}

interface AdminUserDto {
  id: Uuid; email: string; role: Role; status: UserStatus; statusReason: string | null;
  failedLoginAttempts: number; lockedUntil: IsoDateTime | null;
  lastLoginAt: IsoDateTime | null; createdAt: IsoDateTime;
}

interface AdminAccountDto extends AccountDto { customerId: Uuid; statusReason: string | null; }

interface AdminCustomerDetailsDto {
  customer: CustomerProfileDto & { pesel: string };    // full PESEL for admin
  user: AdminUserDto;
  accounts: AdminAccountDto[];
}

interface StatusChangeRequest { reason?: string | null; }   // required (5–255) for block

interface BranchCashRequest { iban: string; amount: Money; title?: string | null; currency?: string | null; }

interface LedgerEntryDto {
  id: Uuid; accountId: Uuid; iban: string; direction: EntryDirection;
  amount: Money; balanceAfter: Money;
}

interface AdminPartyDto extends PartyDto { accountId: Uuid; systemCode: SystemAccountCode | null; }

interface AdminTransactionDto extends Omit<TransactionDto, 'sender' | 'recipient'> {
  sender: AdminPartyDto; recipient: AdminPartyDto;
  initiatedBy: { userId: Uuid; email: string; role: Role } | null;
  entries: LedgerEntryDto[];
}

interface AuditEventDto {
  id: Uuid; occurredAt: IsoDateTime; actorUserId: Uuid | null; actorEmail: string | null;
  actorRole: string | null; action: string; targetType: string | null; targetId: Uuid | null;
  ipAddress: string | null; details: Record<string, unknown> | null;
}

interface SystemAccountDto { id: Uuid; systemCode: SystemAccountCode; iban: string; name: string; balance: Money; currency: string; }

interface LedgerIntegrityDto {
  checkedAt: IsoDateTime; ok: boolean;
  totalBalance: Money;                 // must be "0.00"
  balancesMatchEntries: boolean; entriesBalanced: boolean;
  systemAccounts: SystemAccountDto[];
}

interface JobRunResponse {
  job: 'INTEREST_ACCRUAL' | 'INTEREST_CAPITALIZATION'; parameter: string;
  processedAccounts: number; skippedAccounts: number; transactionsCreated: number;
  startedAt: IsoDateTime; finishedAt: IsoDateTime;
}
```

---

## 8. Mocking for frontend

Until an endpoint is implemented, the frontend uses an HTTP interceptor (`MockApiInterceptor`, enabled by `environment.useMocks`) returning the example payloads from this document. Mock data lives in `src/app/core/api/mocks/`. When the backend endpoint is merged into `develop`, the mock is removed in the same sprint.
