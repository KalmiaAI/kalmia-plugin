# Kalmia for coding agents

[Kalmia](https://www.kalmia.dev) traces your AI agents and finds what went wrong.
It triages every run, investigates failures down to the root cause, and alerts you.

This repository connects Kalmia to your coding agent. It contains two things:

- **The Kalmia MCP server.** Your agent can query traces, read root-cause verdicts and grouped issues, compare variants and eval runs, and start investigations and sweeps.
- **The `kalmia-setup` skill.** Your agent adds Kalmia tracing to a codebase: it installs the SDK, wraps the LLM client, groups runs into traces, checks that a trace arrived, and sets up issue digests.

Kalmia works with OpenTelemetry, the Vercel AI SDK, the OpenAI and Anthropic SDKs, and plain HTTP.

## Before you start

You need a Kalmia API key.
[Sign up free](https://www.kalmia.dev/sign-up), then create a read-and-write key under Settings > API keys.
It starts with `kal_live_`.

## Install

### Claude Code

```
/plugin marketplace add KalmiaAI/kalmia-plugin
/plugin install kalmia@kalmia
```

Claude Code asks for your API key when the plugin is enabled.
The setup skill is then available as `/kalmia:kalmia-setup`.

To add only the MCP server, without the plugin:

```bash
claude mcp add --transport http kalmia https://www.kalmia.dev/api/mcp \
  --header "Authorization: Bearer $KALMIA_API_KEY"
```

### Cursor

Add this to `.cursor/mcp.json` in your project, or to `~/.cursor/mcp.json` for every project, and set `KALMIA_API_KEY` in your environment:

```json
{
  "mcpServers": {
    "kalmia": {
      "url": "https://www.kalmia.dev/api/mcp",
      "headers": { "Authorization": "Bearer ${env:KALMIA_API_KEY}" }
    }
  }
}
```

### Any agent that supports skills

```bash
npx skills add KalmiaAI/kalmia-plugin
```

### Any other MCP client

- URL: `https://www.kalmia.dev/api/mcp`
- Transport: streamable HTTP
- Header: `Authorization: Bearer <your API key>`

## What the MCP server can do

| Group | Tools |
| --- | --- |
| Read your traces | `query_traces`, `aggregate_traces`, `compare_metrics`, `get_trace`, `search_trace_content`, `get_span_detail`, `get_monitoring_settings` |
| Read what Kalmia found | `get_verdict`, `list_anomaly_findings`, `list_issues`, `get_issue`, `compare_variants`, `list_recent_runs`, `get_run_results`, `get_case_diagnosis`, `compare_runs` |
| Take action (each asks for your approval) | `analyze_trace`, `investigate_trace`, `annotate_trace`, `run_sweep`, `configure_monitoring`, `connect_slack`, `connect_teams` |

Some actions use credits from your plan.
See [pricing](https://www.kalmia.dev/pricing).

## Things to try

- "Set up Kalmia tracing in this repo."
- "What are the most common issues in my agent's traces this week?"
- "Why did trace `<id>` fail? Show me the root cause and the suggested fix."
- "Compare failure rates between my two prompt variants."

## Links

- [Docs](https://www.kalmia.dev/docs)
- [Setup guide for agents](https://www.kalmia.dev/setup.md)
- [Privacy](https://www.kalmia.dev/privacy), [terms](https://www.kalmia.dev/terms), [security](https://www.kalmia.dev/security)
- Questions: team@kalmia.dev

## License

MIT. See [LICENSE](LICENSE).
