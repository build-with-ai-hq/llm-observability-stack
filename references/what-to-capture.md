# What to capture, and how

Vendor-neutral reference for the per-call record, OpenTelemetry GenAI conventions, agent trace structure, PII redaction, sampling, feedback linkage, and evals on production samples.

## Contents

- [The per-call record](#the-per-call-record)
- [Cardinality rules](#cardinality-rules)
- [OpenTelemetry GenAI semantic conventions](#opentelemetry-genai-semantic-conventions)
- [Trace structure for agents](#trace-structure-for-agents)
- [App-side spans in Python and TypeScript](#app-side-spans-in-python-and-typescript)
- [Measuring TTFT in app code](#measuring-ttft-in-app-code)
- [PII redaction](#pii-redaction)
- [Sampling policy](#sampling-policy)
- [Linking feedback to traces](#linking-feedback-to-traces)
- [Evals on production samples](#evals-on-production-samples)

## The per-call record

One record per LLM call, emitted on success and on failure. If a gateway produces this record, the app only needs to add the attribution fields as headers.

| Field | Type | Why |
|---|---|---|
| `request_id` | string | Primary key. Returned to the client; feedback joins on it. |
| `trace_id`, `span_id`, `parent_span_id` | string | Places the call inside a request or agent run. |
| `session_id` / `conversation_id` | string | Groups turns of a conversation or steps of a run. |
| `timestamp_start`, `timestamp_end` | time | Ordering and duration. |
| `operation` | enum | `chat`, `embeddings`, `text_completion`, `execute_tool`, ... |
| `requested_model` | string | What the code asked for. |
| `provider`, `response_model` | string | What actually served it (differs under aliases, routing, fallbacks). |
| `is_fallback`, `fallback_index`, `retry_count` | bool/int | Silent degradation detection. |
| `input_tokens`, `output_tokens` | int | Usage. |
| `cached_input_tokens`, `reasoning_tokens` | int | Explains cost changes when caching or reasoning effort changes. |
| `cost_usd` | float | Computed at capture time from the price table in effect. |
| `latency_ms_total` | float | End-to-end from the caller's view. |
| `ttft_ms` | float | Streams only. Request sent to first content chunk. |
| `provider_latency_ms` | float | Separates provider time from your/gateway overhead. |
| `stream` | bool | Never mix streaming and non-streaming in one latency percentile. |
| `status` | enum | `success`, `error`. |
| `error_type` | enum | Normalized: caller / policy / provider (rate_limited, timeout, server_error, auth) / internal. |
| `provider_error_code`, `http_status` | string/int | Raw detail for debugging. |
| `finish_reason` | string | `stop`, `length`, `tool_calls`, `content_filter`. |
| `tool_calls` | list | Names (always) and arguments (gated as content). |
| `user_id` (hashed), `tenant_id` | string | Cost and quality attribution. |
| `feature` / `endpoint` | string | Which product surface. Most cost questions are per feature. |
| `prompt_name`, `prompt_version` | string | Ties regressions to prompt changes. |
| `environment`, `app_version`, `region` | string | Deploy correlation. |
| `input_messages`, `output` | json | Content. Redacted, sampled, retained briefly. |
| `feedback` | joined | Thumbs, edits, regenerate; joined by `request_id`. |
| `eval_scores` | joined | Written later by the eval job. |

## Cardinality rules

| Sink | Allowed | Not allowed |
|---|---|---|
| Metric labels (Prometheus, OTLP metrics) | provider, model, operation, feature, environment, team, error_type, status | user ID, session ID, request ID, prompt text, raw error messages |
| Span attributes | all of the above plus user/session/request IDs | full documents larger than your backend's attribute limit |
| Log/record store | everything | unredacted secrets |

Rule of thumb: if a label can take more than a few hundred values, it does not belong on a metric.

## OpenTelemetry GenAI semantic conventions

The GenAI conventions are still marked "development" in OpenTelemetry. Names have changed between releases (for example `gen_ai.system` became `gen_ai.provider.name`, and `gen_ai.usage.prompt_tokens` became `gen_ai.usage.input_tokens`). Pin the semconv version you emit, and check attribute names on a real span before writing queries. Instrumentation libraries usually require `OTEL_SEMCONV_STABILITY_OPT_IN=gen_ai_latest_experimental` to emit the newest names.

Span names: `{gen_ai.operation.name} {gen_ai.request.model}` for model calls (for example `chat gpt-4o-mini`), `execute_tool {gen_ai.tool.name}` for tools, `invoke_agent {gen_ai.agent.name}` for agents. Span kind `CLIENT` for remote calls, `INTERNAL` for in-process tools.

Core attributes:

| Attribute | Example |
|---|---|
| `gen_ai.operation.name` | `chat`, `embeddings`, `text_completion`, `execute_tool`, `invoke_agent` |
| `gen_ai.provider.name` | `openai`, `anthropic`, `gcp.vertex_ai`, `aws.bedrock` |
| `gen_ai.request.model` / `gen_ai.response.model` | requested vs served model |
| `gen_ai.request.temperature`, `gen_ai.request.max_tokens`, `gen_ai.request.top_p` | parameters |
| `gen_ai.response.id` | provider response ID |
| `gen_ai.response.finish_reasons` | `["stop"]` |
| `gen_ai.usage.input_tokens`, `gen_ai.usage.output_tokens` | usage |
| `gen_ai.conversation.id` | conversation or session ID |
| `gen_ai.agent.name`, `gen_ai.agent.id` | agent identity |
| `gen_ai.tool.name`, `gen_ai.tool.call.id`, `gen_ai.tool.type` | tool spans |
| `error.type` | error class on failed spans |
| `server.address`, `server.port` | endpoint called |

Content (opt-in in the conventions, because it is sensitive): `gen_ai.input.messages`, `gen_ai.output.messages`, `gen_ai.system_instructions`, `gen_ai.tool.call.arguments`, `gen_ai.tool.call.result`. Many instrumentations gate this on `OTEL_INSTRUMENTATION_GENAI_CAPTURE_MESSAGE_CONTENT=true`.

Metrics defined by the conventions (Development status; names have changed between versions, so check the version you emit): token counters `gen_ai.client.inference.usage.input_tokens`, `.output_tokens`, `.cache_read.input_tokens`, `.cache_write.input_tokens` and `.reasoning.output_tokens`; per-operation token histograms `gen_ai.client.inference.operation.input_tokens` and `.output_tokens`; `gen_ai.client.inference.duration` (seconds) for model calls and `gen_ai.client.embeddings.duration` for embeddings (`gen_ai.client.operation.duration` covers other operations and should not be reported for inference or embeddings); and server-side `gen_ai.server.time_to_first_token` and `gen_ai.server.time_per_output_token`.

Custom attributes that the conventions do not cover (cost, feature, prompt version) should use your own namespace, for example `app.feature`, `app.prompt.version`, `app.cost_usd`.

## Trace structure for agents

```
trace: one user turn or one agent run
└─ invoke_agent support-agent          (root: run_id, session.id, user hash, feature, final status, step_count)
   ├─ retrieval kb.search               (query hash, top_k, doc IDs, latency)
   ├─ chat gpt-4o-mini                  (step=1, tokens, cost, finish_reason=tool_calls)
   ├─ execute_tool lookup_order         (tool.call.id, args [gated], result size, error)
   ├─ chat gpt-4o-mini                  (step=2, finish_reason=tool_calls)
   ├─ execute_tool refund_order         (side-effecting: always record outcome)
   ├─ invoke_agent billing-subagent     (nested run, same trace)
   │  └─ chat claude-sonnet ...
   └─ chat gpt-4o-mini                  (step=N, finish_reason=stop)
```

Record on the root span: total tokens and cost (sum of children), step count, loop exit reason (`final_answer`, `max_steps`, `error`, `user_cancelled`), and which tools were called. Alert on step_count above a ceiling and on `max_steps` exits; both indicate loops.

Multi-turn: either one trace per turn with a shared `session.id`/`gen_ai.conversation.id`, or one trace per session if the backend handles long traces. One trace per turn is the safer default.

## App-side spans in Python and TypeScript

Use these for the parts a gateway cannot see: the agent loop, tools, retrieval. If calls go through a gateway that exports OTel, do not also create a duplicate span per LLM call; propagate `traceparent` instead so the gateway's span nests under your step span.

```python
# pip install opentelemetry-api opentelemetry-sdk opentelemetry-exporter-otlp opentelemetry-instrumentation-httpx
from opentelemetry import trace
from opentelemetry.instrumentation.httpx import HTTPXClientInstrumentor

HTTPXClientInstrumentor().instrument()   # OpenAI/Anthropic Python SDKs use httpx: traceparent is injected
tracer = trace.get_tracer("support-agent")

def run_agent(session_id: str, user_hash: str, question: str):
    with tracer.start_as_current_span("invoke_agent support-agent") as run:
        run.set_attributes({
            "gen_ai.operation.name": "invoke_agent",
            "gen_ai.agent.name": "support-agent",
            "gen_ai.conversation.id": session_id,
            "app.user_hash": user_hash,
            "app.feature": "support-chat",
        })
        for step in range(1, MAX_STEPS + 1):
            resp = call_model(messages)                      # gateway span joins this trace
            calls = resp.choices[0].message.tool_calls or []
            if not calls:
                run.set_attribute("app.exit_reason", "final_answer")
                run.set_attribute("app.step_count", step)
                return resp
            for call in calls:
                with tracer.start_as_current_span(f"execute_tool {call.function.name}") as t:
                    t.set_attributes({
                        "gen_ai.operation.name": "execute_tool",
                        "gen_ai.tool.name": call.function.name,
                        "gen_ai.tool.call.id": call.id,
                    })
                    try:
                        result = TOOLS[call.function.name](call.function.arguments)
                        t.set_attribute("app.tool.result_bytes", len(str(result)))
                    except Exception as e:
                        t.record_exception(e)
                        t.set_status(trace.Status(trace.StatusCode.ERROR))
                        t.set_attribute("error.type", type(e).__name__)
                        result = {"error": str(e)}
                    messages.append(tool_message(call.id, result))
        run.set_attribute("app.exit_reason", "max_steps")
        run.set_attribute("app.step_count", MAX_STEPS)
```

```ts
// npm i @opentelemetry/api  (plus your SDK/exporter setup and an undici/http instrumentation for traceparent)
import { trace, SpanStatusCode } from "@opentelemetry/api";

const tracer = trace.getTracer("support-agent");

export async function runAgent(sessionId: string, userHash: string, messages: Msg[]) {
  return tracer.startActiveSpan("invoke_agent support-agent", async (run) => {
    run.setAttributes({
      "gen_ai.operation.name": "invoke_agent",
      "gen_ai.agent.name": "support-agent",
      "gen_ai.conversation.id": sessionId,
      "app.user_hash": userHash,
      "app.feature": "support-chat",
    });
    try {
      for (let step = 1; step <= MAX_STEPS; step++) {
        const resp = await callModel(messages);
        const calls = resp.choices[0].message.tool_calls ?? [];
        if (calls.length === 0) {
          run.setAttributes({ "app.exit_reason": "final_answer", "app.step_count": step });
          return resp;
        }
        for (const call of calls) {
          await tracer.startActiveSpan(`execute_tool ${call.function.name}`, async (t) => {
            t.setAttributes({
              "gen_ai.operation.name": "execute_tool",
              "gen_ai.tool.name": call.function.name,
              "gen_ai.tool.call.id": call.id,
            });
            try {
              const result = await TOOLS[call.function.name](call.function.arguments);
              messages.push(toolMessage(call.id, result));
            } catch (e) {
              t.recordException(e as Error);
              t.setStatus({ code: SpanStatusCode.ERROR });
              messages.push(toolMessage(call.id, { error: String(e) }));
            } finally {
              t.end();
            }
          });
        }
      }
      run.setAttributes({ "app.exit_reason": "max_steps", "app.step_count": MAX_STEPS });
    } finally {
      run.end();
    }
  });
}
```

## Measuring TTFT in app code

Gateways measure TTFT at their edge. To measure what the end user sees (including your server and network), time it in the client path too.

```python
import time

t0 = time.perf_counter()
ttft_ms = None
usage = None
stream = client.chat.completions.create(model=MODEL, messages=messages, stream=True,
                                        stream_options={"include_usage": True})
for chunk in stream:
    if ttft_ms is None and chunk.choices and chunk.choices[0].delta.content:
        ttft_ms = (time.perf_counter() - t0) * 1000
    if chunk.usage:
        usage = chunk.usage                      # final chunk carries usage
total_ms = (time.perf_counter() - t0) * 1000
```

```ts
const t0 = performance.now();
let ttftMs: number | undefined;
let usage: OpenAI.CompletionUsage | undefined;
const stream = await client.chat.completions.create({
  model: MODEL, messages, stream: true, stream_options: { include_usage: true },
});
for await (const chunk of stream) {
  if (ttftMs === undefined && chunk.choices[0]?.delta?.content) ttftMs = performance.now() - t0;
  if (chunk.usage) usage = chunk.usage;
}
const totalMs = performance.now() - t0;
```

Note: with tool calls or reasoning models, the first chunk may be a tool-call delta or reasoning; decide whether TTFT means "first visible token" or "first byte" and record it consistently.

## PII redaction

Order of preference:

1. **Do not capture** content for regulated routes (per-route or per-request content opt-out). Metadata is still recorded.
2. **Redact at the edge** (in-process or at the gateway) before content reaches any sink.
3. **Restrict access** to content in the backend and keep retention short.

A regex redactor is a floor, not a guarantee. It catches structured identifiers (emails, phones, cards, keys) and misses names and addresses; use an NER-based detector (for example Microsoft Presidio, self-hosted) for free text.

```python
import re, hashlib

PATTERNS = [
    ("EMAIL", re.compile(r"[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}")),
    ("API_KEY", re.compile(r"\b(sk-[A-Za-z0-9_-]{20,}|sk-ant-[A-Za-z0-9_-]{20,}|AKIA[0-9A-Z]{16}|AIza[0-9A-Za-z_-]{35})\b")),
    ("CARD", re.compile(r"\b(?:\d[ -]?){13,19}\b")),
    ("PHONE", re.compile(r"\+?\d[\d\s().-]{8,}\d")),
    ("IPV4", re.compile(r"\b(?:\d{1,3}\.){3}\d{1,3}\b")),
]

def redact(text: str) -> str:
    for label, rx in PATTERNS:
        text = rx.sub(f"[{label}]", text)
    return text

def hash_id(user_id: str, salt: str) -> str:
    return hashlib.sha256((salt + user_id).encode()).hexdigest()[:16]
```

```ts
import { createHash } from "node:crypto";

const PATTERNS: [string, RegExp][] = [
  ["EMAIL", /[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}/g],
  ["API_KEY", /\b(sk-[A-Za-z0-9_-]{20,}|sk-ant-[A-Za-z0-9_-]{20,}|AKIA[0-9A-Z]{16}|AIza[0-9A-Za-z_-]{35})\b/g],
  ["CARD", /\b(?:\d[ -]?){13,19}\b/g],
  ["PHONE", /\+?\d[\d\s().-]{8,}\d/g],
  ["IPV4", /\b(?:\d{1,3}\.){3}\d{1,3}\b/g],
];

export const redact = (s: string) => PATTERNS.reduce((t, [label, rx]) => t.replace(rx, `[${label}]`), s);
export const hashId = (id: string, salt: string) => createHash("sha256").update(salt + id).digest("hex").slice(0, 16);
```

Redaction caveats:
- Redact both the prompt and the output; models echo input.
- Redact tool arguments and tool results; they often carry the most PII (order lookups, CRM records).
- Keep the redaction function versioned and test it with fixtures.
- Header-captured values and dimension values are usually exported regardless of content settings. Never put PII in dimension headers.

## Sampling policy

| Data | Rate | Notes |
|---|---|---|
| Metrics | 100% | Aggregates only. |
| Metadata record (no content) | 100% | Needed for cost attribution and feedback lookup. |
| Content + full trace, normal traffic | 1 to 10% | Per feature; raise for new features or after a prompt change. |
| Errors, fallbacks, retries > 0 | 100% | |
| Latency above p95 for its feature | 100% | |
| Negative feedback | 100% | Retained after the fact: requires the content to still exist, so keep a short raw buffer (hours to days) before sampling it away, or retain content for all requests briefly. |
| Regulated routes | 0% content | Metadata only. |

With an OpenTelemetry Collector, use the `tail_sampling` processor with `status_code` (ERROR), `latency` (threshold), and `probabilistic` policies, and route by attribute (for example `app.feature`).

## Linking feedback to traces

1. Generate `request_id` (UUID) in your backend before the LLM call. Send it upstream (for example as `x-request-id`) so the gateway record and your record share it.
2. Return `request_id` to the client with the response; store it with the message in UI state.
3. Feedback endpoint writes `{request_id, user_hash, kind: thumbs_up|thumbs_down|edit|regenerate|copy, comment, ts}`.
4. Join feedback to records and traces by `request_id`. Add the feedback as a span event or attribute in your tracing backend if it supports late writes; otherwise join in the warehouse.
5. Negative feedback triggers: retain full trace, add to review queue, add to eval candidate set.

Implicit signals are often more plentiful than thumbs: regenerate clicks, copy events, the user rephrasing within 30 seconds, conversation abandonment right after an answer.

## Evals on production samples

Daily (or hourly) job:

1. **Sample**: pull yesterday's successful records, stratified by `feature` and `model`; include all negative-feedback and fallback records; cap per stratum.
2. **Score** with cheap deterministic checks first (schema/JSON validity, length bounds, required citations present, forbidden phrases), then an LLM judge with a task-specific rubric returning a score and a reason. Use a different, stronger model than the one under test where possible.
3. **Store** scores keyed by `request_id`, with judge model, rubric version, and timestamp.
4. **Track** mean score and failure rate per feature and per `prompt_version` as time series; alert on drops.
5. **Promote** failures into the offline regression dataset after human review.

Self-hosted version against a Bifrost log store (`GET /api/logs`). Inspect one log first (`curl -s 'http://localhost:8080/api/logs?limit=1' | jq '.logs[0] | keys'`) and adjust the field names in `extract()` to match your version.

```python
import json, random, datetime as dt, requests
from openai import OpenAI

BIFROST = "http://localhost:8080"
judge = OpenAI(base_url=f"{BIFROST}/openai", api_key="sk-bf-...")   # judge also goes through the gateway
JUDGE_MODEL = "gpt-4o"                                               # use the project's chosen judge model

RUBRIC = """Score the assistant answer from 1-5 for: correct, grounded in the provided context,
follows instructions, no unsafe content. Return JSON {"score": int, "reason": str}."""

def fetch(start, end, limit=500):
    r = requests.get(f"{BIFROST}/api/logs", params={
        "status": "success", "objects": "chat.completion",
        "start_time": start, "end_time": end, "limit": limit, "offset": 0,
    }, timeout=30)
    r.raise_for_status()
    return r.json()["logs"]

def extract(log):
    # Field names: verify against `jq '.logs[0] | keys'`
    return log.get("id"), log.get("input_history"), log.get("output_message")

def score(inp, out):
    r = judge.chat.completions.create(
        model=JUDGE_MODEL, response_format={"type": "json_object"},
        messages=[{"role": "system", "content": RUBRIC},
                  {"role": "user", "content": f"CONVERSATION:\n{json.dumps(inp)[:12000]}\n\nANSWER:\n{json.dumps(out)[:4000]}"}],
        extra_headers={"x-bf-dim-feature": "eval-judge"},   # keep judge cost attributable and separate
    )
    return json.loads(r.choices[0].message.content)

end = dt.datetime.now(dt.timezone.utc).replace(microsecond=0)
start = end - dt.timedelta(days=1)
logs = fetch(start.isoformat().replace("+00:00", "Z"), end.isoformat().replace("+00:00", "Z"))
for log in random.sample(logs, min(100, len(logs))):
    rid, inp, out = extract(log)
    if not inp or not out:
        continue                                    # content logging disabled for this request
    s = score(inp, out)
    print(json.dumps({"request_id": rid, **s}))     # write to your eval store instead
```

```ts
import OpenAI from "openai";

const BIFROST = "http://localhost:8080";
const judge = new OpenAI({ baseURL: `${BIFROST}/openai`, apiKey: process.env.BIFROST_VK ?? "sk-bf-..." });
const JUDGE_MODEL = "gpt-4o"; // use the project's chosen judge model
const RUBRIC = `Score the assistant answer from 1-5 for: correct, grounded, follows instructions, safe.
Return JSON {"score": number, "reason": string}.`;

const end = new Date();
const start = new Date(end.getTime() - 24 * 3600 * 1000);
const qs = new URLSearchParams({
  status: "success", objects: "chat.completion",
  start_time: start.toISOString(), end_time: end.toISOString(), limit: "500", offset: "0",
});
const { logs } = await (await fetch(`${BIFROST}/api/logs?${qs}`)).json();

const sample = logs.sort(() => Math.random() - 0.5).slice(0, 100);
for (const log of sample) {
  const [id, input, output] = [log.id, log.input_history, log.output_message]; // verify field names first
  if (!input || !output) continue;
  const r = await judge.chat.completions.create(
    {
      model: JUDGE_MODEL,
      response_format: { type: "json_object" },
      messages: [
        { role: "system", content: RUBRIC },
        { role: "user", content: `CONVERSATION:\n${JSON.stringify(input).slice(0, 12000)}\n\nANSWER:\n${JSON.stringify(output).slice(0, 4000)}` },
      ],
    },
    { headers: { "x-bf-dim-feature": "eval-judge" } },
  );
  console.log(JSON.stringify({ request_id: id, ...JSON.parse(r.choices[0].message.content ?? "{}") }));
}
```

Judge hygiene:
- Calibrate the judge on 50 to 100 human-labeled examples before trusting it; report agreement.
- Version the rubric; a rubric change resets the baseline.
- Do not let the judge see the feature's own system prompt if it would bias toward instruction-following over correctness.
- Exclude the judge's own traffic from product dashboards (tag it with a dedicated feature dimension).
