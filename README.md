# APCinema Backend

Backend for the APCinema cinema platform. Built as multiple NestJS services communicating over gRPC. Clients only talk to the **gateway** over HTTP; everything else is internal.

## What's inside

| Service / package | Role |
|---|---|
| **gateway-service** | HTTP API, Swagger, proxies requests to microservices via gRPC |
| **auth-service** | Authentication: OTP via phone or email, accounts in PostgreSQL, codes in Redis |
| **contracts** | Protobuf contracts and generated TypeScript types (`@apcinema/contracts`) |
| **common** | Shared utilities (`@apcinema/shared`) — gRPC → HTTP mapping, etc. |
| **core** | Shared dev tooling (`@apcinema/core`) — currently mostly Prettier config |
| **docker** | Local infrastructure: PostgreSQL and Redis |

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
          ▼
┌───────────────────┐       ┌─────────────┐
│   auth-service    │◄─────►│    Redis    │  OTP codes (5 min TTL)
│   :50051          │       └─────────────┘
└─────────┬─────────┘
          │
          ▼
   ┌─────────────┐
   │ PostgreSQL  │  accounts (phone / email)
   └─────────────┘
```

Only one domain is implemented so far — **auth**:

- `POST /auth/otp/send` — send an OTP to a phone number or email
- `POST /auth/otp/verify` — verify the code and receive tokens

API docs: `http://<host>/docs` (Swagger), OpenAPI YAML: `/openapi.yaml`.

## Requirements

- **Node.js** 20+ (the project uses `@types/node` v24)
- **npm**
- **Docker** and **Docker Compose** — for PostgreSQL and Redis
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

### 3. Set up auth-service

```bash
cd auth-service
npm install
```

```bash
cp auth-service/.env.example auth-service/.env
```

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
cp gateway-service/.env.example gateway-service/.env
```

Run:

```bash
npm run start:dev
```

### 5. Verify

```bash
# health check
curl http://localhost:3000/health

# send OTP (in dev the code is logged by auth-service)
curl -X POST http://localhost:3000/auth/otp/send \
  -H "Content-Type: application/json" \
  -d '{"identifier": "+421950353687", "type": "phone"}'
```

Swagger: [http://localhost:3000/docs](http://localhost:3000/docs)

## Environment variables

### gateway-service

| Variable | Description |
|---|---|
| `HTTP_HOST` | Public service URL (used in logs) |
| `HTTP_PORT` | HTTP server port |
| `HTTP_CORS` | Allowed origins, comma-separated |
| `AUTH_GRPC_URL` | auth-service address (`host:port`) |
| `SWAGGER_TITLE` | Swagger page title |
| `SWAGGER_DESCRIPTION` | API description in Swagger |
| `SWAGGER_VERSION` | API version in Swagger |

### auth-service

| Variable | Description |
|---|---|
| `DATABASE_URL` | Connection string for Prisma CLI |
| `POSTGRES_*` | PostgreSQL connection settings at runtime |
| `REDIS_*` | Redis connection settings |

The auth-service gRPC port is hardcoded in `auth-service/src/main.ts` (`localhost:50051`). If you change it, update `AUTH_GRPC_URL` in the gateway as well.

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
├── auth-service/       # gRPC authentication microservice
├── gateway-service/    # HTTP gateway
├── contracts/          # proto + generated types
├── common/             # @apcinema/shared
├── core/               # @apcinema/core
├── docker/             # docker-compose for local development
├── .gitmodules         # submodule repository links
└── README.md
```

Each service is a separate git submodule with its own repository. Changes in `contracts` or `common` may require running `npm install` in dependent services.

## Development

### Useful commands

In each service (`auth-service`, `gateway-service`):

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
- In dev mode, OTP is logged to the auth-service console (`console.debug`); SMS/email delivery is not wired up yet.
- Prisma migrations: currently using `prisma db push`; once `prisma/migrations` exists, switch to `prisma migrate dev`.

## License

Services are private (`UNLICENSED`). The `@apcinema/contracts` and `@apcinema/core` packages are published separately.
