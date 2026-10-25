# Dashboards and alerts

PromQL panels, alert rules, threshold guidance, and an on-call runbook for LLM traffic. Queries use Bifrost's Prometheus metrics (see [bifrost-telemetry.md](bifrost-telemetry.md#prometheus-metrics)); if you emit OpenTelemetry GenAI metrics from app code instead, map `bifrost_upstream_latency_seconds` to `gen_ai_client_operation_duration_seconds` and token counters to `gen_ai_client_token_usage` (exact exported names depend on your OTLP-to-Prometheus translation).

Before using a query, run `curl -s http://<bifrost>:8080/metrics | grep <metric>` to confirm label names and values in your version.

## Contents

- [Dashboard layout](#dashboard-layout)
- [Traffic and errors](#traffic-and-errors)
- [Latency and TTFT](#latency-and-ttft)
- [Tokens and cost](#tokens-and-cost)
- [Routing, fallbacks, keys](#routing-fallbacks-keys)
- [Agents and tools](#agents-and-tools)
- [Alert rules](#alert-rules)
- [Choosing thresholds](#choosing-thresholds)
- [Runbook: alert to action](#runbook-alert-to-action)

## Dashboard layout

One dashboard, top to bottom, with template variables `$provider`, `$model`, `$feature` (custom label), `$team`:

1. **Health row**: request rate, error rate excluding caller errors, p95 latency, p95 TTFT, cost per hour, active requests.
2. **Errors**: errors by `error_type`, errors by provider and model, `_OTHER` count.
3. **Latency**: p50/p95/p99 provider latency by model; TTFT by model; inter-token latency; gateway overhead.
4. **Spend**: cost per hour by feature/team/model, tokens per minute, cost per request, output/input token ratio.
5. **Resilience**: fallback share, retries, key rotations by reason, key health.
6. **Quality** (from your eval store): eval score by feature and prompt version, negative-feedback rate.

Link panels to the logs: a Grafana data link to `http://<bifrost>:8080/logs` (or your trace backend filtered by `provider`/`model`) closes the loop from graph to request.

## Traffic and errors

```promql
# Requests per second by provider and model
sum by (provider, model) (rate(bifrost_upstream_requests_total[5m]))

# Error rate excluding caller mistakes and your own policy refusals (the number to alert on)
sum(rate(bifrost_error_requests_total{error_type!~"(caller|policy)_.*"}[5m]))
  / sum(rate(bifrost_upstream_requests_total[5m]))

# Error rate per provider
sum by (provider) (rate(bifrost_error_requests_total{error_type!~"(caller|policy)_.*"}[5m]))
  / sum by (provider) (rate(bifrost_upstream_requests_total[5m]))

# Errors by fault domain and type
sum by (error_type) (rate(bifrost_error_requests_total[5m]))

# Provider 429s (need more upstream capacity / keys / spread)
sum by (provider, model) (rate(bifrost_error_requests_total{error_type="provider_rate_limited"}[5m]))

# Your own governance limits refusing traffic (limits too tight, or abuse)
sum by (error_type, virtual_key_name) (rate(bifrost_error_requests_total{error_type=~"policy_.*"}[5m]))

# Caller mistakes by team and model (bad model names, malformed requests)
sum by (error_type, team_name, model) (rate(bifrost_error_requests_total{error_type=~"caller_.*"}[5m]))

# Unclassified failures (should be flat zero)
sum(rate(bifrost_error_requests_total{error_type="_OTHER"}[5m]))
```

## Latency and TTFT

```promql
# p95 provider latency by model (successful requests only)
histogram_quantile(0.95,
  sum by (le, provider, model) (rate(bifrost_upstream_latency_seconds_bucket{is_success="true"}[5m])))

# p50 / p99: change 0.95 to 0.5 / 0.99

# p95 time to first token (streams) by model
histogram_quantile(0.95,
  sum by (le, provider, model) (rate(bifrost_stream_first_token_latency_seconds_bucket[5m])))

# p95 inter-token latency (streaming throughput degradation)
histogram_quantile(0.95,
  sum by (le, provider, model) (rate(bifrost_stream_inter_token_latency_seconds_bucket[5m])))

# Mean provider latency (cheap, for stat panels)
sum(rate(bifrost_upstream_latency_seconds_sum[5m])) / sum(rate(bifrost_upstream_latency_seconds_count[5m]))

# Gateway overhead p99 in milliseconds (should be tiny relative to provider time)
histogram_quantile(0.99, sum by (le) (rate(bifrost_overhead_latency_microseconds_bucket[5m]))) / 1000

# In-flight requests
sum by (method) (bifrost_active_requests)
```

Notes:
- `bifrost_upstream_latency_seconds` covers both streaming and non-streaming. For streams it is total stream time; break down by `method` to keep them apart.
- Compare latency per output token when evaluating model or prompt changes: longer answers are slower by construction.

## Tokens and cost

```promql
# Cost per hour by model (USD)
sum by (provider, model) (increase(bifrost_cost_total[1h]))

# Cost per hour by feature (requires prometheus_labels: ["feature"] and x-bf-dim-feature)
sum by (feature) (increase(bifrost_cost_total[1h]))

# Cost per day by team / customer (virtual keys with team_id / customer_id)
sum by (team_name) (increase(bifrost_cost_total[1d]))
sum by (customer_name) (increase(bifrost_cost_total[1d]))

# Cost per request by model
sum by (provider, model) (rate(bifrost_cost_total[5m]))
  / sum by (provider, model) (rate(bifrost_upstream_requests_total[5m]))

# Tokens per minute
sum by (model) (rate(bifrost_input_tokens_total[5m])) * 60
sum by (model) (rate(bifrost_output_tokens_total[5m])) * 60

# Average input tokens per request (prompt bloat, runaway context)
sum by (model) (rate(bifrost_input_tokens_total[15m]))
  / sum by (model) (rate(bifrost_upstream_requests_total[15m]))

# Output/input ratio
sum(rate(bifrost_output_tokens_total[5m])) / sum(rate(bifrost_input_tokens_total[5m]))

# Cache hit rate (semantic/direct cache)
sum(rate(bifrost_cache_hits_total[5m])) / sum(rate(bifrost_upstream_requests_total[5m]))

# Cost burn vs same hour last week
sum(increase(bifrost_cost_total[1h])) / sum(increase(bifrost_cost_total[1h] offset 1w))
```

## Routing, fallbacks, keys

```promql
# Share of successful requests served by a fallback (fallback_index 0 = primary)
sum(rate(bifrost_success_requests_total{fallback_index!="0"}[5m]))
  / sum(rate(bifrost_success_requests_total[5m]))

# Fallbacks by primary provider/model being bypassed: use logs (x-bifrost-routing-info-primary-*), or by served model:
sum by (provider, model) (rate(bifrost_success_requests_total{fallback_index!="0"}[5m]))

# p95 retries per request
histogram_quantile(0.95, sum by (le, provider) (rate(bifrost_request_retries_bucket[5m])))

# Key rotations by reason (rate limits, bad credentials, quota)
sum by (provider, fail_reason) (rate(bifrost_key_rotation_events_total[5m]))

# Keys most often rate-limited
topk(5, sum by (provider, key_name) (rate(bifrost_key_rotation_events_total{fail_reason="rate_limit_error"}[1h])))

# Unhealthy keys right now
bifrost_provider_key_up == 0

# Traffic by calling client (Claude Code, Cursor, API, ...)
sum by (app) (rate(bifrost_upstream_requests_total[5m]))
```

## Agents and tools

Agent-level signals (steps per run, exit reasons) come from your app spans or logs; gateway metrics show tool calls executed through Bifrost's MCP gateway:

```promql
# MCP tool call rate and failure rate by tool
sum by (mcp_client, mcp_tool_name) (rate(bifrost_mcp_client_operation_duration_seconds_count[5m]))
sum by (mcp_tool_name) (rate(bifrost_mcp_client_operation_duration_seconds_count{error_type!=""}[5m]))

# p95 tool latency
histogram_quantile(0.95, sum by (le, mcp_tool_name) (rate(bifrost_mcp_client_operation_duration_seconds_bucket[5m])))
```

If you emit an `agent_steps` histogram or `agent_runs_total{exit_reason}` counter from app code, alert on `exit_reason="max_steps"` share and on p99 steps.

## Alert rules

```yaml
groups:
  - name: llm-gateway
    rules:
      - alert: LLMHighErrorRate
        expr: |
          sum by (provider) (rate(bifrost_error_requests_total{error_type!~"(caller|policy)_.*"}[5m]))
            / sum by (provider) (rate(bifrost_upstream_requests_total[5m])) > 0.05
        for: 5m
        labels: { severity: page }
        annotations:
          summary: "LLM error rate {{ $value | humanizePercentage }} on {{ $labels.provider }}"

      - alert: LLMProviderRateLimited
        expr: |
          sum by (provider) (rate(bifrost_error_requests_total{error_type="provider_rate_limited"}[5m]))
            / sum by (provider) (rate(bifrost_upstream_requests_total[5m])) > 0.02
        for: 10m
        labels: { severity: warning }
        annotations:
          summary: "{{ $labels.provider }} returning 429 on {{ $value | humanizePercentage }} of requests"

      - alert: LLMUnclassifiedErrors
        expr: sum(increase(bifrost_error_requests_total{error_type="_OTHER"}[15m])) > 0
        labels: { severity: warning }
        annotations:
          summary: "LLM failures with no error classification"

      - alert: LLMLatencyP95High
        expr: |
          histogram_quantile(0.95,
            sum by (le, model) (rate(bifrost_upstream_latency_seconds_bucket{is_success="true"}[10m]))) > 20
        for: 10m
        labels: { severity: warning }
        annotations:
          summary: "p95 latency for {{ $labels.model }} is {{ $value }}s"

      - alert: LLMTTFTRegression
        expr: |
          histogram_quantile(0.95, sum by (le, model) (rate(bifrost_stream_first_token_latency_seconds_bucket[10m])))
            > 2 * histogram_quantile(0.95, sum by (le, model) (rate(bifrost_stream_first_token_latency_seconds_bucket[10m] offset 1d)))
        for: 15m
        labels: { severity: warning }
        annotations:
          summary: "p95 TTFT for {{ $labels.model }} doubled vs yesterday"

      - alert: LLMFallbackShareHigh
        expr: |
          sum(rate(bifrost_success_requests_total{fallback_index!="0"}[10m]))
            / sum(rate(bifrost_success_requests_total[10m])) > 0.10
        for: 10m
        labels: { severity: warning }
        annotations:
          summary: "{{ $value | humanizePercentage }} of requests served by fallbacks"

      - alert: LLMCostBurnHigh
        expr: sum(increase(bifrost_cost_total[1h])) > 50        # set to (daily budget / 24) * 2
        for: 15m
        labels: { severity: warning }
        annotations:
          summary: "LLM spend ${{ $value | printf \"%.2f\" }} in the last hour"

      - alert: LLMCostAnomaly
        expr: |
          sum(increase(bifrost_cost_total[1h]))
            > 3 * sum(increase(bifrost_cost_total[1h] offset 1w))
          and sum(increase(bifrost_cost_total[1h])) > 5
        for: 30m
        labels: { severity: warning }
        annotations:
          summary: "Hourly LLM spend is over 3x the same hour last week"

      - alert: LLMInputTokensPerRequestJump
        expr: |
          (sum by (model) (rate(bifrost_input_tokens_total[30m])) / sum by (model) (rate(bifrost_upstream_requests_total[30m])))
            > 2 * (sum by (model) (rate(bifrost_input_tokens_total[30m] offset 1d)) / sum by (model) (rate(bifrost_upstream_requests_total[30m] offset 1d)))
        for: 30m
        labels: { severity: info }
        annotations:
          summary: "Avg input tokens/request for {{ $labels.model }} doubled (prompt or context bloat)"

      - alert: LLMProviderKeyDown
        expr: min by (provider, key_name) (bifrost_provider_key_up) == 0
        for: 10m
        labels: { severity: warning }
        annotations:
          summary: "Key {{ $labels.key_name }} on {{ $labels.provider }} failing"

      - alert: BifrostMetricsMissing
        expr: absent(up{job="bifrost"} == 1)
        for: 5m
        labels: { severity: page }
        annotations:
          summary: "Bifrost metrics not scraped (gateway down or scrape auth failing)"
```

## Choosing thresholds

| Signal | Starting threshold | Adjust by |
|---|---|---|
| Error rate (non-caller, non-policy) | 5% for 5m page; 2% for 15m ticket | Your SLO; providers have routine 0.1 to 1% noise |
| Provider 429 share | 2% for 10m | Higher if retries/fallbacks absorb them without user impact |
| p95 latency | 2x the 7-day baseline per model, or an absolute SLO (e.g. 20s non-stream) | Separate stream vs non-stream; reasoning models need their own baseline |
| p95 TTFT | 2x yesterday, or absolute (e.g. 3s for chat UIs) | Long prompts raise TTFT; segment by feature |
| Fallback share | 10% for 10m | Any sustained fallback is a quality and cost change |
| Cost per hour | 2x (daily budget / 24) | Weekday/weekend seasonality: compare to `offset 1w` |
| Input tokens per request | 2x yesterday | Catches prompt bloat, runaway agent context, retrieval returning too much |
| `_OTHER` errors | any | Should always be zero |

Use `for:` durations long enough to ride out single-minute provider blips; LLM error bursts are spiky.

## Runbook: alert to action

| Symptom | First query | Likely cause | Action |
|---|---|---|---|
| Error rate up, `provider_rate_limited` | 429s by provider/model; key rotations | Upstream quota | Add keys/weights, spread across providers, add fallbacks, lower concurrency |
| Error rate up, `provider_overloaded` / `provider_server_error` | errors by provider | Provider incident | Confirm on provider status page; fallbacks to another provider; pin traffic away |
| `provider_timeout` | latency p99 by model | Slow provider or oversized prompts | Check input tokens per request; raise timeout only if outputs are legitimately long |
| `provider_auth_failed` / `provider_billing` | errors by key | Key revoked or account out of credit | Rotate key; top up; key rotation should hide it if other keys exist |
| `policy_budget_exceeded` / `policy_rate_limited` | errors by virtual key | Your limits | Raise limit or investigate abuse by that key/team |
| `caller_*` spike | errors by team, model | Bad deploy (wrong model name, malformed request) | Roll back the calling service |
| `bifrost_dropped` | active requests, queue | Gateway queue full | Scale gateway replicas; raise pool size |
| TTFT up, total latency flat | TTFT by model, input tokens/request | Longer prompts, provider queueing | Trim context, enable prompt caching, try another region/provider |
| Cost spike | cost by feature/team/model; input tokens/request | New feature, prompt change, agent loop, abuse | Find top request in logs (`min_cost` filter), check agent step counts, set budgets |
| Fallback share up | served model by fallback_index | Primary degraded | Check primary provider; verify fallback model quality is acceptable |
| Negative feedback up, no errors | eval scores by prompt version | Quality regression (prompt/model change) | Diff prompt versions; roll back; add failing samples to regression set |

To go from a graph to a request: filter the Bifrost logs (`/logs` UI or `GET /api/logs` with `providers`, `models`, `status=error`, `min_latency`, `min_cost`, time range), open the record, then follow its trace ID into your tracing backend.
