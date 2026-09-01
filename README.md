# create-custom-mcp

A drop-in skill for standing up a community or custom MCP: pick an implementation, review it, run it behind a shared auth gateway (OAuth 2.1 PKCE + API key), keep it alive across reboot, wire consumers, then do a **final as-built security review**.

Two copies live in this repo:

| Path | For |
|---|---|
| [`create-custom-mcp/skills/create-custom-mcp/`](create-custom-mcp/skills/create-custom-mcp/) | **Grok Build** (and any agent that reads `SKILL.md`) |
| [`grok-bot/create-custom-mcp/`](grok-bot/create-custom-mcp/) | **Grok Bot** (tool names and connector wiring for that app) |

No hostnames, tokens, or account IDs. Use at your own risk. MIT.

## Install: Grok Build

Copy the skill folder:

```bash
mkdir -p ~/.grok/skills
cp -R create-custom-mcp/skills/create-custom-mcp ~/.grok/skills/create-custom-mcp
```

Or, from this repo as a personal marketplace:

```bash
grok plugin marketplace add larry-fuqua/create-custom-mcp
grok plugin install create-custom-mcp --trust
```

Then `/create-custom-mcp` or ask Grok Build to add, wrap, or host a custom MCP.

## Install: Grok Bot

Copy `grok-bot/create-custom-mcp/` into your Grok Bot skills (or ask a Bot to import that folder). Invoke with `/` or `@`. Catalog connectors still win when one exists; this skill is for community/custom servers.

## What it insists on

- **Phase 2** reviews vendor source and the *plan*. **Phase 8** reviews the *running* stack. Phase 2 passing does not skip Phase 8.
- The gateway **API key** stays on the gateway host. Paste it on the **consent page** during OAuth PKCE. Do not put it in a Connector URL or in `Authorization` / `x-api-key` headers.

## License

MIT. Not affiliated with xAI, Cursor, or any MCP vendor.
