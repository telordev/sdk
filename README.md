# Simse SDKs

Official client libraries for the [Simse API](https://api.simse.dev) — the
Anthropic-wire-compatible model API served by `simse-warp`.

| Language    | Package            | Directory                      | Install (when published)            |
|-------------|--------------------|--------------------------------|-------------------------------------|
| TypeScript  | `@telordev/simse`  | [`typescript/`](./typescript)  | `npm i @telordev/simse`             |
| Python      | `simse`            | [`python/`](./python)          | `pip install simse`                 |
| Rust        | `simse`            | [`rust/`](./rust)              | `cargo add simse`                   |

All three are modelled on the official Anthropic SDKs and target the same wire
contract, so the same request/response shapes and streaming semantics hold
across languages.

## Contract

- **[`CONTRACT.md`](./CONTRACT.md)** — the canonical, human-readable API contract
  (the single source of truth the SDKs and the gateway implement).
- **[`openapi.yaml`](./openapi.yaml)** — OpenAPI 3.1 spec for the surface.

## Authentication

All clients authenticate with a platform API key (`sk_…`). The key is sent as
both `x-api-key` (Anthropic-style) and `Authorization: Bearer`. Default base URL
is `https://api.simse.dev`. The key is read from `SIMSE_API_KEY` (then
`ANTHROPIC_API_KEY`) and the base URL from `SIMSE_BASE_URL` when not passed
explicitly.

## Quickstart

```ts
// TypeScript
import { Simse } from "@telordev/simse";
const client = new Simse({ apiKey: process.env.SIMSE_API_KEY });
const msg = await client.messages.create({
  model: "zoysia",
  max_tokens: 1024,
  messages: [{ role: "user", content: "Hello, Simse" }],
});
console.log(msg.content);
```

```python
# Python
from simse import Simse
client = Simse()  # reads SIMSE_API_KEY
msg = client.messages.create(
    model="zoysia",
    max_tokens=1024,
    messages=[{"role": "user", "content": "Hello, Simse"}],
)
print(msg.content)
```

```rust
// Rust
use simse::{Client, CreateMessage, Message};
let client = Client::new(std::env::var("SIMSE_API_KEY")?);
let msg: Message = client.messages().create(
    CreateMessage::new("zoysia", 1024)
        .user("Hello, Simse"),
).await?;
println!("{:?}", msg.content);
```

## Models

`rye` (Qwen3.5-4B) · `zoysia` (Qwen3.5-9B, default). List them with
`client.models.list()`.

## Surface

Each SDK covers the full platform surface: `messages` (create / stream /
count_tokens), `models`, `account` / `usage` / `billing`, `sessions` (agentic
prompt loop with streaming), `memories`, `plugins`, project-management
(`tasks` / `projects` / `todos` / `schedules` / `workflows`), and `flags`.

See each package's own `README.md` for language-specific usage, streaming
helpers, error handling, and examples.
