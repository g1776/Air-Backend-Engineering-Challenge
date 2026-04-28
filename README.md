"# Air-Backend-Engineering-Challenge"

## Core Design Decision for batching notifications

For this notification system, I would model notification batching as a mutable, time-extended window keyed by (user_id, event_type, group_id) where each event either updates an existing open batch or creates a new one, and batch closure is handled asynchronously.

## Event-Driven Architecture

```mermaid
sequenceDiagram
participant U as User
participant API as Backend API
participant DB as Database
participant B as Batcher Logic
participant W as Background Worker
participant N as Notification Service
participant WS as Web Client (WS/SSE)
participant Q as Email Queue
participant E as Email Worker

    %% --- Event Ingestion ---
    U->>API: Trigger event (e.g., create asset)
    API->>DB: Insert into notification_events

    %% --- Batch Aggregation ---
    API->>B: Process event for batching
    B->>DB: Lookup open batch (by batch_key)

    alt Open batch exists and active
        B->>DB: Increment count + extend window_end
    else No open batch
        B->>DB: Create new batch (window_end = now + 2 min)
    end

    %% --- Batch Finalization ---
    loop Every ~30s
        W->>DB: Query batches where window_end <= now AND status = open
        DB-->>W: Return expired batches
        W->>DB: Mark batch as closed
        W->>N: Send batch for notification generation
    end

    %% --- Notification Generation ---
    N->>N: Construct notification payload

    %% --- Fanout ---
    N->>WS: Push in-app notification
    N->>Q: Enqueue email job

    %% --- Email Delivery ---
    E->>Q: Consume job
    E->>E: Send email via provider
```
