# eligibility-pipeline

[![PyPI](https://img.shields.io/pypi/v/eligibility-pipeline)](https://pypi.org/project/eligibility-pipeline/)
[![Python](https://img.shields.io/pypi/pyversions/eligibility-pipeline)](https://pypi.org/project/eligibility-pipeline/)

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

### 4. Configure Azure Functions

```bash
cp azure_functions/local.settings.json.example azure_functions/local.settings.json
# Edit local.settings.json — see "Azure Functions local setup" below
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

The deduplication hash is only enforced once per payload that has been **successfully processed**. A `PARSE_FAILURE` row does not block resubmission — the pipeline will process the message normally on the next attempt.

---

## Azure Functions — local setup

Copy the example settings file and fill in real values:

```bash
cp azure_functions/local.settings.json.example azure_functions/local.settings.json
```

`local.settings.json` is gitignored and never committed.

### Required keys

| Key | Description |
|-----|-------------|
| `FUNCTIONS_WORKER_RUNTIME` | Must be `python`. |
| `AzureWebJobsStorage` | Storage connection string. Use `UseDevelopmentStorage=true` with Azurite for local dev. |
| `DATABASE_URL` | PostgreSQL connection string. Matches `docker-compose.yml` defaults: `postgresql+psycopg://edi:edi@localhost:5432/eligibility` |
| `LOG_LEVEL` | `DEBUG` locally, `INFO` in production. |
| `AZURE_SERVICEBUS_CONNECTION_STRING` | Service Bus namespace connection string. Required for the queue trigger; omit for HTTP-only local testing. |
| `AZURE_SERVICEBUS_QUEUE_NAME` | Name of the inbound queue (e.g. `edi-inbound`). |
| `AZURE_STORAGE_ACCOUNT` | Storage account name for raw message archive. Optional for local dev. |
| `AZURE_STORAGE_CONTAINER` | Blob container for raw message archive (e.g. `edi-raw`). Optional for local dev. |

### Running locally

```bash
# 1. Start local stack (Postgres + migrations + Azurite + Functions host)
docker compose up -d

# 2. Send a test message
curl -X POST http://localhost:7071/api/process \
     --data-binary @samples/270_request.edi \
     -H "Content-Type: text/plain"
```

If you want to run the Functions host directly on your machine instead of Docker:

```bash
cd azure_functions
func start
```

For compose-based local testing, the Service Bus trigger is disabled by default
(`AzureWebJobs.ingest_from_service_bus.Disabled=true`) so HTTP tests can run
without a live Service Bus namespace.

### HTTP integration tests (optional)

From the repo root, with `docker compose up -d` running:

```bash
pytest -m azure_http
```

Optional: `AZURE_FUNCTIONS_HTTP_BASE_URL`, `AZURE_FUNCTIONS_FUNCTION_KEY` (see `tests/test_azure_functions_http.py`).
