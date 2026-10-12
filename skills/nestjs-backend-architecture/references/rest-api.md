# REST API Design

Contents: [Principles](#principles) · [URIs](#uris) · [Methods and status codes](#methods-and-status-codes) · [Headers](#headers) · [Query parameters](#query-parameters) · [Payload](#payload) · [Pagination](#pagination) · [Errors: RFC 9457](#errors-rfc-9457) · [Versioning](#versioning) · [Anti-patterns](#anti-patterns)

## Principles

A well-designed API can be guessed by someone who never read it. That comes from three rules: **resources are nouns** (the URL names the thing, the HTTP method is the action); **each method has fixed semantics** (above all, whether it is idempotent); **the contract is uniform** (same error format, same pagination, same naming on every endpoint). A mediocre convention applied everywhere beats a perfect one applied sometimes.

## URIs

| Rule | Do | Avoid |
|------|----|-------|
| Plural nouns | `/accounts`, `/payments` | `/account`, `/getAccounts` |
| kebab-case paths | `/payment-orders` | `/paymentOrders`, `/payment_orders` |
| No verbs | `POST /accounts` | `POST /createAccount` |
| Shallow hierarchy (max 2 levels) | `/accounts/:id/transactions` | `/banks/:b/accounts/:a/cards/:c/limits/:l` |
| Opaque ids (UUID) | `/accounts/9f1b4c…` | `/accounts/42` (enumerable) |
| No extension, no trailing slash | `/accounts` | `/accounts.json`, `/accounts/` |

Non-CRUD business actions (`capture`, `refund`, `block`): prefer turning the verb into a resource when it has a life of its own — `POST /payments/:id/refunds` creates a refund you can later `GET /refunds/:id`. Use an action sub-resource (`POST /payments/:id/capture`) only when there is nothing to query afterwards. Both are `POST` and take an `Idempotency-Key`.

## Methods and status codes

| Method | Semantics | Idempotent | Typical success |
|--------|-----------|------------|-----------------|
| `GET` | Read, never change state | Yes (and safe) | `200` |
| `POST` | Create / start a process | No — hence `Idempotency-Key` | `201` + `Location` |
| `PUT` | Replace the whole resource | Yes | `200` / `204` |
| `PATCH` | Partial update | Treat as no | `200` / `204` |
| `DELETE` | Remove | Yes | `204` |

`2xx` = it worked; `4xx` = the client erred, do not retry as-is; `5xx` = we erred, may retry. Clients, gateways and our own retry policies rely on that split — a wrong status induces a wrong retry on the other side.

| Status | When |
|--------|------|
| `200` | Read, or write with a response body |
| `201` | Created; return `Location: /v1/payments/:id` |
| `204` | Success with no body — a command with no `Output` |
| `400` | Malformed request, or value-object validation failed |
| `401` / `403` | Missing or invalid credential / authenticated but not allowed |
| `404` | Not found — **or exists but belongs to someone else** |
| `409` | State conflict, invalid transition, idempotency key reused with another payload |
| `422` | Valid input rejected by a business rule (limit, eligibility) |
| `429` | Rate limited; return `Retry-After` |
| `500` | Bug or unexpected infrastructure failure — never for client errors |
| `503` | Dependency unavailable or circuit open; return `Retry-After` |

The business error → status mapping lives in one `Record` (`controllers.md`).

## Headers

Request:
- `Authorization: Bearer <jwt>` — never a token in the query string (it ends up in logs).
- `Idempotency-Key: <uuid>` — mandatory on money-moving writes (`idempotency.md`).
- `X-Correlation-Id` — inherited if present, generated if not, echoed in the response (`logging.md`).
- `If-Match: "<etag>"` — optimistic concurrency on updates; stale → `412`.
- `Content-Type: application/json` — reject others with `415`.

Response:
- `Location` with `201`.
- `X-Correlation-Id` — what the client pastes into a support ticket.
- `Cache-Control: no-store` by default on financial data.
- `Retry-After` with `429` and `503`.

Headers carry metadata about the communication, never business data.

## Query parameters

Query strings vary how a collection is read — filter, sort, paginate, select fields. They never identify a resource (that is the path) and never carry sensitive data: a CPF in a URL lands in proxy logs and browser history.

| Use | Convention | Example |
|-----|------------|---------|
| Filter | `field=value`, camelCase | `?status=SETTLED&currency=BRL` |
| Range | `From` / `To` suffixes | `?createdFrom=2026-01-01&createdTo=2026-06-30` |
| Sort | `sort`, `-` for desc, comma for multi | `?sort=-createdAt,fullName` |
| Search | `q` | `?q=ada` |
| Pagination | `limit` + `cursor`, or `page` + `pageSize` | see below |

Validate the query like the body: coerce types, cap `limit`/`pageSize` (≤ 100), allowlist sortable fields. An unvalidated query is a DoS vector (`pageSize=1000000000`) and an `ORDER BY` injection vector.

## Payload

| Topic | Convention |
|-------|------------|
| Field names | `camelCase` everywhere |
| Dates | ISO 8601 UTC with `Z`: `"2026-06-09T14:30:00Z"` |
| Money | `{ "amountCents": 1500, "currency": "BRL" }` — integer + currency, never a float |
| Ids | Opaque strings (UUID) |
| Enums | Stable upper-case strings: `"SETTLED"`, never magic numbers |
| Missing vs null | Responses always include the field (`null` when empty). In `PATCH`, absent = untouched, `null` = clear |
| Booleans | Affirmative names: `isActive`, `hasPendingRefund` |
| Envelope | Item: the resource directly. Collection: `{ data, meta }`. Never `{ success: true, data }` — the status already says it |

The payload is a contract, not a mirror of the database. Responses are built from the use-case `Output` or a query-service DTO; internal fields (hashes, audit flags, provider ids) never leak by "serializing the whole object".

## Pagination

- **Cursor (keyset)** — default for app-facing collections (statements, notifications). Stable under concurrent inserts and constant cost: `WHERE (created_at, id) < ($1, $2) ORDER BY created_at DESC, id DESC`. The cursor is opaque base64; the client never builds or parses it. Cannot jump to page 7 or show a cheap total.
- **Offset** — for backoffice screens where "page 3 of 12" is a requirement. Suffers page drift under writes and `OFFSET 100000` cost.

```json
{ "data": [], "meta": { "nextCursor": "eyJjcmVhdGVkQXQiOiIyMDI2LTA2LTA5In0", "limit": 20 } }
{ "data": [], "meta": { "page": 3, "pageSize": 20, "total": 231 } }
```

Always cap the page size.

## Errors: RFC 9457

Every error body is `application/problem+json`:

```json
{
  "type": "https://api.example.com/errors/insufficient-limit",
  "title": "excede o limite diário da conta",
  "status": 422,
  "code": "INSUFFICIENT_LIMIT",
  "correlationId": "7c3e…",
  "timestamp": "2026-06-09T14:30:00Z"
}
```

- `code` is the stable machine contract — the `BusinessError.type`. Clients switch on it, never on `title`.
- `type` is a stable URI that can point to documentation.
- Validation errors add an `errors` extension: `[{ "field": "email", "message": "formato inválido" }]`.
- Never expose stack traces, SQL or internal messages in `detail`. Log them; do not return them.

Built by the global `ResultInterceptor` + `BusinessErrorFilter` for business errors, and by the catch-all `UnhandledExceptionFilter` for everything else (`controllers.md`).

## Versioning

Major version in the URI (`/v1/payments`): explicit, visible in every log and `curl`. The version changes only on a breaking change, and breaking changes are the exception.

| Safe in the same version | Breaking (needs v2 or a new field) |
|--------------------------|-------------------------------------|
| Add an optional request field | Make an optional field required |
| Add a response field | Remove or rename a response field |
| Add an endpoint, or an input enum value | Change a field's type or format |
| Relax a validation | Tighten a validation that used to pass |

Tolerant reader on both sides: the server strips unknown request fields on public edges; clients ignore new response fields. Document that output enums may gain values — a client with an exhaustive `switch` on a response enum breaks on the first addition.

## Anti-patterns

- `200` with `{ success: false }`; `500` for a client validation error; a `GET` that changes state.
- Verbs in URLs; sequential ids; CPF or tokens in query strings.
- Serializing entities or ORM models; money as float; mixing camelCase and snake_case.
- Money-moving writes without `Idempotency-Key`; collections without a page-size cap; stack traces in error bodies.
