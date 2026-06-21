# Simse Public API — Canonical Contract (v1)

This is the **single source of truth** for the `api.simse.dev` public REST API and
for the TypeScript / Python / Rust SDKs. It is modelled on the Anthropic Messages
API: the flagship surface (`POST /v1/messages`) is **wire-compatible** with
Anthropic's Messages API, and the SDKs mirror the official Anthropic SDK
ergonomics (`client.messages.create` / `.stream` / `.count_tokens`,
`client.models.list` / `.retrieve`, typed errors, retries, pagination).

Gateway implementation: `simse-warp` (`warp/` crate). Backend models: `rye`
(Qwen3.5-4B) and `zoysia` (Qwen3.5-9B, default).

## Scope boundary — what is (and isn't) the public API

The SDK covers the **public, key-authenticated `/v1` developer API** ONLY:
`messages`, `models`, `count_tokens`, `sessions`, `memories`, `plugins`, `pm`,
`agents`, `usage` (read), `flags`, `account`/`billing` (read). These are the
endpoints below.

The following are **deliberately NOT in the SDK** — they are not part of the
public API:
- **Auth** (Google SSO / magic-link login / token refresh) — the public API is
  authenticated by `sk_` **keys**, not the consumer login flow. There are no
  `/auth/*` endpoints on the public API surface.
- **Payments / billing management** (`/payments/*`) — consumer/console concern.
- **Remote rendezvous** (`/v1/remote/*`) — the `simse-cli` ↔ web-dashboard
  internal transport, not a developer capability.

First-party clients (the web app, the CLI) use the SDK for public-API access and
talk to auth/payments/remote surfaces directly.

---

## 1. Connection & auth

- **Base URL:** `https://api.simse.dev`
- **Auth:** an `sk_…` platform API key, sent as either:
  - `x-api-key: sk_…`  (Anthropic-style — preferred for the Messages API), **or**
  - `Authorization: Bearer sk_…`
- **Version header:** `anthropic-version: 2026-06-01` (also accepts `2023-06-01`).
  Optional; the gateway defaults to the latest when absent.
- **Content type:** `application/json` on request bodies.
- `anthropic-beta` is accepted and ignored (no beta gating today).

Every response carries:
- `request-id: req_…` — unique per request; echo it in bug reports.
- `anthropic-ratelimit-requests-limit` / `-remaining` / `-reset` — the per-client
  request budget (`-reset` is an RFC 3339 UTC timestamp).
- On `429`: `retry-after: <seconds>`.

---

## 2. Errors

Errors for the Messages/Models surfaces use the **Anthropic error envelope**:

```json
{
  "type": "error",
  "error": { "type": "invalid_request_error", "message": "..." },
  "request_id": "req_..."
}
```

| HTTP | `error.type`            | Meaning                                  |
|------|-------------------------|------------------------------------------|
| 400  | `invalid_request_error` | Malformed/invalid request.               |
| 401  | `authentication_error`  | Missing/invalid API key.                 |
| 403  | `permission_error`      | Key lacks permission.                    |
| 404  | `not_found_error`       | Resource not found.                      |
| 409  | `api_error`             | Conflict (e.g. concurrent update). **Retried** by the SDKs (see below). |
| 413  | `request_too_large`     | Body exceeds the size limit.             |
| 422  | `api_error`             | Unprocessable entity (semantically invalid). Not retried. |
| 429  | `rate_limit_error`      | Rate limit hit — see `retry-after`.      |
| 500  | `api_error`             | Internal error.                          |
| 502  | `api_error`             | Upstream/gateway failure (a non-`Unavailable` downstream error maps to `502 bad_gateway`). Retryable as ≥500. |
| 503  | `overloaded_error`      | Backend unavailable/overloaded (retry).  |

The platform/dashboard surfaces (`/v1/sessions`, `/v1/memories`, `/v1/plugins`,
`/v1/usage`, PM) use the legacy shape `{"error":{"code":"...","message":"..."}}`
with the same HTTP statuses. **SDKs must parse both shapes** (prefer `error.type`,
fall back to `error.code`).

**SDK error classes** (mirror Anthropic): `APIError` (base, has `.status`,
`.requestId`, `.type`) → `BadRequestError` (400), `AuthenticationError` (401),
`PermissionDeniedError` (403), `NotFoundError` (404), `ConflictError` (409),
`RequestTooLargeError` (413), `UnprocessableEntityError` (422),
`RateLimitError` (429), `InternalServerError` (500), `OverloadedError`
(503), plus `APIConnectionError` / `APITimeoutError` for transport failures.

**Retries:** retry on `408`, `409`, `429`, and `≥500` with exponential backoff
(default `max_retries = 2`; honor `retry-after`). `409` (conflict) is retried;
`422` (unprocessable entity) is not. Do not retry other 4xx.

---

## 3. Messages API (flagship — Anthropic wire-compatible)

### `POST /v1/messages`

**Request body:**

| Field            | Type                              | Req | Notes |
|------------------|-----------------------------------|-----|-------|
| `model`          | string                            | ✅  | `"rye"` or `"zoysia"`. |
| `messages`       | `Message[]`                       | ✅  | Non-empty. Each `{role:"user"\|"assistant", content: string \| ContentBlock[]}`. |
| `max_tokens`     | integer > 0                       | ✅  | Max tokens to generate. |
| `system`         | string \| `TextBlock[]`           |     | System prompt. |
| `temperature`    | number 0..1                       |     | |
| `top_p`          | number 0..1                       |     | |
| `top_k`          | integer                           |     | |
| `stop_sequences` | string[]                          |     | |
| `stream`         | boolean                           |     | Default false. |
| `tools`          | `Tool[]`                          |     | `{name, description?, input_schema}`. |
| `tool_choice`    | `{type:"auto"\|"any"\|"tool"\|"none", name?}` | | |
| `metadata`       | `{user_id?: string}`              |     | Accepted; opaque. |

**Input content blocks:**
- `{"type":"text","text":"..."}`
- `{"type":"image","source":{"type":"base64","media_type":"image/png","data":"<b64>"}}`
- `{"type":"tool_use","id":"toolu_...","name":"...","input":{...}}` (echo assistant turn)
- `{"type":"tool_result","tool_use_id":"toolu_...","content": string | Block[], "is_error"?: bool}`
- `{"type":"thinking","thinking":"..."}` (echo a prior assistant thinking turn; the
  gateway accepts it and echoes the text back into the prompt context). Note: a
  `{"type":"redacted_thinking",...}` block is currently dropped (ignored).

**Response — `Message` object (non-stream):**

```json
{
  "id": "msg_...",
  "type": "message",
  "role": "assistant",
  "model": "zoysia",
  "content": [
    {"type": "text", "text": "..."},
    {"type": "tool_use", "id": "toolu_...", "name": "get_weather", "input": {"city":"SF"}}
  ],
  "stop_reason": "end_turn",
  "stop_sequence": null,
  "usage": {"input_tokens": 12, "output_tokens": 34}
}
```

`stop_reason` ∈ `end_turn` | `max_tokens` | `stop_sequence` | `tool_use`.

**Streaming (`stream:true`)** — Server-Sent Events, each
`event: <type>\ndata: <json>\n\n`, in order:

1. `message_start` — `{"type":"message_start","message":{...Message with empty content, usage.input_tokens}}`
2. `ping` — `{"type":"ping"}`
3. `content_block_start` — `{"type":"content_block_start","index":0,"content_block":{"type":"text","text":""}}`
4. `content_block_delta` (repeated) — `{"type":"content_block_delta","index":0,"delta":{"type":"text_delta","text":"..."}}`
   - tool args stream as `{"type":"input_json_delta","partial_json":"..."}` on a `tool_use` block.
5. `content_block_stop` — `{"type":"content_block_stop","index":0}`
6. `message_delta` — `{"type":"message_delta","delta":{"stop_reason":"end_turn","stop_sequence":null},"usage":{"output_tokens":34}}`
7. `message_stop` — `{"type":"message_stop"}`
8. mid-stream errors: `event: error\ndata: {"type":"error","error":{"type":"...","message":"..."}}`

The `ping` is emitted immediately after `message_start`, before the first
`content_block_start` (matching the Anthropic wire and warp's actual emission).

`message_start.usage.input_tokens` is an estimate; `message_delta.usage.output_tokens`
is authoritative. The SDK streaming helper **accumulates** events into a final
`Message` (text from `text_delta`, tool input from `input_json_delta` fragments).

### `POST /v1/messages/count_tokens`

Same body minus `max_tokens` / `stream`. Returns `{"input_tokens": N}` (an
estimate — there is no dedicated tokenizer endpoint yet).

---

## 4. Models API (Anthropic-shaped)

### `GET /v1/models`
Query: `limit` (default 20), `before_id`, `after_id` (cursor pagination — the
hosted catalog is small, so `has_more` is currently always `false`).

```json
{
  "data": [
    {"id":"zoysia","type":"model","display_name":"Zoysia (Qwen3.5 9B)","created_at":"2026-01-01T00:00:00Z","max_input_tokens":131072,"max_tokens":8192},
    {"id":"rye","type":"model","display_name":"Rye (Qwen3.5 4B)","created_at":"2026-01-01T00:00:00Z","max_input_tokens":131072,"max_tokens":8192}
  ],
  "has_more": false,
  "first_id": "zoysia",
  "last_id": "rye"
}
```
(The same object also carries legacy `models` / `acp_providers` fields for the
console — SDKs read `data`.)

### `GET /v1/models/{model_id}`
Returns a single `Model` object, or `404 not_found_error`.

---

## 5. Account / usage / billing

- `GET /v1/account` → `{"id":"pa_...","user_id":"user_...","plan":"free"}`
- `GET /v1/usage` → `{"period":"current","requests":N,"tokens":N,"by_model":{"zoysia":N}}`
- `GET /v1/usage/dashboard` → rich per-model view: `{plan, periodStart, models:[{model, includedInputTokens, includedOutputTokens, extraInputTokens, extraOutputTokens, extraSpendCents, requestCount, multiplier}], billing:{extraUsageEnabled, extraUsageCapCents, creditsBalanceCents, extraSpendThisPeriodCents, planIncludedTokens}, compute?}`
- `GET /v1/billing` → `{"plan":"free","status":"active","limits":{...},"current_usage":{...}}`
- `GET /v1/agents` → subagent run history: `{"agents":[{id, description, status, started_at, completed_at, duration_ms, turns, input_tokens, output_tokens, error}]}`

---

## 6. Sessions API (agentic prompt loop)

A session is a persisted, model-driven conversation (the agent reaches tools via
the orchestrator). Distinct from the stateless Messages API.

- `POST /v1/sessions` `{model?, title?}` → `{"id","model","status":"active","created_at"}`
- `GET /v1/sessions` → `{"sessions":[{"id","title","message_count","created_at","updated_at","status"}]}`
- `GET /v1/sessions/{id}` → session object
- `DELETE /v1/sessions/{id}` → `{}`
- `POST /v1/sessions/{id}/messages` `{content, stream?}`
  - non-stream → `{"message":{"role":"assistant","content":"..."},"usage":{"input_tokens","output_tokens"}}`
  - stream → SSE lines `data: {json}\n\n` (`[DONE]` terminal) with events:
    `{"type":"delta","delta":"..."}`, `{"type":"tool_call","id","name","input"}`,
    `{"type":"tool_result","id","output","is_error"}`,
    `{"type":"done","status","text","usage":{"input_tokens","output_tokens"}}`, `{"type":"error","message"}`

  Usage is Anthropic-style `input_tokens`/`output_tokens` (no `total_tokens` — compute client-side). The OpenAI-shaped `/v1/chat/completions` endpoint does **not** exist; the only raw-inference surface is the Anthropic `/v1/messages` above.
- `POST /v1/sessions/{id}/resume` → resumes with history
- `POST /v1/sessions/{id}/abort` → cancels an in-flight prompt

---

## 7. Memories API

- `GET /v1/memories` → `{"memories":[...]}`
- `POST /v1/memories` `{text, metadata?}` → created memory
- `DELETE /v1/memories/{id}` → `{}`
- `GET /v1/memories/stats` → stats object

(User-scoped; the gateway forces the scope to the authenticated key's user.)

---

## 8. Plugins / marketplace API

- `GET /v1/plugins` → installed tools + plugins
- `GET /v1/plugins/registry` → marketplace listing
- `GET /v1/plugins/registry/{id}` → one registry entry
- `GET /v1/plugins/installed` → installed plugins
- `POST /v1/plugins/install` `{plugin_name, ...}` → result
- `POST /v1/plugins/uninstall` `{plugin_name}` → result

---

## 9. Project-management API (session-scoped)

- Tasks: `GET/POST /v1/sessions/{id}/tasks`, `GET/PATCH/DELETE /v1/sessions/{id}/tasks/{task_id}`,
  `POST .../move`, `PUT .../checklist`, `POST .../deps`, `DELETE .../deps/{blocks_task_id}`
- Projects: `GET/POST /v1/sessions/{id}/projects`, `PATCH/DELETE /v1/sessions/{id}/projects/{project_id}`
- Todos (read-only): `GET /v1/sessions/{id}/todos`
- Schedules: `GET/POST /v1/schedules`, `PATCH/DELETE /v1/schedules/{task_id}`
- Workflows: `GET/POST /v1/workflows`, `GET/PATCH/DELETE /v1/workflows/{workflow_id}`,
  `POST /v1/workflows/lint`, `POST /v1/workflows/{workflow_id}/run`,
  `GET /v1/workflows/runs/{run_id}`, `POST /v1/workflows/runs/{run_id}/cancel`

---

## 10. Feature flags

- `GET /v1/flags` → `{"flags":{"v1.chat":true, ...}}` (which features the key may use)

---

## SDK design requirements (all three languages)

1. **Client construction** with `api_key` (default from `SIMSE_API_KEY` /
   `ANTHROPIC_API_KEY` env), `base_url` (default `https://api.simse.dev`,
   overridable via `SIMSE_BASE_URL`), `timeout`, `max_retries`, `default_headers`.
2. **Resource namespaces:** `messages` (`.create`, `.stream`, `.count_tokens`),
   `models` (`.list`, `.retrieve`), `account`, `usage`, `billing`, `sessions`
   (`.create`, `.list`, `.retrieve`, `.delete`, `.prompt`/`.stream`, `.resume`,
   `.abort`), `memories`, `plugins`, `pm` (tasks/projects/todos/schedules/
   workflows), `flags`.
3. **Streaming helpers:** `messages.stream(...)` yields typed events and
   accumulates a final `Message` (expose `.text_stream` / `.on("text")` /
   `finalMessage()` idioms per language).
4. **Typed models** for requests/responses + content blocks (discriminated union
   on `type`).
5. **Typed error hierarchy** + automatic retry/backoff honoring `retry-after`.
6. **Pagination** helper for `models.list` (returns the `data` array; the cursor
   fields are exposed).
7. **README** with quickstart (sync + streaming + tools), **examples/**, and a
   **test suite** that runs offline (mock the HTTP layer — do not hit the live
   API). Package metadata ready to publish (do NOT publish):
   - TS: `package.json` (`@telordev/simse`), `tsconfig.json`, `tsup`/`vitest`, ESM+CJS, `.d.ts`.
   - Python: `pyproject.toml` (`simse`), sync `Simse` + `AsyncSimse`, `httpx`, `pytest`, type hints + `py.typed`.
   - Rust: `Cargo.toml` (`simse`), `reqwest` + `tokio`, `serde`, `thiserror`, streaming via `futures::Stream`, `cargo test`.
8. Idiomatic naming per language (camelCase TS, snake_case Python/Rust). JSON wire
   names are as above (snake_case).
