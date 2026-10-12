# Repositories & Ports

Contents: [Ports are abstract classes](#ports-are-abstract-classes) · [The repository port](#the-repository-port) · [The adapter](#the-adapter) · [Wiring](#wiring) · [Port only for external I/O](#port-only-for-external-io) · [Test fakes](#test-fakes)

## Ports are abstract classes

**Never `interface`.** An interface vanishes at compile time, so it cannot be a DI token — which forces `Symbol` tokens plus `@Inject(TOKEN)` on every constructor.

An `abstract class` survives at runtime and **is its own token**:

```ts
export abstract class PaymentRepository {
  abstract save(payment: Payment): Promise<void>;
  abstract findById(paymentId: string): Promise<Payment | null>;
  abstract totalSettledToday(accountId: string): Promise<Money>;
}
```

Injection is then by type, with no decorator:

```ts
constructor(private readonly repository: PaymentRepository) {}
```

No `Symbol`. No `@Inject`. The class is the contract and the token at once.

Ports live flat in `contexts/<ctx>/domain/` (`payment.repository.ts`, `pix-gateway.ts`). They are declared by the inside and implemented by the outside — the compile-time arrow points inward even though the runtime call goes outward.

## The repository port

Keep it aggregate-shaped: `save`, `findById`, plus domain-level questions a command needs (`totalSettledToday`). Screen-specific finders belong in the query service — see `cqs.md`.

The port speaks domain types only: it takes and returns `Payment` and `Money`, never ORM models, never `QueryRunner`, never `Observable`.

## The adapter

Implements the port, maps domain ↔ persistence model, and drains the aggregate's events into the outbox in the same transaction. It lives in `contexts/<ctx>/adapters/` with its persistence model.

```ts
@Injectable()
export class TypeOrmPaymentRepository implements PaymentRepository {
  constructor(private readonly dataSource: DataSource) {}

  async save(payment: Payment): Promise<void> {
    await this.dataSource.getRepository(PaymentModel).save(PaymentMapper.toModel(payment));
  }

  async findById(paymentId: string): Promise<Payment | null> {
    const model = await this.dataSource.getRepository(PaymentModel).findOneBy({ id: paymentId });

    if (!model) {
      return null;
    }

    return PaymentMapper.toDomain(model);
  }

  async totalSettledToday(accountId: string): Promise<Money> {
    const rows = await this.dataSource.query<{ total: string }[]>(
      `SELECT COALESCE(SUM(amount_cents), 0) AS total
         FROM payments
        WHERE account_id = $1 AND status = 'SETTLED' AND created_at >= date_trunc('day', now())`,
      [accountId],
    );

    return PaymentMapper.toMoney(Number(rows[0].total));
  }
}
```

- `implements`, not `extends` — the port is a contract, not inherited behavior.
- The mapper (`PaymentMapper`, beside the adapter) uses `Payment.restore(...)` to rebuild the aggregate without re-validating or re-recording events.
- Persistence failures throw. The adapter never turns a database error into a `BusinessError`.

**When the context publishes events** (level 2+), `save` also drains the aggregate's events into the outbox, in the same transaction as the state:

```ts
async save(payment: Payment): Promise<void> {
  await this.dataSource.transaction(async (manager) => {
    await manager.save(PaymentModel, PaymentMapper.toModel(payment));

    for (const event of payment.pullEvents()) {
      await manager.insert(OutboxModel, OutboxMapper.fromDomainEvent(event, payment.id));
    }
  });
}
```

Do not add this to repositories of contexts that publish nothing. If the project uses a unit of work, the adapter reads `transactionHost.tx` instead of opening its own transaction (`transactions.md`).

Shared low-level concretes (the `DataSource` configuration, migrations, the outbox model) live in global `infrastructure/`.

## Wiring

The module binds each port to its adapter. Internal collaborators are plain providers.

```ts
@Module({
  imports: [InfrastructureModule, AcmeIntegrationModule],
  controllers: [PaymentsController],
  providers: [
    CreatePaymentUseCase,
    RefundPaymentUseCase,
    PaymentsQueryService,
    PaymentLimitService,
    FeeCalculator,
    ReconcilePaymentsCron,
    { provide: PaymentRepository, useClass: TypeOrmPaymentRepository },
    { provide: PixGateway, useClass: AcmePixGateway },
    { provide: AccountLimits, useClass: ShadowAccountLimits },
  ],
})
export class PaymentsModule {}
```

`PaymentLimitService` and `FeeCalculator` have no constructor dependencies, so Nest instantiates them without `@Injectable()` and the domain stays free of framework imports.

## Port only for external I/O

Define a port for: database, vendor/HTTP gateway, another service, queue/broker, cache, clock or id generation (when tests need control).

Do **not** define a port for: pure logic, calculators, mappers, formatters. Inject those as concrete classes. Abstraction without a second implementation is a cost with no benefit — an extra file, an extra indirection, and a test double where the real object would be simpler and truer.

Extract a port when: a second implementation appears, a fake is genuinely needed for tests, or the collaborator starts doing I/O.

## Test fakes

Because the port is an abstract class, a fake is a few readable lines — no mocking framework needed:

```ts
export class InMemoryPaymentRepository extends PaymentRepository {
  private readonly payments = new Map<string, Payment>();

  async save(payment: Payment): Promise<void> {
    this.payments.set(payment.id, payment);
  }

  async findById(paymentId: string): Promise<Payment | null> {
    return this.payments.get(paymentId) ?? null;
  }

  async totalSettledToday(): Promise<Money> {
    return Money.zero();
  }
}
```

Fakes live beside the specs that use them, never in production code paths.
