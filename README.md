# spring-boot-jwt-auth

> Spring Boot 3 · Spring Security 6 · JWT access/refresh tokens · RBAC · PostgreSQL · Docker

A production-ready JWT authentication and authorization service built with Spring Boot 3 and Spring Security 6. Implements stateless token-based authentication with role-based access control, refresh token rotation, and token revocation on logout.

---

## Tech Stack

| Layer | Technology |
|---|---|
| Language | Java 17 |
| Framework | Spring Boot 3.1.4 |
| Security | Spring Security 6 |
| Token | JJWT 0.11.5 |
| Database | PostgreSQL |
| ORM | Spring Data JPA / Hibernate |
| API Docs | SpringDoc OpenAPI 3 (Swagger UI) |
| Build | Maven |
| Container | Docker / Docker Compose |

---

## Features

- **JWT Authentication** — stateless access tokens signed with HMAC-SHA256
- **Refresh Token Rotation** — long-lived refresh tokens stored in DB, single-use with automatic rotation
- **Token Revocation** — all active tokens invalidated on logout
- **Role-Based Access Control** — three roles with granular permission sets
- **Auditing** — `createdBy` / `lastModifiedBy` fields auto-populated via Spring Data Auditing
- **OpenAPI Docs** — Swagger UI available at `/swagger-ui/index.html`

---

## Roles & Permissions

| Role | Permissions |
|---|---|
| `USER` | Basic authenticated access |
| `MANAGER` | `management:read`, `management:create`, `management:update`, `management:delete` |
| `ADMIN` | All MANAGER permissions + `admin:read`, `admin:create`, `admin:update`, `admin:delete` |

---

## Project Structure

```
src/main/java/com/authservice/security/
├── auth/               # Registration, login, refresh token endpoints
├── config/             # Security filter chain, JWT service, logout handler
├── token/              # Token entity, repository, type enum
├── user/               # User entity, roles, permissions, user service
├── demo/               # Protected demo endpoints (USER, MANAGER, ADMIN)
├── book/               # Sample CRUD resource (requires auth)
└── auditing/           # Spring Data Auditing configuration
```

---

## API Reference

### Authentication

| Method | Endpoint | Description | Auth Required |
|---|---|---|---|
| `POST` | `/api/v1/auth/register` | Register new user | No |
| `POST` | `/api/v1/auth/authenticate` | Login and receive tokens | No |
| `POST` | `/api/v1/auth/refresh-token` | Rotate refresh token | Bearer (refresh) |
| `POST` | `/api/v1/auth/logout` | Revoke all tokens | Bearer |

### Protected Endpoints

| Method | Endpoint | Required Role |
|---|---|---|
| `GET` | `/api/v1/demo-controller` | USER |
| `GET/POST/PUT/DELETE` | `/api/v1/management/**` | MANAGER |
| `GET/POST/PUT/DELETE` | `/api/v1/admin/**` | ADMIN |

---

### Register

```http
POST /api/v1/auth/register
Content-Type: application/json

{
  "firstname": "John",
  "lastname": "Doe",
  "email": "john.doe@example.com",
  "password": "password123",
  "role": "USER"
}
```

**Response:**
```json
{
  "access_token": "eyJhbGciOiJIUzI1NiJ9...",
  "refresh_token": "eyJhbGciOiJIUzI1NiJ9..."
}
```

### Login

```http
POST /api/v1/auth/authenticate
Content-Type: application/json

{
  "email": "john.doe@example.com",
  "password": "password123"
}
```

### Refresh Token

```http
POST /api/v1/auth/refresh-token
Authorization: Bearer <refresh_token>
```

### Logout

```http
POST /api/v1/auth/logout
Authorization: Bearer <access_token>
```

---

## Getting Started

### Prerequisites

- Java 17+
- Maven 3.8+
- Docker & Docker Compose

### 1. Clone the repository

```bash
git clone https://github.com/your-username/spring-boot-jwt-auth.git
cd spring-boot-jwt-auth
```

### 2. Start PostgreSQL with Docker

```bash
docker-compose up -d
```

This starts:
- **PostgreSQL** on port `5432` (user: `username`, password: `password`)
- **pgAdmin** on port `5050` (http://localhost:5050)

### 3. Configure application properties

`src/main/resources/application.yml`:

```yaml
spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/jwt_security
    username: username
    password: password
  jpa:
    hibernate:
      ddl-auto: create-drop
    show-sql: true

application:
  security:
    jwt:
      secret-key: your-256-bit-secret-key-here
      expiration: 86400000        # 1 day (ms)
      refresh-token:
        expiration: 604800000     # 7 days (ms)
```

### 4. Run the application

```bash
./mvnw spring-boot:run
```

The service starts on `http://localhost:8080`.

**Swagger UI:** `http://localhost:8080/swagger-ui/index.html`

---

## How JWT Flow Works

```
Client                    Server
  |                          |
  |-- POST /register ------->|
  |<-- access_token + -------|
  |    refresh_token         |
  |                          |
  |-- GET /protected ------->| (Authorization: Bearer <access_token>)
  |   (access denied after   |
  |    token expiry)         |
  |                          |
  |-- POST /refresh-token -->| (Authorization: Bearer <refresh_token>)
  |<-- new access_token  ----|
  |    new refresh_token     | (old refresh token is revoked)
  |                          |
  |-- POST /logout --------->| (all tokens revoked in DB)
  |<-- 200 OK ---------------|
```

- Access tokens are short-lived (default: 24h)
- Refresh tokens are long-lived (default: 7 days) and stored in the database
- On each token refresh, the old refresh token is revoked and a new pair is issued
- On logout, all user tokens (access + refresh) are marked as expired and revoked

---

## Token Storage

Tokens are persisted in the `token` table:

| Column | Type | Description |
|---|---|---|
| `id` | `BIGINT` | Primary key |
| `token` | `VARCHAR` | JWT string |
| `token_type` | `VARCHAR` | `BEARER` |
| `expired` | `BOOLEAN` | Manually expired |
| `revoked` | `BOOLEAN` | Revoked on logout |
| `user_id` | `BIGINT` | FK to `_user` table |

---

## Running Tests

```bash
./mvnw test
```

---

## License

MIT
