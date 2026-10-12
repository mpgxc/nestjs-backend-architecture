# Workers: Cron Jobs and Queue Consumers

Contents: [Workers are input adapters](#workers-are-input-adapters) · [Placement and naming](#placement-and-naming) · [Cron worker](#cron-worker) · [Batch use-case](#batch-use-case) · [Queue consumer](#queue-consumer) · [Where Result stops](#where-result-stops) · [Rules](#rules)

## Workers are input adapters

A cron trigger, a queue consumer and an HTTP controller are the same kind of thing: an **input adapter**. Each translates an external trigger — a request, a schedule tick, a queued message — into a call to a use-case with the `Input` already assembled. None of them hold business logic, and the use-case does not know or care which one called it.

If the same reconciliation later also needs a manual "reprocess now" endpoint or a queue trigger, add another thin adapter calling the same use-case. The use-case, the domain service and the repository never change.

## Placement and naming

Business workers live in `contexts/<ctx>/workers/`:

```
contexts/recharges/
├─ recharges.controller.ts
├─ workers/
│  ├─ reconcile-recharges.cron.ts      ReconcileRechargesCron
│  └─ recharge-requested.consumer.ts   RechargeRequestedConsumer
├─ use-cases/
│  └─ reconcile-recharges.use-case.ts
└─ ...
```

Operational jobs that are not business (outbox relay, cleanup of published outbox rows, cache warmers) live in `infrastructure/`.

## Cron worker

`ScheduleModule.forRoot()` once in `AppModule` (from `@nestjs/schedule`). The worker is a normal provider, constructor-injected with the use-case; it only declares the schedule and builds the `Input` from configuration.

```ts
const RECONCILE_BATCH_SIZE = 100;

@Injectable()
export class ReconcileRechargesCron {
  constructor(private readonly reconcileRecharges: ReconcileRechargesUseCase) {}

  @Cron(CronExpression.EVERY_MINUTE, { name: 'reconcile-recharges' })
  async run(): Promise<void> {
    await this.reconcileRecharges.execute({ batchSize: RECONCILE_BATCH_SIZE });
  }
}
```

- No `try/catch` in the worker. Infrastructure failures propagate so the scheduler and observability register the run as failed.
- With several ECS replicas, every replica runs the cron. The use-case must make that safe: the repository's `claimPending` takes a lease atomically — `UPDATE recharges SET lease_until = now() + interval '2 minutes' WHERE id IN (SELECT id ... WHERE lease_until < now() FOR UPDATE SKIP LOCKED LIMIT $1) RETURNING *` — so the claim survives after the short transaction ends. Each item is also guarded by a conditional transition. Alternatively run schedules from a single scheduled task (EventBridge → ECS task or Lambda) instead of in-process `@Cron`.
- Never let two runs overlap on the same item: the claim above handles it.

## Batch use-case

A scheduled use-case legitimately processes a batch inside one `execute()`. It is still one use-case per action — the action is "reconcile the pending batch", not "reconcile one row". Do not split it into one use-case per item.

```ts
export type ReconcileRechargesInput = {
  batchSize: number;
};

@Injectable()
export class ReconcileRechargesUseCase {
  constructor(
    private readonly repository: RechargeRepository,
    private readonly provider: TopupGateway,
    private readonly retryPolicy: RechargeRetryPolicy,
    private readonly logger: Logger,
  ) {}

  async execute(input: ReconcileRechargesInput): Promise<void> {
    const pending = await this.repository.claimPending(input.batchSize);

    this.logger.log({ message: 'reconcile-recharges.started', count: pending.length }, ReconcileRechargesUseCase.name);

    for (const recharge of pending) {
      await this.reconcileOne(recharge);
    }

    this.logger.log({ message: 'reconcile-recharges.finished', count: pending.length }, ReconcileRechargesUseCase.name);
  }

  private async reconcileOne(recharge: Recharge): Promise<void> {
    const status = await this.provider.consultStatus(recharge.externalId);

    if (status.isErr) {
      const decision = this.retryPolicy.decide(recharge);
      recharge.registerFailedAttempt(decision);
      this.logger.log({ message: 'reconcile-recharges.attempt-failed', rechargeId: recharge.id, reason: status.error.type, decision }, ReconcileRechargesUseCase.name);
      await this.repository.save(recharge);
      return;
    }

    const applied = recharge.applyProviderStatus(status.value);

    if (applied.isErr) {
      this.logger.log({ message: 'reconcile-recharges.status-rejected', rechargeId: recharge.id, reason: applied.error.type }, ReconcileRechargesUseCase.name);
      return;
    }

    await this.repository.save(recharge);
  }
}
```

- The loop body is a private method so `execute` stays at one indentation level.
- One item's business failure is logged and skipped; it never aborts the batch.
- The retry-count threshold decision (`RechargeRetryPolicy`) is a domain service — the use-case orchestrates, the domain decides.
- An infrastructure exception on one item propagates and fails the run; the scheduler logs it once, and the next tick picks the remaining items up. If one bad item must not block the others, catch per item in the loop, log at `error` with the item id (this *is* the edge for that item), and continue — but never convert the exception into a business outcome.

## Queue consumer

Same shape: parse the envelope, dedupe, map to `Input`, delegate. Details on topology, routing by event type, DLQ and idempotency are in `messaging-pubsub.md` and `idempotency.md`.

## Where Result stops

At an HTTP edge, `Result` maps to a status code. At a worker edge nobody is waiting for a response:

- The **use-case** logs the outcome — start, each business branch, end — so the worker does not inspect the `Result` beyond what it needs.
- A consumer that receives `Err` treats it as a final answer: log, mark processed, acknowledge.
- A thrown infrastructure error propagates out of the worker uncaught: the scheduler marks the run failed; SQS redelivers the message.

## Rules

- Workers are thin: schedule or message in, `Input` out, call the use-case.
- Business workers in `contexts/<ctx>/workers/`; operational jobs in `infrastructure/`.
- `<action>.cron.ts` / `<event>.consumer.ts`.
- Safe under multiple replicas: claim with `SKIP LOCKED` or conditional transitions.
- Batch loops delegate the body to a private method; per-item business failures never abort the batch.
