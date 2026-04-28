# Air-Backend-Engineering-Challenge

This design models notifications as two first-class trigger types sharing one delivery pipeline:

- activity-based summaries through mutable, time-extended batches
- time-based deadline watches for expected activity that never occurs

Incoming events are ingested synchronously, mapped to either batch updates or watch state transitions, and finalized asynchronously before fanout to delivery channels.

> [!IMPORTANT]
> The core objective is to keep the write path simple and low-cost while still producing timely, user-friendly notifications. The system does this by maintaining minimal mutable state (open batches and active watches), then moving finalization and delivery to asynchronous workers.

## Design Principles

- **Single open batch per key** — avoids broad locking and keeps state management localized
- **Sliding aggregation window** — extends `window_end` as new events arrive, so bursts collapse into a single notification
- **Deadline watch lifecycle** — creates, cancels, and fires watches for time-based conditions (for example, share link not viewed in 5 days)
- **Asynchronous finalization** — removes delivery work from the hot path and improves API responsiveness
- **Low-contention writes** — event processing updates only the relevant batch or watch state for the current key/resource
- **Channel decoupling** — realtime and email delivery are fanout concerns, not part of ingestion-time logic

## Trigger Model

The design intentionally supports both trigger classes upfront:

- **Activity-based trigger:** group noisy event bursts into a single summarized notification via an open batch window
- **Time-based trigger:** create a watch at time T, cancel on expected activity, or notify at `fire_at` if no activity occurs

Example time-based rule:

- **Asset Share Link Not Viewed** — when a newly created share link receives no view event in 5 days, generate a notification

Both trigger types converge into the same notification generation and delivery fanout path.

## Scaling and Extensibility

The system is designed to scale by keeping the synchronous write path narrow and pushing aggregation finalization and delivery into background work. In practice, that creates a straightforward scaling path:

- keep event ingestion lightweight and synchronous inside the API
- scale batch finalization workers independently from the API fleet
- scale notification fanout and email delivery independently from both ingestion and finalization
- partition mutable state by recipient, channel, and grouping key so hot activity stays localized to the affected batch rows

This model is also extensible because notification rules are driven by canonical event fields and grouping policy rather than by asset-specific tables or hardcoded one-off flows. The current examples use assets, boards, comments, and share links, but the same pattern extends cleanly to other domains such as collections, workspaces, approvals, review tasks, or external integrations.

Supporting a new notification pattern typically means adding three things:

1. A new canonical `notification_type` and, if needed, a new raw event mapping.
2. A grouping rule for each channel, such as `user + workspace`, `project`, or `approval_request`.
3. A payload builder that renders the finalized batch or fired watch into the user-facing notification.

The existing model already supports that evolution because `notification_events` stores a canonical subject, optional container, and raw trigger resource, while `notification_batches` stores channel-specific grouping state rather than a single fixed batch formula. That means the system can move beyond assets and boards without redesigning the storage model each time a new product surface appears.

If notification policy becomes large enough to manage outside application code, the current structure can also absorb a future policy/configuration layer without changing the batch, watch, or notification lineage described in the rest of this design.

## Sections

- [Event Flow and Architecture](EVENT_FLOW_AND_ARCHITECTURE.md)
- [Data Model](DATA_MODEL.md)
- [API Design](API_DESIGN.md)
