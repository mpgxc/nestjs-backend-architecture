# Principles & Design Patterns

Contents: [Use when the problem exists](#use-when-the-problem-exists) · [SOLID in this codebase](#solid-in-this-codebase) · [DRY, KISS, YAGNI](#dry-kiss-yagni) · [GoF patterns that actually show up](#gof-patterns-that-actually-show-up) · [Do not force](#do-not-force)

Object Calisthenics lives in `code-style.md`.

## Use when the problem exists

Principles point a direction; patterns are named solutions to recurring problems. The most common mistake is applying them before the problem shows up, because they look elegant. Every pattern costs files, layers and reading effort; that cost pays only when the problem it solves is already real. Read each pattern below as "when this problem appears, this is the solution" — never "always do it this way".

Almost every pattern here is the same idea seen from different angles: **depend on a contract, not on an implementation**.

## SOLID in this codebase

- **S — Single responsibility.** One use-case per action; the controller does HTTP; a notifier sends e-mail. A class with three reasons to change gets split.
- **O — Open/closed.** A new provider is a new adapter class bound in a module or registered in a factory map — not a new `case` in a growing `switch`.
- **L — Liskov.** Every adapter honors the whole port contract, behavior included. If the port promises business outcomes in `Result` and availability as an exception, an adapter that throws for a business decline breaks every caller even though it compiles.
- **I — Interface segregation.** Small ports per capability (`TopupGateway`, `BoletoGateway`), not one `PaymentProvider` with twenty methods half of which throw "not supported".
- **D — Dependency inversion.** Use-cases depend on ports (abstract classes in `domain/`), never on `AcmeClient` or `HttpService`.

## DRY, KISS, YAGNI

- **DRY** is about knowledge, not similar-looking code. "Acme success means `errorCode === '000'`" lives in one mapper. Two blocks that look alike by coincidence stay separate.
- **KISS** — a clear `if` beats a Strategy with three classes when there is one case.
- **YAGNI** — no `PixGateway` port "because PIX is coming someday". Create it when PIX enters the roadmap.

The **rule of three** reconciles them: write it the first time, tolerate the repetition the second time, abstract on the third — when you have seen the real pattern and will not abstract the wrong thing from two coincidences. Adding an abstraction later is cheaper than removing a wrong one.

## GoF patterns that actually show up

| Problem | Pattern | Where it lives here |
|---------|---------|---------------------|
| Integrate a vendor without leaking its model | **Adapter** | `adapters/acme-pix.gateway.ts` implementing a port — the ACL (`integration.md`) |
| Choose the implementation by a runtime key (provider id) | **Strategy + simple Factory** | A factory provider with a `Record<ProviderId, TopupGateway>` |
| Add retry / circuit breaker / logging around calls without touching them | **Decorator** | Composed resilience policies wrapping the client (`resilience.md`) |
| Many react to one fact | **Observer** | Integration events over SNS → SQS between services (`messaging-pubsub.md`) |
| A pipeline of checks before the handler | **Chain of Responsibility** | Nest middleware, guards, pipes and interceptors |
| Build an object with many optional parts | **Builder** | Test data builders; rarely needed in production code with value objects |
| Exactly one shared instance | **Singleton** | Already what a Nest provider is. Never hand-roll `getInstance()` — global state that DI and tests cannot replace |

Strategy + Factory for providers:

```ts
@Injectable()
export class TopupGatewayFactory {
  private readonly gateways: Record<ProviderId, TopupGateway>;

  constructor(acme: AcmeTopupGateway, globex: GlobexTopupGateway) {
    this.gateways = {
      ACME: acme,
      GLOBEX: globex,
    };
  }

  forProvider(providerId: ProviderId): TopupGateway {
    return this.gateways[providerId];
  }
}
```

The map is typed `Record<ProviderId, ...>`, so adding a provider id without a gateway is a compile error — no runtime "not supported" branch needed. The use-case receives the port, never the concrete class.

Adapter vs Decorator: both wrap an object. The adapter **changes** the contract (vendor language → domain language); the decorator **keeps** it and adds behavior. That is why decorators stack and adapters do not.

## Do not force

- One case → a clear `if` beats a Strategy.
- Hypothetical future → YAGNI.
- Nest already solves it → do not reimplement Singleton or a DI container.
- Coincidental duplication → do not force DRY.
- A layer that only forwards calls → remove it; it hides what the code does.
