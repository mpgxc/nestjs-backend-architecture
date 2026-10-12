# Bounded Contexts & Events

Contents: [Bounded context](#bounded-context) · [The same word, different models](#the-same-word-different-models) · [Domain vs integration events](#domain-vs-integration-events) · [Communication between contexts](#communication-between-contexts) · [Anti-corruption layer](#anti-corruption-layer)

## Bounded context

A bounded context is the boundary within which a model is coherent and its terms mean one thing. It is the **strategic** side of DDD — where you draw service and module boundaries. In code, each one is a folder under `src/contexts/`; across services, each service owns one or a few contexts.

Inside: one ubiquitous language, one model, consistency enforced by aggregates.
Across: translation, asynchronous events, explicit contracts.

Only business goes into `contexts/`. Health checks, metrics and the outbox relay are operational and live in `infrastructure/`.

## The same word, different models

`Payment` in the **payments** context: amount, PIX key, status, limit invariants.
`Payment` in the **ledger** context: a pair of debit and credit entries.
`Payment` in **notifications**: a receipt to send.

None is "the real" Payment. Do **not** build a shared canonical model across contexts — that produces a class with forty fields, owned by nobody, that cannot change without breaking three teams. Never share entity classes or a "common types" package between services. Each context owns its model and translates at the boundary.

## Domain vs integration events

- **Domain event** — internal to the service, recorded by the aggregate (`record(...)`), never published by it. May reference domain concepts.
- **Integration event** — crosses the boundary. Primitives only, past tense, versioned, wrapped in the envelope from `outbox.md`. A public contract.

The aggregate records; the repository drains the events into the outbox in the same transaction, mapping each domain event to its integration event; the relay publishes. That mapping (`OutboxMapper.fromDomainEvent`) is the one place where an internal fact becomes a public contract — review it like an API change.

## Communication between contexts

- **Async via events** (default) — the publisher does not know the subscribers; contexts stay independent, tolerate each other's downtime and scale separately. Cost: eventual consistency.
- **Replicated read model** — when a context needs another's data to answer reads, keep a local replica fed by events instead of calling the owner on every request. See `platform/read-models.md`.
- **Sync via a port** — only when an answer is needed *now* and must be exact (balance to authorize a debit, a write that belongs to the owner). Costs coupling and availability: their outage becomes yours. Wrap it in a port + adapter with timeout and circuit breaker (`resilience.md`) and machine-to-machine authentication (`platform/service-auth.md`).

Default to events. Reach for a synchronous call only when the flow genuinely cannot proceed without the exact answer.

## Anti-corruption layer

When consuming another context's model (or a vendor's), translate at the edge instead of letting their shape spread. The adapter implementing your port **is** the ACL: their DTO enters, your domain object leaves, and their field names stop right there.

Without it, a rename on their side becomes a refactor across your domain. With it, exactly one file changes. See `integration.md`.
