# Domain: Entities, Value Objects, Domain Services

Contents: [Purity](#purity) · [Match the model to the problem](#match-the-model-to-the-problem) · [Value objects](#value-objects) · [Entity](#entity) · [Recording events](#recording-events) · [Transitions: tell, don't ask](#transitions-tell-dont-ask) · [First-class collections](#first-class-collections) · [Domain services](#domain-services) · [Entity vs domain service](#entity-vs-domain-service)

All files below live flat in `contexts/<ctx>/domain/`.

## Purity

The domain imports nothing from NestJS, TypeORM, Axios or `infrastructure/`. It may import `commons/` (`Result`, `BusinessError`) and other domain files. That is what makes it testable with `new` and nothing else.

Consequences:
- No `@Injectable()` in `domain/`. A domain service with no constructor dependencies is registered as a plain provider in the module and Nest instantiates it without the decorator.
- No ORM decorators on entities. The persistence model (`payment.model.ts`) lives in `adapters/` and the repository adapter maps between the two.
- Props types that the domain needs are declared in the domain, never imported from a use-case.

## Match the model to the problem

Model only as much as the rules demand (architecture levels in `SKILL.md`):

- **Level 0 — no invariants** (preferences, catalog entries, configuration): the domain is a plain exported `type` plus the repository port. No class, no factory, no `restore`, no value objects, no events. The use-case validates nothing beyond what the DTO already validated.
- **Level 1 — real rules**: an entity class with a factory and transition methods. Value objects **only where a value has its own rule** — money, CPF/CNPJ, PIX key, barcode, phone. A `name`, a `description`, an `externalId` stay strings.
- **Level 2+ — the context publishes events**: the entity also extends `AggregateRoot` and records events for the outbox.

Promote a context to the next level when a rule appears, not before.

```ts
export type NotificationPreferences = {
  accountId: string;
  emailEnabled: boolean;
  pushEnabled: boolean;
  updatedAt: Date;
};

export abstract class NotificationPreferencesRepository {
  abstract findByAccountId(accountId: string): Promise<NotificationPreferences | null>;
  abstract save(preferences: NotificationPreferences): Promise<void>;
}
```

That is a complete level-0 domain. Do not wrap it in a class to look like the rest.

## Value objects

Immutable, no identity, validated on construction, compared by value. Private constructor plus a static factory returning `Result` — an invalid value cannot exist.

```ts
export class Money {
  private constructor(private readonly cents: number) {}

  static fromCents(cents: number): Result<Money, BusinessError> {
    if (!Number.isInteger(cents)) {
      return Result.Err({ type: 'INVALID_AMOUNT', message: 'valor deve ser inteiro em centavos' });
    }

    if (cents < 0) {
      return Result.Err({ type: 'INVALID_AMOUNT', message: 'valor não pode ser negativo' });
    }

    return Result.Ok(new Money(cents));
  }

  static zero(): Money {
    return new Money(0);
  }

  add(other: Money): Money {
    return new Money(this.cents + other.cents);
  }

  percentage(rate: number): Money {
    return new Money(Math.round(this.cents * rate));
  }

  isGreaterThan(other: Money): boolean {
    return this.cents > other.cents;
  }

  isZero(): boolean {
    return this.cents === 0;
  }

  equals(other: Money): boolean {
    return this.cents === other.cents;
  }

  toCents(): number {
    return this.cents;
  }
}
```

- Money is always **integer cents**. Floats are never acceptable for currency. Conversion to a vendor's decimal format happens in the adapter.
- `Money` itself accepts zero; "a payment must be positive" is a rule of `Payment`, not of money in general.
- `add` and `percentage` (with a non-negative rate) cannot break the invariant, so they return `Money`, not `Result`. Rounding happens once, inside the value object. Only operations that can break an invariant return `Result`.
- `Cpf`, `Cnpj`, `PixKey`, `Barcode`, `PhoneNumber` follow the same shape. Validate once at the edge of the domain; trust them everywhere after.
- No value object without a rule. A wrapper that only holds a string adds a file and an indirection and protects nothing.

## Entity

An entity has identity and protects its invariants: private constructor, a factory that validates, a `restore` that rebuilds from persistence without re-validating.

```ts
export type PaymentStatus = 'PENDING' | 'SETTLED' | 'REFUNDED';

export type CreatePaymentProps = {
  accountId: string;
  amount: Money;
  pixKey: PixKey;
};

export class Payment {
  private constructor(
    readonly id: string,
    readonly accountId: string,
    readonly amount: Money,
    readonly pixKey: PixKey,
    private status: PaymentStatus,
  ) {}

  static create(props: CreatePaymentProps): Result<Payment, BusinessError> {
    if (props.amount.isZero()) {
      return Result.Err({ type: 'INVALID_AMOUNT', message: 'pagamento deve ter valor positivo' });
    }

    return Result.Ok(new Payment(randomUUID(), props.accountId, props.amount, props.pixKey, 'PENDING'));
  }

  static restore(snapshot: PaymentSnapshot): Payment {
    return new Payment(snapshot.id, snapshot.accountId, snapshot.amount, snapshot.pixKey, snapshot.status);
  }

  currentStatus(): PaymentStatus {
    return this.status;
  }
}
```

Rules:
- **One transaction, one aggregate.** Needing to change two atomically usually means the boundary is wrong; when it is not, use a unit of work (`transactions.md`); across services, a saga.
- **Reference other aggregates by id**, never by object.
- **Keep them small.** Fields that are only read for screens belong to the query service, not the entity.

## Recording events

Only when the context **publishes** integration events (level 2+). Until then, entities do not extend anything and repositories do not touch an outbox.

When it does, the entity extends the project's own small `AggregateRoot` in `commons/` (not `@nestjs/cqrs`, so the domain stays free of framework imports) and records events in its factory and transitions:

```ts
export class Payment extends AggregateRoot {
  static create(props: CreatePaymentProps): Result<Payment, BusinessError> {
    if (props.amount.isZero()) {
      return Result.Err({ type: 'INVALID_AMOUNT', message: 'pagamento deve ter valor positivo' });
    }

    const payment = new Payment(randomUUID(), props.accountId, props.amount, props.pixKey, 'PENDING');
    payment.record({ type: 'payment.created', payload: { paymentId: payment.id, amountCents: props.amount.toCents() } });

    return Result.Ok(payment);
  }
}
```

```ts
export type DomainEvent = {
  type: string;
  payload: Record<string, unknown>;
};

export abstract class AggregateRoot {
  private events: DomainEvent[] = [];

  protected record(event: DomainEvent): void {
    this.events.push(event);
  }

  pullEvents(): DomainEvent[] {
    const pulled = this.events;
    this.events = [];

    return pulled;
  }
}
```

- **Events are recorded, not published.** The repository drains `pullEvents()` into the outbox in the same transaction. See `outbox.md`.
- `restore` never re-records events.

## Transitions: tell, don't ask

Every state change is a method on the entity that checks its own precondition. The use-case tells; it never reads the status and decides.

```ts
refund(): Result<void, BusinessError> {
  if (this.status !== 'SETTLED') {
    return Result.Err({ type: 'INVALID_TRANSITION', message: 'só pagamentos liquidados podem ser estornados' });
  }

  this.status = 'REFUNDED';

  return Result.Ok();
}
```

The transition mutates the entity and returns `Result<void, ...>`. The use-case then saves the same instance. In a context that publishes events, the transition also calls `this.record(...)` right after changing the state.

## First-class collections

When a list carries a rule, wrap it so the rule has a home:

```ts
export class Installments {
  private constructor(private readonly items: Installment[]) {}

  static create(items: Installment[], total: Money): Result<Installments, BusinessError> {
    if (items.length === 0) {
      return Result.Err({ type: 'INVALID_INSTALLMENTS', message: 'informe ao menos uma parcela' });
    }

    let sum = Money.zero();

    for (const item of items) {
      sum = sum.add(item.amount);
    }

    if (!sum.equals(total)) {
      return Result.Err({ type: 'INVALID_INSTALLMENTS', message: 'soma das parcelas difere do total' });
    }

    return Result.Ok(new Installments(items));
  }

  count(): number {
    return this.items.length;
  }
}
```

A plain list with no rule stays a plain array — do not wrap for symmetry. Same for the other Object Calisthenics rules: they are signals for review, not quotas to fill (`code-style.md`).

## Domain services

Stateless. For a rule that spans two or more aggregates, or needs data the entity does not hold. Pure — receives what it needs as arguments, does no I/O.

```ts
export class PaymentLimitService {
  assertWithinDailyLimit(payment: Payment, todayTotal: Money, dailyLimit: Money): Result<void, BusinessError> {
    const projected = todayTotal.add(payment.amount);

    if (projected.isGreaterThan(dailyLimit)) {
      return Result.Err({ type: 'INSUFFICIENT_LIMIT', message: 'excede o limite diário da conta' });
    }

    return Result.Ok();
  }
}
```

The use-case fetches `todayTotal` through a port and passes it in. If the domain service starts needing a repository, the I/O belongs to the use-case, not the service.

## Entity vs domain service

- Rule uses only **one** aggregate's state → method on the entity (`Payment.create`, `payment.refund()`).
- Rule spans **two or more** aggregates, or needs external data (daily limit, another account's balance) → domain service.
- A retry threshold, an eligibility decision, a fee calculation: still domain — entity method or domain service, never inline in the use-case.

Never put the rule in the use-case. If the use-case grows `if`s about *what is allowed*, the logic escaped the domain.
