# Idempotency

Contents: [Why it is mandatory](#why-it-is-mandatory) · [Layer 1: dedupe at the edge](#layer-1-dedupe-at-the-edge) · [The claim-before-process trap](#the-claim-before-process-trap) · [Layer 2: idempotent operations](#layer-2-idempotent-operations) · [Projections: timestamp guard](#projections-timestamp-guard) · [API idempotency keys](#api-idempotency-keys) · [Rules](#rules)

## Why it is mandatory

At-least-once delivery means duplicates are a **certainty, not an edge case**: the relay may publish twice, SQS may deliver twice, a consumer may crash after the write but before the delete, a client may retry a timed-out request.

Processing the same message twice must have the same effect as processing it once. In fintech this is the difference between a payment and a double payment.

Defend in **two layers** — never trust just one.

## Layer 1: dedupe at the edge

Skip messages already processed successfully, and mark a message as processed **only after** the work succeeded:

```ts
@Injectable()
export class PaymentCreatedConsumer {
  constructor(
    private readonly processed: ProcessedMessages,
    private readonly postEntry: PostLedgerEntryUseCase,
    private readonly logger: Logger,
  ) {}

  @SqsMessageHandler('ledger-payment-created', false)
  async handle(message: Message): Promise<void> {
    const event = parseEnvelope<PaymentCreatedData>(message);

    if (await this.processed.contains(event.eventId)) {
      this.logger.debug({ message: 'payment-created.duplicate-skipped', eventId: event.eventId }, PaymentCreatedConsumer.name);
      return;
    }

    const result = await this.postEntry.execute({ paymentId: event.data.paymentId, amountCents: event.data.amountCents, sourceEventId: event.eventId });

    if (result.isErr) {
      this.logger.log({ message: 'payment-created.rejected', eventId: event.eventId, reason: result.error.type }, PaymentCreatedConsumer.name);
    }

    await this.processed.add(event.eventId);
  }
}
```

`ProcessedMessages` is a port backed by a `processed_messages` table with a unique `message_id` (or Redis with a TTL). A business rejection is a final answer, so the message is marked processed and acknowledged; a thrown infrastructure error skips `add` and SQS redelivers.

Two concurrent deliveries can both pass `contains` before either calls `add`. That window is real and is exactly why Layer 2 exists.

When the stack supports a transaction shared between the use-case and the consumer (for example `@nestjs-cls/transactional`), insert into `processed_messages` in the **same transaction** as the business write and treat the unique violation as "duplicate". That closes the window completely.

## The claim-before-process trap

```ts
// ✗ never do this
const firstTime = await redis.setnx(messageId, '1');

if (!firstTime) {
  return;
}

await this.postEntry.execute(input);
```

If `execute` throws after the claim, SQS redelivers — and the redelivery is discarded as a duplicate. The message is **lost**, silently. A claim must either be part of the same transaction as the work, or be written only after the work succeeds.

## Layer 2: idempotent operations

The dedupe store can expire, fail or race. Design the write so replaying it is harmless anyway:

- **UPSERT on a business key**, not a blind INSERT.
- **Conditional transitions**: `UPDATE ... SET status = 'SETTLED' WHERE id = $1 AND status = 'PENDING'` — a replay touches zero rows. Entity transitions that check their precondition (`payment.settle()` returning `INVALID_TRANSITION`) give the same protection at the domain level.
- **Never** `balance = balance + x` unguarded. Post a ledger entry keyed by the source event id and let a unique constraint reject the duplicate.
- Reuse a **natural idempotency key** where one exists (the aggregate id, the source event id), so a retry is provably the same logical operation.

Layer 1 is the optimization. Layer 2 is the correctness guarantee.

## Projections: timestamp guard

A consumer that maintains a local replica (`platform/read-models.md`) is idempotent and order-tolerant by comparing timestamps: apply the event only if it is newer than what is stored.

```ts
async execute(input: ApplyAccountChangeInput): Promise<void> {
  const shadow = await this.shadows.findById(input.accountId);

  if (shadow && shadow.sourceUpdatedAt >= input.occurredAt) {
    return;
  }

  await this.shadows.upsert(AccountShadowMapper.fromEvent(input));
}
```

Duplicates and late arrivals become no-ops; the replica never moves backwards.

## API idempotency keys

The same idea at the HTTP edge. Every endpoint that moves money accepts an `Idempotency-Key` header:

- Store the key with a hash of the request payload and the response.
- Same key + same payload → return the stored response, do not execute again.
- Same key + different payload → `409 Conflict`.
- Key in flight (first request still running) → `409` so the client backs off.

Send an idempotency key to vendors too when they support it, derived from our own id — see `integration.md`.

## Rules

- Every consumer is idempotent. No exceptions.
- Mark processed after success, or inside the same transaction — never claim before.
- Every saga step and every compensation is idempotent (`platform/saga.md`).
- Prefer natural keys over generated ones when they identify the same logical operation.
- Money-moving endpoints require `Idempotency-Key`.
