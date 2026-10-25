# llm-observability-stack

[![skills.sh](https://skills.sh/b/build-with-ai-hq/llm-observability-stack)](https://skills.sh/build-with-ai-hq/llm-observability-stack)

An Agent Skill that sets up production observability for LLM applications and agents: per-call records, traces, metrics, dashboards, alerts, PII controls, feedback linkage, and evals on production traffic.

## Problems this solves

- A user reports a bad answer and nobody can find the exact prompt, model, and output.
- Latency spiked and you cannot tell whether it is time-to-first-token, total generation time, the provider, or your code.
- Provider errors, 429s, retries, and fallbacks happen silently.
- Token and cost spend jumped with no attribution to a user, team, feature, or prompt version.
- A multi-step agent run is scattered across unrelated logs instead of one trace.
- Prompts and completions containing PII are written to plain logs and exported to third-party backends.
- Every service and language instruments SDK calls differently, or not at all.
- Nothing scores production traffic; quality regressions are found by users.

## Install

```bash
npx skills add build-with-ai-hq/llm-observability-stack
```

## What the skill does

When loaded, the agent:

1. Audits the codebase with grep commands for LLM call sites, existing instrumentation, prompts in logs, streams missing usage, swallowed errors, and missing user/session/feature attribution.
2. Applies fixes in order of impact: a single capture point, a fixed per-call record, TTFT vs total latency, error classification and fallback visibility, cost attribution with safe cardinality, agent trace structure using OpenTelemetry GenAI conventions, PII redaction, sampling, feedback linkage, and production evals.
3. Sets up gateway-level capture with Bifrost (open-source AI gateway) where it removes per-SDK instrumentation: request logs and log API, Prometheus metrics, OpenTelemetry export to any collector, per-request dimensions, content-logging controls, and correlation response headers.
4. Builds dashboards and alert rules with example PromQL, and verifies the setup end to end.

Code examples are in Python and TypeScript; gateway config is JSON; checks are curl.

## Contents

```
llm-observability-stack/
  SKILL.md                          # workflow: diagnose, core practices, Bifrost setup, verify, common mistakes
  references/
    what-to-capture.md              # per-call record, cardinality, OTel GenAI conventions, agent traces, TTFT, PII redaction, sampling, feedback, production evals
    bifrost-telemetry.md            # logs and log store, content logging, dimensions, Prometheus, OTel, response headers
    dashboards-and-alerts.md        # PromQL panels, alert rules, thresholds, runbook
  README.md
  LICENSE
```

## Links

- Bifrost: https://github.com/maximhq/bifrost (docs: https://docs.getbifrost.ai)
- OpenTelemetry GenAI semantic conventions: https://opentelemetry.io/docs/specs/semconv/gen-ai/

## License

MIT. See [LICENSE](LICENSE).
