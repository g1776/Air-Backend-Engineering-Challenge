# Air-Backend-Engineering-Challenge

This system delivers user notifications from two trigger classes through one pipeline:

- activity bursts, summarized with sliding-window batches
- time-based absence conditions, modeled as watches

The write path stays lightweight: events are recorded synchronously, while finalization and delivery run asynchronously.

## Core Flow

1. Ingest event and write immutable event log row.
2. Resolve recipient preferences.
3. Update or create channel-specific open batches, and create/cancel watches when applicable.
4. Background workers close expired batches and fire eligible watches.
5. Generate finalized notifications and deliver by channel (in-app, email).

## Key Design Decisions

- Event log is append-only for auditability and replay safety.
- Batches are channel-specific to support different grouping rules and debounce windows.
- Preferences are enforced at aggregation and again at finalization to handle mid-window opt-out.
- Idempotent worker transitions protect against retry duplicates.
- PostgreSQL is the default state store; DynamoDB is an optional scale optimization for hot batch paths.

## Representative Scenarios

### Burst activity, same user

UserA uploads many assets quickly.

- In-app groups by `user + board`.
- Email groups by `board`.
- Expired windows produce summarized notifications instead of per-event noise.

### Time-based reminder

A share link is created and never viewed.

- A watch is created on share-link creation.
- If no canceling view event arrives before deadline, the watch fires.
- Eligible channels receive reminder notifications.

### Preference change mid-window

User disables email for a notification type after events were already batched.

- Events remain recorded for auditability.
- Finalization re-checks preferences.
- Email is suppressed while other enabled channels can still deliver.

## Sections

- [Event Flow and Architecture](EVENT_FLOW_AND_ARCHITECTURE.md)
- [Data Model](DATA_MODEL.md)
- [API Design](API_DESIGN.md)
