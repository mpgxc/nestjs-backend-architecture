# Transactions & Unit of Work

Contents: [Default: one aggregate, one save](#default-one-aggregate-one-save) · [When one transaction must cover more](#when-one-transaction-must-cover-more) · [The UnitOfWork port](#the-unitofwork-port) · [Repositories join the transaction](#repositories-join-the-transaction) · [Rules](#rules)

## Default: one aggregate, one save

Most commands change one aggregate. The repository's `save` opens its own short transaction (state, plus outbox rows when the context publishes events) and that is all the use-case needs. Do not add a unit of work for symmetry.

## When one transaction must cover more

Real cases that need several writes to succeed or fail together, in the **same database and the same service**:

- a ledger entry and the balance it changes;
- a business write and the `processed_messages` row of the consumer that triggered it (`idempotency.md`);
- a parent and children owned by different repositories (order and items, boleto batch and boletos).

First ask whether the boundary is wrong — if two things must always change together, they may be one aggregate saved by one repository. If not, use a unit of work. Across services, never: that is a saga (`platform/saga.md`).

## The UnitOfWork port

A tiny port in `commons/`, so the use-case states the transactional boundary explicitly without importing TypeORM:

```ts
export abstract class UnitOfWork {
  abstract run<T>(work: () => Promise<T>): Promise<T>;
}
```

The adapter in `infrastructure/database/` uses `@nestjs-cls/transactional` with the TypeORM adapter, which keeps the active transaction in `AsyncLocalStorage`:

```ts
@Injectable()
export class ClsUnitOfWork extends UnitOfWork {
  constructor(private readonly transactionHost: TransactionHost<TransactionalAdapterTypeOrm>) {
    super();
  }

  run<T>(work: () => Promise<T>): Promise<T> {
    return this.transactionHost.withTransaction(work);
  }
}
```

The use-case draws the boundary in plain sight:

```ts
async execute(input: PostLedgerEntryInput): Promise<Result<void, BusinessError>> {
  const amount = Money.fromCents(input.amountCents);

  if (amount.isErr) {
    return amount;
  }

  const balance = await this.balances.findByAccountId(input.accountId);

  if (!balance) {
    return Result.Err({ type: 'BALANCE_NOT_FOUND', message: 'saldo da conta inexistente' });
  }

  const credited = balance.credit(amount.value);

  if (credited.isErr) {
    return credited;
  }

  await this.unitOfWork.run(async () => {
    await this.entries.save(LedgerEntry.credit(input.accountId, amount.value, input.sourceEventId));
    await this.balances.save(balance);
  });

  return Result.Ok();
}
```

All business decisions happen **before** the transaction opens; the transaction only contains writes. Keep it short: no HTTP calls, no vendor calls, no waiting inside it.

## Repositories join the transaction

When a project adopts the unit of work, every repository adapter reads the entity manager from the transaction host instead of the `DataSource`. Outside a `run`, it behaves as a normal non-transactional manager; inside, it joins the open transaction automatically.

```ts
@Injectable()
export class TypeOrmBalanceRepository implements BalanceRepository {
  constructor(private readonly transactionHost: TransactionHost<TransactionalAdapterTypeOrm>) {}

  async save(balance: Balance): Promise<void> {
    await this.transactionHost.tx.save(BalanceModel, BalanceMapper.toModel(balance));
  }
}
```

A repository whose `save` drains events into the outbox no longer opens its own transaction: inside `run` it joins the caller's; outside, wrap that single `save` in `run` in the use-case.

## Rules

- One aggregate per command is the default; a unit of work is the exception with a reason.
- Same database, same service only. Across services, saga.
- Decide first, then open the transaction with writes only.
- Adopt it project-wide (all adapters on `transactionHost.tx`) or not at all — mixing `DataSource` and `transactionHost` silently splits transactions.
