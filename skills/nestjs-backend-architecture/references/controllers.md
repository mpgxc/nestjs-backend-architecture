# Controllers & DTOs

Contents: [One controller per context](#one-controller-per-context) · [Result to HTTP: interceptor + filter](#result-to-http-interceptor--filter) · [The status map](#the-status-map) · [Output is the response contract](#output-is-the-response-contract) · [DTO vs Input](#dto-vs-input) · [Rules](#rules)

## One controller per context

The controller belongs to the **context**, not to a use-case. It routes writes to use-cases and reads to the query service. It lives at the context root: `contexts/payments/payments.controller.ts`.

```ts
@Controller('v1/payments')
export class PaymentsController {
  constructor(
    private readonly createPayment: CreatePaymentUseCase,
    private readonly refundPayment: RefundPaymentUseCase,
    private readonly queries: PaymentsQueryService,
  ) {}

  @Post()
  @HttpCode(HttpStatus.CREATED)
  @ApiCreatedResponse({ type: CreatePaymentResponse })
  create(@Body() body: CreatePaymentRequest): Promise<Result<CreatePaymentOutput, BusinessError>> {
    return this.createPayment.execute({
      accountId: body.accountId,
      amountCents: body.amountCents,
      pixKey: body.pixKey,
    });
  }

  @Post(':id/refunds')
  @HttpCode(HttpStatus.NO_CONTENT)
  refund(@Param('id', ParseUUIDPipe) id: string): Promise<Result<void, BusinessError>> {
    return this.refundPayment.execute({ paymentId: id });
  }

  @Get()
  list(@Query() query: ListPaymentsQuery): Promise<PaymentListResponse> {
    return this.queries.listByAccount(query.accountId, query.cursor, query.limit);
  }
}
```

Why not one controller per use-case: the HTTP surface of a context is one cohesive thing (`/v1/payments`, `/v1/payments/:id/refunds`). Splitting it scatters routing and hides the API shape.

The controller does two things only: map DTO → `Input` and return what the use-case returns. No `if` on the `Result`, no business decision, no repository, no `try/catch` — the global interceptor and filters handle the rest.

URI, status, header, pagination and error-body conventions are in `rest-api.md`.

## Result to HTTP: interceptor + filter

A filter alone cannot see a `Result`: filters catch thrown exceptions, and the controller *returns* a value. Two global pieces in `commons/http/` do the translation, so no controller ever knows a business status code:

1. **`ResultInterceptor`** inspects what the handler returned. `Ok` → its `value` becomes the response body. `Err` → it throws `BusinessErrorException` carrying the `BusinessError`. Anything that is not a `Result` (a query-service DTO) passes untouched.
2. **`BusinessErrorFilter`** catches `BusinessErrorException`, looks the status up in the status map and writes an RFC 9457 `application/problem+json` body.

```ts
export class BusinessErrorException extends Error {
  constructor(readonly error: BusinessError) {
    super(error.message);
    this.name = BusinessErrorException.name;
  }
}
```

```ts
@Injectable()
export class ResultInterceptor implements NestInterceptor {
  intercept(_context: ExecutionContext, next: CallHandler): Observable<unknown> {
    return next.handle().pipe(map((data) => this.unwrap(data)));
  }

  private unwrap(data: unknown): unknown {
    if (!isResult(data)) {
      return data;
    }

    if (data.isErr) {
      throw new BusinessErrorException(data.error as BusinessError);
    }

    return data.value;
  }
}
```

```ts
@Catch(BusinessErrorException)
export class BusinessErrorFilter implements ExceptionFilter {
  constructor(
    private readonly cls: ClsService,
    private readonly logger: Logger,
  ) {}

  catch(exception: BusinessErrorException, host: ArgumentsHost): void {
    const response = host.switchToHttp().getResponse<Response>();
    const status = BUSINESS_ERROR_STATUS[exception.error.type];

    this.logger.debug({ message: 'http.business-error', code: exception.error.type, status }, BusinessErrorFilter.name);

    response
      .status(status)
      .type('application/problem+json')
      .json({
        type: `https://api.example.com/errors/${exception.error.type.toLowerCase().replaceAll('_', '-')}`,
        title: exception.error.message,
        status,
        code: exception.error.type,
        correlationId: this.cls.getId(),
        timestamp: new Date().toISOString(),
      });
  }
}
```

Registered once in `AppModule` as providers, so both get dependency injection (prefer this over `app.useGlobalFilters(new ...)`):

```ts
@Module({
  providers: [
    { provide: APP_INTERCEPTOR, useClass: ResultInterceptor },
    { provide: APP_FILTER, useClass: UnhandledExceptionFilter },
    { provide: APP_FILTER, useClass: BusinessErrorFilter },
  ],
})
export class AppModule {}
```

`UnhandledExceptionFilter` (`@Catch()`, everything else) turns infrastructure failures into a problem+json `500` — or `503` with `Retry-After` for `ProviderUnavailableError` — logs them at `error`, and never leaks a stack trace or internal message. Nest tries `APP_FILTER` providers in reverse registration order, so the specific `BusinessErrorFilter` is registered **last** to be tried first.

The interceptor only applies to HTTP. Workers never see it: at a worker edge the `Result` already stopped in the use-case (`workers.md`).

## The status map

One `Record` in `commons/http/business-error-status.ts` maps every business error type to a status. Because the key type is the `BusinessErrorType` union, adding a new error type without a status is a compile error — the mapping cannot drift.

```ts
export const BUSINESS_ERROR_STATUS: Record<BusinessErrorType, HttpStatus> = {
  INVALID_AMOUNT: HttpStatus.BAD_REQUEST,
  INVALID_PIX_KEY: HttpStatus.BAD_REQUEST,
  PAYMENT_NOT_FOUND: HttpStatus.NOT_FOUND,
  INVALID_TRANSITION: HttpStatus.CONFLICT,
  INSUFFICIENT_LIMIT: HttpStatus.UNPROCESSABLE_ENTITY,
};
```

Choosing statuses:
- Input that fails a value-object rule → `400`.
- Resource missing, **or existing but owned by someone else** → `404`. Never `403` for another customer's resource; it confirms existence.
- State conflict (invalid transition, duplicate, idempotency key reused with a different payload) → `409`.
- Valid input rejected by a business rule (limit, eligibility) → `422`.

## Output is the response contract

With the interceptor, the use-case `Output` becomes the response body as-is. Design it as the API contract:

- Minimal: only what the client needs (`{ paymentId }`), never the aggregate, never internal ids or flags.
- Named for the client: the field the client receives is the field the `Output` declares.
- A command with no `Output` returns `Result<void, BusinessError>` and pairs with `@HttpCode(HttpStatus.NO_CONTENT)`.
- `@nestjs/swagger` cannot read `Promise<Result<...>>`: declare the response type explicitly with `@ApiCreatedResponse` / `@ApiOkResponse` and a response DTO class mirroring the `Output`.

Escape hatch, the exception and not the rule: when an endpoint genuinely needs a body different from the `Output`, the controller awaits the result, throws `new BusinessErrorException(result.error)` on `isErr`, and builds the response explicitly. The same filter handles it.

## DTO vs Input

The DTO is an HTTP concern: wire shape plus `class-validator` decorators (or a Zod schema). It lives in `<ctx>.dto.ts`, beside the controller. Validation of format (required, type, max length, UUID) happens here; validation of business meaning happens in value objects.

The use-case's `Input` is a domain-facing type. The controller maps DTO → `Input` field by field — never passes the DTO itself. They often look similar, and that is fine: the DTO changes when the API contract changes; the `Input` changes when the use-case's needs change.

## Rules

- One controller per context, at the context root.
- Map DTO → `Input` explicitly; never pass the DTO to the use-case.
- Controllers return the use-case `Result` directly; `ResultInterceptor` + `BusinessErrorFilter` are the only place `Result` becomes HTTP.
- The status map is the single source of truth for business error → status.
- The use-case `Output` is the response contract: minimal, client-named, documented with an explicit Swagger response type.
- No business logic, no repository, no `try/catch` in the controller.
