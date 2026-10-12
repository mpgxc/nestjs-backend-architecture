# Pub/Sub Messaging & Queues

Contents: [Topology](#topology) · [Why a queue per consumer](#why-a-queue-per-consumer) · [Integration events](#integration-events) · [Consumers](#consumers) · [Transient vs permanent failures](#transient-vs-permanent-failures) · [DLQ](#dlq) · [Ordering](#ordering)

## Topology

Fan-out with SNS → SQS. The producer publishes once to a topic; each interested consumer owns its own queue subscribed to it:

```
outbox → relay → SNS topic ──┬──▶ SQS ledger-payment-created   → PaymentCreatedConsumer (ledger)
                             ├──▶ SQS notify-payment-created   → PaymentCreatedConsumer (notifications)
                             └──▶ SQS <new consumer gets its own queue>
```

The producer knows nothing about its consumers. Adding one requires no producer change — that is the point of pub/sub. SNS **distributes** (one to many); SQS **isolates** (each consumer its own pace, retries and DLQ).

## Why a queue per consumer

Never one shared queue, because each consumer needs:
- **independent retry and DLQ policy** — a poison message for one consumer must not block another;
- **independent scaling** — the ledger may need 10 workers while notifications need 1;
- **independent failure** — one slow consumer never backs up the others.

Name queues `<consumer-service>-<event>` so ownership is obvious.

## Integration events

The payload is a **stable, versioned contract**, wrapped in the envelope described in `outbox.md`:

- Named in the **past tense** — facts, not commands: `PaymentCreated`, `AccountStatusChanged`.
- Specific over generic: `AccountStatusChanged` and `AccountLimitChanged` let consumers react to what matters; a single `AccountUpdated` for everything forces every consumer to diff.
- **Self-contained**: carry the new state the consumer needs. A thin `{ accountId }` event forces a call back to the producer and reintroduces the synchronous coupling events were meant to remove.
- Primitives only. No domain objects, no ORM entities.
- Versioned. Adding an optional field is safe; removing or renaming one breaks consumers you cannot see.

Domain events are different: internal, recorded by the aggregate, drained to the outbox. See `domain.md` and `bounded-contexts.md`.

## Consumers

A consumer is a driving adapter in `contexts/<ctx>/workers/`: it parses the envelope, dedupes, maps to the use-case `Input` and delegates. It holds no business logic.

```ts
@Injectable()
export class AccountEventsConsumer {
  constructor(
    private readonly applyAccountChange: ApplyAccountChangeUseCase,
    private readonly closeAccountShadow: CloseAccountShadowUseCase,
    private readonly logger: Logger,
  ) {}

  @SqsMessageHandler('cards-account-events', false)
  async handle(message: Message): Promise<void> {
    const event = parseEnvelope<AccountEventData>(message);

    if (event.eventType === 'AccountClosed') {
      await this.closeAccountShadow.execute({ accountId: event.aggregateId, occurredAt: event.occurredAt });
      return;
    }

    if (event.eventType === 'AccountUpdated' || event.eventType === 'AccountStatusChanged') {
      await this.applyAccountChange.execute({ accountId: event.aggregateId, occurredAt: event.occurredAt, data: event.data });
      return;
    }

    this.logger.debug({ message: 'account-events.ignored', eventId: event.eventId, eventType: event.eventType }, AccountEventsConsumer.name);
  }

  @SqsConsumerEventHandler('cards-account-events', 'processing_error')
  onProcessingError(error: Error, message: Message): void {
    this.logger.error({ message: 'account-events.processing-error', messageId: message.MessageId, error }, AccountEventsConsumer.name);
  }
}
```

- Library: `@ssut/nestjs-sqs`. `@SqsMessageHandler` receives the **message** (route and delegate here); `@SqsConsumerEventHandler` receives **consumer lifecycle events** (errors, timeouts) — use it for logs and metrics only.
- When one queue carries several event types, route on `eventType` with one guard per type and delegate each to its own use-case. Unknown types are logged and acknowledged — tolerant reader.
- `parseEnvelope` lives in `infrastructure/messaging/` and also restores the `correlationId` into the logging context (`logging.md`), so the consumer's logs join the producer's trace.
- Dedupe by `eventId` before delegating — see `idempotency.md`.

## Transient vs permanent failures

- **Transient** (timeout, 503, deadlock, circuit open) → let it throw. The message is not deleted and SQS redelivers after the visibility timeout.
- **Business rejection** (the use-case returned `Err`) → final answer. Log with `reason`, mark processed, return normally so the message is deleted.
- **Permanent technical** (malformed payload, unknown version) → do not retry forever. Log with the `eventId` and let it reach the DLQ after `maxReceiveCount`, or move it there explicitly.

## DLQ

Configure `maxReceiveCount` (typically 3–5) with backoff so a poison message lands in a DLQ instead of blocking the queue forever.

A DLQ with no alarm and no redrive plan is where messages go to die. Alarm on DLQ depth > 0, and know how you will redrive after the fix. Because consumers are idempotent, redrive is safe.

## Ordering

The standard broker does not guarantee order. When a consumer needs it (state replicas, status transitions), use an SNS FIFO topic + SQS FIFO queues with `MessageGroupId = aggregateId` and `MessageDeduplicationId = eventId`: order within each aggregate, parallelism across aggregates. Otherwise make handlers order-independent with a timestamp guard — prefer that when the throughput cost of FIFO matters.
