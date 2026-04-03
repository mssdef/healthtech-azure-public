# healthtech-azure

[![PyPI](https://img.shields.io/pypi/v/eligibility-pipeline)](https://pypi.org/project/eligibility-pipeline/)
[![Python](https://img.shields.io/pypi/pyversions/eligibility-pipeline)](https://pypi.org/project/eligibility-pipeline/)
[![CI](https://github.com/mssdef/edi-pipeline-gh/actions/workflows/ci.yml/badge.svg)](https://github.com/mssdef/edi-pipeline-gh/actions/workflows/ci.yml)

Nx monorepo — healthcare eligibility EDI pipeline (Phase 1) and patient portal / case management (Phase 2).

> **Portfolio MVP** — synthetic 270/271 fixtures, no real PHI, no production clearinghouse integration.

---

## Architecture

```mermaid
flowchart TB
  subgraph nx [Nx Monorepo]
    direction TB
    subgraph apps
      FN[apps/edi-processor\nAzure Function]
      BE[apps/api-gateway\nNestJS API]
      FE[apps/patient-portal\nReact SPA]
    end
    subgraph libs
      EP[libs/eligibility-pipeline\nPython EDI core]
      DTO[libs/shared-dto\nZod / TS types]
      FHIR[libs/fhir-mappers\nCoverage · Patient · EpisodeOfCare]
      AUTH[libs/auth-guards\nJWT + RBAC]
    end
  end

  Provider -->|X12 270| SB[Azure Service Bus]
  HTTP[HTTP POST] --> FN
  SB --> FN --> EP
  EP --> PG[(PostgreSQL)]
  FE -->|REST / JSON| BE
  BE --> FHIR --> PG
  BE --> DTO
  FE --> DTO
```

### Phase 1 — EDI pipeline

```mermaid
flowchart LR
    Provider -->|X12 270| SB[Azure Service Bus]
    HTTP[HTTP POST] -->|raw EDI body| Fn
    SB -->|queue trigger| Fn[Azure Function]
    Fn -->|ProcessRequest| Svc[process_message]
    Svc -->|SHA-256 hash| Idempotency{duplicate?}
    Idempotency -->|yes| Drop[return DUPLICATE]
    Idempotency -->|no| Parse[parse_edi]
    Parse -->|fail| Raw[(RawEdiMessage\n+ PipelineRun PARSE_FAILURE)]
    Parse -->|ok| Map[x12_to_domain mapper]
    Map --> Commit[(RawEdiMessage\nEligibilityTransaction\nCoverageSnapshot\nPipelineRun SUCCESS)]
```

---

## Monorepo layout

```
apps/
  edi-processor/           ← Azure Functions HTTP + Service Bus triggers
  api-gateway/             ← NestJS: Patient, Coverage, Case, Audit modules
  patient-portal/          ← React SPA: login, dashboard, cases
libs/
  eligibility-pipeline/    ← Python 3.11+ EDI parse → validate → persist
  shared-dto/              ← Zod schemas + TS types shared across apps
  fhir-mappers/            ← FHIR R4: Patient, Coverage, EpisodeOfCare
  auth-guards/             ← JWT guard + @Roles RBAC decorator
specs/
  db-schema.prisma         ← source-of-truth Prisma schema (Phase 2)
  api-gateway.openapi.yaml ← generated OpenAPI spec (Phase 2)
  shared-dto.ts            ← Zod type definitions (Phase 2)
```

---

## Phase 1 — EDI pipeline

### Installation

```bash
pip install eligibility-pipeline
# with Azure Functions support:
pip install eligibility-pipeline[azure]
```

### Local setup

**Prerequisites:** Python 3.11+, Docker, [Azure Functions Core Tools](https://learn.microsoft.com/azure/azure-functions/functions-run-local)

```bash
# 1. Install Python dependencies
python -m venv .venv && source .venv/bin/activate
pip install -e "libs/eligibility-pipeline[dev,azure]"

# 2. Start PostgreSQL
make up

# 3. Apply migrations
alembic upgrade head

# 4. Configure Functions host
cp apps/edi-processor/local.settings.json.example apps/edi-processor/local.settings.json
# edit DATABASE_URL and any Azure keys

# 5. Start Functions host
cd apps/edi-processor && func start

# 6. Send a test message
curl -X POST http://localhost:7071/api/process \
     --data-binary @libs/eligibility-pipeline/samples/270_request.edi \
     -H "Content-Type: text/plain"
```

### HTTP API

#### `POST /api/process`

Send a raw X12 EDI string as the request body.

**Response — `ProcessResponse` JSON:**

```json
{
  "status": "SUCCESS",
  "raw_id": "a1b2c3d4-...",
  "errors": [],
  "transaction_set_id": "270"
}
```

| HTTP code | Meaning |
|-----------|---------|
| `200` | Processed successfully |
| `400` | Parse failure — bad EDI payload |
| `409` | Duplicate — same payload already processed |
| `500` | Unexpected server error |

### Testing

```bash
# Unit tests (no database)
make test

# Integration tests (requires Docker)
make up
pytest -m integration

# Full suite including Azure Functions HTTP tests (requires func start)
make test-strict
```

### Replaying a failed message

A `PARSE_FAILURE` row in `pipeline_run` stores the raw payload for replay. The deduplication hash is only enforced for successfully processed messages — re-POSTing a failed payload will reprocess it normally.

1. Find the failed row: `SELECT * FROM pipeline_run WHERE status = 'PARSE_FAILURE';`
2. Fix the root cause.
3. Re-POST the original body to `POST /api/process`.

### Azure Functions — local settings

`local.settings.json` is gitignored. Copy the example:

```bash
cp apps/edi-processor/local.settings.json.example apps/edi-processor/local.settings.json
```

| Key | Required | Description |
|-----|----------|-------------|
| `FUNCTIONS_WORKER_RUNTIME` | Yes | `python` |
| `AzureWebJobsStorage` | Yes | Use `UseDevelopmentStorage=true` with Azurite locally |
| `DATABASE_URL` | Yes | `postgresql+psycopg://edi:edi@localhost:5432/eligibility` |
| `LOG_LEVEL` | Yes | `DEBUG` locally, `INFO` in production |
| `AZURE_SERVICEBUS_CONNECTION_STRING` | Queue trigger only | Service Bus namespace connection string |
| `AZURE_SERVICEBUS_QUEUE_NAME` | Queue trigger only | e.g. `edi-inbound` |

---

## Phase 2 — Patient portal & case management

### Local setup

**Prerequisites:** Node 20+, Python 3.11+, Docker

```bash
# Install Node dependencies (Nx, NestJS, React tooling)
npm install

# Install Python lib
pip install -e "libs/eligibility-pipeline[dev]"

# Start full local stack
make up
alembic upgrade head

# Run NestJS API
npx nx serve api-gateway

# Run React SPA (separate terminal)
npx nx serve patient-portal
```

### API

NestJS API gateway at `http://localhost:3000`:

| Endpoint | Description |
|----------|-------------|
| `POST /auth/login` | JWT login |
| `GET /patients` | List patients |
| `GET /patients/:id` | Patient detail |
| `GET /fhir/Patient/:id` | FHIR R4 Patient resource |
| `GET /fhir/Coverage?patient=:id` | FHIR R4 Coverage resource |
| `GET /cases` | List cases |
| `POST /cases` | Open new case |
| `GET /cases/:id` | Case detail with notes and documents |
| `POST /cases/:id/notes` | Add case note |

Full OpenAPI spec: `specs/api-gateway.openapi.yaml`

### Testing

```bash
# All TypeScript tests
npx nx run-many -t test

# Specific project
npx nx test api-gateway
npx nx test fhir-mappers
npx nx test patient-portal

# Affected only (CI mode)
npx nx affected -t test
```

---

## CI / CD

GitHub Actions (`.github/workflows/`):

| Workflow | Trigger | Jobs |
|----------|---------|------|
| `ci.yml` | push / PR to main | lint, typecheck, test (Python), publish to PyPI on `v*` tag |
| `ci-ts.yml` | push / PR to main | `nx affected` lint + test (TypeScript) |
| `deploy.yml` | merge to main | build React SPA → Azure Static Web Apps; build NestJS → Azure Container Apps |

---

## Skills demonstrated

**Phase 1 (Python):**
- Python 3.11+ application layout, typing, Pydantic, SQLModel, PostgreSQL transactions
- X12 270/271 EDI parsing and validation at portfolio depth
- Azure Functions HTTP + Service Bus triggers, idempotent pipeline design

**Phase 2 (TypeScript / full-stack):**
- NestJS module architecture, Prisma, JWT + RBAC, OpenAPI generation
- FHIR R4 resource mapping (`Patient`, `Coverage`, `EpisodeOfCare`)
- React with TanStack Query, React Hook Form + Zod, Zustand, shadcn/ui
- Nx monorepo: `affected` builds, shared libs, cross-language workspace
- Azure: Static Web Apps, Container Apps, Managed Identity
