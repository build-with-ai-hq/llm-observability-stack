# Bifrost telemetry reference

Configuration for capturing LLM telemetry at the gateway with open-source Bifrost: request logs, content controls, per-request dimensions, Prometheus metrics, OpenTelemetry export, the Maxim plugin, and response headers. Doc pages are linked per section; check them when a field here does not match your version.

## Contents

- [How capture works](#how-capture-works)
- [Request logs and the log store](#request-logs-and-the-log-store)
- [Content logging and PII](#content-logging-and-pii)
- [Attribution: dimensions, logging headers, sessions](#attribution-dimensions-logging-headers-sessions)
- [Prometheus metrics](#prometheus-metrics)
- [OpenTelemetry export](#opentelemetry-export)
- [Maxim plugin: tracing and evals on production traffic](#maxim-plugin-tracing-and-evals-on-production-traffic)
- [Response headers](#response-headers)
- [Full config.json example](#full-configjson-example)

## How capture works

Every request through Bifrost (OpenAI-compatible `/v1/*` and drop-in routes `/openai`, `/anthropic`, `/genai`, `/langchain`, `/litellm`) passes through plugins. The logging plugin and the telemetry (Prometheus) plugin run in gateway mode by default and write asynchronously, off the request path. Add `otel` and `maxim` plugins to export elsewhere. Nothing changes in app code except the base URL and optional headers.

## Request logs and the log store

Docs: https://docs.getbifrost.ai/features/observability/default.md

Captured per request: input messages, parameters, provider and model, output messages and tool calls, tokens, cost, latency, status and error details, retry/key attempt trail, and metadata from logging headers.

```json
{
  "client": {
    "enable_logging": true,
    "disable_content_logging": false,
    "logging_headers": ["X-Tenant-ID", "X-Correlation-ID"]
  },
  "logs_store": {
    "enabled": true,
    "type": "sqlite",
    "config": { "path": "./logs.db" }
  }
}
```

For production volume use Postgres (database must be UTF8):

```json
{
  "logs_store": {
    "enabled": true,
    "type": "postgres",
    "config": {
      "host": "env.PG_HOST", "port": "5432", "user": "bifrost", "password": "env.PG_PASSWORD",
      "db_name": "bifrost", "ssl_mode": "require"
    }
  }
}
```

If `env.` substitution is not accepted for a field in your version, inject the value at deploy time instead.

Access:

- UI: `http://localhost:8080/logs`. Live stream, filters, request/response inspection. The **Group** toggle folds requests by session (`N turns`) and fallback attempts under the original request.
- API: `GET /api/logs` with filters:

| Filter | Example |
|---|---|
| `providers`, `models` | `openai,anthropic` / `gpt-4o-mini` |
| `status` | `success,error,processing` |
| `objects` | `chat.completion,embedding` |
| `start_time`, `end_time` | RFC3339 |
| `min_latency`, `max_latency` | ms |
| `min_tokens`, `max_tokens`, `min_cost`, `max_cost` | numbers (cost in USD) |
| `content_search` | text in messages |
| `tool_call_names` | `get_weather,search` |
| `request_id` | exact lookup; overrides time range |
| `limit`, `offset` | pagination |
| `roots_only=true`, `group_sessions=true` | fold fallbacks and sessions; rows carry `child_count`, `session_total_cost`, `session_total_tokens` |

The response includes `logs`, `pagination`, and `stats` (`total_requests`, `success_rate`, `average_latency`, `total_tokens`, `total_cost`).

Useful queries:

```bash
# Everything that failed in the last hour
curl -s "http://localhost:8080/api/logs?status=error&start_time=$(date -u -v-1H +%Y-%m-%dT%H:%M:%SZ 2>/dev/null || date -u -d '1 hour ago' +%Y-%m-%dT%H:%M:%SZ)&limit=50" | jq '.stats, .logs[0]'
# Slow requests (> 10 s)
curl -s "http://localhost:8080/api/logs?min_latency=10000&limit=50" | jq '.stats'
# Expensive requests (> $0.50 each)
curl -s "http://localhost:8080/api/logs?min_cost=0.5&limit=50" | jq '.stats'
# The exact request behind a user complaint (your x-request-id)
curl -s "http://localhost:8080/api/logs?request_id=<id>" | jq '.logs[0]'
```

A log's ID is its request ID: either the `x-request-id` you sent or a generated UUID, also returned as `x-bifrost-request-id`. Real-time updates are also available over WebSocket at `ws://localhost:8080/ws`.

## Content logging and PII

Docs: https://docs.getbifrost.ai/features/observability/content-logging.md

Metadata (model, provider, tokens, cost, latency, status, attribution) is always logged. Content (messages, tool definitions and calls, parameters, raw bytes) is controlled per destination:

| Layer | Setting | Scope |
|---|---|---|
| Global | `client.disable_content_logging: true` | Log store only |
| Per virtual key | `disable_content_logging` on the key (`true`/`false`/`null` = inherit) | Log store; `true` also strips content from OTel export |
| Per request | header `x-bf-disable-content-logging: true` | Ignored unless `client.allow_per_request_content_storage_override: true` |
| Per exporter | OTel profile `disable_content_logging`, `disable_root_span_content` | That exporter only |
| Metrics | n/a | Never carry content |

Pattern "content on, suppressed for sensitive requests":

```json
{
  "client": {
    "enable_logging": true,
    "disable_content_logging": false,
    "allow_per_request_content_storage_override": true
  }
}
```

```python
client.chat.completions.create(model=MODEL, messages=msgs,
    extra_headers={"x-bf-disable-content-logging": "true"})
```

```ts
await client.chat.completions.create({ model: MODEL, messages: msgs },
  { headers: { "x-bf-disable-content-logging": "true" } });
```

Pitfalls:

- `client.disable_content_logging` does **not** stop content going to your OTel collector. Set `disable_content_logging` on the OTel profile too.
- With per-request override enabled, a caller can also send `x-bf-disable-content-logging: false` to force content on. For strict no-content workloads keep the override off and audit virtual keys for `disable_content_logging: false`.
- Values captured via `client.logging_headers`, `x-bf-lh-*`, OTel `request_headers`, and `x-bf-dim-*` are exported regardless of content flags. Never put PII or secrets in them.
- Content-free records still support cost attribution, latency, error analysis, and feedback joins. Only evals and debugging need content.

## Attribution: dimensions, logging headers, sessions

Docs: https://docs.getbifrost.ai/providers/request-options.md

| Header | Goes to | Use for |
|---|---|---|
| `x-bf-dim-<name>: <value>` | log metadata, OTel span attributes, Prometheus label `<name>` (only if declared in `client.prometheus_labels`), Maxim tags | feature, team, environment, tenant tier: low-cardinality dimensions |
| `x-bf-lh-<name>: <value>` | log metadata only | high-cardinality IDs you want searchable in logs, not in metrics |
| `client.logging_headers: ["X-Tenant-ID"]` | log metadata | capturing headers your services already send |
| `x-bf-session-id: <id>` | OTel `session.id` on the root span; session grouping in logs and traces; session affinity | multi-turn conversations, agent runs |
| `x-request-id: <id>` | becomes the log ID and `x-bifrost-request-id` | feedback join key |
| `traceparent` (W3C) | Bifrost spans join the caller's trace; echoed as `x-bifrost-trace-id` | distributed tracing |
| `x-bf-vk: sk-bf-...` | virtual key: `virtual_key_*`, `team_*`, `customer_*` labels | per-team/customer cost |

Dimension names `path` and `method` are reserved and ignored for metrics. `x-bf-prom-*` is the deprecated Prometheus-only form; prefer `x-bf-dim-*`.

Coding agents (Claude Code, Codex CLI, OpenCode) already send their own session header; Bifrost adopts it when `x-bf-session-id` is absent.

## Prometheus metrics

Docs: https://docs.getbifrost.ai/features/observability/prometheus.md and https://docs.getbifrost.ai/features/telemetry.md

`GET /metrics` is exposed by the telemetry plugin, enabled by default. With Bifrost auth enabled, scrape with basic auth (`admin_username` / `admin_password`), over `https` when TLS is on.

```yaml
scrape_configs:
  - job_name: bifrost
    metrics_path: /metrics
    scrape_interval: 15s
    static_configs:
      - targets: ["bifrost-host:8080"]
    # basic_auth:
    #   username: '<admin_username>'
    #   password: '<admin_password>'
```

Key metrics:

| Metric | Type | Notes |
|---|---|---|
| `bifrost_upstream_requests_total` | counter | requests to providers |
| `bifrost_success_requests_total` | counter | |
| `bifrost_error_requests_total` | counter | labels `status_code`, `error_type` |
| `bifrost_upstream_latency_seconds` | histogram | provider latency; label `is_success` |
| `bifrost_stream_first_token_latency_seconds` | histogram | TTFT for streams |
| `bifrost_stream_inter_token_latency_seconds` | histogram | streaming throughput |
| `bifrost_overhead_latency_microseconds` | histogram | gateway's own overhead |
| `bifrost_input_tokens_total`, `bifrost_output_tokens_total` | counter | |
| `bifrost_cost_total` | counter | USD |
| `bifrost_cache_hits_total` | counter | label `cache_type` (direct/semantic) |
| `bifrost_active_requests` | gauge | in-flight, label `method` |
| `bifrost_request_retries` | histogram | retries per request |
| `bifrost_key_rotation_events_total` | counter | labels `provider`, `requested_model`, `key_name`, `fail_reason` |
| `bifrost_provider_key_up` | gauge | per-key health 1/0 |
| `bifrost_mcp_client_operation_duration_seconds` | histogram | MCP tool calls; labels `mcp_client`, `mcp_tool_name`, `error_type` |
| `http_requests_total`, `http_request_duration_seconds` | counter/histogram | transport level, label `path` is the route template |

Base labels on request-level LLM metrics: `provider`, `model`, `alias`, `method`, `app` (client classified from User-Agent), `virtual_key_id`/`virtual_key_name`, `team_id`/`team_name`, `customer_id`/`customer_name`, `project_id`/`project_name`, `routing_engine_used`, `routing_rule_id`/`routing_rule_name`, `complexity_tier`, `selected_key_id`/`selected_key_name`, `fallback_index` (0 = primary), plus custom labels. `user_id`/`user_name` are off by default (`user_labels_enabled` on the telemetry plugin) because of cardinality.

`error_type` values are prefixed by fault domain: `caller_*` (bad request, unknown model, cancelled), `policy_*` (your budgets, rate limits, blocked models), `provider_*` (`provider_rate_limited`, `provider_overloaded`, `provider_server_error`, `provider_timeout`, `provider_auth_failed`, `provider_connection_failed`, ...), `bifrost_*` (`bifrost_dropped`, `bifrost_internal`), and `_OTHER` (unclassified; should stay at zero).

Custom labels from dimensions:

```json
{
  "client": {
    "prometheus_labels": ["feature", "team", "environment"]
  }
}
```

Then send `x-bf-dim-feature: ticket-summary` etc. Only declared names become labels; undeclared `x-bf-dim-*` values still reach logs, spans, and Maxim.

Multi-node: scraping behind a load balancer misses nodes. Scrape each pod directly (Kubernetes service discovery) or enable the Push Gateway:

```json
{
  "plugins": [
    {
      "name": "telemetry",
      "enabled": true,
      "config": {
        "push_gateway": {
          "enabled": true,
          "push_gateway_url": "env.PUSHGATEWAY_URL",
          "job_name": "bifrost",
          "push_interval": 15
        }
      }
    }
  ]
}
```

Example PromQL is in [dashboards-and-alerts.md](dashboards-and-alerts.md).

## OpenTelemetry export

Docs: https://docs.getbifrost.ai/features/observability/otel.md

The `otel` plugin sends traces to any OTLP collector (self-hosted Collector, Grafana Tempo, Jaeger, Honeycomb, New Relic, Datadog via its OTLP intake, and others) using the GenAI conventions (`trace_type: "genai_extension"`).

```json
{
  "plugins": [
    {
      "enabled": true,
      "name": "otel",
      "config": {
        "service_name": "bifrost",
        "collector_url": "http://otel-collector:4318/v1/traces",
        "trace_type": "genai_extension",
        "protocol": "http",
        "headers": { "Authorization": "env.OTEL_API_KEY" },
        "group_traces_by_session": false,
        "disable_content_logging": false,
        "excluded_attributes": ["gen_ai.request.tools"]
      }
    }
  ]
}
```

Newer versions (v1.5.8+) also accept a `profiles` array inside `config` (one object per destination, each with `enabled` and the fields above), which lets you send full content to an internal collector and metadata-only to a third-party backend. Use `protocol: "grpc"` with `collector_url: "host:4317"` for OTLP/gRPC; `tls_ca_cert` for private CAs.

Fields worth knowing:

| Field | Effect |
|---|---|
| `group_traces_by_session` | requests with the same `x-bf-session-id` share one trace (deterministic trace ID). An incoming `traceparent` always wins. |
| `disable_content_logging` | drops messages, tool definitions, and tool args/results from every span; metadata stays |
| `disable_root_span_content` | drops the duplicated content on the root span only |
| `excluded_attributes` | exact attribute names to drop (typos silently do nothing) |
| `request_headers` | header name patterns copied to root span as `http.request.header.*`; exported regardless of content flags |
| `export_raw_payloads` | attach raw provider bodies; requires provider `store_raw_request_response` and content enabled |
| `OTEL_RESOURCE_ATTRIBUTES` env | e.g. `deployment.environment=production,service.version=1.2.3` on every span |

What spans carry: span name by request type (`gen_ai.chat`, `gen_ai.embedding`, ...), `gen_ai.provider.name`, `gen_ai.request.model`, request parameters, messages and tool calls (unless disabled), token usage, `gen_ai.usage.cost`, timing, `session.id`, and on streams:

- `bifrost.response.time_to_first_token_ms`: request received to first chunk (what callers see); on both the LLM span and the root span. Use this for TTFT dashboards.
- `gen_ai.response.time_to_first_chunk` (seconds): provider-only streaming behaviour.

Failures carry `bifrost.error.type` (same vocabulary as the Prometheus `error_type` label), `error.type`, `gen_ai.error`, `gen_ai.error.code`, including requests refused before reaching a provider. Headers forwarded with `x-bf-eh-<name>` appear as `gen_ai.request.extra_header.<name>`.

Token attribute names in Bifrost spans may differ from the latest semconv names (check a received span: older `gen_ai.usage.prompt_tokens`/`completion_tokens` vs newer `input_tokens`/`output_tokens`) before writing backend queries.

Joining app traces: instrument your HTTP client so `traceparent` is sent to Bifrost; the gateway span then nests under your agent step span, and `x-bifrost-trace-id` in the response confirms the trace ID.

## Maxim plugin: tracing and evals on production traffic

Docs: https://docs.getbifrost.ai/features/observability/maxim.md

Maxim (getmaxim.ai) is an evaluation and observability platform for AI agents. The Bifrost `maxim` plugin forwards every request and response to a Maxim log repository as traces and generations. In Maxim you then run evaluators on production logs, curate failing logs into datasets for offline experiments, review with human annotation, and alert when quality or latency degrade. Use it when you want production scoring without building and running your own eval job.

```json
{
  "plugins": [
    {
      "enabled": true,
      "name": "maxim",
      "config": {
        "api_key": "env.MAXIM_API_KEY",
        "log_repo_id": "your-default-log-repo-id"
      }
    }
  ]
}
```

If `env.` is not accepted for `api_key` in your version, inject it at deploy time.

Per-request headers:

| Header | Effect |
|---|---|
| `x-bf-maxim-log-repo-id` | send this request to a different repository (e.g. per product) |
| `x-bf-maxim-session-id` | group traces into a Maxim session (conversation / agent run) |
| `x-bf-maxim-trace-id`, `x-bf-maxim-trace-name` | attach the generation to your own trace ID / name |
| `x-bf-maxim-generation-id`, `x-bf-maxim-generation-name` | name the generation (e.g. `rerank-step`) |
| `x-bf-maxim-<tag>` | any other suffix becomes a tag |
| `x-bf-dim-<name>` | also becomes a Maxim tag (lower priority than an explicit `x-bf-maxim-<name>`) |

Repository selection: header first, then `log_repo_id` from config; if neither is set, the request is not sent to Maxim. Supported request types: chat and text completions (plus Agent Gateway operations).

Linking feedback: use your `request_id` as `x-bf-maxim-trace-id` (or send it as a tag) so feedback collected in your app can be matched to the Maxim trace.

## Response headers

Docs: https://docs.getbifrost.ai/providers/response-headers.md

Every inference response carries:

| Header | Meaning |
|---|---|
| `x-bifrost-request-id` | your `x-request-id` or a generated UUID; equals the log ID |
| `x-bifrost-trace-id` | trace ID (inherited from `traceparent` or generated) |
| `x-bifrost-routing-info-provider`, `x-bifrost-routing-info-model` | who actually served the attempt |
| `x-bifrost-routing-info-requested-model`, `-requested-provider` | what the caller asked for |
| `x-bifrost-routing-info-is-fallback: true` | a fallback served (or returned the error); with `-primary-provider`, `-primary-model` |
| `x-bifrost-fallback-index` | which fallback (from 1) |
| `x-bifrost-routing-info-key` | name of the stored key used (never the secret) |
| `x-bifrost-upstream-latency-ms` | time waiting on the provider, summed over attempts; on streams only until the stream opened |
| `x-bifrost-request-type` | e.g. `chat_completion_stream` |
| `x-bifrost-cache-*` | semantic cache result |

Legacy `x-bifrost-provider` and `x-bifrost-resolved-model` are still sent; prefer `routing-info`. Browsers on another origin cannot read these headers (not in `Access-Control-Expose-Headers`); read them server side. App pattern: log `is-fallback` and `upstream-latency-ms` alongside your own end-to-end timing to split provider time from your own.

## Full config.json example

```json
{
  "client": {
    "enable_logging": true,
    "disable_content_logging": false,
    "allow_per_request_content_storage_override": true,
    "logging_headers": ["X-Tenant-ID"],
    "prometheus_labels": ["feature", "team", "environment"]
  },
  "logs_store": {
    "enabled": true,
    "type": "postgres",
    "config": { "host": "postgres", "port": "5432", "user": "bifrost", "password": "change-me", "db_name": "bifrost", "ssl_mode": "disable" }
  },
  "providers": {
    "openai": { "keys": [{ "name": "openai-key-1", "value": "env.OPENAI_API_KEY", "models": ["gpt-4o-mini", "gpt-4o"], "weight": 1.0 }] }
  },
  "plugins": [
    {
      "enabled": true,
      "name": "otel",
      "config": {
        "service_name": "bifrost",
        "collector_url": "http://otel-collector:4318/v1/traces",
        "trace_type": "genai_extension",
        "protocol": "http",
        "group_traces_by_session": true
      }
    },
    {
      "enabled": true,
      "name": "maxim",
      "config": { "api_key": "env.MAXIM_API_KEY", "log_repo_id": "your-log-repo-id" }
    }
  ]
}
```

Keep the provider list and models to what the project already uses. The telemetry plugin (`/metrics`) is on without being listed; add a `telemetry` entry only to change its config (push gateway, `user_labels_enabled`, `overhead_breakdown_enabled`).
