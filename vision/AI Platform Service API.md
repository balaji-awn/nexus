# AI Platform Service API (assumed contract)

This is the contract Nexus will code against. It's based on what we know about the AI Platform Service:

- agents are pre-created on the platform
- an agent is invoked by name, with the tenant in a header
- invocation is async: the call returns an execution ID, which is polled with exponential backoff
- a catalog lists all agents

Paths, header names and field names below are **assumptions**. Nexus keeps all of them in one adapter module plus config, so when the real API differs, only that module changes.

## Conventions

- Base URL: configured per environment (`AI_PLATFORM_BASE_URL`), e.g. `http://ai-platform-service.<namespace>.svc.cluster.local`
- Content type: `application/json`
- Headers on every call:

| Header | Required | Meaning |
| --- | --- | --- |
| `X-Tenant-Id` | yes | Tenant the call runs for. Taken from the incoming Nexus request. |
| `Authorization` | assumed | Service credential for Nexus (`Bearer <token>`), if the platform requires one. |
| `X-Request-Id` | no | Nexus request ID, for cross-service tracing. |

## Catalog

### `GET /agents`

Lists every agent. The catalog is the same for all tenants. Assumed to be paginated.

Query parameters: `page_token` (optional), `page_size` (optional, default 100).

```json
{
  "agents": [
    {
      "name": "ip-reputation-lookup",
      "description": "Looks up the reputation of an IPv4/IPv6 address across threat-intel feeds.",
      "input_schema": {
        "type": "object",
        "properties": { "ip": { "type": "string" } },
        "required": ["ip"]
      },
      "output_schema": {
        "type": "object",
        "properties": {
          "ip": { "type": "string" },
          "verdict": { "type": "string", "enum": ["malicious", "suspicious", "clean", "unknown"] },
          "sources": { "type": "array", "items": { "type": "string" } }
        }
      },
      "model": "provider/model-name",
      "version": "3",
      "updated_at": "2026-10-30T12:00:00Z"
    }
  ],
  "next_page_token": null
}
```

Nexus uses `version` and/or `updated_at` to detect changed agents during catalog sync.

### `GET /agents/{agent_name}`

Returns one agent in the same shape as an item above. `404` if it doesn't exist.

## Executions

### `POST /agents/{agent_name}/executions`

Submits an execution. The body is the agent's input and must match its `input_schema`.

Request body:

```json
{ "input": { "ip": "1.2.3.4" } }
```

Response `202 Accepted`:

```json
{ "execution_id": "exe_01HZX...", "status": "queued" }
```

### `GET /executions/{execution_id}`

Polls an execution.

```json
{
  "execution_id": "exe_01HZX...",
  "agent_name": "ip-reputation-lookup",
  "status": "succeeded",
  "output": { "ip": "1.2.3.4", "verdict": "malicious", "sources": ["feed-a", "feed-b"] },
  "error": null,
  "created_at": "2026-11-09T10:00:00Z",
  "completed_at": "2026-11-09T10:00:04Z"
}
```

`status` values:

| Status | Terminal | Meaning |
| --- | --- | --- |
| `queued` | no | Accepted, not started. |
| `running` | no | In progress. |
| `succeeded` | yes | `output` is set and matches the agent's `output_schema`. |
| `failed` | yes | `error` is set. |
| `cancelled` | yes | Cancelled before completing (only if cancellation exists). |

When `failed`:

```json
{ "error": { "code": "AGENT_ERROR", "message": "Upstream feed timed out", "retryable": true } }
```

### `DELETE /executions/{execution_id}` (if supported)

Cancels a running execution. Nexus uses it when a request hits its budget or is abandoned. If the platform has no cancellation, Nexus just stops polling.

## Errors (non-2xx)

```json
{ "error": { "code": "INVALID_INPUT", "message": "input.ip is required" } }
```

| HTTP | Nexus behaviour |
| --- | --- |
| `400` / `422` | Input invalid: do not retry; report as a failed step. |
| `401` / `403` | Configuration error: fail the request and alert. |
| `404` | Unknown agent or execution: trigger a catalog resync; fail the step. |
| `429` | Rate limited: honour `Retry-After`, then retry. |
| `5xx`, network errors | Retry with backoff, up to a limit. |

## Polling strategy (Nexus side)

- First poll after **0.5 s**, then the interval doubles each time, capped at **10 s** per interval, with ±20% jitter.
- Honour `Retry-After` on any poll response that has one.
- Each agent has an overall **timeout** (default 120 s, configurable per agent). When it's hit, Nexus cancels if possible and marks the step as timed out.
- Polling is done by one shared poller in each Nexus process, not a loop per call, so thousands of open executions don't each hold a coroutine and a connection.
- Submission is **not** retried blindly: a submit that times out may still have created an execution. A submit is retried only when the response proved it wasn't accepted (`4xx` other than `429`, or connection refused).
- Execution IDs are saved in the LangGraph checkpoint as soon as they're returned. A restarted Nexus resumes polling those IDs instead of resubmitting, so no agent call runs twice.

## Things to confirm with the platform team

- Actual paths, header names and field names
- Catalog pagination and any change feed or "updated since" filter
- Whether cancellation exists
- Rate and concurrency limits per tenant or per caller, and whether `Retry-After` is sent
- How long execution results stay readable after completion
- Whether `output` is guaranteed to match `output_schema`, or whether Nexus must validate (Nexus will validate regardless)
