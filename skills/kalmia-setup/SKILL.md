---
name: kalmia-setup
description: Set up Kalmia tracing in a codebase - install the SDK, wrap the LLM client, group runs into traces, verify a trace arrived, connect the Kalmia MCP server, and set up issue digests (email, Slack, or Microsoft Teams). Use when asked to add tracing, observability, monitoring, or Kalmia to an LLM agent, or to instrument agent runs.
---

# Set up Kalmia in this codebase

These instructions are written for a coding agent working inside a user's repository.
Follow them in order.
Each step states what to do and how to tell it worked.
Stop and ask the user only when a step says so.

Kalmia captures each run of an LLM agent as a trace: every model call, tool call, and the messages between them.
Setup means installing a small SDK, wrapping the LLM client once, and grouping each run under one trace.

## Before you start

You need a workspace API key.
It looks like `kal_live_...`.
If the user's prompt did not include one, ask for it now.
The key is shown once, when it is created: on the workspace onboarding screen, or by rotating the key under workspace settings (rotation revokes the old key).
Never commit the key.
Never echo it in a command, and never print it in your final summary.

Read `https://www.kalmia.dev/llms.txt` if you need anything beyond this page.

## Step 1 - Pick the integration path

Inspect the repository and choose exactly one path from this table.
Check the dependency manifest (`pyproject.toml`, `requirements.txt`, `package.json`) and the imports where the LLM client is constructed.

| What the repo uses | Path |
| --- | --- |
| Python with `openai` or `anthropic` | A - Python SDK |
| TypeScript/JavaScript with `openai` or `@anthropic-ai/sdk` | B - TypeScript SDK |
| TypeScript/JavaScript with the Vercel AI SDK (`ai` package) | C - Vercel AI SDK middleware |
| LangChain, LlamaIndex, CrewAI, or any app that already emits OpenTelemetry spans | D - OpenTelemetry exporter |
| None of the above, or the LLM calls happen in a service you cannot edit | E - Direct ingestion |

If the repo has both a Python and a TypeScript service that call models, do the one the user pointed you at.
Mention the other in your summary.

## Step 2 - Install

Path A: use the project's existing dependency manager.

| Manager | Command |
| --- | --- |
| uv (`uv.lock`) | `uv add kalmia` |
| Poetry (`poetry.lock`) | `poetry add kalmia` |
| pip (`requirements.txt`) | `pip install kalmia`, then add `kalmia` to the requirements file |

The package has no dependencies of its own.

Path B and C:

```sh
npm install kalmia-sdk
```

The npm package is `kalmia-sdk`, not `kalmia`.
Use the repo's package manager if it is not npm (check for `pnpm-lock.yaml`, `yarn.lock`, `bun.lockb`).

Path D has two cases; check before installing anything:

- The app already emits OpenTelemetry spans for its LLM calls (an existing tracer provider, OpenInference or `gen_ai.*` instrumentation): install nothing new. You only add an exporter in Step 4.
- The app uses a supported framework but has no OTel instrumentation yet: install the framework's OpenInference instrumentation package plus an OTLP/HTTP exporter.

Never add a second instrumentation to an app that already has one; that double-reports every call.
Kalmia needs no SDK on this path.

Path E: nothing to install.

## Step 3 - Configure the key

Put the key in the environment, not in code.

1. Add `KALMIA_API_KEY=kal_live_...` to the local env file the project already uses (`.env`, `.env.local`).
2. Make sure that file is ignored by git. If it is not in `.gitignore`, add it before writing the key. If the file is already tracked by git (`git ls-files --error-unmatch <file>` succeeds), stop and tell the user before continuing.
3. If the project keeps an `.env.example`, add `KALMIA_API_KEY=` to it with an empty value.

The SDKs read `KALMIA_API_KEY` automatically.
The default destination is `https://www.kalmia.dev`, so no base URL setting is needed.

## Step 4 - Instrument

Make the smallest change that captures a full run.
Do not change which model is called, its parameters, or the prompts.
Do not add tracing to test files.

### Path A - Python SDK

Initialize once at process startup, then wrap the client where it is constructed:

```python
from kalmia import init_logger, wrap_anthropic, wrap_openai, traced

init_logger()  # reads KALMIA_API_KEY from the environment

client = wrap_anthropic(anthropic.Anthropic())  # or wrap_openai(openai.OpenAI())
```

Then wrap the function that runs one agent turn or one request so every call inside it lands in one trace:

```python
@traced(name="my-agent-run")
def run(prompt):
    ...
```

`traced` also works as a context manager: `with traced(name="my-agent-run") as span:`.
Without it, each model call becomes its own standalone trace.

If tool execution is a function you can wrap, capture it as a tool span:

```python
with traced(name=tool_name, span_type="tool") as span:
    result = run_tool(args)
    span.log(input=args, output=result)
```

Streaming responses are captured automatically.
If the code offloads model calls to threads, copy the context in: `executor.submit(contextvars.copy_context().run, fn)`.

### Path B - TypeScript SDK

```ts
import { initLogger, wrapAnthropic, wrapOpenAI, traced } from "kalmia-sdk";

initLogger({}); // reads KALMIA_API_KEY from the environment; the argument is required (an empty object is fine)

const client = wrapAnthropic(new Anthropic()); // or wrapOpenAI(new OpenAI())
```

Wrap the body of the function that runs one agent turn:

```ts
await traced(async (span) => {
  // ... the existing run logic, using the wrapped client ...
}, { name: "my-agent-run" });
```

Tool spans: `await span.traced(toolName, async (t) => { ...; t.log({ input, output }); }, { type: "tool" })`.

The trace is posted when the `traced` callback resolves.
In a serverless handler, await it before returning.
In a script, let the process exit normally; the SDK flushes on `beforeExit`, so do not call `process.exit()` right after the run.

### Path C - Vercel AI SDK

```ts
import { wrapLanguageModel } from "ai";
import { initLogger, wrapAISDK, endAISDKSession } from "kalmia-sdk";

initLogger({}); // reads KALMIA_API_KEY from the environment; the argument is required (an empty object is fine)

const model = wrapLanguageModel({
  model: baseModel,
  middleware: [wrapAISDK({ sessionID })], // omit sessionID for one trace per call
});
```

Use `model` with `generateText` / `streamText` exactly as before.
If the app has a conversation or session id, pass it as `sessionID` so all calls in that session form one trace, and call `await endAISDKSession(sessionID)` when the session ends.
Tracing is observation-only and never alters model output.

### Path D - OpenTelemetry exporter

If the app already has a tracer provider, add one OTLP/HTTP exporter pointed at Kalmia to it and change nothing else.
Otherwise, instrument and export. Python example with OpenInference for LangChain:

```python
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor
from opentelemetry.exporter.otlp.proto.http.trace_exporter import OTLPSpanExporter
from openinference.instrumentation.langchain import LangChainInstrumentor

provider = TracerProvider()
provider.add_span_processor(BatchSpanProcessor(OTLPSpanExporter(
    endpoint="https://www.kalmia.dev/api/otel/v1/traces",
    headers={"Authorization": "Bearer " + os.environ["KALMIA_API_KEY"]},
)))
LangChainInstrumentor().instrument(tracer_provider=provider)
```

TypeScript: `new OTLPTraceExporter({ url: "https://www.kalmia.dev/api/otel/v1/traces", headers: { Authorization: `Bearer ${process.env.KALMIA_API_KEY}` } })` from `@opentelemetry/exporter-trace-otlp-proto`, registered on a `NodeTracerProvider`.

OTLP/HTTP only, protobuf or JSON.
OTLP/gRPC is not accepted.

### Path E - Direct ingestion

POST traces in Kalmia's format to `https://www.kalmia.dev/api/v1/traces/ingest` with `Authorization: Bearer $KALMIA_API_KEY`.
The full schema is at `https://www.kalmia.dev/api/v1/openapi.json`.
Minimal shape: a trace with `id`, `input`, `output`, and `spans`, where each span has `id`, `span_id`, `root_span_id`, `name`, `input`, `output`, `span_attributes: { type: "llm" }`, and `metrics`.

## Step 5 - Run it once

Run the agent the way the user normally runs it, for one real request.
If you cannot run it (no credentials for the model provider, needs a UI), ask the user to run it once and tell you when done.

## Step 6 - Verify a trace arrived

Do not declare success from the code change alone.
Note the time before the run in Step 5 as an ISO timestamp, then ask the API for traces since then:

```sh
curl -s "https://www.kalmia.dev/api/v1/traces?from=2026-01-01T00:00:00Z&limit=5" \
  -H "Authorization: Bearer $KALMIA_API_KEY"
```

Replace `from` with the timestamp you noted.
Success: the JSON response has a non-empty `traces` array.
If the run's output included a `[kalmia]` line naming a trace id, that id should be in the list.

Try up to three times, five seconds apart.
If no matching trace appears, stop polling and diagnose with this table:

| Response | Meaning | Fix |
| --- | --- | --- |
| 401 | The key is wrong or not reaching the process | Check the env file is loaded where the agent runs |
| 403 write-only key | The key can ingest but not read | Ask the user to confirm the trace in the dashboard, which updates live |
| 200 with empty `traces` | Nothing was sent | The process exited before flushing, the run was not inside `traced`, or the SDK logged a warning naming a dropped trace id. Check the run's output for `[kalmia]` warnings |

Rate limits on this endpoint are generous, but do not poll beyond the three tries.

## Step 7 - Optional: connect the MCP server

Once traces flow, the user's coding agent can read them, run root-cause analysis, and query evals through Kalmia's MCP server.
Offer this to the user.
If they accept, add it to the agent's MCP config.
Step 8 does not need it.
For Claude Code:

```sh
claude mcp add --transport http kalmia https://www.kalmia.dev/api/mcp \
  --header "Authorization: Bearer $KALMIA_API_KEY"
```

For any host that reads `.mcp.json`:

```json
{
  "mcpServers": {
    "kalmia": {
      "type": "http",
      "url": "https://www.kalmia.dev/api/mcp",
      "headers": { "Authorization": "Bearer ${KALMIA_API_KEY}" }
    }
  }
}
```

The MCP server needs a read-and-write key.
A write-only key is rejected.

## Step 8 - Digests: confirm delivery and offer Slack or Teams

This step is part of setup; do it even if the user's request only mentioned verifying a trace.
It is one API read and at most two questions, and it is how the user finds out about issues without opening the dashboard.
Only Step 7 is optional.

A sweep is how Kalmia finds issues on its own: it audits the recent runs, confirms what went wrong, and writes a digest with an agent health score.
The digest is emailed to the workspace owners; quiet sweeps send nothing.
A sweep costs 2 credits for each trace it checks (a new account has 150 credits), so a new workspace starts with scheduled sweeps off and nothing is spent until the user says so.
The reads and switches in this step are plain API calls with the key, like Step 6; connecting a channel is a link the user opens. No MCP needed.

1. Read the current state:

```sh
curl -s "https://www.kalmia.dev/api/v1/monitoring" -H "Authorization: Bearer $KALMIA_API_KEY"
```

   The JSON has `monitoring` (null when never configured), `slack`, `teams`, and `delivering`.
   If `monitoring.enabled` is already true, sweeps are on a schedule: tell the user the cadence and skip to item 3.
2. Offer a schedule. Ask the user: "Kalmia can sweep your agent's runs on a schedule and email you a digest of confirmed issues. A sweep costs 2 credits for each trace it checks, up to 250 traces a sweep unless you change that in Settings, so what a schedule costs follows how many runs your agent sends. If your credits cover only part of a sweep, it checks that part. Want weekly, daily, or no schedule for now?"
   If they pick one, send it as the cadence (`weekly` or `daily`):

```sh
curl -s -X PATCH "https://www.kalmia.dev/api/v1/monitoring" -H "Authorization: Bearer $KALMIA_API_KEY" \
  -H "Content-Type: application/json" -d '{"enabled":true,"cadence":"weekly","emailDigests":true}'
```

   Then tell the user, in these words or close to them: "Sweeps are on. When one finds confirmed issues, Kalmia emails the workspace owners. Say so if you'd rather keep them in-app only and I'll turn email off."
   If they want email off, PATCH `{"emailDigests":false}` the same way.
   If they want no schedule, tell them: "No schedule. When your agent has some real runs in Kalmia, open the Scan tab and click Run a sweep to get your first digest. It states what it will check and cost before it starts." Do not start a sweep yourself during setup: the workspace only holds the test trace so far, and a sweep over one trace finds nothing.
3. Ask: "Want digests posted to a Slack or Microsoft Teams channel as well?"
   If not, or if `slack.connected` / `teams.connected` is already true for the one they want, skip to Step 9.
4. If they want a channel, give them this link and tell them to open it signed in to Kalmia:

   `https://www.kalmia.dev/dashboard?view=settings&section=monitoring`

   For Slack, the page has a "Connect Slack" button that opens Slack's approval page; they click Allow and are brought back connected.
   Only Slack workspace owners can install apps that are not in the Slack Marketplace; if the user is not one, a Slack owner must do this part.
   For Teams (needs a Microsoft 365 work or school account; personal Teams has no Workflows), the page has a "Connect Teams" field.
   Microsoft has no approval link, so they create the webhook in Teams first: open the channel menu, choose Workflows, add "Send webhook alerts to a channel" (older tenants call it "Post to a channel when a webhook request is received"), copy the URL it shows, and paste it into that field.
   Wait for the user to say they finished.

   Fallback, only if the user would rather stay in the terminal than open the settings page:

   - Slack: get the sign-in email of the Kalmia workspace owner who will approve it (usually the user), then:

```sh
curl -s -X POST "https://www.kalmia.dev/api/v1/monitoring/slack-link" -H "Authorization: Bearer $KALMIA_API_KEY" \
  -H "Content-Type: application/json" -d '{"ownerEmail":"<that email>"}'
```

     Give the user the returned `url`.
     It opens Slack's approval page, is valid for 30 minutes, and must be opened by that owner in a browser signed in to Kalmia.
   - Teams: have the user create the Workflows webhook as above and give you the URL, then:

```sh
curl -s -X POST "https://www.kalmia.dev/api/v1/monitoring/teams" -H "Authorization: Bearer $KALMIA_API_KEY" \
  -H "Content-Type: application/json" -d '{"webhookUrl":"<that URL>","label":"<channel name>"}'
```

     Kalmia posts a "Kalmia connected" card to the channel to prove the webhook works and only then stores it; an `error` means the URL was wrong or the workflow is off.
5. Read `https://www.kalmia.dev/api/v1/monitoring` again and confirm `slack.connected` or `teams.connected` is true.
   There is nothing else to switch on: from the next sweep, every digest posts to the connected channel.

How to tell it worked: if the user chose a schedule, the final read reports `monitoring.enabled` true and `monitoring.emailDigests` true; `delivering` is true if a channel was connected.
The first digest arrives after the first sweep; the dashboard's Scan tab shows it as well.

## Step 9 - Report

Tell the user, in a few lines:

- which path you chose and why
- which files changed
- where the key lives, without printing it
- whether the verification succeeded, with the trace id if it did
- whether digests are on, and where they go (email, Slack channel, or off)
- what to look at next: `https://www.kalmia.dev/dashboard`

## Rules

- Keep the change minimal. Tracing wraps existing code; it does not restructure it.
- Never commit `KALMIA_API_KEY`.
- Never change model, parameters, or prompts.
- One `init_logger` / `initLogger` per process, at startup.
- If anything in this page conflicts with what you find in the repo, prefer the repo's conventions for where config and env live, and this page for the Kalmia calls.
- Do not skip Step 8 or Step 9. Step 7 is the only optional step.

## Keeping this skill current

The canonical copy of this procedure is `https://www.kalmia.dev/setup.md`.
If a step here fails against the current SDK, fetch that page and follow it instead.
