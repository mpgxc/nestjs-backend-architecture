# Project Structure

Contents: [The tree](#the-tree) · [Inside a context](#inside-a-context) · [Where does this file go](#where-does-this-file-go) · [Global infrastructure and commons](#global-infrastructure-and-commons) · [Dependency rule](#dependency-rule) · [Naming](#naming) · [Why not layer-first](#why-not-layer-first)

## The tree

```
src/
├─ contexts/                              # one folder per bounded context — only business lives here
│  └─ payments/
│     ├─ payments.controller.ts           # ONE controller per context
│     ├─ payments.dto.ts                  # HTTP request/response DTOs (edge only)
│     ├─ payments.module.ts               # composition root of the context
│     ├─ use-cases/                       # COMMANDS — flat, one file per action
│     │  ├─ create-payment.use-case.ts    # Input + Output + class
│     │  ├─ create-payment.use-case.spec.ts
│     │  └─ refund-payment.use-case.ts
│     ├─ domain/                          # flat: entities, value objects, domain services, ports
│     │  ├─ payment.ts                    # aggregate root
│     │  ├─ money.ts                      # value object
│     │  ├─ pix-key.ts                    # value object
│     │  ├─ payment-limit.service.ts      # domain service
│     │  ├─ payment.repository.ts         # port (abstract class)
│     │  └─ pix-gateway.ts                # port (abstract class)
│     ├─ workers/                         # non-HTTP input adapters
│     │  ├─ reconcile-payments.cron.ts
│     │  └─ boleto-paid.consumer.ts
│     ├─ services/                        # only if needed — query service lives here
│     │  └─ payments.query-service.ts
│     └─ adapters/                        # only if needed — context-specific port implementations
│        ├─ payment.model.ts              # TypeORM entity (persistence model)
│        ├─ typeorm-payment.repository.ts
│        └─ acme-pix.gateway.ts        # ACL over the global Acme client
├─ infrastructure/                        # GLOBAL — shared concretes, operational concerns
│  ├─ database/                           # DataSource, migrations, transaction helpers
│  ├─ http/                               # base HTTP setup, correlation middleware
│  ├─ logger/                             # Pino adapter for Nest's LoggerService, redaction
│  ├─ messaging/                          # SNS publisher, SQS consumer setup
│  ├─ outbox/                             # outbox table model + relay
│  ├─ integrations/
│  │  ├─ acme/                         # low-level client, DTOs, typed API error, resilience policy
│  │  └─ globex/
│  ├─ health/                             # liveness/readiness — not domain
│  └─ infrastructure.module.ts
├─ commons/                               # GLOBAL — Result, BusinessError, AggregateRoot, HTTP error mapping
│  ├─ result.ts
│  ├─ business-error.ts
│  ├─ unit-of-work.ts                     # port, only if the project needs multi-write transactions
│  ├─ aggregate-root.ts                   # only if some context publishes events
│  └─ http/
│     ├─ business-error-status.ts         # Record<BusinessErrorType, HttpStatus>
│     ├─ business-error.exception.ts
│     ├─ result.interceptor.ts            # global: unwraps Ok, throws on Err
│     ├─ business-error.filter.ts         # global: Err → status + problem+json
│     └─ unhandled-exception.filter.ts    # global: infra failures → 500/503
└─ app.module.ts
```

## Inside a context

| Path | What lives there | Required? |
|------|------------------|-----------|
| `<ctx>.controller.ts` | The single HTTP controller of the context. Routes to use-cases (writes) and to the query service (reads). | When the context has HTTP endpoints |
| `<ctx>.dto.ts` | HTTP DTOs with validation decorators or Zod schemas. Edge only. | With the controller |
| `<ctx>.module.ts` | Composition root: providers, port → adapter bindings. | Always |
| `use-cases/` | One `<action>.use-case.ts` per write action, flat, no subfolder per use-case. Spec file beside each command. | Always |
| `domain/` | Flat. Entities, aggregate roots, value objects, domain services and ports (abstract classes). No `value-objects/`, `entities/` or `ports/` subfolders. | Always |
| `workers/` | Cron jobs (`<action>.cron.ts`) and queue consumers (`<event>.consumer.ts`). Thin input adapters calling use-cases. | When the context has scheduled or async entry points |
| `services/` | The context's query service (`<ctx>.query-service.ts`). Application-level helpers that are neither a use-case nor a domain rule. | Only if needed |
| `adapters/` | Implementations of the context's ports: TypeORM repository + its model, vendor ACL adapters. | Only if needed |

The tree above is a level-2 context. A level-0 context is just the controller, the DTO file, the module, `use-cases/`, a `domain/` with a type and a port, and `adapters/` with the TypeORM repository — no `workers/`, often no `services/` until the first list endpoint. Create a folder when the first file for it appears.

`domain/` stays flat until it hurts. When a context has so many files that `domain/` is hard to scan, that is usually a sign the context should be split — not that it needs subfolders.

## Where does this file go

Ask in order:

1. A **write action**? → `use-cases/<action>.use-case.ts`
2. A **read** for a screen, list or report? → the query service in `services/`
3. A **business rule** on one aggregate? → method on the entity in `domain/`
4. A rule spanning several aggregates or needing data the entity does not hold? → domain service in `domain/`
5. A value with its own validation (money, document, key, barcode)? → value object in `domain/`
6. A **contract for external I/O** (database, vendor, queue)? → port (abstract class) in `domain/`
7. An **implementation of a port** used only by this context? → `adapters/`
8. A **shared concrete** (DataSource, vendor low-level client, SNS publisher, logger)? → `infrastructure/`
9. A **schedule or queue entry point**? → `workers/`
10. An **HTTP concern** (DTO, route)? → context root, beside the controller
11. Something **operational** (health check, metrics, outbox relay)? → `infrastructure/`, never `contexts/`

## Global infrastructure and commons

The line for `infrastructure/` is: **is it reused across contexts, or is it operational?**

- `infrastructure/integrations/acme/` — the Acme client (auth, base URL, timeout, circuit breaker, DTOs, `AcmeApiError`). Payments and recharges both use it. **Global.**
- `contexts/payments/adapters/acme-pix.gateway.ts` — implements the `PixGateway` port, maps `Payment` ↔ Acme DTOs, translates `AcmeApiError` into business outcomes. Only payments cares. **Context.**

That is the two-layer integration pattern — see `integration.md`.

`commons/` holds what every context imports and that has no I/O: `Result`, `BusinessError`, the `AggregateRoot` base, and the HTTP translation of business errors (status map, `ResultInterceptor`, exception filters). Nothing in `commons/` imports from `contexts/` or `infrastructure/`.

## Dependency rule

```
controller / workers → use-cases → domain ← adapters → infrastructure
                                     ↑
                                  commons
```

- `domain/` imports nothing from NestJS, TypeORM, Axios, `infrastructure/` or `use-cases/`. Only `commons/` and other files in `domain/`.
- Use-cases import `domain/` and `commons/`. Never an adapter, never `infrastructure/`.
- Adapters import the port they implement (from `domain/`) and global `infrastructure/`.
- A context never imports another context's internals. Cross-context communication goes through events or a port — see `bounded-contexts.md`.

A domain type that a use-case needs (`CreatePaymentProps`) is declared in `domain/`. The domain never imports a use-case `Input` type — that would point the arrow outward.

## Naming

| Kind | File | Class |
|------|------|-------|
| Use-case | `create-payment.use-case.ts` | `CreatePaymentUseCase` |
| Query service | `payments.query-service.ts` | `PaymentsQueryService` |
| Entity / aggregate | `payment.ts` | `Payment` |
| Value object | `money.ts` | `Money` |
| Domain service | `payment-limit.service.ts` | `PaymentLimitService` |
| Port | `payment.repository.ts`, `pix-gateway.ts` | `PaymentRepository`, `PixGateway` |
| Adapter | `typeorm-payment.repository.ts`, `acme-pix.gateway.ts` | `TypeOrmPaymentRepository`, `AcmePixGateway` |
| Cron worker | `reconcile-payments.cron.ts` | `ReconcilePaymentsCron` |
| Queue worker | `boleto-paid.consumer.ts` | `BoletoPaidConsumer` |

## Why not layer-first

A global `controllers/ services/ repositories/` layout means one feature change touches three distant folders, and unrelated features sit next to each other while related code sits far apart. The context folder keeps everything one change needs in one place, and deleting a context is one `rm -rf`.

The trade-off is some duplication across contexts. Accept it. Premature sharing couples things that only look alike and is much harder to undo than duplication. Extract to `infrastructure/` or `commons/` on the third real repetition, not the second.
