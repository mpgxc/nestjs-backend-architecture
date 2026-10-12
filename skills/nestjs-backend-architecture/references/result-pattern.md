# Result Pattern

Contents: [The type](#the-type) · [Why not exceptions](#why-not-exceptions) · [Narrowing](#narrowing) · [Threading Result](#threading-result) · [BusinessError](#businesserror) · [Where Result stops](#where-result-stops) · [Rules](#rules)

## The type

A discriminated union modeled on Rust's `Result`. This is the canonical implementation, in `commons/result.ts` — use it as-is:

```ts
export type Result<T, E> = Ok<T> | Err<E>;

type Ok<T> = {
  kind: 'ok';
  isOk: true;
  isErr: false;
  value: T;
};

type Err<E> = {
  kind: 'err';
  isOk: false;
  isErr: true;
  error: E;
};

const Ok = <T = void>(value?: T): Ok<T> => ({
  kind: 'ok',
  isOk: true,
  isErr: false,
  value: value as T,
});

const Err = <E>(error: E): Err<E> => ({
  kind: 'err',
  isOk: false,
  isErr: true,
  error,
});

export const Result = {
  Ok,
  Err,
};

export const isResult = (value: unknown): value is Result<unknown, unknown> => {
  if (typeof value !== 'object' || value === null) {
    return false;
  }

  const kind = (value as { kind?: unknown }).kind;

  return kind === 'ok' || kind === 'err';
};
```

`isResult` is used by the HTTP `ResultInterceptor` to tell a returned `Result` apart from a plain DTO (`controllers.md`). Domain and use-case code never need it — they know their types.

Construct through the namespace: `Result.Ok(payment)`, `Result.Ok()` for a step that produces nothing, `Result.Err({ type: 'INVALID_AMOUNT', message: '...' })`.

The shape keeps four things on purpose:
- `kind` — the discriminant; reads well in a `switch` and in logs.
- `isOk` / `isErr` — literal-typed guards. Either one narrows the union.
- `value` / `error` — present only on the matching variant, so the compiler refuses access on the wrong branch.

It is a plain object, not a class: it serializes to JSON as-is and crosses a queue or process boundary without losing anything.

`Ok<T>` and `Err<E>` are not exported. Signatures declare `Result<T, E>`; the variants are an implementation detail.

## Why not exceptions

A business outcome ("insufficient limit") is an expected result, not an exceptional condition. Modeling it as a `throw`:

- **Hides it from the signature.** `Promise<Payment>` lies — it can also blow up and the compiler never says so. `Promise<Result<Payment, BusinessError>>` states the whole truth.
- **Gets caught by accident**, three layers up, by a `catch` meant for a network error.
- **Conflates two different things.** "The business said no" and "the database is down" need different handling, logging and retry semantics.

Reserve `throw` for infrastructure failures (timeout, connection lost, malformed vendor response, circuit open). Those propagate to the retry, queue or saga machinery — see `integration.md` and `resilience.md`.

## Narrowing

Guard on `isErr` and return early. Below the guard the value is narrowed to `Ok<T>`:

```ts
const payment = Payment.create({ accountId, amount, pixKey });

if (payment.isErr) {
  return payment;
}

payment.value;
```

`kind` narrows the same way and reads better in a `switch` when both branches carry work:

```ts
switch (result.kind) {
  case 'ok': {
    return this.notifySuccess(result.value);
  }
  case 'err': {
    return this.notifyRejection(result.error);
  }
}
```

Prefer the guard form in use-cases and domain code. Use `isOk` only when the success branch is the short one.

## Threading Result

Each failing step short-circuits; the happy path continues below:

```ts
async execute(input: CreatePaymentInput): Promise<Result<CreatePaymentOutput, BusinessError>> {
  const amount = Money.fromCents(input.amountCents);

  if (amount.isErr) {
    return amount;
  }

  const pixKey = PixKey.create(input.pixKey);

  if (pixKey.isErr) {
    return pixKey;
  }

  const payment = Payment.create({ accountId: input.accountId, amount: amount.value, pixKey: pixKey.value });

  if (payment.isErr) {
    return payment;
  }

  await this.repository.save(payment.value);

  return Result.Ok({ paymentId: payment.value.id });
}
```

Returning the narrowed `Err` directly (`return amount`) is type-safe: `Err<BusinessError>` is assignable to `Result<T, BusinessError>` for any `T`. No re-wrapping.

A step that produces nothing is `Result<void, BusinessError>` and succeeds with `Result.Ok()`.

## BusinessError

The error channel carries a semantic union, not a string. It lives in `commons/business-error.ts`:

```ts
export type BusinessErrorType =
  | 'INVALID_AMOUNT'
  | 'INVALID_PIX_KEY'
  | 'PAYMENT_NOT_FOUND'
  | 'INVALID_TRANSITION'
  | 'INSUFFICIENT_LIMIT';

export type BusinessError = {
  type: BusinessErrorType;
  message: string;
};
```

- `type` is for machines — the HTTP edge maps it to a status, tests assert on it, logs carry it as `reason`.
- `message` is for humans. Nobody parses it to decide what happened.
- The object is written **inline at the return site**: `return Result.Err({ type: 'PAYMENT_NOT_FOUND', message: 'pagamento inexistente' })`. No error classes, no factory objects.
- Adding a variant to the union forces the status map (`controllers.md`) to cover it — the compiler keeps the HTTP mapping exhaustive.
- Name types `SUBJECT_PROBLEM` in upper snake case (`ACCOUNT_BLOCKED`, `BOLETO_ALREADY_PAID`). Keep them domain words, never HTTP words (`NOT_FOUND` alone is too vague; `PAYMENT_NOT_FOUND` says what).

## Where Result stops

At the input adapter:
- **HTTP controller** — returns the use-case `Result` as-is. The global `ResultInterceptor` unwraps `Ok` into the response body and turns `Err` into a `BusinessErrorException`, which the global `BusinessErrorFilter` maps to status + problem+json through the single status map. See `controllers.md`.
- **Cron / queue worker** — no caller waits for a response. The use-case already logged the business outcome; the worker returns. Thrown infrastructure errors propagate so the scheduler or SQS register the failure. See `workers.md`.

A `Result` never reaches the client as `{ kind, isOk, value }` — the interceptor always unwraps it. Do not read `.error.type` anywhere except the status map, logs and tests.

## Rules

- Construct via `Result.Ok(...)` / `Result.Ok()` / `Result.Err(...)`. Never hand-build the object literal.
- Declare `Result<T, E>` in signatures; never `Ok<T>` or `Err<E>`.
- Guard with `if (x.isErr) { return x; }` — braces always, see `code-style.md`.
- Return the narrowed `Err` as-is when short-circuiting.
- `Result` is for business outcomes. Infrastructure failures still `throw`.
- Every value-object factory, entity factory or transition, domain service and use-case that can fail a business rule returns `Result`.
