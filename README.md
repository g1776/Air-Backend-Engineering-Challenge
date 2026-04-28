# Air Backend Engineering Challenge

A notification system design for Air, covering two trigger classes, a shared delivery pipeline, and a REST API for consuming notifications.

## Documents

- [Event Flow and Architecture](EVENT_FLOW_AND_ARCHITECTURE.md) — sequence diagram, AWS component mapping, worker concurrency, and real-time delivery
- [Data Model](DATA_MODEL.md) — Relational database schema with grouping policy, lifecycle semantics, and concurrency guarantees
- [API Design](API_DESIGN.md) — REST contract for the notification feed, preference management, and operational semantics

## System Overview

The system handles two distinct trigger classes through a shared pipeline:

- **Activity bursts** — incoming events are aggregated into sliding-window batches, grouped by channel-specific rules (e.g., `user + board` for in-app, `board` for email). A background worker closes expired batches and finalizes notifications.
- **Time-based absences** — a watch is created when an event should trigger a future reminder. If no canceling event arrives before the deadline, the watch fires and a notification is generated.

The write path is kept lightweight: events are recorded synchronously; finalization and delivery run asynchronously.

## Core Flow

1. Ingest event and write an immutable event log row.
2. Resolve recipient preferences.
3. Upsert channel-specific open batches; create or cancel watches as applicable.
4. Background workers close expired batches and fire eligible watches.
5. Finalize notifications and deliver by channel (in-app via WebSocket, email via SQS + Lambda).

## Key Design Decisions

- **Append-only event log** — all raw events are preserved for auditability and replay safety.
- **Channel-specific batches** — in-app and email have different grouping keys and debounce windows, so each channel gets its own batch row.
- **Two-phase preference check** — preferences are enforced at aggregation and again at finalization, so a mid-window opt-out is always respected.
- **Idempotent worker transitions** — `SKIP LOCKED` and `ON CONFLICT DO NOTHING` protect against double-firing under concurrent workers.
- **PostgreSQL for all state** — a single store keeps the deployment simple and the transactional guarantees strong; see [Data Model](DATA_MODEL.md) for the schema.

## Representative Scenarios

### Burst activity

A user uploads many assets in quick succession.

- In-app batch groups by `user + board`; email batch groups by `board`.
- Expired windows produce one summarized notification per channel instead of per-event noise.

### Time-based reminder

A share link is created and never viewed.

- A watch is created on share-link creation.
- If no canceling view event arrives before the deadline, the watch fires.
- Eligible channels receive a reminder notification.

### Preference change mid-window

A user disables email for a notification type after events have already been batched.

- Raw events remain recorded for auditability.
- Finalization re-checks preferences before generating the notification.
- Email is suppressed; other enabled channels still deliver.

## Trade-offs and Scope Boundaries

### What I chose and why

**PostgreSQL for everything.** Using a single relational store keeps deployment simple and lets the schema express relationships and constraints directly. The trade-off is that high write volume on `notification_batches` (many concurrent users producing events) would eventually push against PostgreSQL's row-lock throughput. A dedicated key-value store (e.g., DynamoDB or Redis) on the hot upsert path would help at that scale, but adds operational complexity that isn't justified at the problem size described here.

**Synchronous event ingestion.** Events are written to the DB in the request path so the response can confirm receipt. The trade-off is that a slow batch upsert under a traffic spike adds latency to the caller. An alternative is to publish events to a queue and write asynchronously, but that makes receipt acknowledgement weaker and adds a layer of infra. The current approach favors simplicity and correctness.

**EventBridge Scheduler for finalization.** A managed scheduler avoids running a polling loop, but it requires one scheduled rule per active batch/watch, which creates overhead if the number of open windows is very large. A single polling worker with a short tick interval is simpler operationally and easier to reason about at moderate scale; the design notes this trade-off without prescribing a hard cutover point.

**Redis pub/sub for WebSocket fan-out.** Redis is introduced solely to support multi-instance WebSocket delivery. If the service runs on a single instance, Redis is unnecessary. The design calls it optional for that reason and notes that sticky sessions reduce no-op publishes without affecting correctness.

### What I didn't design

**Schema migration strategy.** The spec defines the target schema but says nothing about how it gets there — no migration tooling, no versioning approach, no zero-downtime column additions. In practice this would use something like `node-pg-migrate` or Flyway with backward-compatible migrations applied before a deploy.

**Token validation mechanics.** The API requires a Bearer token on every request and scopes all data access to the authenticated user, but the spec doesn't define whether tokens are JWTs (validated locally) or opaque session tokens (validated against a session store). The right choice depends on Air's existing auth infrastructure.

**`notification_events` retention.** The table is append-only and grows indefinitely. A production system would need a retention policy — either a rolling delete (e.g., keep 90 days) or archival to cold storage (e.g., S3 + Athena) — along with table partitioning by `created_at` to keep query performance stable as the table grows.

**Push notifications (mobile).** The design covers in-app (WebSocket) and email. A native mobile channel would follow the same pipeline — add a `push` channel row in `notification_batches`, finalize the same way, and deliver via APNs/FCM. The shape of the extension is clear but the channel itself was out of scope.

**Multi-tenancy and org-level permissions.** The design assumes `recipient_id` maps directly to an authenticated user in a single-tenant context. If Air's permission model involves workspace roles, org-level notification routing, or notification delegation, those concepts would need to be surfaced in the schema and the preference model.
