# eligibility-pipeline

[![PyPI](https://img.shields.io/pypi/v/eligibility-pipeline)](https://pypi.org/project/eligibility-pipeline/)
[![Python](https://img.shields.io/pypi/pyversions/eligibility-pipeline)](https://pypi.org/project/eligibility-pipeline/)
[![CI](https://github.com/mssdef/edi-pipeline-gh/actions/workflows/ci.yml/badge.svg)](https://github.com/mssdef/edi-pipeline-gh/actions/workflows/ci.yml)

X12 270/271 eligibility pipeline — parse, validate, persist, and process healthcare eligibility transactions via Azure Functions.

Healthcare providers send X12 270 eligibility inquiry messages to payers, who respond with X12 271 eligibility responses. This pipeline ingests those messages, validates and parses them, persists them to PostgreSQL, and exposes the processing logic as an Azure Function (HTTP trigger + Service Bus trigger).

> **Note:** This is a portfolio MVP using synthetic 270/271 fixtures. Not a production clearinghouse integration.

---

## Architecture

```mermaid
flowchart LR
    Provider -->|X12 270| SB[Azure Service Bus]
    HTTP[HTTP POST] -->|raw EDI body| Fn
    SB -->|queue trigger| Fn[Azure Function\nfunction_app.py]
    Fn -->|ProcessRequest| Svc[process_message\nservice]
    Svc -->|SHA-256 hash| Idempotency{duplicate?}
    Idempotency -->|yes| Drop[return DUPLICATE]
    Idempotency -->|no| Parse[parse_edi]
    Parse -->|fail| Raw[(RawEdiMessage\n+ PipelineRun\nPARSE_FAILURE)]
    Parse -->|ok| Map[x12_to_domain mapper]
    Map --> Commit[(RawEdiMessage\nEligibilityTransaction\nCoverageSnapshot\nPipelineRun SUCCESS)]
```

---

## Installation

```bash
pip install eligibility-pipeline
```

With Azure Functions support:

```bash
pip install eligibility-pipeline[azure]
```

---

## Setup (development)

### Prerequisites

- Python 3.11+
- Docker (for PostgreSQL)
- [Azure Functions Core Tools](https://learn.microsoft.com/azure/azure-functions/functions-run-local) (for `func start`)

### 1. Create a virtual environment and install dependencies

```bash
python -m venv .venv
source .venv/bin/activate       # Windows: .venv\Scripts\activate
pip install -e ".[dev,azure]"
```

### 2. Start PostgreSQL

```bash
docker compose up -d
```

### 3. Apply database migrations

```bash
alembic upgrade head
```

### 4. Configure the Functions host

```bash
cp azure_functions/local.settings.json.example azure_functions/local.settings.json
# Fill in DATABASE_URL and any Azure keys — see Required keys below
```

### 5. Start the Functions host

```bash
cd azure_functions
func start
```

### 6. Send a test message

```bash
curl -X POST http://localhost:7071/api/process \
     --data-binary @samples/270_request.edi \
     -H "Content-Type: text/plain"
```

---

## HTTP API

### `POST /api/process`

Send a raw X12 EDI string as the request body.

```bash
curl -X POST http://localhost:7071/api/process \
     --data-binary @samples/270_request.edi \
     -H "Content-Type: text/plain"
```

**Response** — `ProcessResponse` JSON:

```json
{
  "status": "SUCCESS",
  "raw_id": "a1b2c3d4-...",
  "errors": [],
  "transaction_set_id": "270"
}
```

**Status codes:**

| Code | Meaning |
|------|---------|
| `200` | Message processed successfully |
| `400` | Parse failure — bad EDI payload |
| `409` | Duplicate — same payload already processed |
| `500` | Unexpected server error |

---

## Testing

### Unit tests (no database required)

```bash
pytest
```

Runs parse and model tests. No Docker, no Azure.

### Integration tests (requires PostgreSQL)

```bash
docker compose up -d
pytest -m integration
```

Tests the full `process_message` service against a real database: happy path (270 + 271), duplicate detection, parse failure behaviour, and row counts.

---

## Replaying a failed message

When a message lands in `PARSE_FAILURE`, the raw payload is committed to `raw_edi_message` so it can be replayed once the underlying issue is fixed.

**To replay:**

1. Identify the failed message via the `pipeline_run` table (`status = 'PARSE_FAILURE'`).
2. Fix the root cause (parser config, schema issue, malformed fixture, etc.).
3. Re-POST the same raw EDI body:

```bash
curl -X POST http://localhost:7071/api/process \
     --data-binary @the_failed_message.edi \
     -H "Content-Type: text/plain"
```

The deduplication hash is only enforced for payloads that have been **successfully processed**. A `PARSE_FAILURE` row does not block resubmission — the pipeline will process the message normally on the next attempt.

---

## Azure Functions — local settings

`local.settings.json` is gitignored and never committed. Copy the example and fill in values:

```bash
cp azure_functions/local.settings.json.example azure_functions/local.settings.json
```

### Required keys

| Key | Required | Description |
|-----|----------|-------------|
| `FUNCTIONS_WORKER_RUNTIME` | Yes | Must be `python`. |
| `AzureWebJobsStorage` | Yes | Storage connection string. Use `UseDevelopmentStorage=true` with Azurite for local dev. |
| `DATABASE_URL` | Yes | PostgreSQL connection string. Default: `postgresql+psycopg://edi:edi@localhost:5432/eligibility` |
| `LOG_LEVEL` | Yes | `DEBUG` locally, `INFO` in production. |
| `AZURE_SERVICEBUS_CONNECTION_STRING` | Queue trigger only | Service Bus namespace connection string. Omit for HTTP-only local testing. |
| `AZURE_SERVICEBUS_QUEUE_NAME` | Queue trigger only | Name of the inbound queue (e.g. `edi-inbound`). |
| `AZURE_STORAGE_ACCOUNT` | Optional | Storage account name for raw message archive. |
| `AZURE_STORAGE_CONTAINER` | Optional | Blob container for raw message archive (e.g. `edi-raw`). |

---

## Project structure

```
src/eligibility_pipeline/
  __init__.py              ← __version__ = "0.1.0"
  py.typed                 ← PEP 561 marker
  cli.py                   ← eligibility-pipeline CLI entry point
  parse.py                 ← parse_edi + ParseResult / ParseError
  models.py                ← ProcessRequest / ProcessResponse / ErrorDetail
  settings.py              ← pydantic-settings Settings
  db/
    models.py              ← SQLModel tables (RawEdiMessage, EligibilityTransaction,
    session.py               CoverageSnapshot, PipelineRun) + enums
    migrations/            ← Alembic env + initial migration
  services/
    process_message.py     ← core ingest service
  mappers/
    x12_to_domain.py       ← parsed X12 → domain models
azure_functions/
  function_app.py          ← Service Bus + HTTP triggers
  host.json
  requirements.txt         ← eligibility-pipeline[azure]
  local.settings.json.example
samples/
  270_request.edi          ← synthetic 270 inquiry fixture
  271_response.edi         ← synthetic 271 response fixture
tests/
  test_parse.py
  test_process_message.py
.github/workflows/ci.yml   ← lint + test + publish
```
