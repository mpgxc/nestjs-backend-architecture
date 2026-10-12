# Logging & Observability

Contents: [Structured events](#structured-events) · [Levels](#levels) · [The logger port](#the-logger-port) · [Correlation](#correlation) · [Logs and traces](#logs-and-traces) · [What never gets logged](#what-never-gets-logged) · [Aggregation and retention](#aggregation-and-retention) · [Rules](#rules)

## Structured events

A log line is an **event with fields**, not a sentence. JSON to stdout, consumed by the platform.

```json
{
  "timestamp": "2026-06-11T14:30:02.123Z",
  "level": "info",
  "message": "recharge.confirmed",
  "context": "ConfirmRechargeUseCase",
  "service": "payments-service",
  "version": "1.4.2",
  "env": "production",
  "correlationId": "7c3e…",
  "trace_id": "0af7651916cd43dd8448eb211c80319c",
  "rechargeId": "7f3a…",
  "amountCents": 1500,
  "durationMs": 847
}
```

- `message` is a stable event name: `<action>.<stage>` (`create-payment.started`, `create-payment.rejected`).
- Business data goes in typed keys, never interpolated into the message. That is what makes `amountCents > 0 AND durationMs > 1000` a query instead of a regex.
- Agree on field names across services (`accountId`, not `account_id` here and `acct` there); follow OpenTelemetry semantic conventions where they exist.

## Levels

| Level | Use for | Production |
|-------|---------|------------|
| `debug` | Diagnostic detail: payload shapes, ignored events, duplicates skipped | Off or sampled |
| `log` (info) | Lifecycle and business outcomes support will ask about: started, finished, rejected with `reason` | On — the bulk of logs |
| `warn` | Anomalies the system survived: retry fired, circuit opened, outbox publish failed and will retry | On |
| `error` | An operation failed and someone must look: persistence failed, message headed to DLQ, dependency down after retries | On |
| `fatal` | Unrecoverable; the last line before the process exits | On |

A handled business rejection ("insufficient balance", "invalid key") is never `error`. Logging expected outcomes as errors buries the real ones and pages someone at 3 a.m. for nothing.

Level comes from `LOG_LEVEL` per environment. A good logger checks the level before serializing, so a disabled `debug` costs almost nothing.

## The logger port

Do not invent a new logging abstraction. NestJS already has one: the `LoggerService` interface implemented by the adapter, and the `Logger` class code depends on.

```ts
@Injectable()
export class PinoLoggerAdapter implements LoggerService {
  constructor(
    private readonly pino: PinoLogger,
    private readonly cls: ClsService,
  ) {}

  log(message: unknown, context?: string): void {
    this.pino.info(this.enrich(message, context));
  }

  warn(message: unknown, context?: string): void {
    this.pino.warn(this.enrich(message, context));
  }

  error(message: unknown, context?: string): void {
    this.pino.error(this.enrich(message, context));
  }

  debug(message: unknown, context?: string): void {
    this.pino.debug(this.enrich(message, context));
  }

  verbose(message: unknown, context?: string): void {
    this.pino.trace(this.enrich(message, context));
  }

  fatal(message: unknown, context?: string): void {
    this.pino.fatal(this.enrich(message, context));
  }

  private enrich(message: unknown, context?: string): Record<string, unknown> {
    const fields = typeof message === 'object' && message !== null ? message : { message };

    return { ...fields, context, correlationId: this.cls.getId() };
  }
}
```

```ts
const app = await NestFactory.create(AppModule, { bufferLogs: true });
app.useLogger(app.get(PinoLoggerAdapter));
```

The global infrastructure module provides Nest's `Logger` class (`providers: [Logger], exports: [Logger]`, `@Global()`). Classes inject it and log inline, passing their own name as context:

```ts
constructor(private readonly logger: Logger) {}

this.logger.log({ message: 'confirm-recharge.finished', rechargeId: recharge.id }, ConfirmRechargeUseCase.name);
```

Swapping Pino for another backend is a new `implements LoggerService` class and one `useLogger` line. Tests pass `new Logger()` or a silent fake.

Never `console.log`: it writes synchronously to stdout and blocks the event loop under load. Enforce it with a lint rule.

## Correlation

One request produces logs in the controller, the use-case, the repository, the outbox relay and, later, the consumer in another service. A shared id turns those scattered events into one story.

- **Generate at the edge**: middleware reads `X-Correlation-Id` (or `X-Request-Id`) or generates a UUID, and echoes it in the response.
- **Carry it in-process** with `AsyncLocalStorage` (via `nestjs-cls`), not by passing it through every signature and not with request-scoped providers (one instance per request costs latency).
- **Propagate across services**: HTTP header on outbound calls; `correlationId` stored on the outbox row and published in the event envelope; restored into the CLS context by `parseEnvelope` in the consumer.

Without the last step, the worker starts a "new" story and the causal link is lost — the classic event-driven blind spot.

## Logs and traces

- **Correlation id** — business scope you control, can span several requests of one journey.
- **Trace id / span id** — OpenTelemetry's technical trace of one request, propagated with the W3C `traceparent` header.

Use both. Inject `trace_id` and `span_id` into every log emitted inside an active span so the observability backend jumps from a slow span to its logs and back. With OTel, keep using Pino; the OTel instrumentation bridges it.

Metrics say *that* something is wrong, traces say *where*, logs filtered by `trace_id` say *why*.

## What never gets logged

LGPD and PCI-DSS make this a compliance rule, not a preference:

- Full card number (PAN), CVV, CPF/CNPJ in clear, passwords, tokens, secrets, `Authorization` headers, cookies.
- Whole request or response bodies of payment endpoints.

Defense in layers: redact at the source (Pino `redact` with paths such as `req.headers.authorization`, `*.cpf`, `*.cardNumber`, `*.token`, censor `[REDACTED]`), redact again in the collector, and log only partial values when needed operationally (last 4 digits). Log `customerId`, never the document.

## Aggregation and retention

The service only writes JSON to stdout; collecting and routing is the platform's job (CloudWatch Logs on ECS/Lambda, or an OTel Collector / Fluent Bit forwarding elsewhere). Filter health-check noise in the collector and sample high-volume debug.

Retention is a cost and compliance decision: PCI DSS v4.0 requires audit trails for at least 12 months, with the last 3 months immediately available.

## Who logs what

One fact, one line — never the same failure logged at several layers:

| What | Logged by | Level |
|------|-----------|-------|
| Request in/out (method, route, status, duration) | `pino-http` middleware | `log` |
| Business facts (`started`, `rejected` + reason, `finished`) | The use-case, inline | `log` |
| Anomalies survived (retry, circuit opened, deferred registration) | The adapter, policy or use-case that reacted | `warn` |
| Unexpected failure on HTTP | `UnhandledExceptionFilter` | `error` |
| Unexpected failure in a consumer | The consumer's `processing_error` handler | `error` |
| Unexpected failure in a cron | The scheduler (or the per-item catch in a batch) | `error` |

Use-cases and adapters do not log-and-rethrow: the edge already logs the error with the `correlationId` that joins it to the use-case's `started` line.

## Rules

- Structured JSON, stable `message` event names, typed fields.
- Each failure is logged once, at the edge.
- Inject Nest's `Logger`; log inline with the class name as context. No logging helpers, no `console.log`.
- `correlationId` on every line, propagated through HTTP, outbox and queues.
- Business rejections at `log`, never `error`.
- Redact at the source; never log documents, card data, tokens or secrets.
