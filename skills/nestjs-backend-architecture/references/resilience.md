# Resilience: Timeout, Retry, Circuit Breaker

Contents: [The failure that spreads](#the-failure-that-spreads) · [Timeout](#timeout) · [Retry](#retry) · [Circuit breaker](#circuit-breaker) · [Composing the policies](#composing-the-policies) · [What the domain sees](#what-the-domain-sees) · [Pitfalls](#pitfalls)

## The failure that spreads

A slow dependency is worse than a dead one. If Acme answers every call only at the timeout, each request holds a connection and memory for the full timeout waiting for an error that was certain. The pool drains, internal queues swell, and the whole service slows down — including flows that have nothing to do with Acme. Retrying alone makes it worse: every retry is one more stuck call.

Three policies, always together, at the **client** layer (`infrastructure/integrations/<vendor>/`), never in the adapter or the use-case:

## Timeout

Every outbound call has a timeout. A call without one can hold a connection until the pool starves. Set it below the caller's own budget (an HTTP request with a 10 s gateway limit cannot wait 30 s on a vendor).

## Retry

- Only for **idempotent** or **idempotency-keyed** operations. Retrying a non-idempotent POST duplicates money movement.
- Only for availability faults: timeout, network error, `5xx`, `429` (honor `Retry-After`). Never for `4xx` business or validation responses.
- Bounded (2–3 attempts) with exponential backoff **and jitter**.

## Circuit breaker

A small state machine in front of the calls to one dependency:

- **CLOSED** — calls pass; failures are counted. Past the threshold it opens.
- **OPEN** — calls fail immediately without touching the network (fail fast). Stays open for a cooldown (e.g. 30 s).
- **HALF_OPEN** — lets one probe call through. Success closes the circuit; failure reopens it.

Calibration starting point: open on 5 consecutive availability faults (or 50% failure in a 10 s window with at least 5 calls), half-open after 30 s.

**Only availability faults count**: timeout, network, `5xx`, `429`. A `422` "invalid amount" is a business answer from a healthy provider — counting it would cut valid traffic for other customers because of one bad request.

**One breaker per dependency.** Acme has its own, Globex its own. An Acme outage must not cut calls to a healthy Globex.

## Composing the policies

With `cockatiel`:

```ts
const isAvailabilityFault = handleType(AcmeApiError, (error) => error.isAvailabilityFault());

const retryPolicy = retry(isAvailabilityFault, {
  maxAttempts: 2,
  backoff: new ExponentialBackoff({ initialDelay: 200, maxDelay: 2_000 }),
});

const breakerPolicy = circuitBreaker(isAvailabilityFault, {
  halfOpenAfter: 30_000,
  breaker: new ConsecutiveBreaker(5),
});

const timeoutPolicy = timeout(8_000, TimeoutStrategy.Aggressive);

export const acmeResilience = wrap(retryPolicy, breakerPolicy, timeoutPolicy);
```

Order, outside in: **retry → breaker → timeout → call**. The retry re-attempts; each attempt passes through the breaker (which can cut it immediately) and has its own timeout.

The client's private `request` wraps every call in `acmeResilience.execute(...)`, so all endpoints are covered at once (`integration.md`). Emit a metric and a `warn` log on the breaker's `onBreak` / `onReset` — an open circuit is an incident signal.

## What the domain sees

Nothing new. When the breaker is open, `cockatiel` throws `BrokenCircuitError` before calling the network. The adapter translates it — like a timeout or a `5xx` — into the availability exception (`ProviderUnavailableError`), and every caller already knows how to react:

| Caller | What happens when the circuit is open |
|--------|----------------------------------------|
| HTTP request (sync phase) | Fails in milliseconds instead of waiting for the timeout. The global filter returns `503` with `Retry-After`; any balance blocked inline is released by the existing compensation. |
| Queue worker | The exception propagates, the message is not deleted, SQS redelivers later. The worker is not stuck. |
| Reconciliation cron | The item stays pending and is resolved on a later tick, after the circuit closes. |

The breaker gives fail-fast during the outage; saga and reconciliation give recovery afterwards (`platform/saga.md`). No line of the domain knows a circuit exists.

## Pitfalls

- A global breaker for everything.
- Counting `4xx` business responses as failures.
- A breaker without a timeout — it counts failures, but each call still hangs.
- Assuming in-memory breaker state is global. Each ECS replica has its own; that is fine — they all detect the outage within seconds. A distributed breaker (shared state in Redis) is rarely worth it.
- A very short cooldown that hammers a provider that is trying to recover.
- Inventing a fallback. Some flows have one (stale cache for a read); a recharge or a payment does not — fail fast with a clear message and let the saga resume later.
- Synchronous fallbacks to another of our services (`platform/read-models.md`) without their own timeout and breaker — they reopen the coupling through the back door.
