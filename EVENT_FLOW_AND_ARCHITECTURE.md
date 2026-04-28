# Event Flow and Architecture

## Event Flow (Sequence Diagram)

The sequence diagram shows how a single event moves through the system from ingestion to downstream delivery. The flow breaks into six phases:

1. **Event Ingestion:** A user action (e.g., asset CRUD, download, share) triggers an event, which the backend API records and stores.
2. **Preference Resolution:** The system resolves effective user preferences per channel and notification type.
3. **Batch Aggregation:** The system maps the raw event to a canonical notification type, computes one channel-specific grouping key per eligible channel, and evaluates whether an open batch exists. Each affected batch is either updated (count incremented and window extended) or created.
4. **Batch Finalization:** A background worker periodically scans for batches whose time window has expired and marks them as closed.
5. **Notification Generation:** Before insert and fanout, the system re-checks effective user preferences, then transforms eligible finalized items into notification payloads.
6. **Notification Delivery:** The notification is fanned out to downstream channels, including real-time in-app delivery and asynchronous email processing.

Time-based absence rules (for example, "share link not viewed in 5 days") are modeled as a small parallel watch lifecycle: create a watch on share link creation, cancel it on view, and fire it on deadline if still active.

This view is useful for understanding control flow and timing boundaries, especially where synchronous request handling ends and asynchronous processing begins.

```mermaid
sequenceDiagram
    participant U as User
    participant API as Backend API
    participant DB as Database
    participant B as Batcher Logic
    participant T as Time Watch Logic
    participant P as Preference Resolver
    participant W as Background Worker
    participant N as Notification Service
    participant WS as Web Client (WS/SSE)
    participant Q as Email Queue
    participant E as Email Worker

    %% Event Ingestion
    rect rgba(219, 234, 254, 0.5)
        Note over U, DB: Event Ingestion
        U->>API: Trigger event (e.g., create asset)
        API->>DB: Insert into notification_events
        alt Share link created
            API->>T: Create watch (fire_at = now + 5 days)
            T->>DB: Insert into notification_watches
        else Share link viewed
            API->>T: Cancel matching active watch
            T->>DB: Update watch status = cancelled
        end
    end

    %% Batch Aggregation
    rect rgba(237, 233, 254, 0.5)
        Note over API, DB: Batch Aggregation
        API->>B: Process event for batching
        B->>P: Resolve recipient channel preferences
        P->>DB: Query notification_preferences
        B->>B: Compute notification_type + group keys
        B->>DB: Lookup open in-app batch (2 min window)
        B->>DB: Lookup open email batch (10 min window)

        alt Matching batch exists for a channel
            B->>DB: Increment count + extend that channel's window_end
        else No open batch for a channel
            B->>DB: Create channel-specific batch
        end
    end

    %% Batch Finalization
    rect rgba(254, 249, 195, 0.5)
        Note over W, N: Batch Finalization
        loop Every ~30s
            W->>DB: Query channel-specific batches where window_end <= now AND status = open
            DB-->>W: Return expired batches
            W->>DB: Mark batch as closed
            W->>N: Send batch for notification generation
            W->>DB: Query watches where fire_at <= now AND status = active
            DB-->>W: Return expired watches
            W->>DB: Mark watch as fired
            W->>N: Send fired watch for notification generation
        end
    end

    %% Notification Generation
    rect rgba(237, 233, 254, 0.5)
        Note over N: Notification Generation
        N->>P: Re-check effective preferences
        P->>DB: Query notification_preferences
        N->>N: Construct channel-specific notification payload
    end

    %% Notification Delivery
    rect rgba(254, 226, 226, 0.5)
        Note over N, E: Notification Delivery
        N->>WS: Push in-app notification
        N->>Q: Enqueue email job
        E->>Q: Consume job
        E->>E: Send email via provider
    end
```

## Architecture Diagram

The sequence diagram above explains behavior over time. The architecture diagram below translates that behavior into concrete runtime boundaries, showing where responsibilities sit, which components own persistence, and how delivery fans out across realtime and asynchronous channels.

The four stores shown in the state layer are logical domains, not a requirement to run four separate databases.

```mermaid
flowchart TB

subgraph CLIENT["Client Layer"]
    UI["Web App / Client"]
end

subgraph API_LAYER["API Layer"]
    API["Backend API"]
end

subgraph PROCESSING["Processing Layer"]
    BAS["Batch Aggregation Service"]
    TWS["Time Watch Service"]
    PRS["Preference Resolver"]
    BFW["Batch Finalization Worker"]
    NS["Notification Service"]
end

subgraph STATE["State Layer"]
    direction LR
    ES[("Event Store")]
    PS[("Preference Store")]
    BS[("Batch Store")]
    WS2[("Watch Store")]
end

subgraph DELIVERY["Delivery Layer"]
    direction LR
    WS["Realtime Channel (WebSocket / SSE)"]
    EMAILQ["Email Queue"]
    EMAILW["Email Worker"]
    EMAIL["Email Provider"]
end

%% Event Ingestion
UI --> API
API --> ES
API --> BAS
API --> TWS
API --> PRS

%% Batch Aggregation & Finalization
BAS --> BS
BAS --> PRS
PRS --> PS
TWS --> WS2
BFW --> BS
BFW --> WS2
BFW --> NS
NS --> PRS

%% Notification Fanout
NS --> WS
WS --> UI
NS --> EMAILQ
EMAILQ --> EMAILW
EMAILW --> EMAIL

%% Styles
classDef clientStyle  fill:#DBEAFE,stroke:#3B82F6,color:#1E3A5F
classDef apiStyle     fill:#BFDBFE,stroke:#2563EB,color:#1E3A5F
classDef procStyle    fill:#EDE9FE,stroke:#7C3AED,color:#2E1065
classDef stateStyle   fill:#FEF9C3,stroke:#CA8A04,color:#422006
classDef delivStyle   fill:#FEE2E2,stroke:#DC2626,color:#450A0A

class UI clientStyle
class API apiStyle
class BAS,TWS,PRS,BFW,NS procStyle
class ES,PS,BS,WS2 stateStyle
class WS,EMAILQ,EMAILW,EMAIL delivStyle
```

> [!NOTE]
> The state domains are separated conceptually because they have different lifecycles and access patterns. They could be one PostgreSQL database (see [Data Model](DATA_MODEL.md)).

## Implementation in AWS

### Client Layer

| Component        | Primary Recommendation | Alternatives    | Notes                                                                |
| ---------------- | ---------------------- | --------------- | -------------------------------------------------------------------- |
| Web App / Client | S3 + CloudFront        | Amplify Hosting | Static hosting; communicates with API over HTTP and realtime channel |

### API Layer

| Component       | Primary Recommendation | Alternatives                                        | Responsibilities                                                        |
| --------------- | ---------------------- | --------------------------------------------------- | ----------------------------------------------------------------------- |
| Express Backend | ECS on Fargate + ALB   | EKS, EC2 ASG, Lambda + API Gateway (if runtime fit) | Event ingestion, orchestration, writes to state, publishing jobs/events |

### Processing Layer

| Component                 | Primary Recommendation                                     | Alternatives                                                | Responsibilities                                                                                                                                                                                                                                                                                                                             |
| ------------------------- | ---------------------------------------------------------- | ----------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Batch Aggregation Service | In-process in API (initially)                              | Dedicated ECS service behind internal queue at higher scale | Compute notification type, derive grouping keys, upsert open batches                                                                                                                                                                                                                                                                         |
| Time Watch Service        | In-process in API for create/cancel                        | Dedicated service if rule count grows                       | Create and cancel watch rows                                                                                                                                                                                                                                                                                                                 |
| Preference Resolver       | Shared library + read-through cache (ElastiCache optional) | Direct DB reads only                                        | Resolve effective preference for `(channel, notification_type)`. If cached, a preference update must invalidate or bypass the cache entry so that finalization re-checks the latest value — the two-phase preference check (at aggregation and again at finalization) provides a safety net for the window between a cache write and expiry. |
| Batch Finalization Worker | EventBridge Scheduler -> ECS Fargate worker                | EventBridge Scheduler -> Lambda                             | Scan expired batches/watches, close/fire, enqueue notification jobs                                                                                                                                                                                                                                                                          |
| Notification Service      | SQS-triggered ECS worker                                   | Lambda consumer                                             | Re-check preferences, build payloads, write finalized notifications, fanout                                                                                                                                                                                                                                                                  |

### State Layer

| Logical Store      | Recommendation | Why                                                                         |
| ------------------ | -------------- | --------------------------------------------------------------------------- |
| Event Store        | PostgreSQL     | Append-only audit stream, straightforward indexing and retention management |
| Preference Store   | PostgreSQL     | Simple relational overrides/defaults model                                  |
| Batch Store        | PostgreSQL     | Hottest mutable path; use `ON CONFLICT DO UPDATE` for atomic increments     |
| Watch Store        | PostgreSQL     | Deadline scans and cancellation lookups fit naturally in SQL                |
| Notification Store | PostgreSQL     | Durable user-facing record and auditability                                 |

### Delivery Layer

| Component        | Primary Recommendation             | Alternatives                 | Notes                                                                                                                          |
| ---------------- | ---------------------------------- | ---------------------------- | ------------------------------------------------------------------------------------------------------------------------------ |
| Realtime Channel | Socket.io/WebSocket on ECS service | API Gateway WebSocket        | ECS is simpler if API already runs there; API Gateway is stronger for fully decoupled realtime                                 |
| Email Queue      | Amazon SQS                         | EventBridge bus + SQS target | Buffering, retries, and backpressure                                                                                           |
| Email Worker     | SQS-triggered Lambda               | ECS worker                   | Lambda is simplest at lower to moderate steady volume. Must handle provider rate limits and quota exhaustion — see note below. |
| Email Provider   | Amazon SES                         | SendGrid, Mailgun            | Transactional email delivery; providers enforce per-second send rates and monthly quotas.                                      |

> [!NOTE]
> **Provider rate limits and quota exhaustion**
>
> The email worker must be resilient to two distinct provider-side constraints:
>
> - **Per-second send rate** — The provider rejects sends that exceed a rate limit (typically a `429` or equivalent). The worker should use exponential backoff with jitter on these responses rather than immediately retrying, which would re-hit the limit. SQS visibility timeout should be set high enough to absorb the backoff window.
> - **Monthly quota** — Once the quota is exhausted, all sends fail until the quota resets. The worker should detect this condition (distinct error code from a rate limit), stop consuming from the queue, and alert via CloudWatch alarm rather than burning retries into the DLQ. Jobs should remain in the queue so they drain automatically once quota resets.
>
> Both failure modes should route to the DLQ only when retries are genuinely exhausted — not on first contact with a provider-side limit. Quota exhaustion in particular should never be retried automatically; it requires operator intervention or quota increase.

## Operational Considerations

- **Observability** — Track key metrics like batch count, batch age, worker lag, queue depth, delivery failures, and notification volume. Use SQS dead-letter queues to isolate poison notification jobs so they don't block the main worker loop.
- **Queue access control** — The Notification Service IAM role should have `SendMessage` only on the email queue. The Email Worker role should have `ReceiveMessage`, `DeleteMessage`, and `GetQueueAttributes` only. No other principal should be able to publish to the queue.
- **Failure isolation** — Email failures shouldn't affect realtime delivery, and retries shouldn't modify already-closed batches.
- **Scalability path** — Start with aggregation inside the API, then scale finalization and notifications independently as background load increases.
- **Time-based correctness** — Watch cancellation and firing must be idempotent and mutually exclusive so late view events cannot fire duplicate notifications.

## Worker Concurrency

The batch finalization worker runs on a schedule and must be safe to run as multiple concurrent instances — for example, multiple ECS tasks or Lambda invocations triggered at the same time by EventBridge.

**Problem:** Without coordination, two worker instances could both claim the same expired batch, close it twice, and generate duplicate notifications.

**Solution: `SELECT ... FOR UPDATE SKIP LOCKED`**

Each worker instance claims a bounded set of expired rows in a single atomic statement:

```sql
SELECT id FROM notification_batches
WHERE window_end <= now()
  AND status = 'open'
ORDER BY window_end
LIMIT 100
FOR UPDATE SKIP LOCKED;
```

`SKIP LOCKED` causes each worker to bypass rows that another worker has already locked, so two concurrent instances process disjoint sets of batches with no coordination overhead and no blocking. The same pattern applies to the expired-watch scan, substituting `notification_watches`, `fire_at`, and `status = 'active'`.

After claiming rows, each worker closes or fires them within the same transaction before releasing the locks. Because the status transition (`open → closed`, `active → fired`) is the same transaction as the lock, a crash mid-flight leaves the rows locked only for the duration of the transaction; PostgreSQL releases the lock automatically on rollback, and the next worker run picks them up.

**Additional guards:**

- Check the expected current status in the `WHERE` clause of the `UPDATE` as a backstop against any edge-case double-claim.
- Notification insert should use `ON CONFLICT DO NOTHING` keyed on `(batch_id, channel)` or `(watch_id, channel)` as a final backstop against duplicate delivery rows.

## Real-Time Channel Scaling

WebSocket connections are stateful and pinned to a specific server instance. When the API runs as multiple ECS tasks behind a load balancer, a push from one task will not reach a client connected to a different task.

**Problem:** The notification service writes a finalized notification and needs to push it to the user's active WebSocket connection — but any of the `n` API tasks could hold that connection.

**Solution: Redis pub/sub fan-out**

Introduce a Redis pub/sub channel per user (e.g., `notifications:{recipient_id}`). Every API task subscribes to its connected users' channels at connection time and unsubscribes on disconnect.

```
Notification Service
        │
        ▼
Redis PUBLISH notifications:{recipient_id}
        │
        ├── API Task 1 (subscribed, no connection for this user) — no-op
        ├── API Task 2 (subscribed, holds WS connection) — pushes to client ✓
        └── API Task 3 (subscribed, no connection for this user) — no-op
```

Only the task holding the client's socket delivers the push; the others receive and discard. This keeps the API stateless from the load balancer's perspective while allowing any backend component to trigger a real-time push.

For the AWS deployment, use ElastiCache for Redis (single-shard, cluster mode disabled) as the pub/sub broker. ALB sticky sessions are optional — they reduce no-op publishes but aren't required for correctness. Clients should fall back to polling `/notifications/summary` on reconnect to recover any missed pushes.

**WebSocket authentication:**

The bearer token used for REST requests must also authenticate the WebSocket connection. Authentication is enforced at the HTTP upgrade handshake, not per-message:

1. Client sends the upgrade request with the bearer token in the `Authorization` header (or as a query parameter for clients that cannot set headers on WebSocket connections — the token must be short-lived and single-use in that case).
2. The server validates the token before completing the upgrade. If validation fails, the server returns `401` and the connection is not opened.
3. After the upgrade, the authenticated user identity is bound to the connection for its lifetime. No re-authentication per message is required.
4. Token expiry during a long-lived connection: the server must close the connection with code `4001` (application-defined) when it detects the bound token has expired. Clients should reconnect with a fresh token.

Note: tokens passed as query parameters appear in server access logs and browser history; prefer the `Authorization` header where the client library allows it.
