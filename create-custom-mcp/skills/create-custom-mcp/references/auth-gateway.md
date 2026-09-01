# Auth gateway

A small local HTTP reverse proxy in front of every public MCP path.
Clients: Grok app / Bot Connectors and any remote MCP client.

## Route table

Each MCP:

| Field | Example shape |
|---|---|
| Path prefix | `/thing-mcp` |
| Upstream host | loopback, or LAN IP of a dedicated box |
| Upstream port | unique per MCP |
| OAuth client id | `thing-mcp` (stable, documented) |
| Label | shown on the consent HTML page |

`pathname === prefix` or `prefix + '/'…` maps to upstream `/mcp` + remainder
(query string preserved). OPTIONS CORS is not enough: JSON, 401, and
proxied MCP/SSE also send `Access-Control-Allow-Origin` and expose
`WWW-Authenticate` and `mcp-session-id`.

`GET /health` is a **gateway-local** 200 (do not proxy it to a migratable
MCP) so the tunnel watchdog does not depend on a remote box.

Host env per route, default loopback. Unset = this machine. Set = hybrid
flip. After a host-env change, restart the gateway (not the tunnel,
unless the tunnel is down).

## Caller auth

The master secret is one shared API key in a mode-`600` file next to the
gateway. It never goes in chat, never in a Connector URL, and never in Connector `headers`. Stuffing `Authorization: Bearer <api-key>` (or
`x-api-key`) into a Connector copies the master key off the box into that
client’s config, where it does not expire. That is a downgrade from this design.

How a Connector links:

1. Add the remote URL plus the route’s OAuth client id (public client,
   no secret). No API key in headers. (Grok Bot: `AddMcpServer` with
   `auth.CLIENT_ID` only.)
2. The client runs OAuth 2.1 authorization code + PKCE S256. The connect
   card opens the gateway consent page.
3. The user pastes the API key on that **consent page** (the auth prompt
   in the browser, on the gateway host). The gateway mints a code.
4. The client stores issued access/refresh tokens, not the API key.

Gateway still accepts `Authorization: Bearer` **or** `x-api-key` on
requests: either an issued OAuth access token, or the API key from a
machine that is supposed to hold it (the gateway host, a local script).
A remote Connector is not that machine.

- OAuth 2.1: `response_type=code`, PKCE S256, `grant_types` authorization
  code + refresh_token, `token_endpoint_auth_methods=none` (public client).
- Persist issued access/refresh tokens with restrictive permissions.
- `/.well-known/oauth-authorization-server` and
  `/.well-known/oauth-protected-resource/<prefix>` (RFC-style resource
  metadata: `resource`, `authorization_servers`, `bearer_methods_supported`).
- 401 includes
  `WWW-Authenticate: Bearer … resource_metadata="<origin>/.well-known/oauth-protected-resource<prefix>"`.

Allowed `redirect_uri` hosts: loopback, plus the agent’s documented
callbacks (Grok: `grok.com`, `x.ai`, and subdomains).

Issued access tokens are valid on **every** gateway path unless the
gateway binds a token to a prefix. Do not treat a Connector login as
isolating one MCP from the others. The public URL alone is not enough
to link; without the consent-page paste, OAuth cannot finish.

## First MCP vs sibling

**First MCP:** create the gateway process, API key file, well-known
endpoints, consent page, autostart, and the tunnel (next reference).

**Sibling:** new prefix, port, and client id (no collisions). Add a
route + host env default. Autostart **only** the new upstream. Restart
the gateway so it reloads routes. Do not mint a second API key, do not
create a second gateway process, do not touch the tunnel unless it is
down.

Document Connector URL `https://<host>/<prefix>`, client id, API key
file.

Do not run a second gateway on a second public hostname unless the user
wants isolation.
