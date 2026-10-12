# External API Integration

Contents: [Two-layer pattern](#two-layer-pattern) · [Layer 1: low-level client](#layer-1-low-level-client) · [Layer 2: adapter (ACL)](#layer-2-adapter-acl) · [Error translation](#error-translation) · [Configuration and wiring](#configuration-and-wiring) · [Testing](#testing) · [When to skip the ACL](#when-to-skip-the-acl)

## Two-layer pattern

Never let a vendor's payload shape reach the domain. Split every integration in two:

1. **Low-level client** (`infrastructure/integrations/<vendor>/`) — the vendor SDK we write ourselves: endpoints, DTOs mirroring the vendor JSON, auth, base URL, timeout, circuit breaker. Speaks only the vendor's language. Throws a typed `<Vendor>ApiError`, never a raw `AxiosError`. Knows nothing about our domain.
2. **Adapter / ACL** (`contexts/<ctx>/adapters/`) — implements a domain port, maps domain ↔ vendor DTOs, and translates `<Vendor>ApiError` into business outcomes.

```
domain port ← adapter (ACL) ← low-level client ← vendor API
              context            infrastructure/integrations/<vendor>/
```

They are separate because they change for different reasons: the client changes when the vendor's API changes; the adapter changes when our domain's needs change. The client is shared across contexts; the translation is context-specific.

```
infrastructure/integrations/acme/
├─ acme.client.ts          one method per endpoint, DTO in → DTO out
├─ acme-api.error.ts       typed error: httpStatus, code
├─ acme-auth.service.ts    token cache + refresh
├─ acme.resilience.ts      timeout + retry + circuit breaker policy
├─ acme.dto.ts             mirrors the vendor JSON (snake_case, vendor enums)
├─ acme-http.module.ts     HttpModule.registerAsync + own token
└─ acme-integration.module.ts
```

## Layer 1: low-level client

```ts
export class AcmeApiError extends Error {
  constructor(
    readonly httpStatus: number | null,
    readonly code: string | null,
  ) {
    super(`Acme API error (${httpStatus ?? 'network'})`);
  }

  isAvailabilityFault(): boolean {
    if (this.httpStatus === null) {
      return true;
    }

    return this.httpStatus === 429 || this.httpStatus >= 500;
  }
}
```

```ts
@Injectable()
export class AcmeClient {
  constructor(
    private readonly http: AcmeHttpService,
    private readonly auth: AcmeAuthService,
  ) {}

  async createPixPayment(body: AcmePixPaymentRequest, idempotencyKey: string): Promise<AcmePixPaymentResponse> {
    return this.request<AcmePixPaymentResponse>('POST', '/pix/v1/payment', body, { 'Idempotency-Key': idempotencyKey });
  }

  async consultPixPayment(clientCode: string): Promise<AcmePixStatusResponse> {
    return this.request<AcmePixStatusResponse>('GET', `/pix/v1/payment/${clientCode}`);
  }

  private async request<T>(method: Method, url: string, data?: unknown, headers?: Record<string, string>): Promise<T> {
    return acmeResilience.execute(async () => {
      const token = await this.auth.getAccessToken();

      try {
        const response = await firstValueFrom(
          this.http.request<T>({ method, url, data, headers: { ...headers, Authorization: `Bearer ${token}` } }),
        );

        return response.data;
      } catch (error) {
        throw this.toAcmeApiError(error);
      }
    });
  }

  private toAcmeApiError(error: unknown): AcmeApiError {
    if (!isAxiosError(error)) {
      return new AcmeApiError(null, null);
    }

    return new AcmeApiError(error.response?.status ?? null, error.response?.data?.errorCode ?? null);
  }
}
```

- `firstValueFrom`, axios and RxJS die here. Everything above sees `Promise` and `AcmeApiError`.
- Every call goes through one private `request`, so the resilience policy (timeout → retry → circuit breaker) covers every endpoint at once — see `resilience.md`.
- Never log the token, the `Authorization` header or the full request body. Redaction is configured in the logger (`logging.md`).

## Layer 2: adapter (ACL)

Implements the port; translates both directions:

```ts
@Injectable()
export class AcmePixGateway implements PixGateway {
  constructor(
    private readonly client: AcmeClient,
    private readonly logger: Logger,
  ) {}

  async dispatch(payment: Payment): Promise<Result<void, BusinessError>> {
    const request = AcmePixMapper.toPaymentRequest(payment);

    try {
      const response = await this.client.createPixPayment(request, payment.id);

      return AcmePixMapper.toDispatchResult(response);
    } catch (error) {
      return this.translateError(error, payment.id);
    }
  }

  private translateError(error: unknown, paymentId: string): Result<void, BusinessError> {
    if (error instanceof BrokenCircuitError) {
      throw new ProviderUnavailableError('acme', 'circuit open');
    }

    if (!(error instanceof AcmeApiError)) {
      throw error;
    }

    if (error.isAvailabilityFault()) {
      throw new ProviderUnavailableError('acme', `status ${error.httpStatus ?? 'network'}`);
    }

    this.logger.log({ message: 'acme-pix.rejected', paymentId, httpStatus: error.httpStatus, code: error.code }, AcmePixGateway.name);

    return AcmePixMapper.toBusinessError(error);
  }
}
```

- `AcmePixMapper` (beside the adapter) converts cents to the vendor's decimal format, maps vendor status strings to our enums, and maps vendor error codes to `BusinessError` objects with a `PIX_*` type — inline objects, no error classes.
- Our id is sent as the idempotency key: a retry is provably the same operation.
- If the vendor renames a field, exactly one file changes. Nothing in the domain knows Acme exists.

## Error translation

The adapter decides what a vendor failure **means**:

| Vendor failure | Meaning | Becomes |
|----------------|---------|---------|
| 400 / 422 with a validation code | Our request is invalid for the vendor | `Result.Err({ type: 'PIX_INVALID_REQUEST', ... })` |
| Domain refusal code (invalid key, closed account, insufficient funds) | Business outcome | `Result.Err` with a specific type |
| 404 on consult | Unknown at the vendor | `Result.Err({ type: 'PIX_PAYMENT_NOT_FOUND', ... })` |
| 409 idempotency conflict | Same key, different payload — a bug on our side | `throw` |
| 401 / 403 | Credential problem — alert, not business | `throw` |
| 429, 5xx, timeout, network, circuit open | Availability | `throw ProviderUnavailableError` |

`ProviderUnavailableError` is an infrastructure exception in `infrastructure/integrations/`. The use-case does not catch it; it propagates to the retry, queue or saga machinery, and the global HTTP filter maps it to `503` with `Retry-After`.

The port's docstring states this contract — business outcomes in `Result`, availability thrown — and every adapter of that port must honor it (Liskov: an adapter that throws where the contract promised a `Result`, or vice versa, breaks every caller).

## Configuration and wiring

Each vendor gets its own HTTP module and token, because one global `HttpModule` cannot hold several base URLs and timeouts:

```ts
export abstract class AcmeHttpService extends HttpService {}

@Module({
  imports: [
    HttpModule.registerAsync({
      inject: [ConfigService],
      useFactory: (config: ConfigService) => ({
        baseURL: config.getOrThrow<string>('ACME_BASE_URL'),
        timeout: config.get<number>('ACME_TIMEOUT_MS', 8000),
      }),
    }),
  ],
  providers: [{ provide: AcmeHttpService, useExisting: HttpService }],
  exports: [AcmeHttpService],
})
export class AcmeHttpModule {}

@Module({
  imports: [AcmeHttpModule],
  providers: [AcmeClient, AcmeAuthService],
  exports: [AcmeClient],
})
export class AcmeIntegrationModule {}
```

The context module imports `AcmeIntegrationModule` and binds its own port: `{ provide: PixGateway, useClass: AcmePixGateway }`. Credentials come from Secrets Manager through `ConfigService`, never from code or committed env files.

## Testing

- **Use-case tests** — inject a fake implementing the port. A hand-written fake is a few lines and reads better than a mock framework.
- **Adapter tests** — mock the client; throw `AcmeApiError` with each status and assert on the resulting `Result` or thrown error. This is where the translation table is tested.
- **Client tests** — mock the `HttpService`; assert request shape and normalization of `AxiosError` into `AcmeApiError`.
- **Contract drift** — recorded fixtures rot silently. Run a small suite against the vendor sandbox on a schedule, outside the main CI path.

Never mock `HttpService` inside a use-case test: it couples the test to transport and defeats the port.

## When to skip the ACL

If the integration is a trivial generic subdomain (a thin wrapper with no business semantics and no realistic chance of switching vendor), separate mappers and DTO types can be overkill. For payments, PIX, boleto, recharge and core banking — core, swappable vendors, rich error semantics — the full ACL pays for itself immediately.
