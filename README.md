# Secure Banking API Gateway

A microservices-based digital banking platform focused on centralized API security, transactional integrity, and containerized deployment. Users can register, authenticate via JWT or GitHub OAuth2, manage accounts, and execute idempotent money transfers.

```text
Browser ──> Next.js (3000) ──/api/*──> API Gateway (8080) ──> auth-service    (8081) ──> PostgreSQL (authdb)
                                         (JWT check)      └─> account-service (8082) ──> PostgreSQL (accountdb)
                                                                                  └──> Redis (account cache)
```

| Service / Layer | Stack | Key Responsibilities |
|---|---|---|
| `api-gateway` | Spring Cloud Gateway | Request routing and edge-level JWT validation |
| `auth-service` | Spring Boot, Spring Security (OAuth2 Client), JPA, BCrypt, JJWT | Registration, login, token refresh, GitHub OAuth2 flow, short-lived access & refresh tokens |
| `account-service` | Spring Boot, OAuth2 Resource Server, JPA/Hibernate, PostgreSQL, Redis | Account management, atomic transfers, Redis caching, idempotency, optimistic locking |
| `frontend` | Next.js 14, TypeScript, Material UI, Jest | Auth flows, account dashboard, transfer form, paginated transaction history |
| `infra` | Docker, Docker Compose, GitHub Actions | One-command containerized deployment and CI pipeline (tests, build, image push) |

## Quick Start (Docker)

```bash
cp .env.example .env        # Configure DB password and JWT_SECRET (32+ chars)
docker compose up --build
```

Open `http://localhost:3000`, register an account, and create two bank accounts (or a second user) to test transfers using the 12-digit account number.

## Security & OAuth2 Architecture

- **API Gateway & Defence in Depth:** `api-gateway` validates incoming JWTs at the edge, while `account-service` independently verifies bearer tokens as an OAuth2 resource server (`spring-boot-starter-oauth2-resource-server`), checking the HS256 signature, expiry, `type=access` claim, and role authorities.
- **GitHub OAuth2 Login:** `auth-service` acts as an OAuth2 client using Spring Security's authorization-code flow. Authenticated GitHub users are mapped to local accounts (`gh_<login>`) and issued standard access/refresh JWTs delivered via URL fragments so tokens are never sent to backend servers.
- **Account Takeover Mitigations:** Manual registration of `gh_*` usernames is blocked to prevent pre-hijacking, GitHub accounts are never auto-linked to local accounts by email, and temporary OAuth handshake sessions are invalidated immediately after login.

### Optional GitHub OAuth2 Setup
*(Standard username/password authentication works out of the box without this step.)*

1. Go to **GitHub → Settings → Developer settings → OAuth Apps → New OAuth App**.
2. Set **Homepage URL** to `http://localhost:3000` and **Authorization callback URL** to `http://localhost:3000/login/oauth2/code/github`.
3. Add the generated `GITHUB_CLIENT_ID` and `GITHUB_CLIENT_SECRET` to your `.env` file.
4. Run `docker compose up --build`, open `http://localhost:3000`, and select **Continue with GitHub**.

## API Quick Test (cURL)

```bash
# Register a user (returns access & refresh tokens)
curl -s -X POST localhost:8080/api/auth/register -H 'Content-Type: application/json' \
  -d '{"username":"alice","email":"alice@example.com","password":"s3cretPass"}'

# Create an account
TOKEN=<accessToken>
curl -s -X POST localhost:8080/api/accounts -H "Authorization: Bearer $TOKEN" \
  -H 'Content-Type: application/json' -d '{"initialDeposit":500}'

# Execute an idempotent transfer (retrying with the same Idempotency-Key moves money only once)
curl -s -X POST localhost:8080/api/transfers -H "Authorization: Bearer $TOKEN" \
  -H 'Content-Type: application/json' -H 'Idempotency-Key: demo-1' \
  -d '{"fromAccountId":1,"toAccountNumber":"<12 digits>","amount":25.00}'
```

## Local Development (Without Docker)

```bash
docker compose up postgres redis -d
export JWT_SECRET='a-long-random-string-of-at-least-32-characters'
(cd auth-service    && mvn spring-boot:run)
(cd account-service && mvn spring-boot:run)
(cd api-gateway     && mvn spring-boot:run)
(cd frontend        && npm install && npm run dev)
```

PostgreSQL defaults to `bank` / `change-me` (override via `DB_USER` and `DB_PASSWORD`).

## Running Tests

```bash
cd auth-service    && mvn verify     # JUnit + Mockito
cd account-service && mvn verify     # JUnit + Mockito + Testcontainers (requires Docker)
cd frontend        && npm test       # Jest + React Testing Library
```

## Architecture & Design Decisions

- **Atomic Transfers:** Debit, credit, and transfer ledger writes are co-located in `account-service` within a single `@Transactional` boundary to guarantee ACID consistency without the overhead of distributed sagas.
- **Idempotency Enforcement:** Transfer requests require an `Idempotency-Key` header backed by a database `UNIQUE` constraint. Retried requests safely return the original response; the frontend rotates the key only after a successful transfer.
- **Optimistic Locking:** The `Account` entity uses `@Version` concurrency control. Concurrent transfers modifying the same account trigger an HTTP `409 Conflict` for safe client retry rather than silently overwriting balances.
- **Cache-Aside with Redis:** Account lookups are cached in Redis for 10 minutes and evicted *after transaction commit* to prevent stale reads. `AccountLookup` is isolated as a separate Spring bean so caching works cleanly through Spring proxies.
- **Anti-Enumeration Ownership Checks:** Unauthorized requests to another user's account return `404 Not Found` instead of `403 Forbidden` to prevent account ID probing.

## Roadmap & Production Enhancements

- **Token & Key Security:** Migrate token storage from `localStorage` to `HttpOnly` cookies, implement refresh token rotation/revocation, and switch from a shared HS256 secret to asymmetric keys (JWKS) with a dedicated secrets manager.
- **Database & Ledger:** Replace Hibernate `ddl-auto: update` with versioned Flyway migrations and introduce a formal double-entry ledger with audit logging.
- ***Resilience & Observability:*** Add gateway-level rate limiting, graceful fallback handling for Redis outages, OpenAPI/Swagger documentation, and parallel concurrency load tests.
- **Cloud Deployment:** Containerized deployment pipeline targeting AWS (ECS/EC2).
