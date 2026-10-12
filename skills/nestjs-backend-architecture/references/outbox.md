# Transactional Outbox

Contents: [The dual-write problem](#the-dual-write-problem) · [One mechanism: the repository drains events](#one-mechanism-the-repository-drains-events) · [Table and states](#table-and-states) · [The envelope](#the-envelope) · [The relay](#the-relay) · [Gotchas](#gotchas)

## The dual-write problem

Saving state and publishing an event touch two systems. Do them in sequence and a crash in between leaves either a **lost event** (state saved, event never sent) or a **phantom event** (event sent, transaction rolled back). There is no atomic write across a database and a broker.

The outbox makes the event part of the same transaction as the state. Publishing becomes a separate, retryable step. Any service that publishes events consumed by others — and especially a source of truth replicated by others (`platform/read-models.md`) — needs it. A lost event there means a replica that is wrong forever.

## One mechanism: the repository drains events

There is exactly one way events reach the outbox in this codebase:

1. The aggregate **records** domain events (`record(...)`) as transitions happen. It never publishes.
2. The repository adapter, inside the transaction that saves the aggregate, **drains** `pullEvents()` and inserts each one into the `outbox` table.
3. The relay, a separate process, **publishes** pending rows.

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

Do **not** add an in-process event handler (`@EventsHandler`, `@OnEvent`) that writes to the outbox after the fact. That handler runs outside the save transaction and reintroduces the exact dual-write the outbox exists to remove.

## Table and states

```
outbox: id (uuid), event_type, event_version, aggregate_id, payload (jsonb),
        correlation_id, status, occurred_at, published_at
index on (status, occurred_at)
```

State machine: `PENDING → PUBLISHED`.

Never delete a row before the broker acknowledges it — mark it `PUBLISHED`. Prune old published rows with a separate cleanup job. `outbox.id` becomes the `eventId` of the envelope and the consumer's dedupe key.

## The envelope

The payload published is an integration event — a public, versioned contract — wrapped in an envelope that makes ordered, idempotent, traceable delivery possible:

```ts
export type IntegrationEvent<T> = {
  eventId: string;
  eventType: string;
  eventVersion: number;
  occurredAt: string;
  aggregateId: string;
  correlationId: string;
  data: T;
};
```

- `eventId` — unique and stable; the consumer's dedupe key and SQS FIFO `MessageDeduplicationId`.
- `eventType` + `eventVersion` — consumers route by type and tolerate evolution by version.
- `aggregateId` — becomes the FIFO `MessageGroupId`, giving per-aggregate ordering.
- `occurredAt` — what a timestamp guard compares to discard stale events.
- `correlationId` — carries the trace of the request that caused the fact (`logging.md`).
- `data` — primitives only, self-contained enough that the consumer does not have to call back.

## The relay

A poller in `infrastructure/outbox/` publishes pending rows. `FOR UPDATE SKIP LOCKED` lets several replicas run the relay concurrently without double-processing a row.

```ts
const BATCH_SIZE = 50;

@Injectable()
export class OutboxRelay {
  constructor(
    private readonly dataSource: DataSource,
    private readonly publisher: SnsPublisher,
    private readonly logger: Logger,
  ) {}

  @Interval(2000)
  async flush(): Promise<void> {
    await this.dataSource.transaction(async (manager) => {
      const rows = await manager.query<OutboxRow[]>(
        `SELECT * FROM outbox
          WHERE status = 'PENDING'
          ORDER BY occurred_at
          LIMIT $1
          FOR UPDATE SKIP LOCKED`,
        [BATCH_SIZE],
      );

      for (const row of rows) {
        await this.publishRow(manager, row);
      }
    });
  }

  private async publishRow(manager: EntityManager, row: OutboxRow): Promise<void> {
    try {
      await this.publisher.publish(OutboxMapper.toEnvelope(row), { groupId: row.aggregate_id, deduplicationId: row.id });
    } catch (error) {
      this.logger.warn({ message: 'outbox.publish-failed', eventId: row.id, eventType: row.event_type, error }, OutboxRelay.name);
      return;
    }

    await manager.update(OutboxModel, row.id, { status: 'PUBLISHED', publishedAt: new Date() });
  }
}
```

- A row that fails to publish stays `PENDING` and is retried on the next tick; the rest of the batch continues.
- The relay is **at-least-once**: a crash after publish but before commit sends the event again. That is expected. Do not chase exactly-once — make consumers idempotent (`idempotency.md`).
- Keep `BATCH_SIZE` small: the transaction holds row locks while it publishes.
- For lower latency, Postgres `LISTEN/NOTIFY` can wake the relay on insert, with the poller kept as a safety net.

## Gotchas

- **Publish integration events, not your entity.** The payload is a versioned contract other services depend on. Dumping the aggregate couples every consumer to your internal model.
- **Order is not guaranteed** across a standard broker. If a consumer needs ordering, use FIFO topic/queue with `MessageGroupId = aggregateId`, or make handlers order-independent with a timestamp guard.
- **Monitor outbox lag.** Rows sitting in `PENDING` mean the relay is down and events are silently late — alarm on the age of the oldest pending row.
- **Propagate the correlation id.** It is stored on the row and published in the envelope, so the consumer's logs join the original request.
