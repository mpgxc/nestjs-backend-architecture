# Use-Cases

Contents: [What a use-case is](#what-a-use-case-is) · [One file, three things](#one-file-three-things) · [Command output](#command-output) · [Logging and failures](#logging-and-failures) · [Level 0: no rules](#level-0-no-rules) · [Level 1: rules in the domain](#level-1-rules-in-the-domain) · [Level 2+: value objects, domain service, events](#level-2-value-objects-domain-service-events) · [Reacting to an infrastructure failure](#reacting-to-an-infrastructure-failure) · [Internal collaborators](#internal-collaborators) · [Batch use-cases](#batch-use-cases) · [Rules](#rules)

## What a use-case is

A use-case is a **thin orchestrator** for one write action. It sequences steps, propagates `Result`, logs business decisions and talks to the outside through ports. It holds no business rule of its own — rules live in entities, value objects and domain services.

One use-case per action: `CreatePaymentUseCase`, `RefundPaymentUseCase`, `CancelPaymentUseCase`. Never a `PaymentsService` with six public methods.

- Always a concrete class with a single public `execute(input)`. No `UseCase<I, O>` interface, no abstract base class — nobody injects an "abstract use-case".
- Lives in `contexts/<ctx>/use-cases/`, flat, one file per action.
- Transport-agnostic: the same use-case runs from HTTP, a queue consumer or a cron worker.
- As light as its context's level: a level-0 use-case is a few lines. Do not copy the level-2 example into a CRUD context.

## One file, three things

The `Input` type, the `Output` type and the class live together in `<action>.use-case.ts`. No separate `.dto.ts` for the use-case.

The `Input` is not the HTTP DTO. The DTO (validation decorators, wire shape) stays beside the controller; the controller maps DTO → `Input`. The use-case never imports a DTO.

## Command output

A command exists to change state, so by default it returns `Result<void, BusinessError>` and succeeds with `Result.Ok()`.

When the caller genuinely needs something back — the generated id after a create is the classic case — declare a minimal `Output`. Over HTTP the `Output` **is** the response body (the global `ResultInterceptor` unwraps it), so name its fields for the client and never return the aggregate (`controllers.md`).

Reads go through the query service, not a use-case — see `cqs.md`.

## Logging and failures

What a use-case logs, inline (`code-style.md`):
- `<action>.started` with the business identifiers (`accountId`, `paymentId`, `amountCents`);
- `<action>.rejected` with `reason: error.type` on every business guard that returns `Err`;
- `<action>.finished` on success.

What a use-case does **not** do: wrap I/O in a `try/catch` that only logs and rethrows. An infrastructure failure is logged **once**, at the edge — by `UnhandledExceptionFilter` for HTTP (`bootstrap.md`), by the consumer's error handler or the scheduler for workers — with the same `correlationId` as the `started` line, which already carries the business identifiers. A log-and-rethrow in the use-case only duplicates that error line.

A `try/catch` belongs in a use-case when it **does something** with the failure: compensates, records a failure state, releases a reservation. See [Reacting to an infrastructure failure](#reacting-to-an-infrastructure-failure).

## Level 0: no rules

A context with no invariants (preferences, catalog, configuration). The domain is a plain type and a port (`domain.md`); the use-case loads, applies, saves.

```ts
export type UpdateNotificationPreferencesInput = {
  accountId: string;
  emailEnabled: boolean;
  pushEnabled: boolean;
};

@Injectable()
export class UpdateNotificationPreferencesUseCase {
  constructor(private readonly repository: NotificationPreferencesRepository) {}

  async execute(input: UpdateNotificationPreferencesInput): Promise<Result<void, BusinessError>> {
    await this.repository.save({
      accountId: input.accountId,
      emailEnabled: input.emailEnabled,
      pushEnabled: input.pushEnabled,
      updatedAt: new Date(),
    });

    return Result.Ok();
  }
}
```

No value objects, no entity class, no logs beyond the access log — nothing here is a business decision worth recording. It still returns `Result`, so the controller and the error contract stay uniform; the day a rule appears, it becomes a guard without changing any signature.

## Level 1: rules in the domain

The entity owns the rule; the use-case loads, tells, saves, and logs the decisions.

```ts
export type RefundPaymentInput = {
  paymentId: string;
};

/**
 * Estorna um pagamento liquidado. O estorno é um evento novo no histórico,
 * nunca a remoção do pagamento original.
 */
@Injectable()
export class RefundPaymentUseCase {
  constructor(
    private readonly repository: PaymentRepository,
    private readonly logger: Logger,
  ) {}

  async execute(input: RefundPaymentInput): Promise<Result<void, BusinessError>> {
    this.logger.log({ message: 'refund-payment.started', paymentId: input.paymentId }, RefundPaymentUseCase.name);

    const payment = await this.repository.findById(input.paymentId);

    if (!payment) {
      this.logger.log({ message: 'refund-payment.rejected', paymentId: input.paymentId, reason: 'PAYMENT_NOT_FOUND' }, RefundPaymentUseCase.name);
      return Result.Err({ type: 'PAYMENT_NOT_FOUND', message: 'pagamento inexistente' });
    }

    const refunded = payment.refund();

    if (refunded.isErr) {
      this.logger.log({ message: 'refund-payment.rejected', paymentId: input.paymentId, reason: refunded.error.type }, RefundPaymentUseCase.name);
      return refunded;
    }

    await this.repository.save(payment);

    this.logger.log({ message: 'refund-payment.finished', paymentId: input.paymentId }, RefundPaymentUseCase.name);

    return Result.Ok();
  }
}
```

This is the default shape for any command with business rules: entry log → guards that fail fast → writes → exit log → `Result.Ok`. Braces on every block, no `else`, no nesting, no inline comments.

The logger is Nest's own `Logger` class, provided once by the global infrastructure module and backed by the Pino adapter (`logging.md`). Injecting it keeps it replaceable in tests.

## Level 2+: value objects, domain service, events

A money-moving command in a context that publishes events. Value objects for the values with rules, the aggregate, a rule spanning data the entity does not hold, and persistence that also writes the outbox.

```ts
export type CreatePaymentInput = {
  accountId: string;
  amountCents: number;
  pixKey: string;
};

export type CreatePaymentOutput = {
  paymentId: string;
};

/**
 * Cria um pagamento PIX pendente respeitando o limite diário da conta.
 * A liquidação acontece de forma assíncrona a partir do evento `payment.created`.
 */
@Injectable()
export class CreatePaymentUseCase {
  constructor(
    private readonly repository: PaymentRepository,
    private readonly accountLimits: AccountLimits,
    private readonly limitService: PaymentLimitService,
    private readonly logger: Logger,
  ) {}

  async execute(input: CreatePaymentInput): Promise<Result<CreatePaymentOutput, BusinessError>> {
    this.logger.log({ message: 'create-payment.started', accountId: input.accountId, amountCents: input.amountCents }, CreatePaymentUseCase.name);

    const amount = Money.fromCents(input.amountCents);

    if (amount.isErr) {
      this.logger.log({ message: 'create-payment.rejected', accountId: input.accountId, reason: amount.error.type }, CreatePaymentUseCase.name);
      return amount;
    }

    const pixKey = PixKey.create(input.pixKey);

    if (pixKey.isErr) {
      this.logger.log({ message: 'create-payment.rejected', accountId: input.accountId, reason: pixKey.error.type }, CreatePaymentUseCase.name);
      return pixKey;
    }

    const payment = Payment.create({ accountId: input.accountId, amount: amount.value, pixKey: pixKey.value });

    if (payment.isErr) {
      this.logger.log({ message: 'create-payment.rejected', accountId: input.accountId, reason: payment.error.type }, CreatePaymentUseCase.name);
      return payment;
    }

    const todayTotal = await this.repository.totalSettledToday(input.accountId);
    const dailyLimit = await this.accountLimits.dailyPixLimitOf(input.accountId);
    const withinLimit = this.limitService.assertWithinDailyLimit(payment.value, todayTotal, dailyLimit);

    if (withinLimit.isErr) {
      this.logger.log({ message: 'create-payment.rejected', accountId: input.accountId, reason: withinLimit.error.type }, CreatePaymentUseCase.name);
      return withinLimit;
    }

    await this.repository.save(payment.value);

    this.logger.log({ message: 'create-payment.finished', paymentId: payment.value.id }, CreatePaymentUseCase.name);

    return Result.Ok({ paymentId: payment.value.id });
  }
}
```

Notice what is **not** here:
- No call to the PIX provider. Dispatching money before the payment is persisted creates an irreversible effect with no record of it. The aggregate records `payment.created`, the repository writes it to the outbox in the same transaction, and a worker settles it (`outbox.md`).
- No limit arithmetic. The rule is in `PaymentLimitService`; the use-case only fetches the data and passes it in.
- `AccountLimits` is a port in `domain/`. Its adapter can be a guarded call to the accounts service or, in a platform that replicates account data, a query on the local replica (`platform/read-models.md`). The use-case does not know which.

## Reacting to an infrastructure failure

When the use-case must leave the world consistent after a failure, the `try/catch` is legitimate — it reacts, then rethrows so retry or queue machinery still sees the failure:

```ts
try {
  await this.boletoGateway.register(boleto);
} catch (error) {
  boleto.markRegistrationPending();
  await this.repository.save(boleto);
  this.logger.warn({ message: 'issue-boleto.registration-deferred', boletoId: boleto.id }, IssueBoletoUseCase.name);
  throw error;
}
```

Keep the `try` tight around the one call whose failure you handle. The log here is a business fact ("deferred"), at `warn`; the error itself is still logged once at the edge.

## Internal collaborators

A collaborator with no I/O gets **no port**. Inject the concrete class:

```ts
export class FeeCalculator {
  private static readonly PIX_FEE_RATE = 0.0099;

  calculate(amount: Money): Money {
    return amount.percentage(FeeCalculator.PIX_FEE_RATE);
  }
}
```

Extract a port later only if a second implementation appears, a test fake becomes genuinely necessary, or it starts doing I/O.

## Batch use-cases

A use-case triggered by a cron worker may process a batch (`claimPending(limit)` → handle each → persist each) inside one `execute()`. It is still one use-case per action — the action is "reconcile the pending batch". Keep the loop body in a private method so `execute` stays at one indentation level, and let one item's business failure be logged and skipped without aborting the rest. See `workers.md`.

Testing of use-cases (specs with port fakes beside each command) is in `testing.md`.

## Rules

- Pick the lightest shape that fits the context's level.
- Return `Result<T, BusinessError>`; never `throw` for a business outcome.
- Commands return `void` by default; a minimal, client-named `Output` only when the caller needs it.
- Guard and return on the first `Err`, braces always, no `else`, no nesting.
- Log `started`, each `rejected` with `reason`, and `finished` — inline, with business identifiers. Level 0 use-cases may skip them.
- No log-and-rethrow `try/catch`; catch only to react (compensate, record failure state), then rethrow.
- No HTTP, no `Request`, no DTO inside the use-case.
- Business rules belong in the domain. If the use-case grows `if`s about what is allowed, move them.
