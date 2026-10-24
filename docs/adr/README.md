# Architecture Decision Records

An ADR records one significant decision: the context, the decision and its consequences. ADRs are never edited after acceptance — a changed decision gets a new ADR that **supersedes** the old one.

Statuses: `Proposed` → `Accepted` → (`Superseded by ADR-xxxx` | `Deprecated`).

| ADR | Title | Status |
|---|---|---|
| [0001](0001-record-architecture-decisions.md) | Record architecture decisions; documentation as code | Accepted |
| [0002](0002-modular-monolith.md) | Modular monolith, package-by-feature | Accepted |
| [0003](0003-uuid-identifiers.md) | UUID identifiers | Accepted |
| [0004](0004-money-representation.md) | Money representation | Accepted |
| [0005](0005-double-entry-ledger.md) | Double-entry ledger with system accounts | Accepted |
| [0006](0006-account-product-catalog.md) | Account product catalog | Accepted |
| [0007](0007-polish-iban-format.md) | Polish IBAN with fictional sort code | Accepted |
| [0008](0008-stateless-jwt.md) | Stateless JWT authentication, no refresh token in MVP | Accepted |
| [0009](0009-locking-and-idempotency.md) | Pessimistic locking and idempotency keys for money operations | Accepted |
| [0010](0010-problem-details-errors.md) | RFC 9457 Problem Details with stable error codes | Accepted |
| [0011](0011-backend-owned-defaults.md) | Business defaults owned by the backend | Accepted |
| [0012](0012-github-projects-for-planning.md) | GitHub Projects instead of Trello | Accepted |
| [0013](0013-technology-stack.md) | Technology stack and versions | Accepted |
| 0014 | Demo hosting provider | To be written in Sprint 1 |

New ADR: copy [template.md](template.md), next free number, kebab-case title.
