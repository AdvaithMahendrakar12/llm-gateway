# LLM Gateway

One service that every app request to an LLM goes through. Apps call the
gateway; the gateway decides which provider serves the request, enforces
limits and budgets, survives provider failures, and records what everything
cost.

Written in Go. Status: planning.

## The problem

Any company that ships an AI feature calls LLM APIs (OpenAI, Anthropic,
Google, ...) straight from its services. That breaks down in predictable ways:

| Pain | What happens without a gateway |
|---|---|
| **Providers fail** | A provider returns 429 (rate limited) or 5xx/overloaded, and the AI feature goes down with it. Every service has to write its own retries. |
| **Cost blows up** | One buggy loop or one abusive user burns thousands of dollars overnight. There is no per-customer or per-team budget to stop it. |
| **No visibility** | Nobody can answer "which feature or customer costs us the most?" or "how slow is the first token for users?" |
| **Keys are everywhere** | The real provider API key is copied into many services. A leak means rotating it everywhere at once. |
| **Lock-in** | Each provider has a different API, so switching or adding a fallback means code changes in every service. |
| **Paying twice** | Identical requests (same prompt, same model) are sent and billed again. |

A gateway fixes all of these in one place. Apps get **one OpenAI-compatible
API** and **virtual API keys** issued by the gateway; the real provider keys
live only in the gateway.

Existing products in this space: LiteLLM, Portkey, OpenRouter, Helicone,
Cloudflare AI Gateway. This project rebuilds the core to learn how it works.

## Why it is a distributed systems project

Every feature is a failure question:

- A provider dies **in the middle of a streamed response**. What does the client see?
- The client disconnects. Do we keep paying for the upstream generation?
- Three gateway instances run behind a load balancer. How do they share one rate limit?
- Redis is down. Do we **fail open** (allow, risk overspend) or **fail closed** (block, cause an outage)?
- Usage records are written after the response. What if the write is lost? Billing is now wrong.
- Provider A is slow but not down. When do we stop sending it traffic?

## Scope

### Phase 0: Mock provider
A fake LLM server that streams tokens, with knobs for latency, error rate and
"die mid-stream". Development is free (no API bills), and every failure is
reproducible on demand.

### Phase 1: Proxy
- `POST /v1/chat/completions` (OpenAI-compatible), forwarded to one provider
- Non-streaming first, then **streaming (SSE)**, flushing each chunk as it arrives
- Virtual API keys in Postgres, stored **hashed**
- Cancel the upstream call when the client disconnects (`context`)

### Phase 2: Limits and budgets
- Per-key **requests/min** and **tokens/min** in Redis, atomic via a Lua script
- Per-key monthly **budget in dollars**; reject when exhausted
- Shared across all gateway instances

### Phase 3: Reliability
- Timeouts on every upstream call
- Retries with exponential backoff + jitter, only for retryable errors (429, 5xx)
- **Circuit breaker** per provider: stop calling a failing provider, probe it later
- **Fallback** to a second provider when the first is down

### Phase 4: Usage accounting
- Record tokens, cost and latency for every request
- Off the request path: buffer and **batch-write** to Postgres
- Decide what happens to records if the gateway crashes before flushing

### Phase 5: Observability
- Prometheus: requests, errors per provider, **time to first token**, tokens/sec, cost
- Grafana dashboard

### Phase 6: Caching
- Exact-match response cache (same model + same messages + same params)

### Later: deploy on Kubernetes
Helm chart, Terraform, CI/CD, rolling out a new version gradually.

## Running it

Not yet.
