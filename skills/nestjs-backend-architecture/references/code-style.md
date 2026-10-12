# Code Style

Contents: [Non-negotiables](#non-negotiables) · [Braces on every block](#braces-on-every-block) · [Fail fast with guard clauses](#fail-fast-with-guard-clauses) · [No else](#no-else) · [No nested ifs](#no-nested-ifs) · [Object Calisthenics](#object-calisthenics) · [Comments](#comments) · [Logging inline](#logging-inline) · [try/catch only to react](#trycatch-only-to-react) · [No clever code](#no-clever-code) · [Review checklist](#review-checklist)

Every code example in this skill follows these rules. Generated code must follow them too, unless the existing codebase clearly uses a different convention. Rules 1–4 are enforced by ESLint (`curly`, `no-else-return`, `no-lonely-if`, `max-depth`) — see `bootstrap.md`.

## Non-negotiables

1. Every `if`, `for`, `while` and `else` has opening and closing braces, even with one statement.
2. Validate first and leave early (fail fast). The happy path runs at the bottom, unindented.
3. No `else` after a `return`, `throw`, `continue` or `break`.
4. No nested `if`. One level of indentation per method.
5. No inline comments. Comments live only in docstrings (TSDoc).
6. Logs are written inline, structured, at the call site. No helpers that hide them.
7. `try/catch` only to react to a failure (compensate, record state), never just to log and rethrow.

## Braces on every block

A one-line `if` hides where the block ends, and the next person who adds a second line introduces a bug. Always open and close the block.

```ts
// ✗
if (payment.isErr) return payment;
if (!account) return Result.Err({ type: 'ACCOUNT_NOT_FOUND', message: 'conta inexistente' });
for (const item of items) total += item.cents;

// ✓
if (payment.isErr) {
  return payment;
}

if (!account) {
  return Result.Err({ type: 'ACCOUNT_NOT_FOUND', message: 'conta inexistente' });
}

for (const item of items) {
  total += item.cents;
}
```

The `// ✗` / `// ✓` markers in this skill are labels for the reader of the reference, not comments to reproduce in real code.

## Fail fast with guard clauses

Check every reason to stop at the top of the method, each in its own guard, and return immediately. What remains below the guards is the happy path, flat and readable top to bottom.

```ts
async execute(input: RefundPaymentInput): Promise<Result<void, BusinessError>> {
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
}
```

Rules:
- One condition per guard. Do not merge unrelated checks with `||` just to save lines — each failure deserves its own error type and log.
- The guard returns the failing `Result` as-is (`return refunded`). No re-wrapping.
- Order guards from cheapest to most expensive: input shape, value objects, then I/O-backed checks.
- Leave a blank line after each guard block so the steps of the flow are visible.

## No else

After a block that leaves the method, `else` only adds indentation. Invert the condition and return early instead.

```ts
// ✗
if (account.isBlocked()) {
  return Result.Err({ type: 'ACCOUNT_BLOCKED', message: 'conta bloqueada' });
} else {
  await this.repository.save(account);
  return Result.Ok();
}

// ✓
if (account.isBlocked()) {
  return Result.Err({ type: 'ACCOUNT_BLOCKED', message: 'conta bloqueada' });
}

await this.repository.save(account);

return Result.Ok();
```

When the branch chooses a value rather than a flow, use a lookup map instead of an `if/else` chain or a `switch` that keeps growing:

```ts
const FEE_PERCENT_BY_METHOD: Record<PaymentMethod, number> = {
  PIX: 0,
  BOLETO: 0.015,
  CARD: 0.0299,
};

const feePercent = FEE_PERCENT_BY_METHOD[payment.method];
```

A ternary is acceptable only to pick between two simple values with no side effect. Never nest ternaries.

## No nested ifs

Nesting forces the reader to keep every outer condition in their head. Flatten it with guard clauses, or extract the inner block into a method with a name.

```ts
// ✗
if (shadow) {
  if (shadow.updatedAt < event.occurredAt) {
    if (event.data.status !== shadow.status) {
      await this.shadows.upsert(toShadow(event));
    }
  }
}

// ✓
if (shadow && shadow.updatedAt >= event.occurredAt) {
  return;
}

await this.shadows.upsert(toShadow(event));
```

A loop whose body has a decision is a sign that the body is its own operation. Extract it:

```ts
async execute(): Promise<void> {
  const pending = await this.repository.findPending(BATCH_SIZE);

  for (const recharge of pending) {
    await this.reconcile(recharge);
  }
}

private async reconcile(recharge: Recharge): Promise<void> {
  const status = await this.provider.consultStatus(recharge.externalId);

  if (status.isErr) {
    this.logger.warn({ message: 'reconcile.consult-failed', rechargeId: recharge.id, reason: status.error.type }, ReconcileRechargesUseCase.name);
    return;
  }

  recharge.applyProviderStatus(status.value);
  await this.repository.save(recharge);
}
```

## Object Calisthenics

The nine rules by Jeff Bay, adapted. Rules 1 and 2 are hard rules here (sections above). Rules 3 and 9 are strong defaults where a rule exists to protect. The rest are **review signals**: when code trips one, look for a missing concept or a class doing two jobs — but never add structure just to satisfy the table. A level-0 context (`SKILL.md`) has nothing to wrap and nothing to tell.

| # | Rule | How it shows up in this codebase |
|---|------|----------------------------------|
| 1 | One level of indentation per method | Guard clauses + extracted private methods. See [No nested ifs](#no-nested-ifs). |
| 2 | Don't use `else` | Return early. See [No else](#no-else). |
| 3 | Wrap primitives **that have rules** | `Money`, `Cpf`, `PixKey`, `Barcode` as value objects in `domain/`; money and documents never travel as raw primitives inside the domain. A `name`, `description` or `externalId` with no rule stays a `string` — a wrapper with nothing to validate is ceremony. |
| 4 | First-class collections | A list with its own rules becomes a class (`Installments`, `LedgerEntries`) that owns the invariant ("sum must equal total"). A plain list with no rules stays an array. |
| 5 | One dot per line | No `payment.account.owner.document.value`. Ask the nearest object for what you need. Fluent builders and query builders are the accepted exception. |
| 6 | Don't abbreviate | `rechargeAmount`, not `rAmt`; `repository`, not `repo` in new code. A clear name replaces the comment you were about to write. |
| 7 | Keep entities small | A class that does not fit on one screen is doing more than one thing. A use-case `execute` that grows past one screen is orchestrating two actions — split it. |
| 8 | Few instance variables | More than ~5 constructor dependencies in a use-case means it is doing too much. |
| 9 | Tell, don't ask | Call `payment.refund()`; do not read `payment.status` in the use-case and decide the transition there. The rule lives with the data. |

Tell, don't ask, in practice:

```ts
// ✗ the use-case asks for the state and decides the transition
if (recharge.status === 'PENDING' && recharge.amount.cents > 0) {
  recharge.status = 'CONFIRMED';
}

// ✓ the entity owns the transition and protects the rule
export class Recharge {
  confirm(): Result<void, BusinessError> {
    if (this.status !== 'PENDING') {
      return Result.Err({ type: 'INVALID_TRANSITION', message: 'recarga não está pendente' });
    }

    this.status = 'CONFIRMED';

    return Result.Ok();
  }
}
```

## Comments

- No inline comments. Not on call lines, not above blocks, not at the end of lines. If the code needs one, rename the variable, extract a method, or introduce a value object.
- Docstrings (TSDoc) are allowed on exported classes and public methods when they explain **why** or a non-obvious contract — never to restate the signature.
- Port docstrings are the right place to state the contract: what goes in the `Result`, what is thrown.

```ts
/**
 * Gateway de liquidação PIX.
 *
 * Desfechos de negócio (chave inválida, conta destino encerrada) voltam em `Result.Err`.
 * Indisponibilidade do provedor (timeout, 5xx, circuito aberto) é lançada como exceção
 * para que retry, fila ou saga tratem.
 */
export abstract class PixGateway {
  abstract dispatch(payment: Payment): Promise<Result<void, BusinessError>>;
}
```

## Logging inline

Logs are written where the event happens, as a structured object, through the injected `Logger`. No `logReceived()`, `logFailure()` or other helpers that hide what is logged.

```ts
this.logger.log(
  { message: 'create-payment.started', accountId: input.accountId, amountCents: input.amountCents },
  CreatePaymentUseCase.name,
);
```

- `message` is a stable, dot-separated event name (`<action>.<stage>`), not a sentence.
- Business fields go as keys (`paymentId`, `amountCents`), never interpolated into the message.
- A use-case logs business facts: `started` with identifiers, each `rejected` with its `reason`, `finished`. Infrastructure failures are logged once at the edge, not in the use-case.
- Request-level access logs (method, route, status, duration) come from `pino-http`, not from code.
- Levels: `log` for lifecycle and business outcomes support will ask about; `debug` for detail; `warn` for anomalies the system survived (retry, circuit opened); `error` only for failures nobody expected. A handled business rejection is never `error`.
- Never log CPF, full card number, tokens, secrets or authorization headers. See `logging.md`.

## try/catch only to react

An infrastructure failure is logged **once**, at the edge: `UnhandledExceptionFilter` for HTTP, the consumer's error handler or the scheduler for workers. That line carries the `correlationId`, which links it to the use-case's `started` log with the business identifiers. A `try/catch` that only logs and rethrows produces a second copy of the same error and doubles the size of every use-case.

So a `try/catch` appears when the code **does something** with the failure — compensates, records a failure state, releases a reservation — and then rethrows:

```ts
try {
  await this.boletoGateway.register(boleto);
} catch (error) {
  boleto.markRegistrationPending();
  await this.repository.save(boleto);
  this.logger.warn({ message: 'issue-boleto.registration-deferred', boletoId: boleto.id }, IssueBoletoUseCase.name);
  throw error;
}
```

- Keep the `try` tight around the one call whose failure you handle. Business guards stay outside it.
- Never swallow an infrastructure failure and never convert it into a business error — retry, queue or saga machinery needs to see it.
- Adapters are the exception to "do not catch": they catch to **translate** vendor errors (`integration.md`).

## No clever code

- Write explicit, boring TypeScript. No chained ternaries, no `&&` used as an `if`, no `reduce` where a `for` reads better, no metaprogramming to save lines.
- Build the error at the return site: `return Result.Err({ type: 'X', message: '...' })`. No error factory objects (`const PaymentErrors = { notFound: () => ... }`).
- No wrapper layer that only forwards calls.
- Named constants for magic numbers: `const DAILY_LIMIT_CENTS = 500_000`.

## Review checklist

- [ ] Every block has braces.
- [ ] Guards at the top, one condition each, blank line after each.
- [ ] No `else` after a leaving statement; no nested `if`.
- [ ] No inline comment; docstrings only where they explain a contract or a why.
- [ ] Money and documents are value objects; values without rules stay primitives.
- [ ] The use-case does not read entity state to decide a rule (tell, don't ask).
- [ ] Logs are inline, structured, at the right level, without PII; business facts only.
- [ ] No log-and-rethrow `try/catch`; a catch exists only to react, and rethrows.
- [ ] Nothing added that the context's level does not need.
