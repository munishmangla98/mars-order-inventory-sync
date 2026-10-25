# Stock Reservation Pseudocode

## Purpose

Reserve inventory for an order without overselling, including orders containing multiple SKUs. This is illustrative pseudocode, not production-ready code.

## Reservation Algorithm

```text
function reserveOrder(order, idempotencyKey):

    begin database transaction

    existingOrder = findOrderByIdempotencyKey(idempotencyKey)

    if existingOrder exists:
        commit transaction
        return existingOrder

    validate order lines
    combine duplicate SKU lines into total quantities
    sort SKU identifiers in a consistent order

    for each line in sorted order:
        inventory = select inventory row
                    for line.sku_id
                    FOR UPDATE

        if inventory does not exist:
            rollback transaction
            return SKU_NOT_FOUND

        available = inventory.on_hand_qty
                    - inventory.reserved_qty

        if available < line.quantity:
            rollback transaction
            return INSUFFICIENT_STOCK

    create order record

    for each line in sorted order:
        increase inventory.reserved_qty
            by line.quantity

        create reservation record
            with status = RESERVED

    create required outbound sync jobs
        in the outbox table

    commit transaction

    return order confirmation
```

## Important Correctness Rules

1. **Atomicity:** All lines are reserved, or none are. A failed line rolls back the complete transaction.
2. **Concurrency:** Inventory rows are locked before checking availability and updating reservations.
3. **Consistent lock order:** Lock SKUs in sorted order to reduce deadlock risk.
4. **Idempotency:** Enforce a database uniqueness constraint on the idempotency key. Concurrent duplicate requests must be handled by the constraint and transaction retry logic, not only by an initial lookup.
5. **Durable events:** Insert outbox jobs in the same transaction as the order and reservation changes.
6. **No external calls inside the transaction:** Marketplace and ERP requests are handled asynchronously after commit.
7. **Reservation release:** Cancellation before dispatch releases the reservation exactly once.
8. **Dispatch:** Reduce on-hand quantity and release the reservation atomically.
9. **Failure handling:** Retry transient external failures with backoff. Reconcile ambiguous ERP outcomes before repeating potentially non-idempotent operations.

## Implementation Notes

- Use database constraints to prevent negative quantities.
- Define unique keys for external order IDs and processed webhook event IDs.
- Add transaction retry handling for deadlocks or serialization conflicts.
- Keep database transactions short.
- Log a correlation ID for each order and outbound job.
- Test simultaneous reservations for the final available unit, duplicate webhooks, partial external failures, and worker restarts.
