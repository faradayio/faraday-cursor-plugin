# Faraday for Cursor

Configure and operate your [Faraday](https://faraday.ai) account directly from Cursor's Agent, in natural language.

Faraday is a customer context platform: ingest customer data, resolve identities, build audiences (cohorts) and personas, train predictive models (churn, conversion, LTV, lead scoring), and deliver per-person context to your destinations (ad platforms, ESPs, CRMs, warehouses, the Lookup API). This plugin connects Cursor to Faraday's hosted **Configuration MCP** server so the agent can do all of that for you.

## What you can do

- **Orient** — "Connect to my Faraday account and show me what's set up and how it all connects."
- **Explore data** — browse connections, datasets, streams, traits, and the consumer-attribute dictionary.
- **Build audiences** — create and refine cohorts from behavioral and demographic predicates.
- **Model** — train and inspect propensity outcomes, persona sets, and recommenders.
- **Deliver** — assemble scopes (population + payload) and wire up targets to your destinations.

The server exposes a read-and-write toolset over your account; every write is scoped to the account you authenticate as.

## What's included

- **MCP server** — connects Cursor to Faraday's hosted Configuration MCP at `https://mcp.faraday.ai/mcp`.
- **Rules** (`rules/faraday-tool-usage`) — the resource model, the build-order chain, async-polling, and memory/JMESPath patterns the agent should follow when driving Faraday.
- **Skills** — `configure-faraday-account` (end-to-end Connection → Target walkthrough) and `predict-customer-behavior` (focused propensity / lookalike / churn / LTV recipe).

## Install

**One-click:** use the "Add to Cursor" button on the [Cursor Marketplace listing](https://cursor.com/marketplace), or install this plugin from the Marketplace. Cursor will add the server and walk you through signing in to Faraday via OAuth.

**Manual:** add the following to your `~/.cursor/mcp.json` (global) or a project's `.cursor/mcp.json`:

```json
{
  "mcpServers": {
    "faraday": {
      "url": "https://mcp.faraday.ai/mcp"
    }
  }
}
```

Then open **Cursor Settings → Tools & Integrations → MCP Tools**, and complete the sign-in when prompted.

## Authentication

The server uses OAuth — on first use, Cursor opens a browser to sign in to Faraday and authorize access. No API keys or secrets are stored in your Cursor config. Tools run against whichever Faraday account you authenticate as.

## Requirements

- A [Faraday](https://faraday.ai) account.
- Cursor with MCP support.

## Links

- Faraday: https://faraday.ai
- Documentation: https://faraday.ai/docs
- Support: support@faraday.ai

## License

MIT — see [LICENSE](LICENSE).
