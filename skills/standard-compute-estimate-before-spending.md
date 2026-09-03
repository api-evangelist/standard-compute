---
name: standard-compute-estimate-before-spending
description: Size a request with token counting before spending flat-rate compute budget on it, and read model capabilities before sending unsupported parameters.
api: Standard Compute Inference API
operations:
  - anthropic_count_tokens_v1_messages_count_tokens_post
  - list_models_v1_models_get
  - anthropic_messages_v1_messages_post
generated: '2026-09-02'
method: generated
source: >-
  operationIds read from openapi/standard-compute-openapi.json; budget and pacing
  semantics from https://standardcompute.com/fair-use and
  https://standardcompute.com/smart-pacing.
---

# Estimate before you spend

On a per-token API an oversized request costs money you can see itemised. On Standard
Compute it silently eats a fixed monthly budget, and when that budget is gone every
request returns `402` until the period renews. Budget consumption **cannot be
reversed** — there is no cancel, no void and no per-request refund. The only
rehearsal available is token counting, so use it.

## 1. Count tokens first

`POST /v1/messages/count_tokens`
(`anthropic_count_tokens_v1_messages_count_tokens_post`) sizes an Anthropic-format
request without running a completion. This is the closest thing this API has to a
dry-run mode, and it exists only on the Anthropic-compatible path — there is no
OpenAI-path equivalent.

```
curl https://api.stdcmpt.com/v1/messages/count_tokens \
  -H "Authorization: Bearer $STANDARD_COMPUTE_KEY" \
  -H "Content-Type: application/json" \
  -d '{"messages": [{"role":"user","content":"..."}]}'
```

Use it before a large-context call — a codebase dump, a long transcript, a batch
refactor — not before every small one.

## 2. Read capabilities before sending parameters

`GET /v1/models` (`list_models_v1_models_get`) returns `supported_parameters` per
entry. Observed values include `tools`, `tool_choice`, `response_format`, `reasoning`
and `reasoning_effort`, and they are **not uniform across the pool** — some entries
list only `tools` and `tool_choice`. Check the array before sending a parameter
rather than discovering the gap in a failed run. `context_length` on the same record
tells you what will fit.

## 3. Then send the request

`POST /v1/messages` (`anthropic_messages_v1_messages_post`) for the Anthropic wire
format, or `POST /v1/chat/completions` for the OpenAI one.

## 4. Operating rules for a long-running agent

- **Retries cost budget.** There is no idempotency key on this API, so a retried
  completion is a second charge against the budget, not a deduplicated request. Cap
  retries deliberately.
- **A 402 is terminal for the period.** Treat it as a stop condition, not a transient
  error. Retrying it burns wall-clock and clears nothing.
- **There is no in-band budget signal.** No `X-RateLimit-*`, no `RateLimit-*`, no
  `Retry-After`. If your agent needs to know how much budget is left, the only source
  is a human looking at the dashboard.
- **Nothing is logged on the provider side.** Prompt and response content is not
  stored, so if you need the payload of a failed run for debugging, capture it
  yourself before it leaves your process.
