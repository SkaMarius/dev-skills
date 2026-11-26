# Worked example and common mistakes

## Input

"A service that takes orders over HTTP, stores them in Postgres, and notifies a warehouse API when an order is paid."

It's a new service, and nothing about the data blocks the design, so the design goes ahead and records its assumptions. Whether the warehouse API accepts an idempotency key is an external fact, so the design covers both answers instead of asking.

## Output

```markdown
# Orders: design

## Summary
Accepts orders over HTTP, stores them in Postgres, and tells the warehouse API
when an order is paid. One domain (order) and one transport (HTTP), so it uses
the lighter layout; the delivery worker loops over the service's own data, so
it is not a second transport. Trade-off: the HTTP layer shares the business model, which
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
cmd/orders              entry: reads config, builds logger and DB pool, wires
                        packages, starts the server and worker, handles shutdown
internal/orderapi       transport: HTTP handlers, decode, call order, encode,
                        map errors to responses
internal/orderworker    entry: delivery loop, calls order.DeliverPending on a timer
internal/order          business: order model, validation, state rules,
                        Storer and Notifier interfaces, errors
internal/order/orderdb  storage: Postgres implementation of order.Storer
internal/warehouse      outbound client: warehouse API, implements order.Notifier
internal/sqldb          foundation: Postgres connection and transaction helpers

## Dependency rules
- cmd/orders imports everything, for wiring only.
- orderapi and orderworker import order.
- orderdb imports order and sqldb. warehouse imports order.
- order imports only the standard library.
- sqldb imports nothing internal. Nothing imports orderapi, orderworker or cmd/orders.
- No delegates.

## Transactions and side effects
- Mark paid: order loads the order, checks that pending -> paid is allowed, and
  builds the paid order and its outbox entry. It passes both to Storer.MarkPaid,
  which writes them in one transaction. The update only applies while the status
  is still pending; zero rows updated returns order.ErrInvalidState. This guards
  against two concurrent payment requests.
- The warehouse is never called inside a request or inside a transaction.
  order.DeliverPending claims entries with Storer.ClaimPending, which in one
  short transaction selects due entries (SKIP LOCKED), pushes NextAttemptAt
  30s ahead as a lease, and commits. Other instances skip leased entries. order
  then calls the Notifier with the order ID as idempotency key and marks each
  entry delivered. The 30s lease is longer than the 5s call timeout.
- On failure or timeout: increment Attempts and set NextAttemptAt with backoff.
  After 10 attempts, stop retrying and return an error for the worker to log.
- Delivery is at least once, not exactly once. On repeat (crash after the call,
  before marking delivered, so the lease expires and the entry is sent again):
  if the warehouse accepts an idempotency key, it ignores the second call. If
  not, it receives a duplicate and must tolerate it.

## Error handling
- orderdb turns sql.ErrNoRows into order.ErrNotFound and wraps other errors with
  the operation name.
- order returns ErrNotFound, ErrInvalidState and validation errors as-is, and
  wraps everything else with context.
- orderapi maps ErrNotFound to 404, ErrInvalidState to 409, validation errors to
  400 with field messages, and everything else to 500 with a generic message.
- Errors are logged once at the top of each call chain: request middleware (with
  the request ID) and the worker loop.

## Concurrency
- Delivery worker: one goroutine, started by main after the server starts,
  stopped by context cancellation on shutdown; main waits for it to finish.
  It exists so a slow or down warehouse never blocks the payment request.

## Cross-cutting
- Auth: handled by a gateway in front of the service (assumption).
- Migrations: SQL files applied by a separate command before deploy.
- Timeouts: each request gets a context with a 10s deadline; each warehouse call
  gets 5s. Contexts are passed down through every call.
- Observability: structured logs with request ID. Metrics and tracing out of scope.

## Open questions, risks, left out
- Assumption: the payment provider calls POST /orders/{id}/paid.
- Open: does the warehouse API accept an idempotency key? The design works
  either way (see above), but duplicates are possible if not.
- Open: who is alerted when delivery gives up after 10 attempts?
- Risk: the outbox table grows without bound; delivered entries need cleanup.
- Left out: customer and product domains (stored as IDs only), order listing
  and pagination, cancellation after payment. Switch to the full layout if
  customers or products become their own domains or a second transport
  (gRPC, a queue consumer) is added.
```

## Common mistakes

Each item shows a bad design line and the fix.

**Calling an external API inside the request**

Bad: "Mark paid commits the order, then calls the warehouse. If the warehouse fails, return 500."
The order is paid but the client sees an error, and the warehouse is never told.

Good: write an outbox entry in the same transaction and deliver it from a worker with retries (rule 13).

**Business rules in storage**

Bad: "`orderdb.MarkPaid` checks the status and, if the order was pending, also inserts an outbox entry."
The rule "a paid order notifies the warehouse" now lives in storage, where nobody looks for it.

Good: `order` decides the state change and builds the outbox entry. `orderdb` writes both atomically, and its status condition only guards against concurrent requests. (rules 5 and 12)

**Interface defined next to its implementation**

Bad: "`orderdb` defines an `OrderRepository` interface and a `PostgresOrderRepository` that implements it."

Good: "`order` defines `Storer` with the methods it calls. `orderdb` implements it." (rule 9)

**Grab-bag packages**

Bad: "`internal/models` holds all structs, `internal/utils` holds shared helpers."

Good: each model lives in the package that owns it. A helper goes into a foundation package named for what it provides, such as `sqldb`. (rule 8)

**Logging at every level**

Bad: "`orderdb` logs the query error and returns it. `order` logs it and returns it. The handler logs it."
One failure produces three log lines with partial context.

Good: return and wrap on the way up, and log once at the top of the call chain (rule 11).

**Domains importing each other**

Bad: "`order` imports `customer` to check the customer exists, and `customer` imports `order` to count a customer's orders."
That is an import cycle.

Good: `order` may import `customer`. `customer` learns about new orders through a delegate event, and the design says whether that event runs inside the transaction or after commit. Or app combines both for the client. (rule 6)

**Speculative abstraction**

Bad: "A storage interface with Postgres and MongoDB implementations, in case we switch databases."

Good: one Postgres implementation. The interface exists because business code must not depend on storage, not for a second database nobody asked for. (rules 3 and 9)

**A goroutine with no owner**

Bad: "The handler runs `go notifyWarehouse(order)` and returns."
Nothing retries it, and shutdown kills it halfway.

Good: one worker that main starts, stops through context cancellation, and waits for (rules 15 and 16).

**Imposing the default layout on an existing service**

Bad: a feature design for a service that uses `domain/`, `application/` and `infrastructure/` proposes new `business/` and `storage/` packages next to them.

Good: map the roles onto the existing folders and follow the project's conventions (Existing services section).
