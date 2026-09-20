# PayLedger — Digital Wallet & Payment Ledger

[![CI](https://github.com/Prayas-Dev/PayLedger/actions/workflows/ci.yml/badge.svg)](https://github.com/Prayas-Dev/PayLedger/actions/workflows/ci.yml)
[![Java 17](https://img.shields.io/badge/Java-17-007396)](https://adoptium.net/)
[![Spring Boot 4.1](https://img.shields.io/badge/Spring%20Boot-4.1-6DB33F)](https://spring.io/projects/spring-boot)
[![PostgreSQL](https://img.shields.io/badge/PostgreSQL-16-336791)](https://www.postgresql.org/)
[![Maven](https://img.shields.io/badge/Build-Maven-C71A36)](https://maven.apache.org/)
[![License: Apache‑2.0](https://img.shields.io/badge/License-Apache%202.0-blue.svg)](https://opensource.org/licenses/Apache-2.0)

---

## Overview
PayLedger is a **backend‑only** service that provides a reliable digital‑wallet and payment‑ledger API.  It solves the classic engineering challenges of financial systems:
- **Idempotent payment requests** – duplicate client retries never move money twice.
- **Concurrent balance updates** – optimistic locking prevents lost updates.
- **Immutable double‑entry ledger** – every transfer leaves a balanced audit trail.
- **Reliable downstream integration** – a transactional outbox guarantees at‑least‑once event delivery.
- **Secure API** – JWT‑based authentication with scope‑restricted authorization.

The implementation is deliberately small enough to be inspected in full while still demonstrating production‑grade patterns.

---

## Core Features
| Feature | Implementation |
|---|---|
| Wallet / Accounts | `Account` aggregate with versioned balances (`@Version`) persisted in PostgreSQL |
| Money Transfers | `TransferService` executes debits/credits in a single `@Transactional` method |
| Double‑Entry Ledger | `LedgerEntry` entity stores immutable debit and credit rows per transfer |
| Idempotency | `idempotencyKey` column with unique `(client_id, idempotency_key)` constraint |
| Concurrency Control | Optimistic locking (`ObjectOptimisticLockingFailureException`) on `Account` version |
| Authentication & Authorization | Spring Security Resource Server, JWT, scope‑based `@PreAuthorize` checks |
| Immutable Ledger | Ledger rows are never updated or deleted (only inserts) |
| Transactional Outbox | `OutboxEvent` persisted in same transaction, processed by a lock‑skipping publisher |
| Webhooks | HMAC‑SHA256 signed events emitted via the outbox publisher |
| Database Migrations | Flyway scripts version the schema and enforce constraints |
| Observability | Spring Boot Actuator + Micrometer Prometheus metrics |

---
## Architecture

```mermaid
flowchart LR
    Client["Client"] -->|"JWT + Idempotency-Key"| API["REST API"]
    API --> Service["Application / Service Layer"]
    Service --> DB[("PostgreSQL")]

    DB --> Accounts["Versioned Accounts"]
    DB --> Ledger["Immutable Ledger"]
    DB --> Transfers["Transfer Records"]
    DB --> Outbox["Transactional Outbox"]

    Outbox --> Publisher["Outbox Publisher"]
    Publisher --> Webhook["Webhook - HMAC signed"]

    API --> Metrics["Actuator & Micrometer"]
---

## Transfer Flow
1. **Authentication** – The request carries a JWT; Spring Security validates it and extracts scopes.
2. **Request validation** – `TransferController` validates the JSON payload (`@Valid`).
3. **Idempotency check** – `TransferService` looks for an existing transfer with the same `(clientId, idempotencyKey)`.
4. **Account lookup & currency match** – Source and destination accounts are loaded; mismatched currencies reject the request.
5. **Balance validation** – `Account.debit` throws if the source balance would become negative.
6. **Transactional processing** – Inside `@Transactional`:
   - Debit source, credit destination (materialized balances).
   - Persist a `Transfer` record.
   - Insert two `LedgerEntry` rows (debit & credit).
   - Create an `OutboxEvent` for the completed transfer.
7. **Commit** – The transaction commits atomically; any optimistic‑lock or constraint violation rolls back and is reported as a domain error.
8. **Response** – The newly created transfer DTO is returned (or the existing one for a retry).

---

## Double‑Entry Ledger
Each transfer creates **two** immutable ledger rows:
- **Debit** – Decreases the source account balance.
- **Credit** – Increases the destination account balance.

The ledger is never mutated; it provides an immutable audit trail that can be reconciled against the materialized balances.

**Example** – Transfer ₹1,000 from **Account A** to **Account B**:
| Entry | Account | Direction | Amount |
|---|---|---|---|
| 1 | A | DEBIT | 1,000 |
| 2 | B | CREDIT | 1,000 |

The sum of debit amounts always equals the sum of credit amounts, enforcing accounting integrity at the database level.

---

## Idempotency
Clients include an `Idempotency-Key` header.  The service stores `(client_id, idempotency_key)` as a unique constraint in the `transfer` table.  A second request with the same key returns the already‑persisted `Transfer` without performing any balance updates, protecting against network retries.

---

## Concurrency & Consistency
- **ACID transactions** – `@Transactional(isolation = READ_COMMITTED)` guarantees atomic writes.
- **Optimistic locking** – `Account` entities carry a `@Version` column; concurrent updates raise `ObjectOptimisticLockingFailureException` which the service translates into a retry‑safe domain error.
- **Isolation** – `READ_COMMITTED` prevents dirty reads while allowing high throughput.

---

## Security
- **Spring Security** – Configured as a JWT resource server.
- **Scopes** – Endpoints are protected with `@PreAuthorize("hasAuthority('SCOPE_...')")` (e.g., `accounts:read`, `transfers:write`).
- **JWT secret** – Symmetric HS256 secret supplied via `JWT_SECRET` environment variable.
- **Webhook signing** – Outbox events are signed with an HMAC secret (`WEBHOOK_SECRET`).

---

## Transactional Outbox
The `OutboxEvent` entity is written in the same transaction that updates balances and the ledger.  A background publisher claims pending rows with `FOR UPDATE SKIP LOCKED`, publishes the event (e.g., to a webhook), and marks the row as dispatched.  This guarantees that no event is lost even if the application crashes after the business transaction commits.

---

## Database Design
- **PostgreSQL** – Primary data store.
- **JPA / Hibernate** – Object‑relational mapping.
- **Flyway** – Versioned migration scripts (`src/main/resources/db/migration`).
- **Key tables**:
  - `accounts` – versioned aggregate with `balance` and `currency`.
  - `transfers` – idempotent transfer record.
  - `ledger_entry` – immutable debit/credit rows.
  - `outbox_event` – pending integration events.
- **Constraints** – unique `(client_id, idempotency_key)`, foreign keys, and a trigger that prevents updates/deletes on `ledger_entry`.

---

## API Overview
| Method | Endpoint | Purpose |
|---|---|---|
| `POST` | `/api/v1/accounts` | Create a new account (scope: `accounts:write`). |
| `GET`  | `/api/v1/accounts/{id}` | Retrieve account details (scope: `accounts:read`). |
| `POST` | `/api/v1/transfers` | Create a transfer – idempotent (scope: `transfers:write`). |

The OpenAPI specification is available at `src/main/resources/static/openapi.yaml` and can be viewed at `http://localhost:8080/v3/api-docs` when the application runs.

---

## Testing
- **JUnit 5** – Unit and integration tests.
- **Spring Test** – `@SpringBootTest` with `Testcontainers` PostgreSQL.
- **Testcontainers** – Spins up a real PostgreSQL instance for integration tests (`SecurityIntegrationTest`, `TransferIntegrationTest`).
- **Mockito** – Used where appropriate for unit isolation.
- **Coverage** – Tests cover idempotency, optimistic‑locking races, ledger invariants, and security constraints.

---

## Tech Stack
| Category | Technology |
|---|---|
| Language | Java 17 |
| Framework | Spring Boot 4.1 |
| Database | PostgreSQL |
| ORM | Spring Data JPA / Hibernate |
| Migrations | Flyway |
| Security | Spring Security (JWT) |
| Testing | JUnit 5, Mockito, Testcontainers |
| Build | Maven |
| Containerisation | Docker |
| CI | GitHub Actions |

---

## Project Structure
```
src/main/java/com/prayas/payledger
├── account/          # Account aggregate, repository, controller
├── transfer/         # Transfer command, service, controller, ledger
├── outbox/           # Outbox event entity & repository
├── security/         # JWT configuration, method security
├── webhook/          # HMAC‑signed webhook payloads
├── common/           # API error handling utilities
└── PaymentsApplication.java
```

---

## Running Locally
**Prerequisites**
- Java 17 (adopted OpenJDK)
- Maven 3.8+
- Docker & Docker Compose (for PostgreSQL)

**Start the database**
```bash
docker compose -f compose.yaml up -d postgres
```

**Environment variables** (examples):
```bash
export DB_URL=jdbc:postgresql://localhost:5432/payments
export DB_USER=payments
export DB_PASSWORD=payments
export JWT_SECRET='replace-with-32‑byte‑random‑value'
export WEBHOOK_SECRET='replace-with-another‑random‑secret'
```

**Run the application**
```bash
mvn spring-boot:run
```
The API will be reachable at `http://localhost:8080`.

**Execute the test suite**
```bash
mvn test
```
All integration tests use Testcontainers, so Docker must be running.

---

## Engineering Decisions
- **Materialised balances + immutable ledger** – Fast reads via the `accounts` table while the ledger provides an audit‑proof of every movement.
- **Optimistic locking** – Prevents lost updates without heavyweight pessimistic locks; suitable for high‑throughput payment services.
- **Idempotency key** – Guarantees exactly‑once semantics for client‑side retries.
- **Transactional outbox** – Decouples event publishing from the business transaction while preserving atomicity.
- **Flyway migrations** – Immutable, version‑controlled schema evolution.
- **JWT authentication** – Simple to run locally; in production you would swap for an external OIDC provider.
- **Docker/Testcontainers** – Guarantees that developers and CI run against the same PostgreSQL version, catching migration or constraint issues early.

---

## License
Apache-2.0 © 2024‑2026 Prayas Dev.  The original reference implementation was authored by **Asim Abdalla** and is retained under the same license.

---
