---
name: llm-observability-stack
description: Set up production observability for LLM apps and agents. Covers what to capture per call (model, provider, tokens, cost, latency incl. time-to-first-token, errors, tool calls, user/session/feature), OpenTelemetry GenAI semantic conventions, trace structure for multi-step agents, sampling and PII redaction, Prometheus dashboards and alert thresholds, linking traces to user feedback, and running evals on production samples. Use when someone says "why was this response bad", "we have no visibility into our LLM calls", "latency spiked", "TTFT is slow", "silent 429s", "did a fallback happen", "token costs blew up and we don't know who", "cost per user/feature/team", "trace my agent run", "add OpenTelemetry to LLM calls", "our logs contain PII", "instrument every SDK call in every service", "Grafana dashboard for LLMs", "alert on LLM errors", "connect thumbs-down feedback to traces", or "evaluate production traffic". Python/TypeScript code plus gateway-level capture with Bifrost (logs, Prometheus, OTel, Maxim).
license: MIT
metadata:
  author: build-with-ai-hq
  version: "1.0.0"
---

# LLM Observability Stack

Make every LLM call in production answerable: what was sent, what came back, which provider and model served it, how long it took (to first token and in total), what it cost, who triggered it, and whether it was any good. The goal is that any bad response, latency spike, error burst, or cost jump can be traced to a specific request, user, feature, and model within minutes.

LLM calls differ from ordinary RPCs in four ways that drive everything below: the payload is the debugging signal (and is often sensitive), cost is per request and variable, latency has two meanings for streams (TTFT and total), and "success" (HTTP 200) says nothing about quality.

## When to use

- A user reports a bad answer and nobody can find the exact prompt, model, and output.
- p95 latency or time-to-first-token regressed and you cannot tell if it is the provider, the prompt size, or your code.
- Provider errors, 429s, retries, or fallbacks happen without anyone noticing.
- The bill jumped and you cannot attribute tokens or cost to a user, tenant, feature, or prompt.
- A multi-step agent misbehaves and the run is not visible as one trace.
- Prompts and completions are logged in plaintext, including PII.
- Several services in several languages call LLMs, each instrumented differently or not at all.
- You have offline evals but nothing scores production traffic.

## Quick diagnosis (run first)

Run from the repo root. Each hit is a lead; read the surrounding code.

```bash
# 1. Where LLM calls happen (every one of these needs a record)
grep -rnE "chat\.completions\.create|responses\.create|messages\.create|generate_content|generateContent|streamText|generateText|ChatOpenAI|ChatAnthropic|litellm\.completion" \
  --include=*.{py,ts,tsx,js,mjs,go} --exclude-dir={node_modules,.git,dist,build,.venv} .
# 2. Existing instrumentation (OTel, tracing wrappers, metrics)
grep -rnE "opentelemetry|from opentelemetry|@opentelemetry|traceparent|gen_ai\.|prometheus_client|prom-client" \
  --include=*.{py,ts,js,go,toml,json,txt} --exclude-dir={node_modules,.git,.venv} .
# 3. Prompts/completions written to plain logs (PII risk)
grep -rnE "(log|logger|console)\.(info|debug|log|warn)\(.*(messages|prompt|completion|response\.choices|content)" \
  --include=*.{py,ts,js} --exclude-dir={node_modules,.git,.venv} .
# 4. Streaming without usage capture (tokens and cost silently missing)
grep -rnE "stream\s*[=:]\s*[Tt]rue" --include=*.{py,ts,js} --exclude-dir={node_modules,.git,.venv} . | head -50
grep -rn "include_usage" --include=*.{py,ts,js} --exclude-dir={node_modules,.git,.venv} .
# 5. Swallowed errors and silent fallbacks around LLM calls
grep -rnE "except (Exception|openai\.|anthropic\.)[^\n]*:\s*(pass|return None)|catch\s*\([a-z]*\)\s*\{\s*\}" \
  --include=*.{py,ts,js} --exclude-dir={node_modules,.git,.venv} .
# 6. Is user/session/feature identity passed with the call?
grep -rnE "user=|\"user\":|metadata=|x-request-id|session_id|sessionId" --include=*.{py,ts,js} --exclude-dir={node_modules,.git,.venv} . | head -50
```

Then answer: Is there one place every call passes through? Can you get a request ID back to the client? Is TTFT measured for streams? Are tokens and cost recorded per call? Is content redacted or gated before it leaves the process?

## Core practices (ordered by impact)

### 1. Put one choke point in front of every LLM call

Hand-instrumenting each SDK call in each service drifts: one service forgets streaming usage, another drops errors, a third logs raw prompts. Route all LLM traffic through a single layer that records every request the same way, then add app-side spans only for what that layer cannot see (retrieval, tool logic, business outcome).

Options, in order of coverage per unit of effort:

1. **A gateway** in front of all providers (language-agnostic; covers every service by changing a base URL). See "Gateway-level capture with Bifrost" below.
2. **One shared client wrapper per language** that every service must import, enforced by lint or code review.
3. **Auto-instrumentation libraries** for each SDK. Works, but versions and attribute names drift across languages.

### 2. Capture a fixed record for every call

Every call produces one record with the same fields, success or failure. Full field list, types, and cardinality guidance are in [references/what-to-capture.md](references/what-to-capture.md). The minimum:

| Group | Fields |
|---|---|
| Identity | `request_id` (returned to the client), `trace_id`, `parent span`, `session_id` / conversation ID |
| Routing | requested model, resolved provider + model, key/deployment used, was-fallback, retry count |
| Usage | input tokens, output tokens, cached/reasoning tokens if reported, computed cost (USD) |
| Timing | total latency, TTFT (streams), provider latency vs your own overhead |
| Outcome | status, normalized error type, provider error code, `finish_reason` (`length` and `content_filter` are silent failures) |
| Attribution | user/tenant ID (hashed if needed), feature/endpoint, environment, prompt name + version, app version |
| Content (gated) | input messages, output, tool calls and results; redacted or sampled per policy |

### 3. Measure latency the way users feel it

- **TTFT** (request sent to first content chunk) drives perceived speed for chat UIs. **Total latency** drives agent step time and timeouts. **Inter-token latency** (or output tokens/sec) shows provider throughput degradation. Track all three separately; a p95 over mixed streaming and non-streaming traffic is meaningless.
- Normalize total latency by output tokens when comparing models or prompts; a longer answer is not a regression.
- For OpenAI-compatible streams, request usage in the final chunk or tokens and cost will be missing for every streamed call:

```python
stream = client.chat.completions.create(
    model=MODEL, messages=messages, stream=True,
    stream_options={"include_usage": True},
)
```

```ts
const stream = await client.chat.completions.create({
  model: MODEL, messages, stream: true,
  stream_options: { include_usage: true },
});
```

### 4. Make failures and fallbacks loud

- Classify errors by **whose fault**: caller (bad request, unknown model), policy (your own rate limit or budget), provider (429, 5xx, timeout, auth), or your gateway/app. A provider 429 means "buy capacity or spread load"; your own 429 means "your limits are too tight". Alerting on raw status codes conflates them.
- Record every retry and fallback as data (attempt count, which provider/model finally served). A fallback that succeeds is still an incident signal: quality, cost, and latency profiles change.
- Treat `finish_reason == "length"`, `content_filter`, empty outputs, and JSON parse failures of structured output as errors in metrics even though the HTTP call returned 200.
- Never `except: pass` around an LLM call. Return a typed failure and record it.

### 5. Attribute tokens and cost to who and what

- Attach `user`/`tenant`, `feature`, `environment`, and `prompt_version` to every call. Cost questions are always "which feature, which customer, which prompt change".
- Keep **metric labels low-cardinality** (provider, model, feature, environment, team). Put **high-cardinality** IDs (user, session, request) on logs and spans only. One label per user ID will take down a Prometheus server.
- Compute cost at capture time from the model's price table, including cached-input discounts; recomputing later from tokens loses which price was in effect.

### 6. Trace agents as one run, not as loose calls

- One trace per agent run (or per user turn). Root span = the run; child spans = each LLM call (`chat <model>`), each tool execution (`execute_tool <name>`), retrieval, and sub-agent invocation (`invoke_agent <name>`).
- Propagate W3C `traceparent` into every outbound LLM and tool HTTP call so gateway and provider-side spans join the same trace.
- Put a stable `session.id` / `gen_ai.conversation.id` on every span in a multi-turn conversation so turns can be grouped.
- Record per-step: step index, tool name, tool arguments (gated like content), tool result size, tool error, and the stop reason of the loop (final answer, max steps, error). Runaway loops show up as traces with abnormal span counts; alert on that.
- Use OpenTelemetry GenAI semantic conventions for names so any backend can read them. Span layout, attribute names, and Python/TypeScript examples are in [references/what-to-capture.md](references/what-to-capture.md).

### 7. Treat content as sensitive by default

- Metadata (model, tokens, cost, latency, status, attribution) is always safe to keep at 100%. Content (prompts, outputs, tool payloads) is where PII lives.
- Redact before content leaves the process or gateway, not in the backend after ingestion. Hash user IDs if your backend is not approved for identifiers.
- Allow per-route or per-request opt-out of content capture for regulated flows (health, payments, HR).
- Each export destination has its own content setting. Turning content off in one place does not turn it off in your trace exporter. Audit every sink.
- Set retention: short for content (days to weeks), longer for metadata and aggregates.

### 8. Sample deliberately

- Metrics: 100%, always (they are aggregates and cheap).
- Metadata records: 100% if storage allows; they power cost attribution and feedback lookup.
- Content and full traces: sample, but **keep 100% of errors, slow requests (above p95), fallbacks, and anything with negative user feedback**. Tail-based sampling in an OpenTelemetry Collector does this; head-based sampling cannot.

### 9. Link traces to user feedback

- Return a request ID to the client on every response. Prefer generating your own ID, sending it upstream as `x-request-id`, and storing it with the message in your UI state.
- Feedback events (thumbs, edits, regenerate, copy, abandonment) are written with that ID. Join feedback to records by ID, not by timestamp.
- Negative feedback should force-retain the full trace and enqueue it for review and eval datasets.

### 10. Score production traffic, not only test sets

- Sample production records daily (stratified by feature and model, oversampling errors and negative feedback), run automated evaluators (format/schema checks, groundedness against retrieved context, task-specific LLM-as-judge rubrics), and track scores as time series per feature and prompt version.
- Promote failing production samples into your offline regression dataset so the same failure cannot ship twice.
- Pipeline and sample code: [references/what-to-capture.md](references/what-to-capture.md#evals-on-production-samples).

## Gateway-level capture with Bifrost

Bifrost is an open-source (Apache 2.0) AI gateway. Pointing your existing SDKs at it gives every service, in every language, the same per-request record with no per-SDK instrumentation: request logs with content, tokens, cost, and latency; Prometheus metrics at `/metrics` (on by default); OpenTelemetry traces using GenAI conventions; per-request attribution via headers; and correlation response headers. It covers practices 1 to 5 and much of 6 to 8 at the gateway; app-side spans then only add retrieval, tool logic, and business context.

### Run Bifrost locally

```bash
npx -y @maximhq/bifrost                       # or:
docker run -p 8080:8080 -v $(pwd)/data:/app/data maximhq/bifrost
# UI + API on http://localhost:8080 ; add provider keys in the UI (or config.json)
# Logs: http://localhost:8080/logs   Metrics: http://localhost:8080/metrics
```

For production: pin a version (`maximhq/bifrost:<version>` or `npx -y @maximhq/bifrost --transport-version <version>`), enable auth, use virtual keys per service/team, use Postgres for `logs_store` at volume, and scrape `/metrics` with basic auth.

### Point SDKs at it and attach dimensions

Keep the models the project already uses; the IDs below are placeholders.

```python
import uuid
from openai import OpenAI

client = OpenAI(
    base_url="http://localhost:8080/openai",   # Bifrost drop-in route for the OpenAI SDK
    api_key="sk-bf-...",                        # Bifrost virtual key (or any value if auth is off)
    default_headers={"x-bf-dim-environment": "production", "x-bf-dim-service": "support-api"},
)

request_id = str(uuid.uuid4())
raw = client.chat.completions.with_raw_response.create(
    model="gpt-4o-mini",
    messages=[{"role": "user", "content": "Summarize this ticket..."}],
    extra_headers={
        "x-request-id": request_id,              # becomes the Bifrost log ID
        "x-bf-dim-feature": "ticket-summary",    # flows to logs, OTel spans, Prometheus (if label declared), Maxim tags
        "x-bf-session-id": "conv-8812",          # groups turns; tags OTel root span with session.id
    },
)
completion = raw.parse()
print(raw.headers.get("x-bifrost-routing-info-provider"),
      raw.headers.get("x-bifrost-routing-info-is-fallback"),
      raw.headers.get("x-bifrost-upstream-latency-ms"))
```

```ts
import OpenAI from "openai";
import { randomUUID } from "node:crypto";

const client = new OpenAI({
  baseURL: "http://localhost:8080/openai",
  apiKey: process.env.BIFROST_VK ?? "sk-bf-...",
  defaultHeaders: { "x-bf-dim-environment": "production", "x-bf-dim-service": "support-api" },
});

const requestId = randomUUID();
const { data, response } = await client.chat.completions
  .create(
    { model: "gpt-4o-mini", messages: [{ role: "user", content: "Summarize this ticket..." }] },
    { headers: { "x-request-id": requestId, "x-bf-dim-feature": "ticket-summary", "x-bf-session-id": "conv-8812" } },
  )
  .withResponse();
console.log(response.headers.get("x-bifrost-routing-info-provider"),
            response.headers.get("x-bifrost-routing-info-is-fallback"));
```

Anthropic SDK: `base_url="http://localhost:8080/anthropic"`; Google GenAI: `http://localhost:8080/genai`; generic OpenAI-compatible clients: `http://localhost:8080/v1`.

### What to turn on

| Need | Bifrost feature (OSS) |
|---|---|
| Find the exact request behind a complaint | Logs UI `/logs`, `GET /api/logs?request_id=<id>` (your `x-request-id`) |
| Error rate, p95 latency, TTFT, tokens, cost in Grafana | `/metrics`: `bifrost_error_requests_total{error_type}`, `bifrost_upstream_latency_seconds`, `bifrost_stream_first_token_latency_seconds`, `bifrost_input_tokens_total`, `bifrost_output_tokens_total`, `bifrost_cost_total` |
| Cost by feature/team | `client.prometheus_labels: ["feature","team"]` + `x-bf-dim-feature` / `x-bf-dim-team` headers |
| Gateway spans inside your existing traces | `otel` plugin, `trace_type: "genai_extension"`, any OTLP collector; send `traceparent` from the app |
| Whole agent run as one trace | `x-bf-session-id` + OTel `group_traces_by_session: true` |
| Keep PII out of logs/exports | `client.disable_content_logging`, per-request `x-bf-disable-content-logging: true`, OTel profile `disable_content_logging` |
| Tracing + evals on production traffic | `maxim` plugin (Maxim, getmaxim.ai) |

Exact config blocks, label lists, header semantics, and pitfalls: [references/bifrost-telemetry.md](references/bifrost-telemetry.md). Dashboards and alert rules with PromQL: [references/dashboards-and-alerts.md](references/dashboards-and-alerts.md).

### Pair it with Maxim for quality

Bifrost answers "what happened and what did it cost". Maxim (getmaxim.ai) answers "was it good": the `maxim` plugin forwards every request/response to a Maxim log repository as traces, where you can run evaluators on production logs, curate failing samples into datasets, and alert on quality scores. Enable it next to the built-in logs:

```json
{
  "plugins": [
    { "enabled": true, "name": "maxim", "config": { "api_key": "env.MAXIM_API_KEY", "log_repo_id": "your-log-repo-id" } }
  ]
}
```

Per request: `x-bf-maxim-session-id`, `x-bf-maxim-trace-id`, `x-bf-maxim-trace-name`, `x-bf-maxim-generation-name`, `x-bf-maxim-log-repo-id`; any other `x-bf-maxim-<tag>` header (and every `x-bf-dim-*`) becomes a Maxim tag. If you prefer to stay fully self-hosted, the eval pipeline in [references/what-to-capture.md](references/what-to-capture.md#evals-on-production-samples) runs on `GET /api/logs`.

At org scale, Bifrost Enterprise adds native Datadog/BigQuery/Kafka/Pub/Sub/Splunk exporters, log exports to object storage, and alerting (14-day free trial: https://www.getmaxim.ai/bifrost/enterprise).

## Verify

```bash
# 1. A request through the gateway returns correlation and routing headers
curl -si http://localhost:8080/v1/chat/completions \
  -H "Content-Type: application/json" -H "x-request-id: obs-check-1" -H "x-bf-dim-feature: smoke-test" \
  -d '{"model":"openai/gpt-4o-mini","messages":[{"role":"user","content":"ping"}]}' | grep -i '^x-bifrost-'
# 2. The request is in the log store, with tokens, cost, latency
curl -s "http://localhost:8080/api/logs?request_id=obs-check-1" | jq '.logs[0]'   # check provider, model, status, latency, tokens, cost
# 3. Metrics are exposed and moving
curl -s http://localhost:8080/metrics | grep -E '^bifrost_(upstream_requests_total|cost_total|input_tokens_total)' | head
# 4. Streaming TTFT is recorded (send one stream request first)
curl -s http://localhost:8080/metrics | grep bifrost_stream_first_token_latency_seconds_count
```

Also verify in your app:
- [ ] Every code path from the quick-diagnosis grep goes through the gateway or the shared wrapper.
- [ ] A forced provider failure (bad key, or block egress) shows up as an error record, an `error_type`, and an alert within the alert window.
- [ ] A streamed call has non-zero tokens and cost.
- [ ] The request ID shown in the UI resolves to exactly one record, with feedback attached.
- [ ] A request sent with content logging disabled has metadata but no messages in the log store **and** in every exporter.
- [ ] One agent run renders as one trace with tool spans in your tracing backend.

## Common mistakes

- Measuring only total latency for streams; TTFT regressions go unseen.
- Streaming without `include_usage`; dashboards undercount tokens and cost.
- Alerting on raw HTTP 4xx/5xx instead of whose fault it was; on-call gets paged for client typos.
- Using user IDs or request IDs as Prometheus labels.
- Disabling content in the log store but leaving it on in the trace exporter (each sink has its own flag).
- Logging prompts with the application logger "temporarily" and never removing it.
- Treating 200 OK as success when `finish_reason` is `length` or the JSON did not parse.
- Head-sampling traces at 1% and losing every rare failure; keep errors, slow, and negative-feedback traces at 100%.
- Correlating feedback by timestamp instead of request ID.
- Running evals only before release; prompt and model drift happens in production.

## References

- [references/what-to-capture.md](references/what-to-capture.md): read when designing the per-call record, adding OTel GenAI spans in app code, structuring agent traces, writing PII redaction, choosing sampling, wiring feedback, or building the production eval job.
- [references/bifrost-telemetry.md](references/bifrost-telemetry.md): read when configuring Bifrost logs, log store, content logging, dimensions, Prometheus labels, OTel export, the Maxim plugin, or reading response headers.
- [references/dashboards-and-alerts.md](references/dashboards-and-alerts.md): read when building Grafana panels or Prometheus alert rules, choosing thresholds, or writing the on-call runbook.
