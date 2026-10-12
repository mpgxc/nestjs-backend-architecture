---
name: "nestjs-backend-architecture"
description: "Pragmatic architecture and code-style conventions for TypeScript/NestJS backend services in Brazilian fintech. Picks the lightest architecture level that fits (CRUD → domain rules → integration/events → distributed) and covers contexts/ layout, use-cases, CQS with query services, Result pattern with global interceptor/filter, abstract-class ports, DDD, outbox, SQS consumers, idempotency, cron workers, vendor ACL, circuit breaker, logging, REST/RFC 9457, transactions, testing and project bootstrap. Enforces braces on every block, fail-fast guard clauses, no else, no nested ifs, docstring-only comments. Trigger for any NestJS code writing, structuring, review or refactor task."
---

# NestJS Backend Architecture

Conventions for backend services in TypeScript + NestJS. The guiding principle is **pragmatism over ceremony**: start with the simplest structure that works and add abstraction only when concrete pressure justifies it.

## Architecture levels — pick the lowest that fits

Before writing code for a context or feature, decide its level and **state it in one line** ("Nível 1: há regra de transição, sem integração"). Generate only what that level needs. Promote when a new need appears, never in advance.

| Level | When | What it uses | What it does **not** use |
|-------|------|--------------|--------------------------|
| **0 — CRUD** | No invariants, no integration (preferences, catalog, configuration) | Controller, simple use-cases, plain domain `type` + repository port, TypeORM adapter, query service | Value objects, entity classes, domain services, events, outbox |
| **1 — Business rules** | Invariants, state transitions, limits | Entity with factory and transitions, value objects **only for values with rules**, domain services for cross-entity rules | Events, outbox, vendor ports |
| **2 — Integration / publishing** | Calls a vendor or another service, or other services consume its facts | Vendor port + ACL adapter, timeout/retry/circuit breaker, `AggregateRoot` + outbox, idempotent SQS consumers, workers | Sagas, replicated read models |
| **3 — Distributed flow** | A flow spans services and needs compensation, or reads must survive another service's outage | Orchestrated saga with persisted state, event-replicated read models, FIFO ordering, service auth | — |

Most contexts are level 0 or 1. Level 3 is a platform decision, usually already made; its references live in `references/platform/`.

## The defaults (every level)

Apply these unless the user says otherwise:

1. **Contexts, feature-first.** One folder per bounded context in `src/contexts/<ctx>/`. `infrastructure/` and `commons/` are global.
2. **Port only for external I/O.** Database, vendor, another service, queue → port. Internal collaborator with no I/O → plain concrete class. Never abstract for symmetry.
3. **Ports are `abstract class`, never `interface`** — the class is its own DI token. No `Symbol`, no `@Inject(...)`. Ports live flat in `domain/`.
4. **Business errors are values.** Return `Result<T, BusinessError>`, built inline with `Result.Ok(...)` / `Result.Ok()` / `Result.Err({ type, message })`. Guard with `if (x.isErr) { return x; }`. Controllers return the `Result` as-is; a global `ResultInterceptor` + `BusinessErrorFilter` turn it into HTTP. `throw` only for infrastructure failures.
5. **CQS.** Commands → one use-case per action, through the repository port, returning `void` by default. Queries → one query service per context in `services/`, injecting TypeORM's `DataSource`.
6. **Code style.** Braces on every block, fail fast with guard clauses, no `else`, no nested `if`, no inline comments (docstrings only), structured logs inline for business facts, `try/catch` only to react to a failure. See `references/code-style.md` — every generated example must pass its checklist.
7. **Correctness rules are not ceremony** and apply at any level: money in integer cents, no PII in logs, `Idempotency-Key` on money-moving endpoints, idempotent consumers, outbox for every published event.

If a request conflicts with these, follow the user's instruction — but say once, briefly, what the convention would have been and why.

## Orientation

```
src/
├─ contexts/
│  └─ payments/
│     ├─ payments.controller.ts     # ONE controller per context
│     ├─ payments.dto.ts            # HTTP DTOs — edge only
│     ├─ payments.module.ts         # composition root of the context
│     ├─ use-cases/                 # COMMANDS — flat, one file per action (+ .spec.ts)
│     ├─ domain/                    # flat: types/entities, value objects, domain services, ports
│     ├─ workers/                   # level 2+: cron jobs (*.cron.ts), queue consumers (*.consumer.ts)
│     ├─ services/                  # only if needed — the query service
│     └─ adapters/                  # only if needed — port implementations for this context
├─ infrastructure/                  # GLOBAL: database, http, logger, messaging, outbox, integrations/<vendor>, health
├─ commons/                         # GLOBAL: Result, BusinessError, UnitOfWork, AggregateRoot, HTTP error mapping
└─ app.module.ts
```

**Where does this file go?** Write action → `use-cases/`. Read → query service in `services/`. Business rule, value object or port → `domain/` (flat). Implementation of this context's port → `adapters/`. Shared concrete or operational job → `infrastructure/`. Cron or queue entry point → `workers/`. HTTP concern → context root, beside the controller.

## Reference files

Read the file for the topic at hand — do not load them all.

**Start here for any code**
- `references/code-style.md` — braces, fail fast, no else, no nested ifs, Object Calisthenics as review signals, comments, inline logging, try/catch only to react, checklist.
- `references/bootstrap.md` — day-1 setup of a new project: env validation, request validation as problem+json, catch-all filter, migrations, health, graceful shutdown, ESLint rules that enforce the style.

**Levels 0–1: structure and rules**
- `references/project-structure.md` — full tree, context folders, placement questions, dependency rule, naming.
- `references/use-cases.md` — use-case examples per level, Input/Output, logging and failure policy, batch use-cases.
- `references/controllers.md` — one controller returning the `Result`, `ResultInterceptor` + `BusinessErrorFilter`, status map, Output as response contract.
- `references/cqs.md` — query service with `DataSource`, cursor pagination, why the repository is write-side only.
- `references/repositories.md` — abstract-class ports, TypeORM adapter, wiring, test fakes; outbox drain only when publishing.
- `references/result-pattern.md` — `Result` with `isOk`/`isErr`, threading, inline `BusinessError`, where Result stops.
- `references/domain.md` — matching the model to the level, value objects with rules, entities, transitions, domain services, events when publishing.
- `references/transactions.md` — when one transaction must cover several writes; the `UnitOfWork` port.
- `references/testing.md` — domain unit tests, use-case specs with fakes, integration tests on real Postgres, few e2e.
- `references/rest-api.md` — URIs, statuses, headers, payload, pagination, RFC 9457, versioning.
- `references/design-patterns.md` — SOLID, DRY/KISS/YAGNI, rule of three, the GoF patterns that actually appear, when not to.

**Level 2: integration and publishing**
- `references/integration.md` — vendor client in `infrastructure/integrations/<vendor>/` + ACL adapter in the context, error translation, wiring.
- `references/resilience.md` — timeout, retry, circuit breaker with `cockatiel`, what the domain sees.
- `references/outbox.md` — dual write, drain in the repository, envelope, relay.
- `references/messaging-pubsub.md` — SNS → SQS per consumer, event design, consumers, failure kinds, DLQ.
- `references/idempotency.md` — two layers, the claim-before-process trap, API idempotency keys.
- `references/workers.md` — cron and queue workers, multi-replica safety, batch processing.
- `references/logging.md` — structured events, who logs what, Nest `Logger` + Pino, correlation id, PII redaction.
- `references/bounded-contexts.md` — domain vs integration events, choosing events / replicas / sync calls, ACL.
- `references/hexagonal.md` — driving vs driven, gateway vs adapter, service-to-service as the same pattern.

**Level 3: platform (decided once, not per project)**
- `references/platform/saga.md` — orchestrated saga with persisted state, uncertain steps, reconciliation, compensation rules.
- `references/platform/read-models.md` — event-replicated shadows (ECST), guarded fallback, reference vs decision reads.
- `references/platform/service-auth.md` — Client Credentials between services, JWT guard with `aud` and scopes.

## When to escalate

Promote a context only under real pressure: an invariant appears (0 → 1); a vendor call or a consumer of your facts appears (1 → 2); a flow needs compensation across services or reads must survive another service being down (2 → 3). Extract a port when a second implementation appears, a test fake is genuinely needed, or the collaborator starts doing I/O. CQRS with separate stores almost never.

## Style

Match the existing codebase over these conventions when they conflict — consistency within a project beats global purity — except for the correctness rules in default 7, which always apply. Prefer explicit, readable code over clever code: no wrapper layers that only forward calls, no helpers that hide logs, no comments restating the code, nothing the context's level does not need.
