---
name: standard-compute-switch-agent-provider
description: Repoint an OpenAI-compatible or Anthropic-compatible agent at Standard Compute's flat-rate gateway and verify the switch worked.
api: Standard Compute Inference API
operations:
  - list_models_v1_models_get
  - chat_completions_v1_chat_completions_post
generated: '2026-09-02'
method: generated
source: >-
  https://standardcompute.com/getting-started and
  https://standardcompute.com/integrations/claude-code, grounded in operationIds
  read from openapi/standard-compute-openapi.json.
---

# Switch an agent onto Standard Compute

Standard Compute replaces a per-token model provider with a flat monthly budget. The
migration is a base-URL swap — there is no SDK to install, because the gateway speaks
the OpenAI and Anthropic wire formats directly.

## 1. Confirm the gateway is up before changing anything

`GET /v1/models` (`list_models_v1_models_get`) is the **only** unauthenticated
operation on this API, and the provider documents it as the self-service health
check.

```
curl -i https://api.stdcmpt.com/v1/models
```

A `200` returns the routing pool. Read `supported_parameters` on the entries before
assuming tool calling is available — that array is this API's only capability
discovery surface.

## 2. Pick the wire format the agent already speaks

**OpenAI-compatible agents** (Cursor, Cline, Aider, Continue, OpenCode, Kilo Code,
Roo Code, Zed, Codex CLI):

```
OPENAI_BASE_URL=https://api.stdcmpt.com/v1
OPENAI_API_KEY=<your sc_live_ key>
OPENAI_MODEL=standardcompute
```

**Anthropic-compatible agents** (Claude Code):

```
ANTHROPIC_BASE_URL=https://api.stdcmpt.com
ANTHROPIC_API_KEY=<your sc_live_ key>
```

The Anthropic base URL has **no `/v1` suffix** — the client appends `/v1/messages`
itself. Setting `https://api.stdcmpt.com/v1` here produces requests to
`/v1/v1/messages` and they will not resolve. There is also no model variable on this
path; the family tier is selected server-side.

## 3. Send one real request

`POST /v1/chat/completions` (`chat_completions_v1_chat_completions_post`):

```
curl https://api.stdcmpt.com/v1/chat/completions \
  -H "Authorization: Bearer $STANDARD_COMPUTE_KEY" \
  -H "Content-Type: application/json" \
  -d '{"model": "standardcompute", "messages": [{"role":"user","content":"ping"}]}'
```

Send the literal string `standardcompute` as the model. The router picks the
underlying model per request; you do not pin one, and you cannot.

## 4. Handle the two failures that actually occur

- **401** — `{"error": {"message": "Invalid API key", "type": "invalid_request_error"}}`.
  The key is missing, malformed or revoked. Keys are shown in plaintext once at
  creation and cannot be recovered; issue a new one.
- **402** — the plan's monthly compute budget is spent. Requests resume when the
  billing period renews, or immediately on upgrade. **Do not retry a 402 in a loop**;
  it will not clear on its own within the period.

Do **not** port OpenAI 429 backoff logic here expecting it to fire. This gateway does
not return per-minute 429s; under sustained load ahead of budget it paces requests —
they take longer and still succeed.

## 5. Know what you cannot see

There is no budget or usage endpoint and no rate-limit response header. A running
agent gets **no in-band warning** before a 402. If budget exhaustion would be
disruptive, enable optional smart pacing in the dashboard first — it spreads the
remaining budget across the month instead of letting a burst drain it.
