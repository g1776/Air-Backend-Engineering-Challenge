# Air-Backend-Engineering-Challenge

This design models notification batching as a **mutable, time-extended window** keyed by `(user_id, event_type, group_id)`. Incoming events are ingested synchronously, aggregated into an open batch when possible, and finalized asynchronously once the batching window expires.

> [!IMPORTANT]
> The core design objective is to keep the write path simple and cheap while still producing user-friendly summary notifications. The system does that by maintaining at most one open batch per key, extending the window on new activity, and moving finalization and delivery to asynchronous workers.

## Design Principles

- **Single open batch per key** — avoids broad locking and keeps state management localized
- **Sliding aggregation window** — extends `window_end` as new events arrive, so bursts collapse into a single notification
- **Asynchronous finalization** — removes delivery work from the hot path and improves API responsiveness
- **Low-contention writes** — event processing updates only the relevant batch state for the current key
- **Channel decoupling** — realtime and email delivery are fanout concerns, not part of ingestion-time logic

## Sections

- [Event Flow and Architecture](EVENT_FLOW_AND_ARCHITECTURE.md)
- [Data Model](DATA_MODEL.md)
- [API Design](API_DESIGN.md)
