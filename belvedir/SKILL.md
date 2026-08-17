---
name: belvedir
description: Instrument an AI agent with Belvedir observability: install the Belvedir SDK (Node or Python), wrap runs in sessions and tasks, flush spans, and report outcomes. Use when connecting an app to Belvedir or debugging missing traces, tasks, or groups.
---

# Belvedir integration

> This document lives at https://platform.belvedir.ai/agents.md and in the https://github.com/Belvedir/skills repo. It is written for coding agents: everything needed to instrument an app with Belvedir, in one page. It also works as a skill file: install it into the codebase as `.claude/skills/belvedir/SKILL.md` (or your agent's equivalent) to keep it available offline:
>
> ```bash
> mkdir -p .claude/skills/belvedir && curl -fsSL https://raw.githubusercontent.com/Belvedir/skills/main/belvedir/SKILL.md -o .claude/skills/belvedir/SKILL.md
> ```

Belvedir is an observability and self-improvement platform for AI agents. The loop SDK exports an OpenTelemetry span for every LLM call; Belvedir links spans into **sessions**, segments each session into the **tasks** the agent performed, and clusters similar tasks into **groups**. Groups feed the improvement loops (prompt/harness optimization, fine-tuning), so instrumentation quality directly determines improvement quality.

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

### Sessions and tasks (required for Tasks & Groups)

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

Belvedir can also EXECUTE the app's LLM calls: point any OpenAI-compatible client at the router endpoint and swap the provider key for the Belvedir key. Two lines change; the client is used exactly as before. Every project starts with a router (editable on the platform's Routing page), so Belvedir picks the model per request — including the project's fine-tuned models — runs the call, and bills the organization at cost. Pass `model: "auto"`: the router decides, and the request must NOT name a specific provider model, or every trace will be labelled with a model that never ran. Only a project whose routers were all deleted honors an explicit `model` (as a plain metered proxy).

```ts
const client = new openai.OpenAI({
  baseURL: "https://platform.belvedir.ai/api/v1/route",
  apiKey: process.env.BELVEDIR_API_KEY, // was OPENAI_API_KEY
});

const res = await client.chat.completions.create({
  model: "auto", // the router picks; never hard-code a provider model here
  messages,
});
```

```python
client = OpenAI(
    base_url="https://platform.belvedir.ai/api/v1/route",
    api_key=os.environ["BELVEDIR_API_KEY"],  # was OPENAI_API_KEY
)
```

- Pass the session id in an `x-session-id` header (or a `session_id` body field) to pin a conversation to one model.
- Which model served the call comes back in the response's `model` and the `x-belvedir-model` header.
- Keep the tracing `initialize()` from above: routed calls are traced like any other, which is what feeds tasks and training.

## Configuration

| Node / Python | Default | Description |
|---|---|---|
| `apiKey` / `api_key` | required | Your `bv_live_...` key |
| `baseUrl` / `base_url` | `https://platform.belvedir.ai` | Must be a host that serves the ingest API; any other host 404s and spans drop silently |
| `appName` / `app_name` | `belvedir-loop-app` | Your application name |
| `disableBatch` / `disable_batch` | `false` | Export each span immediately instead of batching (useful for local testing) |

## Verify the integration

1. Run traced work inside a session, then flush.
2. Traces appear on the dashboard (Data → Conversations) within seconds.
3. Tasks and groups form **~30 seconds after the session goes quiet**; don't debug earlier than that.

## Common issues

- **HTTP spans arrive but no `anthropic.chat` / `openai.chat` spans** (Node): the patch didn't take. Pass the SDKs via `instrumentModules` AND keep them external (`serverExternalPackages`): both are required; this exact combo has broken live integrations.
- **Spans appear locally but not in production**: the function exited before the batch flushed. `await flush()` before returning.
- **No spans at all**: wrong `baseUrl` (silent 404), `initialize()` ran after the LLM SDK was imported (Node logs a late-init warning), an ESM entry point without `instrumentModules` (no warning — the require hook never sees ESM imports), or the route runs on the Edge runtime.
- **`initialize() failed` warning mentioning `parseKeyPairsIntoRecord`** (Node): a known dependency conflict in `belvedir@0.3.1`. Upgrade: `npm install belvedir@latest` (fixed in 0.3.2).
- **Traces arrive but no tasks or groups form**: work isn't wrapped in `withSession`/`session`, or the session hasn't been quiet for ~30s yet.
- **`reportOutcome` returns false / 404**: the session hasn't been ingested yet: `flush()` first, then retry.

Full docs: https://docs.belvedir.ai
