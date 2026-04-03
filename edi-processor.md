# edi-processor

Azure Functions app — EDI 270/271 eligibility pipeline entry points.

Registers two triggers that both delegate entirely to `libs/eligibility-pipeline`:

| Trigger | Type | Route / Queue |
|---|---|---|
| `ingest_http` | HTTP POST | `/api/process` |
| `ingest_from_service_bus` | Service Bus queue | `edi-inbound` (configurable) |

## Local setup

**Prerequisites:** Python 3.11+, Docker, [Azure Functions Core Tools v4](https://learn.microsoft.com/azure/azure-functions/functions-run-local)

```bash
# 1. Install Python dependencies (from repo root)
pip install -e "libs/eligibility-pipeline[azure]"

# 2. Start PostgreSQL + Azurite
make up           # from repo root

# 3. Apply migrations
alembic upgrade head

# 4. Configure local settings
cp apps/edi-processor/local.settings.json.example apps/edi-processor/local.settings.json
# edit DATABASE_URL if needed

# 5. Start Functions host
cd apps/edi-processor
func start --port 7071
```

## Sending a test message

```bash
# HTTP trigger
curl -X POST http://localhost:7071/api/process \
     --data-binary @libs/eligibility-pipeline/samples/270_request.edi \
     -H "Content-Type: text/plain"

# With custom headers
curl -X POST http://localhost:7071/api/process \
     --data-binary @libs/eligibility-pipeline/samples/270_request.edi \
     -H "Content-Type: text/plain" \
     -H "X-EDI-Source: my-system" \
     -H "X-Correlation-Id: test-001"
```

## HTTP response

`ProcessResponse` JSON:

```json
{
  "status": "SUCCESS",
  "raw_id": "a1b2c3d4-...",
  "transaction_set_id": "270",
  "errors": []
}
```

| HTTP code | `status` | Meaning |
|-----------|----------|---------|
| `200` | `SUCCESS` | Parsed and persisted |
| `400` | `PARSE_FAILURE` | Invalid EDI payload |
| `409` | `DUPLICATE` | Same payload already processed |
| `500` | `DB_ERROR` | Unexpected server error |

## Local settings

`local.settings.json` is gitignored. Copy the example and fill in values:

```bash
cp local.settings.json.example local.settings.json
```

| Key | Required | Description |
|---|---|---|
| `FUNCTIONS_WORKER_RUNTIME` | Yes | Must be `python` |
| `AzureWebJobsStorage` | Yes | Use `UseDevelopmentStorage=true` with Azurite |
| `DATABASE_URL` | Yes | PostgreSQL connection string |
| `LOG_LEVEL` | Yes | `DEBUG` locally, `INFO` in production |
| `AZURE_SERVICEBUS_CONNECTION_STRING` | Queue trigger | Service Bus namespace connection string |
| `AZURE_SERVICEBUS_QUEUE_NAME` | Queue trigger | Queue name — default `edi-inbound` |

## Testing

```bash
# From repo root — unit + integration tests for the pipeline lib
make test

# HTTP trigger tests (func must be running on port 7071)
pytest libs/eligibility-pipeline/tests -m azure_http

# Or via Nx
npx nx test edi-processor
```

## Replaying a failed message

A `PARSE_FAILURE` run stores the raw payload in `raw_edi_message`. The deduplication hash is only set on `SUCCESS` — re-POSTing a previously failed payload will reprocess it normally.

```sql
-- Find failed runs
SELECT r.id, r.error_message, m.received_at
FROM pipeline_run r
JOIN raw_edi_message m ON m.id = r.raw_edi_message_id
WHERE r.status = 'PARSE_FAILURE'
ORDER BY m.received_at DESC;
```

Then re-POST the original body to `POST /api/process`.

## Docker

The full local stack (including this function) runs via Docker Compose:

```bash
make up     # starts db, azurite, migrate, functions
make down   # stop
make logs   # (docker compose logs -f functions)
```

## Project layout

```
function_app.py              ← trigger registrations only; no business logic
host.json                    ← Azure Functions runtime config
requirements.txt             ← eligibility-pipeline[azure]
local.settings.json.example  ← env var template
project.json                 ← Nx targets: start, test
```

All business logic lives in `libs/eligibility-pipeline/`.
