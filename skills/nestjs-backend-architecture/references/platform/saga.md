# Saga Pattern

Contents: [Why sagas](#why-sagas) · [Orchestrated](#orchestrated) · [Persisting saga state](#persisting-saga-state) · [Reconciliation](#reconciliation) · [Choreographed](#choreographed) · [Choosing](#choosing) · [Compensation rules](#compensation-rules)

## Why sagas

A flow spanning several services or aggregates cannot use one transaction, and 2PC is not realistic across HTTP and queue boundaries. A saga splits the work into local transactions, each with a **compensation** that semantically undoes it if a later step fails.

The result is **eventual consistency**, not atomicity: there is a window where the system is partially committed, and the design must tolerate it.

Do not reach for a saga when one transaction suffices. If every write lives in one database, use a transaction.

## Orchestrated

A coordinator drives the steps and compensates completed ones in reverse order. It is a state machine you can test, log and inspect.

Two kinds of failure, handled differently — the same split as everywhere else:
- A step returns `Result.Err` → **business** failure (insufficient balance, provider declined). Compensate and return the `Err`.
- A step **throws** → **infrastructure** failure (timeout, circuit open). The outcome of that step is unknown. Do not compensate blindly: persist the saga as `UNCERTAIN`, rethrow, and let retry or reconciliation resolve it.

```ts
type SagaStep = {
  name: string;
  run: () => Promise<Result<void, BusinessError>>;
  compensate: () => Promise<void>;
};

@Injectable()
export class PayBoletoSaga {
  constructor(
    private readonly accounts: CoreBankingGateway,
    private readonly boletos: BoletoGateway,
    private readonly sagaState: SagaStateRepository,
    private readonly logger: Logger,
  ) {}

  async execute(payment: BoletoPayment): Promise<Result<void, BusinessError>> {
    const steps: SagaStep[] = [
      {
        name: 'block-balance',
        run: () => this.accounts.blockBalance(payment.accountId, payment.amount, payment.id),
        compensate: () => this.accounts.releaseBalance(payment.accountId, payment.id),
      },
      {
        name: 'pay-boleto',
        run: () => this.boletos.pay(payment.barcode, payment.amount, payment.id),
        compensate: () => Promise.resolve(),
      },
      {
        name: 'debit-balance',
        run: () => this.accounts.debitBlocked(payment.accountId, payment.id),
        compensate: () => Promise.resolve(),
      },
    ];

    const completed: SagaStep[] = [];

    for (const step of steps) {
      const result = await this.runStep(payment, step);

      if (result.isErr) {
        await this.compensate(payment, completed);
        return result;
      }

      completed.push(step);
    }

    await this.sagaState.markCompleted(payment.id);

    return Result.Ok();
  }

  private async runStep(payment: BoletoPayment, step: SagaStep): Promise<Result<void, BusinessError>> {
    await this.sagaState.markRunning(payment.id, step.name);

    try {
      return await step.run();
    } catch (error) {
      await this.sagaState.markUncertain(payment.id, step.name);
      this.logger.warn({ message: 'pay-boleto-saga.step-uncertain', paymentId: payment.id, step: step.name }, PayBoletoSaga.name);
      throw error;
    }
  }

  private async compensate(payment: BoletoPayment, completed: SagaStep[]): Promise<void> {
    await this.sagaState.markCompensating(payment.id);

    for (const step of [...completed].reverse()) {
      await step.compensate();
    }

    await this.sagaState.markCompensated(payment.id);
  }
}
```

The irreversible step (`pay-boleto`: money left to the beneficiary) comes after everything reversible; once it succeeds there is nothing left that can fail for business reasons and needs undoing.

## Persisting saga state

An in-memory loop is only correct if the process survives the whole flow. It will not. Persist progress:

```
saga_instances: id, type, aggregate_id, current_step, status, payload, updated_at
status: RUNNING → COMPLETED | COMPENSATING → COMPENSATED | UNCERTAIN | FAILED
```

Record the step **before** executing it and the result **after**. Recovery then knows what *may* have run; combined with idempotent steps, re-running an uncertain step is safe.

For long flows, drive transitions through queues instead of one in-process loop: each step completion enqueues the next, giving durability and retries for free.

**Add timeouts.** A step that never answers must eventually trip compensation or reconciliation, or the saga holds reservations forever.

## Reconciliation

A worker (`workers/reconcile-<flow>.cron.ts`) periodically picks sagas stuck in `RUNNING` or `UNCERTAIN` past a threshold, **consults the provider for the real state** (never assumes), and drives each to `COMPLETED` or compensation. The provider's real state wins; the local state is corrected to match.

This is also how a saga recovers from a provider outage: during the outage the circuit breaker fails fast (`../resilience.md`), flows stay pending, and reconciliation resolves them when the provider is back. Nothing is lost.

## Choreographed

No coordinator — each service reacts to events and publishes its own; the flow emerges from the chain:

```
Payments ──PaymentCreated──▶ Accounts ──BalanceDebited──▶ PixSettlement ──PixSettled──▶ Ledger
                                 ◀──PixFailed (→ refund)──┘
   ◀──BalanceRefunded (→ cancel)──┘
```

Compensation is an event flowing backwards. Low coupling is the appeal; the cost is that no single place describes the flow. Debugging a stuck saga means correlating logs across services — the `correlationId` in every envelope is not optional here.

## Choosing

- **Orchestrated** — many steps, coordinated timeouts and retries, and someone will ask "where is payment 123 stuck?". The default for payment flows.
- **Choreographed** — few steps, genuinely independent services, decoupling matters more than traceability.

Mixed is fine: orchestrate the critical path, let peripheral consumers (notifications, analytics) react choreographically.

## Compensation rules

- **Semantic, not technical.** The inverse of a debit is a credit visible in the statement — never a `DELETE`. Financial history is append-only.
- **Idempotent** — compensations get retried like everything else.
- **Should not fail.** A failed compensation leaves money in limbo. Keep them simple and always possible; if one can fail, it needs its own retry and alarm.
- **Some steps cannot be compensated** — a settled PIX, a paid boleto, a sent notification. Order the saga so irreversible steps come **last**.
- **Beware semantic locks.** Between block and debit the funds are held; other flows must see that reservation or you get double spend.
