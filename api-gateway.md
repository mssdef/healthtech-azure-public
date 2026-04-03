# api-gateway

NestJS REST API — patient portal and case management backend.

## Stack

| Layer | Technology |
|---|---|
| Framework | NestJS 10 |
| ORM | Prisma 5 (PostgreSQL) |
| Auth | `@nestjs/jwt` + Passport JWT |
| API docs | `@nestjs/swagger` (OpenAPI 3) |
| Runtime | Node 20 |

## Local setup

**Prerequisites:** Node 20+, Docker (PostgreSQL via `make up` from repo root)

```bash
# 1. Install dependencies (from repo root)
npm install

# 2. Start PostgreSQL
make up

# 3. Generate Prisma client
cd apps/api-gateway
npx prisma generate

# 4. Run migrations
npx prisma migrate dev

# 5. Serve
npx nx serve api-gateway
# → http://localhost:3000
# → http://localhost:3000/api  (Swagger UI)
```

## Modules

| Module | Routes | Description |
|---|---|---|
| `AuthModule` | `POST /auth/login` | Returns JWT; demo users: `clinician/demo`, `admin/demo` |
| `PatientModule` | `GET /patients`, `GET /patients/:id`, `POST /patients` | Patient CRUD |
| `CoverageModule` | `GET /fhir/Coverage?patient=:id`, `GET /fhir/Coverage/:id` | FHIR R4 Coverage |
| `CaseModule` | `GET /cases`, `POST /cases`, `GET /cases/:id`, `POST /cases/:id/notes`, `PATCH /cases/:id/status` | Case management |
| `AuditModule` | _(internal)_ | Auto-writes `AuditLog` rows on mutations |

All routes except `POST /auth/login` require a Bearer token.

## Auth

```bash
# Obtain token
curl -s -X POST http://localhost:3000/auth/login \
  -H "Content-Type: application/json" \
  -d '{"username":"clinician","password":"demo"}' | jq .access_token

# Use token
curl http://localhost:3000/patients \
  -H "Authorization: Bearer <token>"
```

Demo roles: `clinician`, `admin`. Role-based access enforced with `@Roles()` decorator + `RolesGuard`.

## FHIR endpoints

```bash
# FHIR R4 Patient
GET /fhir/Patient/:id

# FHIR R4 Coverage for a patient
GET /fhir/Coverage?patient=:patientId
```

## Environment variables

| Variable | Required | Default | Description |
|---|---|---|---|
| `DATABASE_URL` | Yes | — | PostgreSQL connection string |
| `JWT_SECRET` | Yes | `dev-secret` | JWT signing key (change in production) |
| `PORT` | No | `3000` | HTTP port |

Set via `.env` in `apps/api-gateway/` or via container environment.

## Database

Schema source of truth: `specs/db-schema.prisma`
App schema: `apps/api-gateway/prisma/schema.prisma`

```bash
# Generate client after schema change
npx prisma generate

# Create and apply migration
npx prisma migrate dev --name <description>

# Reset dev database
npx prisma migrate reset
```

## OpenAPI spec

Swagger UI is served at `/api` when the app is running. To export the spec:

```bash
# After npx nx serve api-gateway is running
curl http://localhost:3000/api-json > specs/api-gateway.openapi.json
```

## Testing

```bash
npx nx test api-gateway
```

Tests use `@nestjs/testing` — no database required for unit tests.

## Project layout

```
src/
  main.ts                  ← bootstrap: ValidationPipe, Swagger, listen
  app.module.ts            ← root module
  prisma/
    prisma.module.ts       ← global PrismaModule
    prisma.service.ts      ← PrismaClient lifecycle
  auth/
    auth.module.ts
    auth.service.ts        ← login, JWT sign
    auth.controller.ts     ← POST /auth/login
    jwt.strategy.ts        ← Passport JWT strategy
    jwt-auth.guard.ts
    roles.decorator.ts
    roles.guard.ts
  patient/
    patient.module.ts
    patient.service.ts
    patient.controller.ts  ← GET/POST /patients, GET /fhir/Patient/:id
  coverage/
    coverage.service.ts
    coverage.controller.ts ← GET /fhir/Coverage
  case/
    case.service.ts
    case.controller.ts     ← case CRUD, notes, status
  audit/
    audit.service.ts       ← log(actor, action, resourceType, resourceId)
prisma/
  schema.prisma            ← Prisma schema (copy of specs/db-schema.prisma)
Dockerfile                 ← multi-stage build for Azure Container Apps
```
