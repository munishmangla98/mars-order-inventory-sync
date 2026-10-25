# Order & Inventory Sync Service — System Design

## 1. Objective

Design a service that keeps inventory consistent across the MARS Cosmetics website, Nykaa, Flipkart, and Blinkit, while integrating with Microsoft Dynamics 365 Business Central.

The service must prevent overselling, handle duplicate events, respect channel-specific API limits, and recover from temporary failures without losing inventory updates.

## 2. Assumptions

- PostgreSQL is the authoritative source for stock quantities and reservations.
- Each SKU has a unique internal identifier, with mappings to external marketplace identifiers.
- The website uses the inventory service directly; external marketplaces communicate through adapters.
- Marketplace APIs may time out, return errors, or temporarily become unavailable.
- Stock updates should normally be propagated within 60 seconds. This target cannot be guaranteed during an external channel outage.
- The stated channel limits are treated as design constraints and must be confirmed against current API documentation before implementation.
- Business Central integration supports a stable external reference or a lookup mechanism for reconciling uncertain requests.
- No live API credentials or production access are assumed.

## 3. Architecture

The service consists of an API layer, PostgreSQL, Redis, background workers, marketplace adapters, and an ERP adapter.

PostgreSQL stores inventory, reservations, order records, processed-event identifiers, and durable outbound jobs. Redis supports caching and rate limiting but is not the source of truth for stock.

The outbox pattern ensures that a business transaction and its corresponding outbound work are committed together. Workers process pending jobs and retry transient failures with backoff.

See `architecture.mmd` for the diagram and `stock_reservation_pseudocode.md` for the reservation algorithm.

## 4. Data Model

### Inventory

- `sku_id`
- `on_hand_qty`
- `reserved_qty`
- `updated_at`
- `version`

Available inventory is calculated as:

`available_qty = on_hand_qty - reserved_qty`

The service must prevent either quantity from becoming negative.

### Orders

- `order_id`
- `channel`
- `external_order_id`
- `status`
- `idempotency_key`
- `created_at`

A unique constraint on the channel and external order identifier prevents the same marketplace order from being processed twice.

### Reservations

- `reservation_id`
- `order_id`
- `sku_id`
- `quantity`
- `status`
- `created_at`

Reservation records provide an audit trail of stock held for each order.

### Outbox Jobs

- job_id
- event_type
- aggregate_id
- payload
- status
- attempt_count
- next_attempt_at
- created_at

A worker claims pending jobs, processes them, and records the outcome. Failed jobs remain durable for retry or manual investigation.

### Processed Events

Store a unique event identifier for each processed webhook or inbound message. The identifier and business changes must be committed in the same transaction.

## 5. Order and Inventory Flows

### Order creation

1. Receive the order and validate its SKU identifiers and quantities.
2. Check the idempotency key or external order reference.
3. Start a PostgreSQL transaction.
4. Lock the required inventory rows in a consistent SKU order.
5. Confirm that sufficient available stock exists for every line.
6. Increase reserved quantities and create reservation records.
7. Store the order and corresponding outbox jobs in the same transaction.
8. Commit the transaction.
9. Return the order result and let workers propagate inventory changes to external systems.

If any line lacks sufficient stock, the transaction must not partially reserve the order.

### Marketplace synchronization

1. A worker claims a pending stock-sync job.
2. The channel adapter maps internal SKUs to external identifiers.
3. The adapter applies the channel-specific request format and rate limit.
4. Successful updates are recorded.
5. Transient errors are retried with exponential backoff and jitter.
6. Persistent failures are moved to a dead-letter or manual-review state and trigger an alert.

Stock-sync jobs should be coalesced by SKU where appropriate so that obsolete intermediate updates do not overwhelm an API. The latest desired stock state must still be delivered.

### ERP integration

ERP jobs use a stable external reference and are retried only according to the integration's idempotency and reconciliation capabilities.

A request timeout does not prove that the ERP rejected the operation. Before repeating an ambiguous operation, the worker should query the ERP by reference or reconcile the result. If neither idempotency nor lookup is available, automatic retries can create duplicate effects and require a controlled exception process.

## 6. Inventory Lifecycle

- Order confirmed: Reserve the required quantity.
- Order cancelled before dispatch: Release the reservation.
- Order dispatched: Reduce on-hand stock and release the associated reservation in one transaction.
- Return received: Restore sellable stock only after inspection and acceptance.
- Stock adjustment: Record the adjustment with an audit trail and publish a new desired stock state.

The exact timing of marketplace order reservation and dispatch deductions should be aligned with the final business rules and ERP accounting flow.

## 7. Channel Constraints

| Channel | Integration approach | Design consideration |
|---|---|---|
| Company website | Direct inventory API | Use transactional reservations |
| Nykaa | One SKU per request | Pace requests around 4 requests/second, below the stated 5/second limit |
| Flipkart | Bulk requests of up to 50 SKUs | Keep below the stated 10 requests/second limit |
| Blinkit | Daily CSV plus urgent REST updates | Handle approximately 2% timeout assumptions and reconcile missed updates |
| Business Central | ERP adapter and background jobs | Use stable references and reconciliation for uncertain outcomes |

These rates are planning assumptions from the assessment brief, not independently verified live API guarantees.

## 8. Reliability and Observability

The service should provide:

- Idempotent processing of orders and webhook events.
- Database transactions for inventory changes.
- Durable outbox jobs for outbound synchronization.
- Exponential backoff with jitter for transient failures.
- Per-channel rate limiting and concurrency controls.
- Reconciliation jobs to compare internal and external stock states.
- Dead-letter handling for jobs that exceed retry limits.
- Structured logs and correlation IDs across orders, jobs, and external requests.

Monitor reservation failures, inventory mismatches, outbox age, retry counts, API error rates, rate-limit responses, and ERP reconciliation exceptions. Alert when synchronization lag exceeds the 60-second target, while distinguishing internal delays from external outages.

## 9. Trade-offs

| Decision | Alternative | Reason |
|---|---|---|
| PostgreSQL as inventory source of truth | Redis-only stock management | Database transactions and constraints provide stronger correctness for reservations |
| PostgreSQL outbox and workers | Kafka from day one | Simpler operational footprint for a small team and the stated workload; Kafka can be introduced if scale or throughput requires it |
| Transactional row locking | Optimistic updates without conflict handling | Clear reservation correctness; contention should be monitored and transactions kept short |
| Retry plus reconciliation for ERP | Blindly retry every timeout | Avoids assuming that an ambiguous request failed when the ERP may already have committed it |

## 10. Security and Operational Controls

- Keep API credentials in a secrets manager or protected environment configuration.
- Use least-privilege credentials for each integration.
- Validate webhook signatures where supported.
- Avoid logging credentials, payment information, or unnecessary customer data.
- Restrict access to operational replay and reconciliation tools.
- Audit manual inventory adjustments and administrative actions.

## 11. AI Usage Note

AI assistance was used to help structure and review this design. The final approach and decisions should be reviewed and understood by the author before submission.
