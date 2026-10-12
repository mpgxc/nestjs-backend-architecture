# Bootstrap: What Every Project Needs on Day 1

Contents: [main.ts](#maints) · [Environment validation](#environment-validation) · [Request validation as problem+json](#request-validation-as-problemjson) · [Catch-all filter](#catch-all-filter) · [Migrations](#migrations) · [Health checks](#health-checks) · [Graceful shutdown](#graceful-shutdown) · [Lint enforces the style](#lint-enforces-the-style) · [Checklist](#checklist)

Build these before the first business feature. They are cheap now and painful to retrofit — and none of them is over-engineering.

## main.ts

```ts
async function bootstrap(): Promise<void> {
  const app = await NestFactory.create(AppModule, { bufferLogs: true });

  app.useLogger(app.get(PinoLoggerAdapter));
  app.useGlobalPipes(app.get(RequestValidationPipe));
  app.enableShutdownHooks();

  await app.listen(app.get(ConfigService).getOrThrow<number>('PORT'));
}

void bootstrap();
```

Global interceptor and filters are registered as `APP_INTERCEPTOR` / `APP_FILTER` providers in `AppModule` (`controllers.md`), so they get dependency injection.

## Environment validation

Fail at boot, not on the first request that reads a missing variable. Validate the whole environment once with a schema:

```ts
const environmentSchema = z.object({
  NODE_ENV: z.enum(['development', 'test', 'production']),
  PORT: z.coerce.number().default(3000),
  LOG_LEVEL: z.enum(['debug', 'info', 'warn', 'error']).default('info'),
  DATABASE_URL: z.string().url(),
  ACME_BASE_URL: z.string().url(),
  ACME_TIMEOUT_MS: z.coerce.number().default(8000),
});

export type Environment = z.infer<typeof environmentSchema>;

@Module({
  imports: [
    ConfigModule.forRoot({
      isGlobal: true,
      validate: (raw) => environmentSchema.parse(raw),
    }),
  ],
})
export class InfrastructureModule {}
```

Secrets come from Secrets Manager / Parameter Store injected as environment variables by ECS or Lambda, never from committed `.env` files.

## Request validation as problem+json

Out of the box, `ValidationPipe` answers with Nest's own error format, so the API ends up with two error shapes. Configure it once so validation failures follow RFC 9457 like every other error:

```ts
@Injectable()
export class RequestValidationPipe extends ValidationPipe {
  constructor() {
    super({
      whitelist: true,
      transform: true,
      exceptionFactory: (errors) => new BadRequestException({
        code: 'VALIDATION_FAILED',
        title: 'Dados de entrada inválidos',
        errors: errors.flatMap((error) => Object.values(error.constraints ?? {}).map((message) => ({ field: error.property, message }))),
      }),
    });
  }
}
```

`whitelist` strips unknown fields (tolerant reader on public edges). The catch-all filter below turns this `BadRequestException` into a problem+json body.

## Catch-all filter

One filter for everything that is not a `BusinessErrorException`. It is the single place where unexpected failures are logged at `error`.

```ts
@Catch()
export class UnhandledExceptionFilter implements ExceptionFilter {
  constructor(
    private readonly cls: ClsService,
    private readonly logger: Logger,
  ) {}

  catch(exception: unknown, host: ArgumentsHost): void {
    const http = host.switchToHttp();
    const request = http.getRequest<Request>();
    const response = http.getResponse<Response>();
    const problem = this.toProblem(exception);

    if (problem.status >= 500) {
      this.logger.error({ message: 'http.unhandled-exception', method: request.method, route: request.route?.path, status: problem.status, error: exception }, UnhandledExceptionFilter.name);
    }

    if (problem.retryAfterSeconds) {
      response.setHeader('Retry-After', String(problem.retryAfterSeconds));
    }

    response
      .status(problem.status)
      .type('application/problem+json')
      .json({ ...problem.body, status: problem.status, correlationId: this.cls.getId(), timestamp: new Date().toISOString() });
  }

  private toProblem(exception: unknown): Problem {
    if (exception instanceof ProviderUnavailableError) {
      return { status: 503, retryAfterSeconds: 30, body: { code: 'PROVIDER_UNAVAILABLE', title: 'Serviço temporariamente indisponível' } };
    }

    if (exception instanceof HttpException) {
      return { status: exception.getStatus(), body: HttpProblemMapper.fromHttpException(exception) };
    }

    return { status: 500, body: { code: 'INTERNAL_ERROR', title: 'Erro interno' } };
  }
}
```

- A `4xx` from Nest itself (`ParseUUIDPipe`, unknown route, validation) is formatted but not logged as an error — it is the client's mistake.
- A `5xx` never exposes the exception message or stack in the body.
- Register it **before** `BusinessErrorFilter` in `AppModule` so the specific filter is tried first.

## Migrations

- TypeORM migrations in `infrastructure/database/migrations/`, generated from the models and **reviewed by a human** before commit.
- `synchronize: false` in every environment except throwaway local databases.
- Run as a separate deploy step (one-off ECS task or pipeline stage) before the new version takes traffic — never on application boot with several replicas racing.
- Backward compatible by default (expand → migrate → contract): add the column, deploy code that writes both, backfill, then drop the old column in a later release. A rollback must not need a reverse migration.

## Health checks

With `@nestjs/terminus`, in `infrastructure/health/`:

- `GET /health/live` — the process is up. No dependencies. This is what the ECS/ALB health check calls; failing it restarts the task.
- `GET /health/ready` — can serve traffic: database reachable. Do **not** check vendors here — an Acme outage must not take every task out of the load balancer.

Exclude health routes from access logs and from authentication.

## Graceful shutdown

ECS sends `SIGTERM` and waits `stopTimeout` (default 30 s) before `SIGKILL`.

- `app.enableShutdownHooks()` so `onModuleDestroy` / `beforeApplicationShutdown` run.
- Stop SQS consumers first (stop polling), let in-flight messages finish, then close the `DataSource`.
- Keep `stopTimeout` longer than the slowest in-flight request or message.
- Anything interrupted anyway is redelivered (SQS) or retried (client with `Idempotency-Key`) — which is why idempotency matters here too.

## Lint enforces the style

Make the house style a build failure instead of a review comment:

```js
rules: {
  curly: ['error', 'all'],
  'no-else-return': ['error', { allowElseIf: false }],
  'no-lonely-if': 'error',
  'no-nested-ternary': 'error',
  'max-depth': ['error', 2],
  'no-console': 'error',
  'max-params': ['error', 5],
  '@typescript-eslint/no-floating-promises': 'error',
  '@typescript-eslint/explicit-function-return-type': 'error',
}
```

`max-depth: 2` allows an `if` inside a loop body; deeper nesting has to be extracted (`code-style.md`). Use TypeScript `strict: true`.

## Checklist

- [ ] Environment validated at boot with a schema.
- [ ] Pino logger with redaction, correlation id via CLS, `pino-http` access log.
- [ ] `ResultInterceptor`, `BusinessErrorFilter`, `UnhandledExceptionFilter`, `RequestValidationPipe` registered globally.
- [ ] Migrations reviewed, `synchronize: false`, run as a deploy step.
- [ ] `/health/live` and `/health/ready`.
- [ ] Shutdown hooks enabled; consumers stop before the database closes.
- [ ] ESLint rules above and `strict: true`.
- [ ] One integration test running against a real Postgres (`testing.md`).
