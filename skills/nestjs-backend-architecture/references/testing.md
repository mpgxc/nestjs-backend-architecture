# Testing

Contents: [What to test where](#what-to-test-where) · [Domain tests](#domain-tests) · [Use-case specs](#use-case-specs) · [Integration tests with a real Postgres](#integration-tests-with-a-real-postgres) · [Adapter tests for vendors](#adapter-tests-for-vendors) · [E2E: few and critical](#e2e-few-and-critical) · [Rules](#rules)

## What to test where

| Layer | Test type | Real dependencies | Where |
|-------|-----------|-------------------|-------|
| Value objects, entities, domain services | Unit | None — `new` and call | `domain/*.spec.ts` |
| Use-cases (commands) | Unit with fakes | None — hand-written port fakes | `use-cases/*.use-case.spec.ts` |
| Repository adapters, query services, migrations | Integration | Postgres in a container | `test/integration/` |
| Vendor adapters (ACL) | Unit | Client mocked | `adapters/*.spec.ts` |
| Critical HTTP flows | E2E | App + Postgres, vendors faked at the port | `test/e2e/` |

Most value per minute comes from domain tests and from integration tests of the SQL you actually wrote. Fakes cannot catch a wrong `JOIN`, a missing index or a migration that does not apply.

## Domain tests

Pure and fast; they are where the business rules live, so they are where the most tests go.

```ts
describe('Payment.refund', () => {
  it('rejeita estorno de pagamento pendente', () => {
    const payment = PaymentBuilder.pending().build();

    const result = payment.refund();

    expect(result.isErr).toBe(true);

    if (result.isErr) {
      expect(result.error.type).toBe('INVALID_TRANSITION');
    }
  });
});
```

Test builders (`PaymentBuilder`) live in `test/builders/` — this is the natural home of the Builder pattern.

## Use-case specs

Every command has a spec beside it. Inject in-memory fakes of the ports (`repositories.md`) and assert on the `Result` and on what was saved. Cover: the happy path, each guard, and the infrastructure failure path when the use-case reacts to it (compensation, failure state).

Do not mock `HttpService`, TypeORM or the `DataSource` in a use-case spec — that couples the test to transport and defeats the port.

## Integration tests with a real Postgres

Run repository adapters and query services against a real Postgres started with Testcontainers, with the real migrations applied:

```ts
describe('PaymentsQueryService (integration)', () => {
  let database: StartedPostgreSqlContainer;
  let dataSource: DataSource;

  beforeAll(async () => {
    database = await new PostgreSqlContainer('postgres:16-alpine').start();
    dataSource = await createTestDataSource(database.getConnectionUri());
    await dataSource.runMigrations();
  });

  afterAll(async () => {
    await dataSource.destroy();
    await database.stop();
  });

  it('pagina por cursor sem repetir itens', async () => {
    await seedPayments(dataSource, { accountId: 'account-1', count: 25 });
    const service = new PaymentsQueryService(dataSource);

    const firstPage = await service.listByAccount('account-1', null, 20);
    const secondPage = await service.listByAccount('account-1', firstPage.meta.nextCursor, 20);

    expect(firstPage.data).toHaveLength(20);
    expect(secondPage.data).toHaveLength(5);
    expect(secondPage.meta.nextCursor).toBeNull();
  });
});
```

- One container per test file (or per run with schema reset), never a shared developer database.
- These tests also prove the migrations apply from zero.

## Adapter tests for vendors

Mock the low-level client; throw `<Vendor>ApiError` with each status and assert the translation table (`integration.md`): which statuses become `Result.Err`, which are thrown as `ProviderUnavailableError`. Recorded fixtures of real vendor responses, including error bodies.

## E2E: few and critical

A handful of HTTP tests (Supertest against the real app and a real Postgres, vendors replaced at the port with `overrideProvider`) for the flows that move money and for the error contract: a business error returns the right status and a problem+json body with `code`; a validation error returns `VALIDATION_FAILED` with `errors`.

## Rules

- Rules are tested in the domain; orchestration in use-case specs; SQL in integration tests.
- Assert on `error.type`, never on `message`.
- Fakes for ports you own; never mocks for libraries you do not own inside use-case tests.
- Every command has a spec; every query service and repository adapter has an integration test.
