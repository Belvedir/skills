---
name: belvedir
description: Instrument an AI agent with Belvedir observability: install the Belvedir SDK (Node or Python), wrap runs in sessions and tasks, flush spans, and report outcomes. Use when connecting an app to Belvedir or debugging missing traces, tasks, or training sets.
---

# Belvedir integration

> This document lives at https://platform.belvedir.ai/agents.md and in the https://github.com/Belvedir/skills repo. It is written for coding agents: everything needed to instrument an app with Belvedir, in one page. It also works as a skill file: install it into the codebase as `.claude/skills/belvedir/SKILL.md` (or your agent's equivalent) to keep it available offline:
>
> ```bash
> mkdir -p .claude/skills/belvedir && curl -fsSL https://raw.githubusercontent.com/Belvedir/skills/main/belvedir/SKILL.md -o .claude/skills/belvedir/SKILL.md
> ```

Belvedir is an observability and self-improvement platform for AI agents. The loop SDK exports an OpenTelemetry span for every LLM call; Belvedir links spans into **sessions**, segments each session into the **tasks** the agent performed, and gathers similar tasks into **training sets**. Training sets feed the improvement loops (prompt/harness optimization, fine-tuning), so instrumentation quality directly determines improvement quality.

## Prerequisites

- A Belvedir API key (`bv_live_...`), minted on the platform's API Keys page (https://platform.belvedir.ai/platform/keys). The raw key is shown **once** at creation. If you don't have it, ask the user to create one; never guess or reuse another project's key.
- Store it in the app's environment (e.g. `BELVEDIR_API_KEY`); never hardcode it or commit it.
- The SDK reads **no environment variables itself**; always pass the key explicitly to `initialize()`.

## Node: `belvedir`

```bash
npm install belvedir
```

Three rules make instrumentation reliable:

1. **`initialize()` must run before any LLM SDK import**: it patches client prototypes. Call it once, at process entry.
2. **Next.js: initialize from `instrumentation.ts` and pass your LLM SDKs via `instrumentModules`.** Bundlers can inline or ESM-load LLM SDKs in ways auto-instrumentation can't see; LLM spans then vanish silently while HTTP spans still arrive. Also keep the SDKs external so every bundle shares one runtime copy: `serverExternalPackages: ["openai", "@anthropic-ai/sdk", "belvedir"]` in `next.config` on Next 15+ (`experimental.serverComponentsExternalPackages` on 13/14; the wrong key for your version is silently ignored). List `belvedir` itself too: a route or server action that bundles its own copy calls `withSession()`/`flush()` on an uninitialized duplicate, so flush silently no-ops and spans drop on serverless.
3. **Edge runtime silently skips init**: any route making LLM calls needs `export const runtime = "nodejs"`.

If the app calls an OpenAI-compatible endpoint with raw `fetch` instead of the `openai`/`@anthropic-ai/sdk` clients (common in agent frameworks with their own HTTP layer), `belvedir@0.4.0+` captures those calls automatically — streamed responses included — so `instrumentModules` is only needed for the client SDKs the app actually imports. Do not rewrite raw-fetch LLM calls to use a client SDK just for tracing.

```ts
// instrumentation.ts  (project root, or src/instrumentation.ts)
export async function register() {
  if (process.env.NEXT_RUNTIME === "nodejs") {
    const { initialize } = await import("belvedir");
    // Only import the LLM SDKs the app actually uses.
    const openai = await import("openai");
    const anthropic = await import("@anthropic-ai/sdk");
    initialize({
      apiKey: process.env.BELVEDIR_API_KEY!,
      appName: "my-ai-app",
      instrumentModules: { openAI: openai.OpenAI, anthropic },
    });
  }
}
```

Do **not** call `initialize()` from inside a route file, and only call it once. In a plain CommonJS Node process, call it at the top of the entry point; `instrumentModules` is then optional as long as it runs before any LLM SDK `require`. **In an ESM entry point (`.mjs`, or `"type": "module"`) ALWAYS pass `instrumentModules`**: static `import`s hoist above the `initialize()` call and dynamic `import()` bypasses the CommonJS require hook, so auto-instrumentation silently produces no LLM spans there — `import` the LLM SDK module and hand it over:

```ts
// index.mjs
import { initialize } from "belvedir";
import * as anthropic from "@anthropic-ai/sdk";
initialize({
  apiKey: process.env.BELVEDIR_API_KEY!,
  appName: "my-agent",
  instrumentModules: { anthropic },
});
```

### AI SDK codebases (the `ai` package)

If the app makes its LLM calls with the AI SDK (`generateText`/`streamText`/`useChat`), one rule decides whether anything is captured: **the calls must go through `@ai-sdk/openai-compatible`**. Its native provider packages and built-in gateway routing use protocols Belvedir doesn't instrument and produce NO LLM spans, with no warning: `@ai-sdk/openai` (Responses API), `@ai-sdk/anthropic` (Messages API), and plain `"provider/model"` model strings (Vercel AI Gateway protocol). Swap those call sites to `createOpenAICompatible` pointed at the same destination (the provider's OpenAI-compatible URL, the gateway's `https://ai-gateway.vercel.sh/v1`, or Belvedir's router per the inference section below) and keep each call site's model. No `instrumentModules` entry is needed for the `ai` package itself — capture happens at the HTTP layer — but `initialize()` must still run first (in ESM, import `ai` dynamically after it; in Next.js, initialize from `instrumentation.ts`):

```ts
import { initialize } from "belvedir";
initialize({ apiKey: process.env.BELVEDIR_API_KEY!, appName: "my-agent" });

const { createOpenAICompatible } = await import("@ai-sdk/openai-compatible");
const { generateText } = await import("ai");

const llm = createOpenAICompatible({
  name: "belvedir",
  baseURL: "https://platform.belvedir.ai/api/v1/route",
  apiKey: process.env.BELVEDIR_API_KEY,
});
const { text } = await generateText({ model: llm("anthropic/claude-sonnet-5"), prompt });
```

Streamed calls (`streamText`) are captured too, final completion included. Sessions, tasks, flush, and outcomes below all apply unchanged.

### Sessions and tasks (required for tasks and training sets)

Wrap each agent run in `withSession` so every LLM and tool span links under one session id. Optionally wrap distinct units of work in `task()` for sharper task boundaries:

```ts
import loop from "belvedir";

await loop.withSession({ sessionId: chatId, userId: user.id }, async () => {
  await loop.task("send_email", async () => {
    await agent.run("email danny the report");
  });
});
```

Without `withSession` raw traces still arrive, but they can't be grouped into sessions or segmented into tasks.

### Serverless: flush before returning

Serverless functions can freeze or exit before the batching span processor exports. `await flush()` at the end of any handler that makes LLM calls:

```ts
import { flush } from "belvedir";

// ... LLM calls ...
await flush();
return Response.json(result);
```

### Report outcomes (ground truth)

When the app knows whether a session actually worked (the order shipped, the test passed, the user accepted), report it. Reported outcomes beat Belvedir's model-judged labels: a `"fail"` keeps every task in that session out of training data.

```ts
import { flush, reportOutcome } from "belvedir";

await flush(); // the session must have arrived first
await reportOutcome(sessionId, orderShipped ? "success" : "fail");
```

`reportOutcome` never throws; it returns `false` (with a warning) if Belvedir hasn't seen the session yet; flush and retry, or report later from a webhook or job.

## Python: `belvedir`

```bash
pip install belvedir
```

Call `initialize()` once at process startup, before the agent makes LLM calls. There is no `instrumentModules` equivalent: Python has no bundler, so auto-instrumentation always sees the installed `anthropic`/`openai`/LangChain/etc. packages. Raw `requests`/`httpx` calls to OpenAI-compatible endpoints are captured automatically on `belvedir>=0.6.0` (streamed responses included) — don't rewrite them to a client SDK just for tracing.

```python
import os
import belvedir as loop

loop.initialize(
    api_key=os.environ["BELVEDIR_API_KEY"],
    app_name="my-agent",
)

with loop.session(session_id=chat_id, user_id=user.id):
    with loop.task("send_email"):
        agent.run("email danny the report")

loop.flush()  # before a short-lived process exits
loop.report_outcome(chat_id, "success" if order_shipped else "fail")
```

Both context managers propagate across `await` boundaries, so they work inside `async def` bodies. `flush()` and `report_outcome()` never raise; failures log warnings under the `belvedir.loop` logger.

## Route inference through Belvedir

Belvedir also EXECUTES the app's LLM calls. Do this for EVERY LLM client in the codebase, one client at a time: keep the model each call site already names, and change only the endpoint and the key — base URL to Belvedir's router, provider key to the Belvedir key. Each model the app uses becomes its own baseline: if the app calls Grok 4.6 for one job and Claude Sonnet 5 for another, the Grok call sites become Belvedir calls with `model: "x-ai/grok-4.6"` and the Sonnet call sites become Belvedir calls with `model: "anthropic/claude-sonnet-5"` — two clients, two baselines, exactly as before. Do NOT rewrite model names to `"auto"`, do NOT consolidate different call sites onto one model, and do NOT pick a different model than the one the code used. Any OpenAI-compatible client works (openai SDK, Vercel AI SDK's OpenAI-compatible provider, raw fetch to `/chat/completions`). For a provider-native xAI or Google GenAI client, replace it with the OpenAI client pointed at Belvedir and carry the model over in Belvedir's provider-prefixed spelling: `grok-4.6` → `x-ai/grok-4.6`, `gemini-2.5-pro` → `google/gemini-2.5-pro`; OpenAI ids stay bare (`gpt-4o-mini`). The router also normalizes bare ids server-side, so a missed one still routes, but write the prefixed form. **Anthropic SDK call sites do NOT swap clients**: change only the base URL — `new Anthropic({ baseURL: "https://platform.belvedir.ai/api", apiKey: BELVEDIR_API_KEY })` — and the request passes through to the Messages API untranslated (`cache_control`, `thinking`, `anthropic-beta` headers, native streaming all intact). That passthrough surface is model-pinned (no `"auto"`); it serves Claude models exactly as Anthropic would. Only rewrite an Anthropic call site to the OpenAI client if the app wants that call ROUTED across tiers rather than pinned.

What happens next: the first call naming a model creates a router anchored on it (Routers page, "From your code") with that model as the ceiling — it answers every conversational call (a person talking to the agent always gets the model the code named), while the project's cheaper Medium/Small tiers serve only easier machine-shaped tasks (classification, extraction, strict-format calls, tool loops) underneath — so the app runs on the models it chose but pays less when a smaller model is enough. Every project also has an auto router; `model: "auto"` hands a call to it outright, which is the opt-in path, not the migration. Which model actually ran comes back in the response's `model` and the `x-belvedir-model` header. Only a project whose routers were all deleted forwards a named `model` as-is (a plain metered proxy). The tiering itself is the project's **Smart routing** permission (on by default); with it off, every call serves exactly the model it names.

```ts
// Before: new OpenAI({ baseURL: "https://openrouter.ai/api/v1", apiKey: process.env.OPENROUTER_API_KEY })
const client = new openai.OpenAI({
  baseURL: "https://platform.belvedir.ai/api/v1/route",
  apiKey: process.env.BELVEDIR_API_KEY, // was OPENROUTER_API_KEY / OPENAI_API_KEY
});

const res = await client.chat.completions.create({
  model: "x-ai/grok-4.6", // unchanged: the model this call site already used
  messages,
});

// Anthropic SDK call sites keep the Anthropic SDK — only the base URL changes:
// requests pass through to the Messages API untranslated (cache_control,
// thinking, beta headers, native streaming intact). Model-pinned by design.
const anthropic = new Anthropic({
  baseURL: "https://platform.belvedir.ai/api", // SDK appends /v1/messages
  apiKey: process.env.BELVEDIR_API_KEY, // was ANTHROPIC_API_KEY
});
const sonnet = await anthropic.messages.create({
  model: "claude-sonnet-5", // unchanged
  max_tokens: 16000,
  messages,
});
```

```python
client = OpenAI(
    base_url="https://platform.belvedir.ai/api/v1/route",
    api_key=os.environ["BELVEDIR_API_KEY"],  # was OPENROUTER_API_KEY / OPENAI_API_KEY
)
res = client.chat.completions.create(model="x-ai/grok-4.6", messages=messages)  # model unchanged
```

- Pass the session id in an `x-session-id` header (or a `session_id` body field) to pin a conversation to one model.
- Which model served the call comes back in the response's `model` and the `x-belvedir-model` header. Token counts are the standard OpenAI `usage` block (`prompt_tokens`, `completion_tokens`, `total_tokens`, plus provider details such as `prompt_tokens_details.cached_tokens`); what it cost (USD, exactly what is billed) is `usage.cost` and the `x-belvedir-cost` header. Streams carry the same `usage` (tokens + cost) on the final chunk; `include_usage` is turned on upstream for you.
- Keep the tracing `initialize()` from above: routed calls are traced like any other, which is what feeds tasks and training.
- **Own gateway for spend visibility:** if the team tracks AI spend in its own aggregator (Vercel AI Gateway, an OpenRouter account), set an **AI gateway** in the dashboard's project settings under **Integrations** (base URL + that gateway's key — for Vercel AI Gateway, `https://ai-gateway.vercel.sh/v1`). Every routed call then executes through that gateway on the team's own account, so its dashboard sees all traffic and spend while Belvedir keeps routing, tracing, and metering (those calls bill on the gateway account, not Belvedir credits). No code change; keep calling the Belvedir router as above. Router model ids pass to the gateway in their public spellings, so keep routers on `provider/model` slugs the gateway serves.
- **Batch inference (offline work, half price):** `POST /api/v1/route/batches` with `{requests: [{custom_id, body: <chat-completions body>}]}` (up to 100,000 Anthropic / 50,000 OpenAI requests per batch; the Sail portion caps at 1,000 requests and ~2 MB serialized; each body names an `anthropic/*`, OpenAI, or Sail-served model, never `"auto"`) submits to Anthropic's/OpenAI's batch tiers (half price) or Sail's deferred flex window; poll `GET /api/v1/route/batches/{id}` until `status: "ended"`, then `GET .../results` — JSONL by default, one result per custom_id per line (`?format=anthropic` / `?format=openai` return the provider's exact results format for single-dialect batches, so migrated parsers run unchanged; `?format=json` returns one envelope). Code using the Anthropic SDK's batch methods needs NO endpoint changes at all: `client.messages.batches.create/retrieve/results/cancel/list/delete` work against the same base URL (`https://platform.belvedir.ai/api`) via the Anthropic-compatible facade at `/api/v1/messages/batches`. SDK batch creates over ~4 MB: use base URL `https://web-18927-96902dd0-7nu4yp6c.onporter.run` (identical API, 256 MB inline). Anthropic-native code can send `{custom_id, params: <Messages API params>}` instead (Anthropic's own batch wire shape, forwarded verbatim; the result carries the native message). Cancel in flight with `POST /api/v1/route/batches/{id}/cancel` (completed requests still return and bill; the rest come back `canceled`). On `POST /api/v1/route/batches` submissions, pass a top-level `idempotency_key` so a retried submission returns the existing batch instead of creating a second one (the Anthropic SDK facade's `batches.create` doesn't carry one). Inline bodies over ~4 MB are rejected at the platform edge — bigger submissions upload first: `POST /api/v1/route/batches/uploads` → PUT the submission JSON to the returned `upload_url` → submit `{input_object}`. Use batches for evals, classification, and backfills, not interactive calls.
- **Embeddings:** `POST /api/v1/route/embeddings` is OpenAI-compatible (`text-embedding-3-small`/`-3-large`/`-ada-002`) — same base URL and key.

## Configuration

| Node / Python | Default | Description |
|---|---|---|
| `apiKey` / `api_key` | required | Your `bv_live_...` key |
| `baseUrl` / `base_url` | `https://platform.belvedir.ai` | Must be a host that serves the ingest API; any other host 404s and spans drop silently |
| `appName` / `app_name` | `belvedir-loop-app` | Your application name |
| `disableBatch` / `disable_batch` | `false` | Export each span immediately instead of batching (useful for local testing) |
| `instrumentModules` (Node only) | — | Your imported LLM SDKs, e.g. `{ openAI: openai.OpenAI, anthropic }`; patches them directly. Required in Next.js and every ESM/bundled environment |
| `instrumentFetch` / `instrument_http` | `true` | Capture raw `fetch` / `requests` / `httpx` calls to OpenAI-compatible `chat/completions` endpoints, streams included (Node 0.4.0+, Python 0.6.0+); official-client requests are skipped so nothing double-counts. Set `false` to opt out |

## Verify the integration

Verify whichever halves you set up.

**Tracing:**
1. Run traced work inside a session, then `flush()`.
2. Raw spans show under Data → Traces immediately; the linked conversation appears under Data → Conversations within seconds.
3. Tasks and training sets form **~30 seconds after the session goes quiet** — don't debug earlier than that.

**Inference:** make one routed call and confirm it came back through Belvedir. Use a model the app actually calls (this also creates that model's router, so don't test with a model you won't use):
1. The response is a normal chat.completion (HTTP 200) and the `x-belvedir-model` response header names the model that served it (with `x-belvedir-cost`, the USD billed). Quick check from the shell:
   ```bash
   curl -sD - -o /dev/null https://platform.belvedir.ai/api/v1/route/chat/completions \
     -H "Authorization: Bearer $BELVEDIR_API_KEY" -H "Content-Type: application/json" \
     -d '{"model":"anthropic/claude-sonnet-5","messages":[{"role":"user","content":"ping"}]}' \
     | grep -i '^x-belvedir-'
   ```
   From an SDK, read the same `x-belvedir-model` header off the response object.
2. On the dashboard the model you named shows on the **Routers** page as an anchored router ("From your code"), and the call appears on the project's **Usage** page.
3. Read errors, don't guess: a `400 … isn't a priced model` means that model id has no Belvedir rate (fix the id — it may not be offered), and a `402` means the organization is out of credits (add credits on the Billing page). Neither is a code bug.

## Common issues

- **HTTP spans arrive but no `anthropic.chat` / `openai.chat` spans** (Node): the patch didn't take. Pass the SDKs via `instrumentModules` AND keep them external (`serverExternalPackages`): both are required; this exact combo has broken live integrations.
- **Spans appear locally but not in production**: the function exited before the batch flushed. `await flush()` before returning.
- **No spans at all**: wrong `baseUrl` (silent 404), `initialize()` ran after the LLM SDK was imported (Node logs a late-init warning), an ESM entry point without `instrumentModules` (no warning — the require hook never sees ESM imports), or the route runs on the Edge runtime.
- **`initialize() failed` warning mentioning `parseKeyPairsIntoRecord`** (Node): a known dependency conflict in `belvedir@0.3.1`. Upgrade: `npm install belvedir@latest` (fixed in 0.3.2).
- **Conversations captured but a streamed final assistant reply is missing**: the app calls an OpenAI-compatible endpoint with raw HTTP and `stream: true` on an SDK older than `belvedir@0.4.0` (Node) / `belvedir==0.6.0` (Python). Upgrade; the raw-HTTP instrumentation records streamed completions.
- **AI SDK app (`ai` package) produces no LLM spans**: the calls use `@ai-sdk/openai` (Responses API), `@ai-sdk/anthropic`, or plain `"provider/model"` gateway strings — none are instrumented. Swap those call sites to `@ai-sdk/openai-compatible` (see the AI SDK section above).
- **Ingest or the router returns 402/429**: not a code bug. 402 from the router = the organization is out of credits (add credits, or turn on Auto refill — off by default — under Organization Settings → Billing); 402 from ingest = the project has no organization. 429 = per-key rate limit (ingest 2 req/s sustained, burst 120; router 25 req/s, burst 300): honor `Retry-After`.
- **Traces arrive but no tasks or training sets form**: work isn't wrapped in `withSession`/`session`, or the session hasn't been quiet for ~30s yet.
- **`reportOutcome` returns false / 404**: the session hasn't been ingested yet: `flush()` first, then retry.

Full docs: https://docs.belvedir.ai
