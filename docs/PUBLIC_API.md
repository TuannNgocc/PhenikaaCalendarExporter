# Phenikaa Calendar Exporter — Public API v1

## Base URL

Use the base URL where the Phenikaa Calendar Exporter server is deployed.
All API responses are non-cacheable (`Cache-Control: no-store`).

## Authentication

Protected endpoints use:

`Authorization: Bearer <API_TOKEN>`

Create a token while signed in to the web application:

`POST /api/v1/tokens`

Send a JSON body with `name` and `scopes`, and include the authenticated
application session cookie plus a valid `X-CSRF-Token` header.

The token secret is returned only once. Store it securely. The database stores
a SHA-256 hash rather than the raw token. Revoked tokens can no longer be used.

Supported scopes:

- `me:read`
- `tokens:read`
- `tokens:write`
- `sessions:read`
- `sessions:write`
- `events:read`
- `sync:write`

## Response format

Successful JSON responses use:

`{"ok": true, "data": {...}}`

Errors use:

`{"ok": false, "error": {"code": "...", "message": "..."}}`

## Health and service status

- `GET /api/v1/health` — public health check.
- `GET /api/v1/status` — public service status; cached for 15 seconds.
- `GET /api/v1/status?refresh=1` — force a status refresh.
- `GET /api/v1/openapi.json` — OpenAPI 3.1 specification.

## One-shot calendar export

`POST /api/v1/export/one-shot?format=json`

This endpoint does not create a persistent session or request Google Calendar
authorization. The request body must be JSON and must be at most 1 MiB.

Example:

`{"start":"2026-09-01","end":"2027-01-31","userId":"YOUR_USER_ID","tokenJWT":"YOUR_CURRENT_TOKEN"}`

Provide either `bootstrapHtml` or both `userId` and `tokenJWT`. Dates accept
`YYYY-MM-DD` or `DD/MM/YYYY`.

Supported `format` values:

- `json` (default): summary and event data in JSON.
- `ics`: calendar file download.
- `xlsx`: spreadsheet download.
- `zip`: JSON, ICS, and XLSX files in one archive.

The one-shot endpoint is rate-limited to 10 requests per minute per client IP.
It does not persist the supplied Phenikaa credentials.

## API tokens

- `POST /api/v1/tokens` — create a token using the browser session and CSRF.
- `GET /api/v1/tokens` — list tokens (`tokens:read`).
- `POST /api/v1/tokens/{tokenId}/revoke` — revoke a token (`tokens:write`).
- `GET /api/v1/me` — token owner information (`me:read`).

## Sessions

- `GET /api/v1/sessions` — list the current token owner's sessions.
- `POST /api/v1/sessions` — create a session (`sessions:write`).
- `GET /api/v1/sessions/{sessionId}` — get session information.
- `PATCH /api/v1/sessions/{sessionId}` — update label, date range, or sync interval.
- `DELETE /api/v1/sessions/{sessionId}` — delete a session and clean up its resources.

Session creation and updates accept JSON fields `label`, `rangeStart`, `rangeEnd`,
and `syncIntervalHours`. Sync interval must be between 0.25 and 720 hours.

The API only exposes sessions owned by the authenticated token owner.

## Events, exports, and sync history

- `GET /api/v1/sessions/{sessionId}/events?start=YYYY-MM-DD&end=YYYY-MM-DD`
  — filter events from the latest stored calendar JSON export.
- `GET /api/v1/sessions/{sessionId}/exports` — list available stored exports.
- `GET /api/v1/sessions/{sessionId}/exports/{filename}`
  — download `calendar.json`, `calendar.ics`, or `calendar.xlsx`.
- `POST /api/v1/sessions/{sessionId}/sync` — request a sync (`sync:write`).
- `GET /api/v1/sessions/{sessionId}/sync/runs` — list recent sync runs.
- `GET /api/v1/sessions/{sessionId}/sync/runs/{runId}` — get a sync run.

## Common HTTP errors

- `400` — invalid request, date, format, or scope.
- `401` — missing, invalid, or expired authentication.
- `403` — insufficient scope or invalid CSRF.
- `404` — resource not found or not owned by the caller.
- `409` — resource conflict or sync could not be queued.
- `413` — request body too large.
- `429` — rate limit exceeded.
- `502` — upstream export or cleanup failure.

## OpenAPI

The machine-readable specification is served at `/api/v1/openapi.json`.
The source definition lives in `server/api_v1.py`.
