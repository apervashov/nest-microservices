# APCinema Backend

<p align="center">
  <a href="https://nestjs.com" title="NestJS"><img src="https://img.shields.io/badge/NestJS-E0234E?style=for-the-badge&logo=nestjs&logoColor=white" alt="NestJS" /></a>
  <a href="https://www.typescriptlang.org" title="TypeScript"><img src="https://img.shields.io/badge/TypeScript-3178C6?style=for-the-badge&logo=typescript&logoColor=white" alt="TypeScript" /></a>
  <a href="https://grpc.io" title="gRPC"><img src="https://img.shields.io/badge/gRPC-244C5A?style=for-the-badge&logo=grpc&logoColor=white" alt="gRPC" /></a>
  <a href="https://protobuf.dev" title="Protocol Buffers"><img src="https://img.shields.io/badge/Protobuf-4285F4?style=for-the-badge&logo=protobuf&logoColor=white" alt="Protocol Buffers" /></a>
  <a href="https://www.postgresql.org" title="PostgreSQL"><img src="https://img.shields.io/badge/PostgreSQL-4169E1?style=for-the-badge&logo=postgresql&logoColor=white" alt="PostgreSQL" /></a>
  <a href="https://redis.io" title="Redis"><img src="https://img.shields.io/badge/Redis-DC382D?style=for-the-badge&logo=redis&logoColor=white" alt="Redis" /></a>
  <a href="https://www.prisma.io" title="Prisma"><img src="https://img.shields.io/badge/Prisma-2D3748?style=for-the-badge&logo=prisma&logoColor=white" alt="Prisma" /></a>
  <a href="https://www.docker.com" title="Docker"><img src="https://img.shields.io/badge/Docker-2496ED?style=for-the-badge&logo=docker&logoColor=white" alt="Docker" /></a>
  <a href="https://www.rabbitmq.com" title="RabbitMQ"><img src="https://img.shields.io/badge/RabbitMQ-FF6600?style=for-the-badge&logo=rabbitmq&logoColor=white" alt="RabbitMQ" /></a>
  <a href="https://swagger.io" title="Swagger"><img src="https://img.shields.io/badge/Swagger-85EA2D?style=for-the-badge&logo=swagger&logoColor=black" alt="Swagger" /></a>
</p>

Backend for the APCinema cinema platform. Built as multiple NestJS services: synchronous calls over **gRPC**, async events over **RabbitMQ**. Clients only talk to the **gateway** over HTTP; everything else is internal.

## What's inside

| Service / package | Role |
|---|---|
| **gateway-service** | HTTP API, Swagger, proxies requests to microservices via gRPC |
| **auth-service** | Authentication: OTP via phone or email, accounts in PostgreSQL, codes in Redis, OTP events via RabbitMQ |
| **user-service** | User profiles: settings, avatars, admin operations; sync via RabbitMQ `auth.account_created` |
| **notifiaction-service** | Consumes async events from RabbitMQ (e.g. `auth.otp_requested`) for SMS/email delivery |
| **contracts** | Protobuf contracts and generated TypeScript types (`@apcinema/contracts`) |
| **common** | Shared utilities (`@apcinema/shared`) — gRPC → HTTP mapping, JWT helpers, etc. |
| **core** | Shared dev tooling (`@apcinema/core`) — currently mostly Prettier config |
| **docker** | Local infrastructure: PostgreSQL, Redis, and RabbitMQ |

## Architecture

```
Client (web / mobile)
        │
        ▼ HTTP
┌───────────────────┐
│  gateway-service  │  :3000 (default)
│  REST + Swagger   │
└─────────┬─────────┘
          │ gRPC
          ├──────────────────────┐
          ▼                      ▼
┌───────────────────┐   ┌───────────────────┐
│   auth-service    │   │   user-service    │
│   :50051          │   │   :50052          │
└─────────┬─────────┘   └─────────┬─────────┘
          │                       │
          ├───────────┐           │
          │           │           │
          ▼           ▼           ▼
   ┌─────────────┐ ┌─────────────┐ ┌─────────────┐
   │ PostgreSQL  │ │    Redis    │ │ PostgreSQL  │
   │  accounts   │ │  OTP codes  │ │    users    │
   └─────────────┘ └─────────────┘ └─────────────┘
          │
          │ RabbitMQ
          ├──────────────────────────────┐
          ▼                              ▼
   ┌─────────────────────┐      ┌─────────────────────┐
   │ notifiaction-service│      │   user-service      │
   │ auth.otp_requested  │      │ auth.account_created│
   └─────────────────────┘      └─────────────────────┘
```

When a user registers, **auth-service** creates an account and emits `auth.account_created` to the `users_queue`. **user-service** consumes the event and creates a profile linked by `accountId`.

When a user requests an OTP, **auth-service** stores the hashed code in Redis, logs it in dev mode, and emits an `auth.otp_requested` event to the `notifications_queue`. **notifiaction-service** consumes that event and will handle actual delivery.

Implemented domains:

**Auth** (`/auth/*`):

- `POST /auth/otp/send` — send an OTP to a phone number or email
- `POST /auth/otp/verify` — verify the code and receive tokens
- `POST /auth/register`, `POST /auth/login`, `POST /auth/refresh`, `POST /auth/logout`
- `GET /auth/me` — auth data only (phone, email, role, verification flags)

**Users** (`/users/*`, `/admin/users/*`):

- `GET /users/me`, `PATCH /users/me`, `POST /users/me/avatar`
- `GET /users/:id`, `GET /users/by-username/:username` — public profiles
- `GET /admin/users`, `GET /admin/users/:id` — admin only
- `PATCH /admin/users/:id/block`, `PATCH /admin/users/:id/unblock`, `DELETE /admin/users/:id`

API docs: `http://<host>/docs` (Swagger), OpenAPI YAML: `/openapi.yaml`.

## Requirements

- **Node.js** 20+ (the project uses `@types/node` v24)
- **npm**
- **Docker** and **Docker Compose** — for PostgreSQL, Redis, and RabbitMQ
- **Git** with submodule support

## Quick start

### 1. Clone the repository with submodules

```bash
git clone --recurse-submodules <repository-url>
cd backend
```

If the repo was already cloned without submodules:

```bash
git submodule update --init --recursive
```

### 2. Start infrastructure

Copy the example env file:

```bash
cp docker/.env.example docker/.env
```

Start the containers:

```bash
cd docker
docker compose up -d
```

Default ports:

- PostgreSQL — `localhost:5433` (container port `5432`)
- Redis — `localhost:6379`
- RabbitMQ — `localhost:5672` (AMQP), management UI — `localhost:15672`

### 3. Set up auth-service

```bash
cd auth-service
npm install
```

```bash
cp .env.example .env
```

Set `RMQ_URL` and `RMQ_QUEUE_NAME` in `.env` to match your RabbitMQ credentials from `docker/.env` (defaults: `amqp://admin:123456@localhost:5672`, queue `notifications_queue`).

Apply the database schema:

```bash
npx prisma db push
```

Run in dev mode:

```bash
npm run start:dev
```

The service listens for gRPC on `localhost:50051`.

### 4. Set up gateway-service

```bash
cd gateway-service
npm install
```

```bash
cp .env.example .env
```

Run:

```bash
npm run start:dev
```

### 5. Set up user-service

```bash
cd user-service
npm install
```

```bash
cp .env.example .env
```

Create the users database (if it does not exist):

```bash
docker exec -it docker-postgres-1 psql -U apcinema -c "CREATE DATABASE apcinema_users;"
```

Apply the database schema:

```bash
npx prisma db push
```

Run:

```bash
npm run start:dev
```

The service listens for gRPC on `localhost:50052` and HTTP (Swagger) on `localhost:3002`.

### 6. Set up notifiaction-service

```bash
cd notifiaction-service
npm install
```

Create `.env` with the same RabbitMQ settings as auth-service:

```env
RMQ_URL=amqp://admin:123456@localhost:5672
RMQ_QUEUE_NAME=notifications_queue
```

Run:

```bash
npm run start:dev
```

The service has no HTTP port — it only listens on the RabbitMQ queue.

### 7. Verify

```bash
# health check
curl http://localhost:3000/health

# send OTP (in dev the code is logged by auth-service; notifiaction-service logs the event)
curl -X POST http://localhost:3000/auth/otp/send \
  -H "Content-Type: application/json" \
  -d '{"identifier": "+421950353687", "type": "phone"}'
```

Swagger: [http://localhost:3000/docs](http://localhost:3000/docs)

RabbitMQ management UI: [http://localhost:15672](http://localhost:15672) (credentials from `docker/.env`)

## Environment variables

### gateway-service

| Variable | Description |
|---|---|
| `HTTP_HOST` | Public service URL (used in logs) |
| `HTTP_PORT` | HTTP server port |
| `HTTP_CORS` | Allowed origins, comma-separated |
| `AUTH_GRPC_URL` | auth-service address (`host:port`) |
| `USER_GRPC_URL` | user-service address (`host:port`) |
| `SWAGGER_TITLE` | Swagger page title |
| `SWAGGER_DESCRIPTION` | API description in Swagger |
| `SWAGGER_VERSION` | API version in Swagger |

### auth-service

| Variable | Description |
|---|---|
| `DATABASE_URL` | Connection string for Prisma CLI |
| `POSTGRES_*` | PostgreSQL connection settings at runtime |
| `REDIS_*` | Redis connection settings |
| `RMQ_URL` | RabbitMQ connection URL (default: `amqp://admin:123456@localhost:5672`) |
| `RMQ_QUEUE_NAME` | Queue for notification events (default: `notifications_queue`) |
| `RMQ_USERS_QUEUE_NAME` | Queue for user profile events (default: `users_queue`) |

The auth-service gRPC port is hardcoded in `auth-service/src/main.ts` (`localhost:50051`). If you change it, update `AUTH_GRPC_URL` in the gateway as well.

### user-service

| Variable | Description |
|---|---|
| `DATABASE_URL` | Connection string for Prisma CLI |
| `POSTGRES_*` | PostgreSQL connection settings at runtime (`POSTGRES_DB=apcinema_users`) |
| `USER_GRPC_URL` | user-service gRPC address (`localhost:50052`) |
| `HTTP_PORT` | HTTP/Swagger port (default: `3002`) |
| `RMQ_URL` | RabbitMQ connection URL |
| `RMQ_USERS_QUEUE_NAME` | Queue for `auth.account_created` events (default: `users_queue`) |
| `AVATAR_STORAGE_PATH` | Local path for avatar files (default: `./uploads/avatars`) |
| `AVATAR_PUBLIC_BASE_PATH` | Public URL prefix for avatars (default: `/avatars`) |
| `AVATAR_MAX_SIZE_MB` | Max avatar upload size in MB (default: `5`) |

The user-service gRPC URL is read from `USER_GRPC_URL`. Update gateway `USER_GRPC_URL` if you change the port.

### notifiaction-service

| Variable | Description |
|---|---|
| `RMQ_URL` | RabbitMQ connection URL — must match auth-service |
| `RMQ_QUEUE_NAME` | Queue name — must match auth-service |

### docker

| Variable | Description |
|---|---|
| `POSTGRES_USER` / `POSTGRES_PASSWORD` | PostgreSQL credentials |
| `REDIS_PASSWORD` | Redis password |
| `RABBITMQ_DEFAULT_USER` / `RABBITMQ_DEFAULT_PASS` | RabbitMQ credentials (used in `RMQ_URL`) |

## Error handling

The gateway maps gRPC errors to a unified HTTP format via `GrpcExceptionFilter`. Auth service error codes:

| Code | When |
|---|---|
| `OTP_EXPIRED` | Code expired or was never requested |
| `OTP_INVALID` | Wrong code |
| `ACCOUNT_NOT_FOUND` | Account not found after verification |

## Repository structure

```
backend/
├── auth-service/           # gRPC authentication microservice
├── user-service/           # gRPC user profile microservice
├── gateway-service/        # HTTP gateway
├── notifiaction-service/   # RabbitMQ consumer for notifications
├── contracts/              # proto + generated types
├── common/                 # @apcinema/shared
├── core/                   # @apcinema/core
├── docker/                 # docker-compose for local development
├── .gitmodules             # submodule repository links
└── README.md
```

`gateway-service`, `auth-service`, `notifiaction-service`, `contracts`, `common`, and `core` are git submodules with their own repositories. Changes in `contracts` or `common` may require running `npm install` in dependent services.

## Development

### Useful commands

In each service (`auth-service`, `user-service`, `gateway-service`, `notifiaction-service`):

```bash
npm run start:dev    # hot-reload
npm run build        # build
npm run lint         # ESLint
npm run test         # unit tests
npm run test:e2e     # e2e tests
```

### Contracts (protobuf)

Sources live in `contracts/proto/`. After changing a `.proto` file:

```bash
cd contracts
npm install
npm run generate
```

The `@apcinema/contracts` package is published to npm; locally, services pull it from `node_modules` or the submodule.

### Local package dependencies

- `@apcinema/shared` → `file:../common`
- `@apcinema/core` → `file:../core` (gateway) or npm version (auth-service)

After changing `common` or `core`, reinstall dependencies in the services:

```bash
npm install
```

## Current limitations

- JWT tokens are placeholders (`access_token` / `refresh_token` in the verify response) — real token issuance is in progress.
- OTP delivery pipeline is wired via RabbitMQ (`auth.otp_requested`), but SMS/email providers are not integrated yet — in dev the code is logged by auth-service and the event is logged by notifiaction-service.
- Prisma migrations: currently using `prisma db push`; once `prisma/migrations` exists, switch to `prisma migrate dev`.

## License

Services are private (`UNLICENSED`). The `@apcinema/contracts` and `@apcinema/core` packages are published separately.
