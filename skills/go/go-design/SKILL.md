---
name: go-design
description: Design a Go service before writing code. Use when starting a new service, a large new feature, or a rewrite, or when asked to design, plan or architect a Go backend or API. Produces a one-page design covering data model and flow, package layout, dependency rules, error handling, and open questions. Not for small changes inside existing packages.
---

# Go service design

Produce a one-page design before any code is written. The output has no implementation code. The go-structure skill turns the design into folders and files.

Rules are tagged [Go] for common Go practice and [Opinion] for an opinionated choice this skill makes on purpose.

## Process

1. Find the data. What comes in, what is stored, what goes out, and which part of the system owns each piece.
2. Decide whether to ask. If the data or the consistency requirements are unclear, ask at most three questions and stop until they are answered. Only ask about what the data is, who owns it, and what must happen atomically. For anything else, pick a reasonable answer and record it as an assumption.
3. Write the design using the template at the end of this file.
4. Check the design against every rule below. Fix each violation, or keep it and name the trade-off in the design.

See `references/example-orders.md` for a complete worked design and a list of common mistakes.

## Rules

### How to think

1. **Decide in this order: integrity, readability, simplicity, performance.** Integrity means the data stays correct and errors are never lost. When two of these conflict, say which one you are giving up. [Opinion]
   Why: a design that is fast or elegant but loses data is wrong, and unnamed trade-offs get forgotten.

2. **Understand the data first.** Describe each workflow as input, transformation, output. If you can't, the data isn't understood yet: ask, don't guess. [Opinion]
   Why: every later decision (models, layers, transactions) depends on the data, and a wrong guess spreads through all of them.

3. **Solve only problems you have.** No "just in case" features, layers or abstractions. [Go]
   Why: speculative structure costs reading time on every change and is usually wrong when the real need arrives.

4. **Note expected load and latency needs, but don't design for performance you haven't measured.** [Go]
   Why: performance guesses made before code exists are usually wrong and make the design harder to read.

### How to split the system

5. **Boundaries go from small to large: packages, layers, domains.** The full layout has four layers plus a foundation: [Opinion]
   - api: transport (HTTP, gRPC). Decodes requests, encodes responses, nothing else.
   - app: request and response models, validation, orchestration across domains.
   - business: rules and core models, one package per domain. Defines the storage interface it needs.
   - storage: implements the business package's storage interface.
   - foundation: reusable code with no business knowledge (database connection, web framework helpers, logger).

   Why: each layer has one reason to change, so a change in transport or storage stays out of the business rules.

6. **Imports only go downward.** A domain may depend on another domain, but never in a cycle. When the dependency would point the other way, use a delegate (an event the other domain subscribes to) instead of an import. Combining domains for a client happens in app. [Opinion]
   Why: one-way imports keep each package testable and replaceable, and prevent the cycles Go refuses to compile.

7. **Each layer has its own models, with an explicit conversion at each boundary.** This costs conversion code, so it's the first thing the lighter layout drops. [Opinion]
   Why: a change to the JSON shape or the table schema doesn't ripple into the business rules.

8. **Name packages for what they provide.** No `utils`, `common`, `models`, `helpers`. [Go]
   Why: the package name is part of every call site (`order.New`), and a vague name collects unrelated code.

9. **Concrete types first.** Add an interface only for a real second implementation or a dependency that needs decoupling (storage, an external API). Define the interface in the package that uses it, not the one that implements it. [Go]
   Why: interfaces nobody needs add indirection, and consumer-defined interfaces stay small.

10. **Use the lighter layout for small services.** With one domain and one transport: a transport package, one business package and its storage package, sharing the business model. Always keep the import direction, consumer-defined interfaces and the error rules. Move to the full layout at two or more domains or two or more transports. [Opinion]
    Why: the full layout pays off when there are several domains and clients to keep apart. Below that it is mostly conversion code.

### Decide up front

11. **Errors: the low level returns, the middle adds context, the top decides.** Low-level packages return the root cause. Layers above add context. The top level (api middleware or `main`) logs the error and decides the response. Decide which errors are safe to show the client and where they map to status codes. Only `main` panics. [Opinion]
    Why: each error gets logged once with full context, and internal details never leak to clients.

12. **Transactions: say where each one starts and ends.** Either inside one domain, or across domains in app. [Opinion]
    Why: transaction scope decides what can be half-written after a failure, which is an integrity question.

13. **Side effects outside the database: plan for failure and repeats.** For every external API call, queue message or email, decide what happens when it fails, times out or runs twice. Options include retry with backoff, an outbox table written in the same transaction, and an idempotency key. [Go]
    Why: the database and the external system can't commit together, so without a plan they drift apart.

14. **Logging and config belong to the application.** `main` reads config and builds the logger, then passes them down. Foundation packages never log or read config. [Opinion]
    Why: reusable code stays reusable, and policy decisions stay in one place.

15. **Use concurrency only when out-of-order work clearly adds value.** For each goroutine in the design, say what starts it and how it stops. [Opinion]
    Why: concurrency adds failure modes and shutdown work, so it has to earn its place.

## Output template

Keep the whole design to about one page. Use these headings in this order.

```markdown
# <Service name>: design

## Summary
Two or three sentences: what the service does, who calls it, and which layout
(full or lighter) it uses and why.

## Data
- Entities: name, key fields, owning domain.
- Workflows: one line each, as input -> transformation -> output.

## Packages
A tree of packages, one line each on what the package provides.

## Dependency rules
Which packages may import which. Note any delegate or event used to avoid a cycle.

## Transactions and side effects
Where each transaction starts and ends. For each external call: what happens on
failure, timeout and repeat.

## Error handling
Where errors are created, where context is added, where they are logged, and how
they map to client responses.

## Concurrency
Each goroutine: why it exists, what starts it, how it stops. Write "None" if none.

## Open questions, risks, left out
- Assumptions made instead of asking.
- Risks.
- What was deliberately left out, and when it would be worth adding.
```
