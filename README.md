# LLM Gateway

One service that every app request to an LLM goes through. Apps call the
gateway; the gateway decides which provider serves the request, enforces
limits and budgets, survives provider failures, and records what everything
cost.

Written in Go. Status: planning.

The interesting part is not the proxy. It is the control plane around it:
shared limits across replicas, a budget that is reserved before the provider
is called, a fallback rule that respects a stream already in progress, and a
usage log that survives a process crash. The same problems show up in any
multi-tenant API platform. Here they are concrete because every mistake is a
line item on a bill.

## The problem

Any company that ships an AI feature calls LLM APIs (OpenAI, Anthropic,
Google, ...) straight from its services. That breaks down in predictable ways:

| Pain | What happens without a gateway |
|---|---|
| **Providers fail** | A provider returns 429 or 5xx, and the AI feature goes down with it. Every service writes its own retries. |
| **Cost blows up** | One buggy loop or one abusive user burns thousands of dollars overnight. There is no per-customer or per-team budget to stop it. |
| **No visibility** | Nobody can answer "which feature or customer costs us the most?" or "how slow is the first token?" |
| **Keys are everywhere** | The real provider API key is copied into many services. A leak means rotating it everywhere at once. |
| **Lock-in** | Each provider has a different API, so switching or adding a fallback means code changes in every service. |
| **Paying twice** | A client timeout retries the same completion, and a repeated prompt is billed again. |

A gateway fixes these in one place. Apps get **one OpenAI-compatible API**
and **virtual API keys** issued by the gateway. The real provider keys live
only in the gateway.

Existing products in this space: LiteLLM, Portkey, OpenRouter, Helicone,
Cloudflare AI Gateway. This project rebuilds the core — streaming proxy,
budgets, failover, durable usage, and the operational surface around them —
to make those failure modes explicit and testable.

## Failure model

Every feature is a decision about a failure. The decisions below are the
design; the phases implement them.

| Failure | What the gateway does |
|---|---|
| Provider dies **after** tokens have been flushed | The client sees a truncated SSE stream and an error event. The gateway does not switch providers mid-stream. Partial usage is still recorded. |
| Provider dies **before** the first byte | Retry if the error is retryable (429, 5xx), then fall back to the next provider. The client sees one response. |
| Client disconnects mid-stream | The upstream request is cancelled via `context`. Tokens the provider already generated are still settled against the budget. |
| Three replicas behind a load balancer | Requests/min, token reservations, and dollar budgets live in Redis and are mutated by a single Lua script, so the limit is one number for the whole fleet. |
| Redis is down and the check is a **rate limit** | Fail open. A limiter outage does not take the product down. |
| Redis is down and the check is a **budget** | Fail closed. The failure mode the budget exists to prevent is overspend. |
| Gateway crashes after the response, before Postgres | The usage event was appended to a Redis stream first. A flusher replays anything not yet in Postgres. A crash loses the response only if the stream append also failed. |
| Provider is slow, not down | A per-process circuit breaker opens, stops sending it traffic, and probes later. Breaker state is local until the proxy and fallback are solid; a shared breaker is a later step. |
| Client retries after a timeout | `Idempotency-Key` returns the original completion and does not call the provider again. |

## Design

### Reserve the budget, then settle it

Tokens and dollars are unknown until the provider responds. Recording cost
after the call means the request that exhausts the budget has already been
billed.

Before the upstream call the gateway estimates an upper bound (input estimate
plus `max_tokens`) and atomically **reserves** that many cents in Redis. The
request is rejected when the reservation does not fit. When the real usage
arrives, the difference is **refunded**. Requests-per-minute stays a plain
counter. Tokens-per-minute and dollars go through reserve / settle / refund.

A Redis pipeline is not enough for this. Pipelines cut round trips; they do
not make read-modify-write atomic. The check and the reserve are one Lua
script.

### Two policies when Redis is down

Fail-open and fail-closed are per check, not a global mode.

- Rate limit store unavailable: allow the request, increment a
  `limiter_fail_open_total` counter, and page on that counter rather than on
  user-facing 500s.
- Budget store unavailable: reject the request. Availability of the AI feature
  yields to the spend cap.

### Fallback stops at the first byte

Once a chunk has been flushed to the client, the response belongs to that
provider. Fallback runs only when the attempt fails with zero bytes written.
The mock provider has two independent knobs — die before the first token, and
die after N tokens — because those are different code paths.

Retries use exponential backoff with jitter, and only for 429 and 5xx.
A 400 is the caller's bug; retrying it spends money to get the same error.

### The circuit breaker is local first

Each process keeps its own closed / open / half-open machine per provider.
Replicas can disagree about a sick upstream for one cooldown window. That is
acceptable while timeouts, retries, and pre-first-byte fallback are still
being proven. Sharing breaker state in Redis comes after, as its own change,
with a test that shows a single open decision draining traffic on every replica.

### Usage is a write-ahead log

The request path appends one usage event to a Redis stream and returns. A
flusher batches those events into Postgres. The stream is the source of
recovery; Postgres is the system of record for "what did this key cost."

Ordering on every cross-store move: append to the stream, then consider the
request accounted for; mark the stream entry flushed only after the Postgres
insert commits. A crash in the middle replays a duplicate, and the usage row
is idempotent on the gateway request id. The worst case is a second insert
attempt, not a missing dollar.

In-memory buffering alone drops the tail on every deploy. That is a stated
loss budget only if someone accepts it. This design does not.

Each row carries: virtual key, team, model, provider, input tokens, output
tokens, reserved cents, settled cents, time to first token, total latency,
and whether the client disconnected.

Prices live in config (dollars per million input tokens, dollars per million
output tokens), not in code. The mock emits a usage chunk so settlement is
testable with no provider bill.

### Keys, identity, and logs

Virtual keys are stored as SHA-256. The secret is shown once at creation.
Lookup hashes the incoming bearer token and compares in constant time. Keys
are never written to logs.

Default request logs are model, key id, provider, status, latency, and token
counts. Prompt and completion bodies stay off unless `LOG_PROMPTS=true`.
Turning that on is an explicit decision, because prompts are customer data.

`Idempotency-Key` is part of the proxy, not a later add-on. A timeout retry
without it is a second billed completion.

### Hot path

Postgres is the system of record for keys and teams. The request path reads
key metadata from an in-process cache with a short TTL, and talks to Redis
for the limit and the reservation. The provider call is the thing the user
waits on; a key lookup should not be a database round trip every time.

## Scope

### Phase 0: Mock provider

A fake LLM server that streams SSE chunks. Knobs:

- latency to first token, and delay between tokens
- error rate (429, 500, 400)
- die before the first token
- die after N tokens
- emit a terminal usage chunk (input tokens, output tokens)

Development stays free of API bills, and every failure above is reproducible
on demand.

### Phase 1: Proxy

- `POST /v1/chat/completions`, OpenAI-compatible, one provider
- Non-streaming first, then streaming, flushing each chunk as it arrives
- Virtual keys in Postgres, stored hashed; secret shown once
- `Idempotency-Key` on the completion
- Cancel the upstream call when the client disconnects (`context`)
- Prompt bodies redacted in logs by default
- Standard library HTTP. No framework.

### Phase 2: Limits and budgets

- Per-key requests/min via one Lua script, shared by every replica
- Per-key tokens/min and monthly budget in cents: reserve, settle, refund
- Fail open on the rate-limit store, fail closed on the budget store
- `Retry-After` and rate-limit headers on 429

### Phase 3: Reliability

- Timeout on every upstream call
- Retry with exponential backoff and jitter, only for 429 and 5xx, only while
  zero bytes have been written
- Per-process circuit breaker per provider (closed / open / half-open)
- Fallback to a second provider only on a pre-first-byte failure

### Phase 4: Usage accounting

- Price table in config
- Append the usage event to a Redis stream on the request path, including
  partial usage when the client or the provider cuts the stream
- Flusher batch-writes to Postgres and only then acknowledges the stream
  entries
- Usage row is idempotent on the gateway request id

### Phase 5: Observability

Prometheus, scraped from each replica:

- requests, errors, and fallbacks per provider
- time to first token (histogram)
- tokens/sec
- cents reserved vs cents settled
- `limiter_fail_open_total` and budget rejections
- stream lag: usage events appended but not yet in Postgres
- breaker state per provider

Grafana is wired from the repo so the dashboard exists on first boot. Counters
are queried with `rate()`. Gauges (breaker state, stream lag, in-flight
requests) are read raw.

### Phase 6: Run it like a service

Docker Compose: Postgres, Redis, gateway (more than one replica), mock
provider, Prometheus, Grafana.

- Graceful shutdown: stop accepting, drain in-flight streams, flush the usage
  stream, then exit
- `make chaos`: Redis killed, mock killed before the first token, mock killed
  after three tokens, gateway killed before the flusher runs
- Load test (vegeta or k6) against the mock, reported as what the gateway
  adds to time-to-first-token
- CI runs `go test -race`

### Phase 7: Exact-match cache

Same model, same messages, same sampling params. A hit skips the provider and
still writes a usage row with `cached=true` and zero provider cost. Streaming
responses are cached only after the stream completes successfully. A partial
stream is never a cache entry.

### After the core is boring

These are real, and they wait until phases 0–6 have a chaos demo that passes:

- A second real provider behind the same adapter interface
- Circuit breaker state shared across replicas
- Kubernetes: Deployment, Service, PodDisruptionBudget, and a horizontal
  scaler driven by in-flight requests or a custom metric. CPU is a weak signal
  for a proxy that spends its time waiting on providers.
- Rolling out a build gradually, with the usage flusher draining before the
  old pods are removed

Helm and Terraform come with that deploy chapter. They do not come before the
failure demo.

## Operability

The thing an on-call engineer should be able to answer from the dashboard:

- Which provider is failing, and did the breaker open?
- Are we failing open on limits? That counter should be near zero.
- Is settled cost tracking reservations, or is the flusher behind?
- What is time to first token, and did a deploy move it?

Health:

- `GET /healthz` — process is up
- `GET /readyz` — Postgres and Redis both answer; 503 names which one did not

Shutdown order is part of the API of the process, not an afterthought. A
deploy that drops in-flight streams or the unflushed tail of the usage stream
is a failed deploy.

## Stack

| Piece | Choice |
|---|---|
| Language | Go (standard library HTTP, `context`, `httptest`) |
| Key store and usage record | Postgres (`pgx`) |
| Limits, reservations, usage WAL, idempotency | Redis (Lua for atomic updates, a stream for usage) |
| Metrics | Prometheus client, Grafana provisioned from the repo |
| Local runtime | Docker Compose |
| Tests | `go test`, race detector, a chaos target |

Provider calls go through one interface. The mock and any real provider are
implementations. Fallback is a chain in front of that interface, not a branch
buried in the HTTP handler.

## Running it

Not yet. The first runnable slice is phase 0 and phase 1: mock provider, one
gateway process, hashed keys, streaming, and cancellation when the client
hangs up.
