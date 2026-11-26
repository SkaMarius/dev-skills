---
name: go-design
description: Design a Go service before writing code. Use when starting a new service, adding a large feature to an existing service, or planning a rewrite, or when asked to design, plan or architect a Go backend or API. Produces a one-page design covering data model and flow, package layout, dependency rules, error handling, and open questions. Not for small changes inside existing packages.
---

# Go service design

Produce a one-page design before any code is written. The output has no implementation code.

The design names the packages and what each one does. Turning that into files, folders and names inside each package is the job of the go-structure skill (planned).

Rules are tagged [Common] for widely agreed practice and [Opinion] for an opinionated choice this skill makes on purpose.

## Existing services

When the service already exists, follow its current layout, naming and conventions. Map the roles in rule 5 onto what is there: `presentation/`, `application/`, `domain/` and `infrastructure/`, for example, are the same roles under other names. Treat [Opinion] rules as defaults that give way to the project's own choices. Keep the import direction, consumer-defined interfaces and the error rules (6, 9, 11) unless the project clearly does otherwise. If the existing layout breaks them, list that as a risk instead of redesigning it, unless asked.

## Process

1. Find the data. What comes in, what is stored, what goes out, and which part of the system owns each piece.
2. Decide whether to ask. If the data or the consistency requirements are unclear, ask at most three questions and wait for the answers. Only ask about what the data is, who owns it, and what must happen atomically. For anything else, pick a reasonable answer and record it as an assumption. If nobody can answer (a non-interactive run), record the questions as blocking assumptions and continue.
   Facts about external systems the user may not know (whether an API supports idempotency keys, its rate limits) are not questions. Design for both answers and say how the design differs.
3. Write the design using the template at the end of this file. Put it in the chat. If the user asks for a file, use the path they give, or `docs/design/<service>.md`.
4. Check the design against every rule below. Fix each violation, or keep it and name the trade-off in the design.

See `references/example-orders.md` for a complete worked design and a list of common mistakes.

## Rules

### How to think

1. **Decide in this order: integrity, readability, simplicity, performance.** Integrity means the data stays correct and errors are never lost. When two of these conflict, say which one you are giving up. [Opinion]
   Why: a design that is fast or elegant but loses data is wrong, and unnamed trade-offs get forgotten.

2. **Understand the data first.** Describe each workflow as input, transformation, output. If you can't, the data isn't understood yet: ask, don't guess. [Opinion]
   Why: every later decision (models, layers, transactions) depends on the data, and a wrong guess spreads through all of them.

3. **Solve only problems you have.** No "just in case" features, layers or abstractions. [Common]
   Why: speculative structure costs reading time on every change and is usually wrong when the real need arrives.

4. **Note expected load and latency needs, but don't design for performance you haven't measured.** [Common]
   Why: performance guesses made before code exists are usually wrong and make the design harder to read.

### How to split the system

5. **Split by role: packages, then layers, then domains.** A new service uses these roles, with these default names: [Opinion]
   - api (transport): decodes requests, encodes responses, nothing else.
   - app (application): request and response models, validation, orchestration across domains, transaction boundaries.
   - business: rules and core models, one package per domain. Defines the interfaces it needs from storage and outbound clients.
   - storage: implements the business package's storage interface. Makes changes atomic, holds no rules.
   - outbound clients: talk to external APIs. Implement an interface the business package defines, the same way storage does.
   - foundation: reusable code with no business knowledge (database connection, web helpers, logger).

   Workers (queue consumers, scheduled jobs, delivery loops) are entry points like api: they call into business and hold no rules.

   Why: each role has one reason to change, so a change in transport, storage or an external API stays out of the business rules.

6. **Imports only go downward.** A domain may depend on another domain, but never in a cycle. When the dependency would point the other way, use a delegate (an event the other domain subscribes to) instead of an import. For each delegate, say whether it runs inside the transaction or after commit, and what happens if it is lost. Combining domains for a client happens in app. [Opinion]
   Why: one-way imports keep each package testable and replaceable, and an event fired after commit is lost on a crash unless the design says otherwise.

7. **Each layer has its own models, with an explicit conversion at each boundary.** This costs conversion code, so it's the first thing the lighter layout drops. [Opinion]
   Why: a change to the wire format or the table schema doesn't ripple into the business rules.

8. **Name packages for what they provide.** No `utils`, `common`, `models`, `helpers`. [Common]
   Why: the package name is part of every call site (`order.New`), and a vague name collects unrelated code.

9. **Concrete types first.** Add an interface only for a real second implementation or a dependency that needs decoupling (storage, an external API). Define the interface in the package that uses it, not the one that implements it. [Common]
   Why: interfaces nobody needs add indirection, and consumer-defined interfaces stay small.

10. **Use the lighter layout for small services.** With one domain and one transport: a transport package, one business package and its storage package, sharing the business model. With no app layer, validation lives in the business package. Always keep the import direction, consumer-defined interfaces and the error rules. Move to the full layout at two or more domains or two or more transports. An entry point counts as a transport if it brings in outside input with its own data shape (HTTP, gRPC, a queue consumer). A loop or scheduled job over the service's own data does not. [Opinion]
    Why: the full layout pays off when there are several domains and clients to keep apart. Below that it is mostly conversion code.

### Decide up front

11. **Errors: the low level returns, the middle adds context, the top decides.** Low-level packages return the root cause. Layers above add context. The top of each call chain (request middleware, worker loop, `main`) logs the error once and decides the response. Decide which errors are safe to show the client and how they map to client responses. Return errors you can return; panic only for programmer errors and startup failures. [Common]
    Why: each error gets logged once with full context, and internal details never leak to clients.

12. **Transactions: say where each one starts and ends.** Business decides what changes together; storage only makes it atomic. A transaction across domains starts at the workflow's entry point in app and is passed down, so each domain gets a storage bound to the same transaction. Keep it as short as the workflow allows, and never hold it open across an external call. [Opinion]
    Why: transaction scope decides what can be half-written after a failure, which is an integrity question.

13. **Side effects outside the database: plan for failure and repeats.** For every external API call, queue message or email, decide what happens when it fails, times out or runs twice. Options include retry with backoff, an outbox table written in the same transaction, and an idempotency key. [Common]
    Why: the database and the external system can't commit together, so without a plan they drift apart.

14. **Logging and config belong to the application.** `main` reads config and builds the logger, then passes them down. Foundation packages never decide on their own to log or read config; they take what they need as parameters. [Opinion]
    Why: reusable code stays reusable, and policy decisions stay in one place.

15. **Use concurrency only when out-of-order work clearly adds value.** [Opinion]
    Why: concurrency adds failure modes and shutdown work, so it has to earn its place.

16. **For every goroutine, say what starts it and how it stops.** [Common]
    Why: a goroutine with no owner leaks, or gets killed halfway through its work on shutdown.

## Output template

Keep the whole design to about one page. Use these headings in this order.

```markdown
# <Service name>: design

## Summary
Two or three sentences: what the service does, who calls it, and which layout
(full, lighter, or the existing one) it uses and why.

## Data
- Entities: name, key fields, owning domain.
- Workflows: one line each, as input -> transformation -> output.

## Packages
A tree of packages, one line each on its role and what it provides.

## Dependency rules
Which packages may import which. For each delegate: inside the transaction or
after commit, and what happens if it is lost.

## Transactions and side effects
Where each transaction starts and ends. For each external call: what happens on
failure, timeout and repeat.

## Error handling
Where errors are created, where context is added, where they are logged, and how
they map to client responses.

## Concurrency
Each goroutine: why it exists, what starts it, how it stops. Write "None" if none.

## Cross-cutting
One line each, or "out of scope": auth, schema migrations, timeouts and context
propagation, observability.

## Open questions, risks, left out
- Assumptions made instead of asking, and blocking assumptions if nobody could answer.
- Risks.
- What was deliberately left out, and when it would be worth adding.
```
