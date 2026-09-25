# Protecting LLM API endpoints

These notes cover protection from misuse, unexpected spend, and unreliable
provider behavior. The running example is a resume builder with summary
enhancement, job-bullet enhancement, and PDF import.

## The core idea

Authentication answers **who is calling**. It does not answer how often they may
call an expensive endpoint, how much data they may send, or what happens when a
request times out. LLM protection needs independent layers.

```text
browser
  -> JWT authentication
  -> input validation and input-size cap
  -> user + IP rate limits
  -> daily quota check
  -> idempotency check
  -> global concurrency semaphore
  -> LLM request: timeout, output cap, bounded retry
  -> usage telemetry and quota update
```

## 1. HTTP outcomes: know what can be retried

| Status / condition | Retry automatically? | Meaning |
| --- | --- | --- |
| 400, 401, 403, 413 | No | Invalid input, identity, authorization, or input size. |
| Application quota 429 | No | The user has no remaining allowance. |
| Provider 429 | Sometimes | The provider is busy or its own limit is reached. Honor `Retry-After`. |
| 500, 502, 503, 504 | Yes, bounded | These can be temporary upstream failures. |
| Network failure / timeout | Yes, carefully | The provider may have completed work even though the response was lost. |

Use `429 Too Many Requests` for a rate limit and `503 Service Unavailable` when
global LLM capacity is full.

## 2. Rate limiting: token bucket

Use a **token bucket** for interactive endpoints. A bucket has a maximum number
of permits and refills over time; every accepted request spends one permit.

```text
Summary enhancement example
capacity: 10 requests
refill: 10 permits every 15 minutes
request cost: 1 permit
```

This permits a small, legitimate burst but prevents indefinite usage. It is more
user-friendly than a fixed window, which can allow a large burst at a boundary.

Apply two buckets to each request:

```text
ai:user:<userId>:summary
ai:ip:<ipAddress>:summary
```

Reject if either bucket is empty. The user bucket constrains an account; the IP
bucket helps slow account-creation and credential-abuse attacks.

### Why Redis?

Vercel/serverless requests can run on separate instances. In-memory JavaScript
state is not shared and can disappear when an instance is recycled. Use Redis so
rate-limit operations are shared and atomic.

### Why not a leaky bucket first?

A leaky bucket smooths work into a queue. It is useful for asynchronous jobs,
but a resume editor expects immediate results. Start with token buckets and a
concurrency limit; add a job queue later for long-running imports.

## 3. Quotas are different from rate limits

A rate limit protects a short period; a daily quota limits total consumption.
For example, a user may have ten requests per 15 minutes but only thirty per day.

Store a durable MongoDB usage record:

```js
{
  userId,
  date: "2026-09-25", // UTC day
  operation: "summary",
  requestCount: 12,
  inputTokens: 4200,
  outputTokens: 860,
  estimatedCost: 0.02,
  lastRequestedAt: Date
}
```

Use an atomic update for the check-and-increment path. Use UTC boundaries so the
limit is consistent regardless of server location.

| Operation | Rate limit | Daily quota |
| --- | --- | --- |
| Summary enhancement | 10 / 15 minutes | 30 |
| Job-bullet enhancement | 20 / 15 minutes | 60 |
| PDF import | 3 / hour | 3 |
| Job tailoring | 5 / day | 5 |

Tune these from legitimate usage and actual provider cost.

## 4. Limit the cost of one request

The browser must not choose a model, temperature, provider parameters, or token
limit. The server owns those values.

Set and validate:

- Maximum characters for summary and job-description input.
- A stricter maximum for extracted PDF text before it reaches the model.
- An allowlist of approved models.
- A maximum generated-output token count per operation.
- A request timeout, such as 25 seconds.

Do not confuse the two meanings of “token”:

- **Token-bucket permit:** permission to make a request.
- **LLM input/output tokens:** provider billing and model-context units.

Both need tracking. Few requests can still be expensive if an input PDF is huge.

## 5. Global concurrency: semaphore

Rate limits do not stop many different users from calling the provider together.
Use a Redis-backed semaphore to allow, for example, ten LLM calls in flight.

```text
acquire slot
  -> call provider
  -> record metadata
  -> release slot in finally
```

Always release in `finally`. Use an expiry/lease as a fallback for process
crashes. If no slot is available, return a clear `503`; do not create an
unbounded queue.

## 6. Retries: exponential backoff with full jitter

Retries can help temporary provider failures, but synchronized retries can make
an outage worse. Use **full jitter**:

```text
delay = random(0, min(maxDelay, baseDelay * 2^attempt))
```

Recommended starting values:

```text
maximum retries: 2
base delay: 500 ms
maximum delay: 15 seconds
```

Prefer `Retry-After` when a provider sends it. Retry only provider 429s,
transient 5xx responses, network failures, and timeouts. Never retry validation,
authentication, authorization, input-size, or daily-quota failures.

The **server** owns automatic upstream retries. The browser disables the action
while it runs and shows an explicit Retry button after final failure.

## 7. Idempotency prevents duplicate bills

A timeout does not prove that the provider did not finish work, and users can
double-click. Give each AI action a client-generated UUID idempotency key.

```text
ai:idempotency:<userId>:<key>
```

Store short-lived Redis state for an in-progress or completed request. A repeat
request with the same key should wait for or return the existing result, not
make another provider call. Scope keys to the authenticated user and endpoint.

## 8. Privacy-safe telemetry

Log enough to detect misuse without recording sensitive resume content.

```text
timestamp, userId, endpoint, model, status, latencyMs,
inputTokens, outputTokens, estimatedCost, providerRequestId
```

Do not log raw resumes, PDF text, job descriptions, prompts, generated text, or
API keys. Set provider spend alerts and define a circuit breaker, such as
temporarily disabling AI calls above a chosen spend or error threshold.

## 9. Testing checklist

- A user and an IP cannot exceed their token buckets.
- Limits are shared across simulated server instances.
- Oversized input is rejected before calling the provider.
- Daily quota enforcement is atomic.
- Semaphore slots release after success, error, and timeout.
- Reused idempotency keys result in at most one provider call.
- Only transient failures use the two bounded, full-jitter retries.
- Logs contain metadata but never prompt or resume text.

## Suggested implementation order

1. Input validation, character caps, server-owned model settings, and timeout.
2. Redis token buckets for user and IP limits.
3. Idempotency keys and Redis global concurrency semaphore.
4. MongoDB daily usage records and quotas.
5. Full-jitter retry wrapper with `Retry-After` support.
6. Usage telemetry, spend alerts, and a circuit breaker.

## Practice exercises

1. Implement a local token bucket and test its refill behavior with a fake clock.
2. Replace its state store with Redis and test two concurrent requests.
3. Write an `AbortController` timeout wrapper around a fake provider call.
4. Simulate two 503s then success; verify only two retries occur.
5. Reuse an idempotency key and verify the provider function runs once.
6. Add a quota record and test that the next request returns 429 at the limit.
