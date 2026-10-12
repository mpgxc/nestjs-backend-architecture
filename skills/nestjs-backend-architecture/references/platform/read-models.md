# Replicated Read Models (Event-Carried State Transfer)

Contents: [The problem](#the-problem) · [Read strategies compared](#read-strategies-compared) · [The solution: a local shadow](#the-solution-a-local-shadow) · [Owner side](#owner-side) · [Consumer side](#consumer-side) · [Shadow miss: guarded fallback](#shadow-miss-guarded-fallback) · [Reference reads vs decision reads](#reference-reads-vs-decision-reads) · [Making it reliable](#making-it-reliable) · [When to use it](#when-to-use-it)

## The problem

`ms.accounts` is the source of truth for accounts. If `ms.cards`, `ms.billet`, `ms.recharge` and `ms.pix` call it synchronously on every request, its availability caps theirs: accounts goes down, everything goes down. Latency adds up, accounts becomes everyone's bottleneck, and no service can deploy or degrade independently.

The wrong question is "how does cards fetch the account from accounts?". The right one is "how does cards have the account data it needs without depending on accounts being up right now?".

## Read strategies compared

| Strategy | Couples availability? | Consistency | Cost |
|----------|----------------------|-------------|------|
| Synchronous REST on every read | Yes, fully | Strong | Cheap to build, fragile to run |
| Cache on the reader side | Partially — expired or cold entries hit the owner | Bounded by TTL | Low |
| BFF / aggregator | Yes — it only moves the dependency | Strong | Medium |
| **Replication by events (ECST)** | **No** — the reader queries its own database | Eventual | Higher to build, robust to run |

The deciding question: *when accounts is down, can the reader still answer?* Only replication says yes, always.

## The solution: a local shadow

The owner keeps writing the truth and **announces every change** as a self-contained integration event. Each interested service keeps a **shadow** — a local, read-only replica of the slice of the account it needs — and reads from its own table.

```
ms.accounts ──outbox──▶ SNS FIFO account-events.fifo ──┬──▶ SQS FIFO cards-account-events  ─▶ cards: account_shadows
                                                       └──▶ SQS FIFO billet-account-events ─▶ billet: account_shadows
```

What is distributed is not the truth — it is the **ability to read** it. Only the owner writes; replicas follow it with a small delay and never lead.

## Owner side

- Publish through the **transactional outbox** (`../outbox.md`). A dual write here means a lost event, and a lost event means a replica wrong forever.
- Specific, past-tense events: `AccountCreated`, `AccountUpdated`, `AccountStatusChanged`, `AccountLimitChanged`, `AccountClosed`. Consumers can react to a block without reacting to an e-mail change.
- **Self-contained payload**: carry the new state (`previousStatus`, `currentStatus`, `reason`). An event with only `{ accountId }` forces a callback and recreates the coupling.
- FIFO with `MessageGroupId = accountId`: order per account, parallelism across accounts.
- Versioned envelope from day one (`eventVersion`).

## Consumer side

The shadow is maintained by a consumer in `workers/` that calls a use-case; the shadow repository and its adapter are ordinary context code.

```ts
export type ApplyAccountChangeInput = {
  accountId: string;
  occurredAt: string;
  data: AccountEventData;
};

@Injectable()
export class ApplyAccountChangeUseCase {
  constructor(
    private readonly shadows: AccountShadowRepository,
    private readonly logger: Logger,
  ) {}

  async execute(input: ApplyAccountChangeInput): Promise<void> {
    const shadow = await this.shadows.findById(input.accountId);

    if (shadow && shadow.sourceUpdatedAt >= input.occurredAt) {
      this.logger.debug({ message: 'apply-account-change.stale-skipped', accountId: input.accountId }, ApplyAccountChangeUseCase.name);
      return;
    }

    await this.shadows.upsert(AccountShadow.fromEvent(input.accountId, input.data, input.occurredAt));
  }
}
```

- The **timestamp guard** makes the projection idempotent and order-tolerant: duplicates and late events become no-ops.
- Only the consumer writes the shadow. No other code in the service updates it.
- The shadow is disposable and rebuildable by replaying events.
- Each service has its own database; nobody reads another service's tables.

Ports that read the shadow (`AccountLimits`, `AccountStatusReader`) live in the consumer context's `domain/`; their adapters query `account_shadows`.

## Shadow miss: guarded fallback

A freshly created account whose event has not arrived yet is a **shadow miss**. Cache-aside: read the shadow; only on a miss, call the owner via REST — as a rare exception, never as the rule.

That fallback reintroduces coupling, so it must be **armored**: short timeout and its own circuit breaker (`../resilience.md`), plus service authentication (`service-auth.md`). Without the armor, a slow accounts service drags consumers down exactly when they fall back most (right after a deploy, with cold shadows).

## Reference reads vs decision reads

Classify every read:

- **Reference** — status, registration data, informative limits, "does the account exist?". A few seconds of delay is harmless → read the shadow.
- **Decision on money** — the balance used to authorize a debit, a limit that must lock an operation at that exact moment. A stale value moves real money wrongly → read the source synchronously, accepting the coupling on that path because correctness demands it.

Most reads are reference reads. The hard and important part is knowing which are not.

## Making it reliable

Three conditions, none optional:

1. **Publishing cannot fail silently** — transactional outbox on the owner.
2. **The projection is idempotent** — timestamp (or version) guard on the consumer.
3. **Divergence is detectable and correctable** — store the source `updatedAt` on the shadow, treat very old shadows of active accounts as suspect, and run a periodic reconciliation that re-syncs shadows from the owner.

## When to use it

Worth it when several services read the same data from another domain at high frequency, their availability must not depend on the owner, the data changes much less than it is read, and eventual consistency is acceptable for those reads.

Avoid it when the read needs strong consistency, only one service reads the data rarely (a guarded REST call is simpler), or the team does not yet have outbox and circuit breaker in place — without them the replication is fragile.

Writes are a different problem: they belong to the owner. When the owner is down, the options are a queued command with guaranteed delivery, graceful degradation, or accepting that the write path is synchronous.
