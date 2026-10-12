# CQS: Commands vs Queries

Contents: [The split](#the-split) · [Layout](#layout) · [Command side](#command-side) · [Query side](#query-side) · [Why the asymmetry](#why-the-asymmetry) · [Rules](#rules)

## The split

Separate by **intent**:

- **Command** (write, changes state) → one dedicated **use-case per action**, through the domain, persisted via the **repository** port. Returns `Result<void, BusinessError>` by default.
- **Query** (read, no effect) → one **query service per context**: one class, N methods, N queries. Injects TypeORM's `DataSource` directly and returns read DTOs.

This is CQS, **not** CQRS: same database, no separate read store, no event-sourced projections. Escalate to separate read models only under real pressure — and when another context needs your data, see `platform/read-models.md`.

## Layout

```
contexts/payments/
├─ payments.controller.ts
├─ use-cases/                      # COMMANDS — one file per action
│  ├─ create-payment.use-case.ts
│  ├─ refund-payment.use-case.ts
│  └─ cancel-payment.use-case.ts
├─ services/
│  └─ payments.query-service.ts    # QUERIES — N methods = N queries
├─ domain/
│  └─ payment.repository.ts        # port, command side only
└─ payments.module.ts
```

## Command side

Goes through the domain and persists the aggregate. See `use-cases.md` for the full shape.

```ts
const payment = await this.repository.findById(input.paymentId);

if (!payment) {
  return Result.Err({ type: 'PAYMENT_NOT_FOUND', message: 'pagamento inexistente' });
}

const refunded = payment.refund();

if (refunded.isErr) {
  return refunded;
}

await this.repository.save(payment);

return Result.Ok();
```

The repository port exposes aggregate-level operations only: `save`, `findById`, and genuinely domain-level questions a command needs (`totalSettledToday`). It never accumulates finders for screens.

## Query side

One service per context, injecting the `DataSource`, returning exactly the shape the screen needs:

```ts
export type PaymentListItem = {
  id: string;
  amountCents: number;
  status: string;
  createdAt: string;
};

export type PaymentListResponse = {
  data: PaymentListItem[];
  meta: { nextCursor: string | null; limit: number };
};

const MAX_LIMIT = 100;

@Injectable()
export class PaymentsQueryService {
  constructor(private readonly dataSource: DataSource) {}

  async listByAccount(accountId: string, cursor: string | null, limit: number): Promise<PaymentListResponse> {
    const pageSize = Math.min(limit, MAX_LIMIT);
    const after = decodeCursor(cursor);

    const rows = await this.dataSource.query<PaymentListItem[]>(
      `SELECT id, amount_cents AS "amountCents", status, created_at AS "createdAt"
         FROM payments
        WHERE account_id = $1
          AND (created_at, id) < ($2, $3)
        ORDER BY created_at DESC, id DESC
        LIMIT $4`,
      [accountId, after.createdAt, after.id, pageSize + 1],
    );

    const hasNext = rows.length > pageSize;
    const data = rows.slice(0, pageSize);

    if (!hasNext) {
      return { data, meta: { nextCursor: null, limit: pageSize } };
    }

    return { data, meta: { nextCursor: encodeCursor(data[data.length - 1]), limit: pageSize } };
  }

  async findDetail(paymentId: string, accountId: string): Promise<PaymentDetail | null> {
    const rows = await this.dataSource.query<PaymentDetail[]>(
      `SELECT id, amount_cents AS "amountCents", status, pix_key AS "pixKey", created_at AS "createdAt"
         FROM payments
        WHERE id = $1 AND account_id = $2`,
      [paymentId, accountId],
    );

    if (rows.length === 0) {
      return null;
    }

    return rows[0];
  }
}
```

- Each method is one query, returning a read DTO — not an entity, not a mapped domain object.
- Always parameterized (`$1`), never string-interpolated. Sort fields from the client go through an allowlist.
- Every collection has a cap on page size. Cursor pagination for app-facing lists, offset for backoffice — see `rest-api.md`.
- Filter by owner (`account_id`) in the query itself, so another customer's resource is simply "not found".
- Plain SQL is fine and often clearer than a query builder. TypeORM's `createQueryBuilder` is fine too when the query is dynamic; pick one per service.
- The controller decides what "not found" means for HTTP (`null` → `404`); the query service does not return `Result`, because a read has no business rule to fail.

## Why the asymmetry

The **repository is a persistence pattern for the domain**: it exists to rebuild and persist **aggregates** with their invariants intact. That only matters when writing.

On reads there is no invariant to protect. Forcing reads through the repository produces:
- bespoke `findByCustomerAndStatusOrderedByDate` methods multiplying on the port,
- pointless row → entity → DTO mapping, where the entity is a costume worn for one line,
- N+1 loads because the aggregate hydrates relations the screen never asked for.

The query service reads the tables and hands back the DTO. Reads are where the aggregate boundary is a liability, so do not pay for it.

## Rules

- Writes: one use-case per action, through the repository port.
- Reads: one query service per context in `services/`, injecting `DataSource`.
- No screen finders on the repository port.
- Parameterized SQL, capped page size, owner filter in the `WHERE`.
