# API Design

> [!IMPORTANT]
> Base path: `/v1`
>
> Auth model: bearer token, self-scoped endpoints.
>
> All connections must use TLS 1.2 or later. HTTP requests must be rejected or permanently redirected to HTTPS at the load balancer.
>
> CORS: the API returns `Access-Control-Allow-Origin` restricted to the application's known origin(s). Wildcard origins (`*`) are not permitted. Preflight requests for mutation endpoints must be handled.

This document specifies the REST contract for:

- notification feed rendering
- summary/badge counts
- seen/read state transitions
- per-channel and per-type preference management

## Quick Navigation

- [Endpoint Matrix](#endpoint-matrix)
- [Common Types](#common-types)
- [Endpoints](#endpoints)
- [Error Contract](#error-contract)
- [Operational Semantics](#operational-semantics)

## Endpoint Matrix

| Method | Path                                                    | Purpose                                      |
| ------ | ------------------------------------------------------- | -------------------------------------------- |
| GET    | /notifications                                          | Paginated notification feed                  |
| GET    | /notifications/summary                                  | Unseen/unread badge summary                  |
| POST   | /notifications/actions/mark-seen                        | Bulk mark selected notifications as seen     |
| POST   | /notifications/actions/mark-read                        | Bulk mark selected notifications as read     |
| POST   | /notifications/actions/mark-all-read                    | Bulk mark all matching notifications as read |
| GET    | /notification-preferences                               | Get effective defaults + overrides           |
| PUT    | /notification-preferences/{channel}/{notification_type} | Upsert preference override                   |
| DELETE | /notification-preferences/{channel}/{notification_type} | Remove preference override                   |

## Common Types

### Enums

| Name              | Values                                                                                                |
| ----------------- | ----------------------------------------------------------------------------------------------------- |
| channel           | in_app, email                                                                                         |
| notification_type | asset_added, comment_created, assets_downloaded, asset_share_link_viewed, asset_share_link_not_viewed |

### Notification Object

| Field             | Type              | Notes                                                                                                                                                                                              |
| ----------------- | ----------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| id                | uuid              | Notification ID                                                                                                                                                                                    |
| recipient_id      | uuid              | Authenticated user ID                                                                                                                                                                              |
| channel           | enum              | in_app or email                                                                                                                                                                                    |
| notification_type | enum              | Canonical notification type                                                                                                                                                                        |
| event_count       | integer           | Aggregated event count; 0 for watch-fired notifications                                                                                                                                            |
| is_seen           | boolean           | `true` once the notification has appeared in the user's feed view. Distinct from read: seen means visible, read means the user actively opened or engaged with it.                                 |
| read_at           | timestamp \| null | Timestamp the user explicitly marked the notification as read; `null` if unread.                                                                                                                   |
| created_at        | timestamp         | Creation time                                                                                                                                                                                      |
| payload           | object            | Render-ready message data; all string fields must be treated as plain text and HTML-escaped before rendering. The server produces sanitized text — callers must not trust this field as safe HTML. |

### Summary Object

| Field                | Type              | Notes                              |
| -------------------- | ----------------- | ---------------------------------- |
| total_unseen         | integer           | Cross-channel unseen total         |
| total_unread         | integer           | Cross-channel unread total         |
| by_channel           | object            | Per-channel counters               |
| last_notification_at | timestamp \| null | Most recent notification timestamp |

### Preference Override Object

| Field             | Type              | Notes                           |
| ----------------- | ----------------- | ------------------------------- |
| channel           | enum              | in_app or email                 |
| notification_type | enum \| \_default | \_default means channel default |
| enabled           | boolean           | Whether delivery is enabled     |
| mute_until        | timestamp \| null | Temporary mute end              |
| updated_at        | timestamp         | Last update timestamp           |

---

## Endpoints

### GET /notifications

Returns the authenticated user's notification feed.

Query Parameters:

| Name              | Type    | Required | Notes                                |
| ----------------- | ------- | -------- | ------------------------------------ |
| channel           | enum    | No       | Filter by channel                    |
| notification_type | enum    | No       | Filter by type                       |
| seen              | boolean | No       | Filter by seen state                 |
| read              | boolean | No       | Filter by read state                 |
| cursor            | string  | No       | Opaque cursor from previous response |
| limit             | integer | No       | Default 20, max 100                  |

Response 200:

```json
{
	"items": [
		{
			"id": "c7276de0-13c4-4f16-91ca-f96304f0565d",
			"recipient_id": "e47ef7e4-2d1f-4fa0-8b74-f181de6eaac9",
			"channel": "in_app",
			"notification_type": "assets_downloaded",
			"event_count": 12,
			"is_seen": false,
			"read_at": null,
			"created_at": "2026-04-28T20:45:00Z",
			"payload": {
				"title": "Your assets were downloaded",
				"body": "12 downloads on Board A"
			}
		}
	],
	"page": {
		"next_cursor": "eyJjcmVhdGVkX2F0Ijoi...",
		"has_more": true
	}
}
```

<details>
<summary>Pagination and ordering semantics</summary>

- Sort: created_at DESC, id DESC
- Cursor boundary: (created_at, id)
- Cursor is opaque and must be treated as a token, not parsed client-side
- Cursors are HMAC-signed server-side (e.g., HMAC-SHA256 with a rotating secret from AWS Secrets Manager). A cursor that fails signature verification is rejected with `400 INVALID_CURSOR`. This prevents parameter tampering that could otherwise bypass the `recipient_id` boundary or skip arbitrary rows.

</details>

### GET /notifications/summary

Returns badge counts and channel rollups.

Query Parameters:

| Name    | Type | Required | Notes                                       |
| ------- | ---- | -------- | ------------------------------------------- |
| channel | enum | No       | If provided, response may be channel-scoped |

Response 200:

```json
{
	"total_unseen": 7,
	"total_unread": 4,
	"by_channel": {
		"in_app": {
			"unseen": 7,
			"unread": 4
		},
		"email": {
			"unseen": 0,
			"unread": 0
		}
	},
	"last_notification_at": "2026-04-28T20:45:00Z"
}
```

### POST /notifications/actions/mark-seen

Bulk mark selected notifications as seen.

Request:

```json
{
	"notification_ids": [
		"c7276de0-13c4-4f16-91ca-f96304f0565d",
		"f5b24c42-b06f-4988-8030-ef49ec57f2e9"
	]
}
```

Notes:

- `notification_ids` must not be empty and is capped at **100 IDs per request**. Requests exceeding this limit are rejected with `400 VALIDATION_ERROR` before any database access. This prevents large `IN (...)` clauses from being used as a denial-of-service vector.
- Each ID must be a valid UUID v4; malformed values are rejected at input validation, not at query time.

Response 200:

```json
{
	"updated_count": 2
}
```

### POST /notifications/actions/mark-read

Bulk mark selected notifications as read.

Request:

```json
{
	"notification_ids": ["c7276de0-13c4-4f16-91ca-f96304f0565d"]
}
```

Notes:

- Same array size constraint as mark-seen: 1–100 UUIDs per request.

Response 200:

```json
{
	"updated_count": 1,
	"read_at": "2026-04-28T21:00:00Z"
}
```

### POST /notifications/actions/mark-all-read

Bulk mark all matching notifications as read.

Request:

```json
{
	"channel": "in_app",
	"before": "2026-04-28T21:00:00Z"
}
```

Notes:

- `channel` and `before` are optional filters, but the server imposes an internal page size of 1,000 rows per call. If more rows match, the response includes `"has_more": true` and the client must call again.
- Without `before`, the cutoff defaults to `now()` at request time, preventing unbounded scans against recently created notifications.
- Without `channel`, the update applies across all channels; callers should prefer supplying `channel` when the intent is channel-specific.

Response 200:

```json
{
	"updated_count": 42,
	"read_at": "2026-04-28T21:00:00Z"
}
```

### GET /notification-preferences

Returns effective preference shape: channel defaults and type-specific overrides.

Response 200:

```json
{
	"defaults": {
		"in_app": {
			"enabled": true,
			"mute_until": null
		},
		"email": {
			"enabled": true,
			"mute_until": null
		}
	},
	"overrides": [
		{
			"channel": "email",
			"notification_type": "assets_downloaded",
			"enabled": false,
			"mute_until": null,
			"updated_at": "2026-04-28T20:00:00Z"
		}
	]
}
```

### PUT /notification-preferences/{channel}/{notification_type}

Upsert one preference override.

Path Parameters:

| Name              | Type              | Notes                             |
| ----------------- | ----------------- | --------------------------------- |
| channel           | enum              | in_app or email                   |
| notification_type | enum \| \_default | \_default updates channel default |

Request:

```json
{
	"enabled": false,
	"mute_until": null
}
```

Response 200:

```json
{
	"channel": "email",
	"notification_type": "assets_downloaded",
	"enabled": false,
	"mute_until": null,
	"updated_at": "2026-04-28T20:00:00Z"
}
```

### DELETE /notification-preferences/{channel}/{notification_type}

Delete one explicit override and fall back to default resolution.

Response 204.

---

## Error Contract

Standard error envelope:

```json
{
	"error": {
		"code": "VALIDATION_ERROR",
		"message": "notification_ids must not be empty",
		"details": {
			"field": "notification_ids"
		},
		"request_id": "4e489352-23e9-4f57-aa8a-a563fd61adcf"
	}
}
```

Status codes:

| Code | Meaning                                            |
| ---- | -------------------------------------------------- |
| 400  | Validation failure                                 |
| 401  | Unauthenticated                                    |
| 403  | Unauthorized resource access                       |
| 404  | Resource not found                                 |
| 409  | Conflict (for example preference version mismatch) |
| 429  | Rate limited                                       |
| 500  | Internal error                                     |

---

## Operational Semantics

> [!NOTE]
> Events remain immutable and recorded regardless of user preference changes. Preference checks affect aggregation eligibility and final delivery eligibility.

<details>
<summary>Consistency</summary>

- Feed reads are read-after-write for the same user in normal operation.
- Summary counts may be briefly eventually consistent if cached.

</details>

<details>
<summary>Idempotency and retries</summary>

- Bulk action endpoints are idempotent by final state.
- Clients may provide Idempotency-Key on mutation endpoints for retry safety.

</details>

<details>
<summary>Authorization and ownership</summary>

All endpoints are self-scoped: the authenticated user's ID is extracted from the bearer token and injected server-side into every query. Clients never pass a `recipient_id` parameter.

Enforcement rules by endpoint:

| Endpoint                                                                     | Enforcement                                                                                                                                                      |
| ---------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| GET /notifications, GET /notifications/summary                               | `WHERE recipient_id = $auth_user_id` applied at query layer                                                                                                      |
| POST /notifications/actions/mark-seen, POST /notifications/actions/mark-read | Server filters `notification_ids` to those owned by the caller before updating; unrecognized IDs are silently excluded (not a 403) to avoid confirming existence |
| POST /notifications/actions/mark-all-read                                    | Same ownership scope; no caller-supplied recipient parameter accepted                                                                                            |
| /notification-preferences (all methods)                                      | All reads and writes scoped to `$auth_user_id`; path params validated against allowed enum values before any DB access                                           |

UUID v4 IDs are unpredictable, but the server-side ownership filter is the authoritative guard — not ID unguessability.

</details>

<details>
<summary>Rate limiting guidance</summary>

Limits are enforced per authenticated user at the API gateway or load balancer layer. Exceeding any limit returns `429` with a `Retry-After` header. The values below are illustrative starting points and should be tuned based on observed traffic.

| Endpoint group                      | Limit       |
| ----------------------------------- | ----------- |
| GET /notifications                  | 60 req/min  |
| GET /notifications/summary          | 120 req/min |
| POST /notifications/actions/\*      | 20 req/min  |
| GET /notification-preferences       | 60 req/min  |
| PUT /notification-preferences/\*    | 10 req/min  |
| DELETE /notification-preferences/\* | 10 req/min  |

Additionally, apply a per-IP limit (e.g., 300 req/min) before authentication is resolved, as a first-line defense against unauthenticated abuse. This limit should be more generous than the per-user limits to avoid impacting legitimate traffic behind shared NAT.

Prefer WebSocket/SSE push to drive badge updates rather than client polling of `/notifications/summary`, which is the most likely polling target.

</details>
