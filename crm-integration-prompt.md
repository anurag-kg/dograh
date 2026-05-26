# CRM → Dograh Integration: Implementation Prompt

Paste this entire prompt into a new Claude session to implement the integration.

---

## Context

You are helping integrate a **custom/internal CRM** with **Dograh** — an open-source voice AI platform (FastAPI backend + Next.js frontend). The goal is to allow the CRM to trigger outbound phone calls and campaigns directly via Dograh's API.

Dograh is running locally at:
- Backend API: `http://localhost:8000`
- Frontend: `http://localhost:3000`
- Source code: `/Users/anuraggupta/Documents/Projects/dograh`

---

## What Needs to Be Built

Build a self-contained **Dograh API client module** that the CRM can import and use to:

1. **Trigger a single outbound call** for a specific contact
2. **Create and start a bulk outbound campaign** from a contact list
3. **Check call/campaign status** to show progress in the CRM
4. **Handle all errors** clearly so the CRM UI can surface them

---

## Dograh API Reference (Exact, from Source)

### Authentication
All requests use an API key in the header:
```
X-API-Key: <api_key>
```
API keys are created in the Dograh UI at Settings → API Keys.

### Trigger a Single Call

```
POST http://localhost:8000/api/v1/public/agent/{trigger_uuid}
Header: X-API-Key: <api_key>
Content-Type: application/json
```

**Request body:**
```json
{
  "phone_number": "+14155552671",
  "initial_context": {
    "any_crm_field": "any_value"
  },
  "telephony_configuration_id": null
}
```
- `phone_number` — required, E.164 format (e.g. `+14155552671`)
- `initial_context` — optional dict; put any CRM fields here (lead ID, name, deal stage, etc.) — they are available to the AI agent during the call
- `telephony_configuration_id` — optional int; omit to use the org default telephony config

**Success response (200):**
```json
{
  "status": "initiated",
  "workflow_run_id": 42,
  "workflow_run_name": "WR-API-4521"
}
```

**Error responses:**
- `401` — Invalid API key
- `402` — Quota exceeded
- `400` — Telephony not configured / invalid phone number
- `403` — API key does not have access to this trigger UUID
- `404` — Trigger UUID not found, workflow not published, or trigger not active

### Alternative: Trigger by Workflow UUID (no trigger node needed)
```
POST http://localhost:8000/api/v1/public/agent/workflow/{workflow_uuid}
```
Same request/response as above.

### Test Mode (uses draft workflow, not published)
```
POST http://localhost:8000/api/v1/public/agent/test/{trigger_uuid}
POST http://localhost:8000/api/v1/public/agent/test/workflow/{workflow_uuid}
```

---

### Create a Campaign (Bulk Calling)

**Step 1 — Create campaign:**
```
POST http://localhost:8000/api/v1/campaign/create
Authorization: Bearer <user_jwt_token>   (NOTE: campaigns use user auth, not API key)
Content-Type: application/json
```

```json
{
  "name": "Q2 Follow-up Campaign",
  "workflow_id": 7,
  "source_type": "csv",
  "source_id": "uploaded-file-id",
  "max_concurrency": 3,
  "retry_config": {
    "enabled": true,
    "max_retries": 2,
    "retry_delay_seconds": 120,
    "retry_on_busy": true,
    "retry_on_no_answer": true,
    "retry_on_voicemail": true
  }
}
```

**Step 2 — Start it:**
```
POST http://localhost:8000/api/v1/campaign/{campaign_id}/start
```

**Step 3 — Monitor progress:**
```
GET http://localhost:8000/api/v1/campaign/{campaign_id}/progress
```

**Step 4 — Pause / Resume:**
```
POST http://localhost:8000/api/v1/campaign/{campaign_id}/pause
POST http://localhost:8000/api/v1/campaign/{campaign_id}/resume
```

---

### Check a Workflow Run (Call Status)
```
GET http://localhost:8000/api/v1/workflow/{workflow_id}/runs/{run_id}
```

**Response includes:**
```json
{
  "id": 42,
  "name": "WR-API-4521",
  "is_completed": true,
  "transcript_url": "https://...",
  "recording_url": "https://...",
  "cost_info": {
    "dograh_token_usage": 0,
    "call_duration_seconds": 87
  },
  "initial_context": { "crm_lead_id": "CRM-123" },
  "gathered_context": { "call_outcome": "interested", "callback_requested": true },
  "annotations": { "qa_score": 8 }
}
```

---

### List Workflows (to let CRM users pick which agent to use)
```
GET http://localhost:8000/api/v1/workflow/fetch
Authorization: Bearer <user_jwt_token>
```

---

## What to Build

### 1. Python Client Module (`dograh_client.py`)

Build a clean Python class `DograhClient` with these methods:

```python
class DograhClient:
    def __init__(self, base_url: str, api_key: str): ...

    def trigger_call(
        self,
        trigger_uuid: str,
        phone_number: str,
        initial_context: dict = None,
        telephony_configuration_id: int = None,
        test_mode: bool = False
    ) -> dict:
        """Returns {"status", "workflow_run_id", "workflow_run_name"} or raises DograhError"""

    def get_call_status(self, workflow_id: int, run_id: int) -> dict:
        """Returns the full workflow run dict"""

    def list_campaigns(self) -> list:
        """Returns list of campaigns"""

    def get_campaign_progress(self, campaign_id: int) -> dict:
        """Returns {"total_rows", "processed_rows", "failed_rows", "state"}"""
```

Requirements:
- Use `httpx` (sync) or `requests` for HTTP calls
- Raise a clear `DograhError(message, status_code)` exception on non-2xx responses
- Log all requests and responses at DEBUG level
- Retry on 5xx errors (max 3 attempts, exponential backoff)
- All methods return plain dicts (no internal Dograh models leak out)

### 2. Configuration (`dograh_config.py`)

```python
DOGRAH_BASE_URL = "http://localhost:8000"
DOGRAH_API_KEY  = ""          # fill from env var DOGRAH_API_KEY
DEFAULT_TRIGGER_UUID = ""     # fill from Dograh UI → Workflow → Trigger node
```

Load from environment variables, never hardcode credentials.

### 3. CRM Integration Module (`crm_dograh_integration.py`)

A higher-level module the CRM service layer calls. It should:

- `initiate_call_for_lead(lead: dict) -> CallResult`
  - Validates lead has a phone number
  - Maps CRM lead fields to `initial_context` (e.g. `lead["id"]` → `crm_lead_id`)
  - Calls `DograhClient.trigger_call()`
  - Returns a `CallResult` dataclass with `workflow_run_id`, `status`, `error`

- `start_campaign_from_leads(name: str, leads: list[dict], workflow_id: int) -> CampaignResult`
  - Converts leads list to a CSV in memory
  - Uploads CSV to Dograh (via MinIO presigned URL or direct upload)
  - Creates and starts the campaign
  - Returns `CampaignResult` dataclass with `campaign_id`, `status`, `error`

### 4. Tests (`test_dograh_integration.py`)

Write tests using `pytest` and `responses` (or `httpx`'s mock transport) that:
- Test `trigger_call` happy path
- Test `trigger_call` with 401 (bad API key)
- Test `trigger_call` with 402 (quota exceeded)
- Test `trigger_call` with 400 (telephony not configured)
- Test `get_call_status` happy path
- Test retry behavior on 500

---

## Project Layout

Place all new files in a new directory at the root of the CRM project. If you don't have the CRM source code in context, create the files as a standalone package that can be dropped in:

```
dograh_integration/
├── __init__.py
├── client.py           # DograhClient class
├── config.py           # Configuration constants
├── integration.py      # CRM-specific high-level module
├── exceptions.py       # DograhError and subclasses
└── tests/
    ├── __init__.py
    └── test_client.py
```

---

## Important Constraints

1. **Phone numbers must be E.164 format** — validate before sending (e.g. `+14155552671`, not `415-555-2671`)
2. **`initial_context` is the CRM data bridge** — everything you put here is available to the AI agent during the call AND in webhook templates after the call. Always include `crm_lead_id` or equivalent so results can be matched back.
3. **`trigger_uuid` vs `workflow_uuid`** — prefer trigger UUID (from the Trigger node in the workflow) for production. Workflow UUID works too but bypasses the trigger state check.
4. **Campaigns use bearer token auth, not API key** — the campaign endpoints use user JWT auth. If the CRM backend needs to start campaigns server-to-server without a user session, raise this as a limitation and note that it needs a service account token.
5. **Do not store API keys in code** — always read from environment variables.

---

## How to Get the Trigger UUID

The person setting this up needs to:
1. Open Dograh UI at `http://localhost:3000`
2. Open the workflow they want to trigger
3. Click the **Trigger** node
4. Copy the UUID from the node settings panel
5. Set it as `DEFAULT_TRIGGER_UUID` in config / env var

---

## Verification Steps

After building, verify with:

```bash
# 1. Set env vars
export DOGRAH_API_KEY="your_key_here"
export DOGRAH_TRIGGER_UUID="your_trigger_uuid_here"

# 2. Run a test call
python -c "
from dograh_integration.client import DograhClient
import os
c = DograhClient('http://localhost:8000', os.environ['DOGRAH_API_KEY'])
result = c.trigger_call(
    trigger_uuid=os.environ['DOGRAH_TRIGGER_UUID'],
    phone_number='+14155552671',
    initial_context={'crm_lead_id': 'TEST-001', 'name': 'Test Lead'},
    test_mode=True   # uses draft, won't make a real call if no telephony configured
)
print(result)
"

# 3. Run tests
python -m pytest dograh_integration/tests/ -v
```

---

## Notes on Local Dev Setup

- Dograh backend must be running: `bash scripts/start_services_dev.sh` from `/Users/anuraggupta/Documents/Projects/dograh`
- Docker services (Postgres, Redis, MinIO) must be up: `docker compose -f docker-compose-local.yaml up -d`
- Postgres is mapped to host port **5433** (not 5432 — native PG18 is on 5432)
- API health check: `curl http://localhost:8000/api/v1/health`
