# Hexagonal Architecture (Ports & Adapters)

Contents: [The shape](#the-shape) · [Driving side](#driving-side) · [Driven side](#driven-side) · [Gateway vs adapter](#gateway-vs-adapter) · [Service-to-service is the same pattern](#service-to-service-is-the-same-pattern) · [Relation to Clean and DDD](#relation-to-clean-and-ddd)

## The shape

The core (domain + use-cases) sits in the middle with **zero dependencies on frameworks or vendors**. Everything else is an adapter on the outside.

- **Driving (primary, input)** — what *calls* the application: the REST controller, a queue consumer, a cron worker. They depend on the core.
- **Driven (secondary, output)** — what the application *calls*: the repository, a vendor gateway, another service. The core declares the port; the adapter implements it.

Dependency inversion means the compile-time arrow points **inward** even though the runtime call goes outward.

Cockburn's test: the core never names an external technology. If `domain/` or `use-cases/` mentions "Acme", "axios", a URL or a table name, the boundary is already broken.

## Driving side

The controller and the workers call the concrete use-case class directly. That is simpler and correct: there is never a second implementation of a use-case, so an "input port" abstract class adds a file and buys nothing.

```ts
@Injectable()
export class BoletoPaidConsumer {
  constructor(private readonly settleBoleto: SettleBoletoUseCase) {}

  @SqsMessageHandler('payments-boleto-paid', false)
  async handle(message: Message): Promise<void> {
    const event = parseEnvelope<BoletoPaidData>(message);

    await this.settleBoleto.execute({ boletoId: event.data.boletoId, paidAt: event.data.paidAt });
  }
}
```

Every driving adapter maps its own input shape (DTO, message envelope, schedule config) to the use-case `Input`. None of them passes its raw payload into the core.

## Driven side

The core declares what it needs, in business language; infrastructure supplies it:

```ts
/**
 * Liquidação de pagamentos PIX no provedor.
 * Desfechos de negócio voltam em Result.Err; indisponibilidade é lançada.
 */
export abstract class PixGateway {
  abstract dispatch(payment: Payment): Promise<Result<void, BusinessError>>;
}
```

`AcmePixGateway implements PixGateway` in `adapters/` calls the global Acme client and translates outcomes — the full adapter is in `integration.md`.

The port name and methods express intent (`dispatch`, `authorize`, `issueInvoice`), never protocol (`postPayment`, `callEndpoint`). Changing transport from HTTP to gRPC or a queue changes the adapter, not the port.

## Gateway vs adapter

The **gateway** is the port — the "what", an abstract class in `domain/`. The **adapter** is the implementation — the "how", in `adapters/`. A gateway with no adapter is a contract nobody fulfils; an adapter with no gateway couples the core to the detail. Together they decouple.

## Service-to-service is the same pattern

Calling another of our own services is no different from calling a vendor: a port in the caller's domain (`BillingGateway.issueInvoice(order)`), an HTTP adapter that maps to the other service's contract and translates its errors, and a driving adapter (controller) on the other side. Two hexagons touching at the edges, never at the cores.

- Never share entity classes or a types package between services; each side owns its model and translates.
- Never share a database.
- Give the HTTP adapter a timeout and a circuit breaker (`resilience.md`) and authenticate the call (`platform/service-auth.md`).
- Before adding a synchronous call, ask whether the data can be replicated by events instead (`platform/read-models.md`).

## Relation to Clean and DDD

Three lenses on the same structure, not competing options:

- **Clean/Onion** — the *dependency direction*: dependencies point inward to the domain.
- **Hexagonal** — the *boundary mechanics*: how the inside talks to the outside via ports and adapters.
- **DDD** — *what goes inside*: aggregates, value objects, events, bounded contexts.

In this codebase they collapse into one layout: the context folder holds the domain (DDD), the port/adapter split (hexagonal), and the dependency rule (clean) is enforced by ports being abstract classes owned by `domain/`.
