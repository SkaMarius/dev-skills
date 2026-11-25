# Worked example and common mistakes

## Input

"A service that takes orders over HTTP, stores them in Postgres, and notifies a warehouse API when an order is paid."

Nothing about the data blocks the design, so the design goes ahead and records its assumptions.

## Output

```markdown
# Orders: design

## Summary
Accepts orders over HTTP, stores them in Postgres, and tells the warehouse API
when an order is paid. One domain (order) and one transport (HTTP), so it uses
the lighter layout. Trade-off: the HTTP layer shares the business model, which
favors simplicity over isolation; a change to the JSON shape touches the model.

## Data
- Order: ID, CustomerID, Items (SKU, Qty, PriceCents), TotalCents,
  Status (pending, paid, cancelled), CreatedAt, PaidAt. Owned by order.
- Outbox entry: ID, OrderID, Event, Attempts, NextAttemptAt, DeliveredAt.
  Owned by order.
- Workflows:
  - Create: order request -> validate, compute total, status pending -> stored order.
  - Get: order ID -> stored order, or not found.
  - Mark paid: order ID -> pending becomes paid, outbox entry written -> paid order.
  - Deliver: undelivered outbox entry -> warehouse API call -> entry marked delivered.

## Packages
cmd/orders            reads config, builds logger and DB pool, wires packages,
                      runs the HTTP server and delivery worker, handles shutdown
internal/orderapi     HTTP handlers: decode, call order, encode, map errors to status codes
internal/order        order model, state rules, Storer and Notifier interfaces, errors
internal/order/orderdb  Postgres implementation of order.Storer
internal/warehouse    warehouse API client, implements order.Notifier
internal/sqldb        foundation: Postgres connection and transaction helpers

## Dependency rules
- cmd/orders imports everything, for wiring only.
- orderapi imports order.
- orderdb imports order and sqldb. warehouse imports order.
- order imports only the standard library.
- sqldb imports nothing internal. Nothing imports orderapi or cmd/orders.

## Transactions and side effects
- Mark paid is one transaction inside orderdb: update the order where status is
  pending, and insert an outbox entry. Zero rows updated means the order wasn't
  pending, which returns order.ErrInvalidState. This also prevents double payment
  under concurrent requests.
- The warehouse is never called inside a request. The delivery worker picks
  undelivered entries (SKIP LOCKED, so several instances can run), calls the
  warehouse with the order ID as idempotency key, and marks the entry delivered.
- On failure or timeout: increment Attempts and set NextAttemptAt with backoff.
  After 10 attempts, stop retrying and log at error level.
- On repeat (crash after the call, before marking delivered): the warehouse
  receives the same idempotency key twice.

## Error handling
- orderdb turns sql.ErrNoRows into order.ErrNotFound and wraps other errors with
  the operation name.
- order returns ErrNotFound, ErrInvalidState and validation errors as-is, and
  wraps everything else with context.
- orderapi maps ErrNotFound to 404, ErrInvalidState to 409, validation errors to
  400 with field messages, and everything else to 500 with a generic message.
- Each request error is logged once, in middleware, with the request ID.
  Worker errors are logged once, in the worker loop.

## Concurrency
- Delivery worker: one goroutine, started by main after the server starts,
  stopped by context cancellation on shutdown; main waits for it to finish.
  It exists so a slow or down warehouse never blocks the payment request.

## Open questions, risks, left out
- Assumption: the payment provider calls POST /orders/{id}/paid, and auth is
  handled by a gateway in front of the service.
- Open: does the warehouse API accept an idempotency key? If not, duplicates
  are possible after a crash.
- Open: who is alerted when delivery gives up after 10 attempts?
- Risk: the outbox table grows without bound; delivered entries need cleanup.
- Left out: customer and product domains (stored as IDs only), order listing
  and pagination, cancellation after payment, metrics. Switch to the full
  layout if customers or products become their own domains or a second
  transport (gRPC, a queue consumer) is added.
```

## Common mistakes

Each item shows a bad design line and the fix.

**Calling an external API inside the request**

Bad: "Mark paid commits the order, then calls the warehouse. If the warehouse fails, return 500."
The order is paid but the client sees an error, and the warehouse is never told.

Good: write an outbox entry in the same transaction and deliver it from a worker with retries (rule 13).

**Interface defined next to its implementation**

Bad: "`orderdb` defines an `OrderRepository` interface and a `PostgresOrderRepository` that implements it."

Good: "`order` defines `Storer` with the methods it calls. `orderdb` implements it." (rule 9)

**Grab-bag packages**

Bad: "`internal/models` holds all structs, `internal/utils` holds shared helpers."

Good: each model lives in the package that owns it. A helper goes into a foundation package named for what it provides, such as `sqldb`. (rule 8)

**Logging at every level**

Bad: "`orderdb` logs the query error and returns it. `order` logs it and returns it. The handler logs it."
One failure produces three log lines with partial context.

Good: return and wrap on the way up, and log once at the top (rule 11).

**Domains importing each other**

Bad: "`order` imports `customer` to check the customer exists, and `customer` imports `order` to count a customer's orders."
That is an import cycle.

Good: `order` may import `customer`. `customer` learns about new orders through a delegate event, or app combines both for the client. (rule 6)

**Speculative abstraction**

Bad: "A storage interface with Postgres and MongoDB implementations, in case we switch databases."

Good: one Postgres implementation. The interface exists because business code must not depend on storage, not for a second database nobody asked for. (rules 3 and 9)

**A goroutine with no owner**

Bad: "The handler runs `go notifyWarehouse(order)` and returns."
Nothing retries it, and shutdown kills it halfway.

Good: one worker that main starts, stops through context cancellation, and waits for (rule 15).
