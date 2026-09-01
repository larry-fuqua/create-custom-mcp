---
name: create-custom-mcp
description: >
  Stand up a community or custom MCP with optional local stdio (same tools
  as HTTP), remote HTTP behind a shared auth gateway (OAuth 2.1 PKCE plus
  API key), optional public hostname, keep-alive across reboot, a security
  review of vendor code and the planned deployment, a final as-built
  security review after install, and wiring to the current
  agent/harness and optional public Connectors. Use when the user asks to
  add, wrap, deploy, migrate, or host an MCP; set up a Connector; share
  one tunnel across MCPs; run /create-custom-mcp; or do a final as-built security
  review after a custom MCP install.
---

# Create a custom MCP

Turn a community (or locally written) MCP into a durable connector: chosen
implementation, security review, process isolation, optional public URL, and
client wiring. Do not copy hostnames, IPs, account names, keys, or token
files from any prior machine into this skill's work product.

Read [references/security-review.md](references/security-review.md),
[references/auth-gateway.md](references/auth-gateway.md), and
[references/keep-alive-and-tunnels.md](references/keep-alive-and-tunnels.md)
when those phases start. Do not paste their checklists into chat.

**Stdio in this skill** means a local harness `command`/`args` entry that
exposes the **same MCP tool list** as the HTTP path. It is in scope only
when deploying **on the machine that will run the vendor**, so that agent
can use those tools over a pipe. Expanding stdio into CLIs, local daemons,
or host files is **out of scope**; mention it only in the closing follow-on
after a **fresh** on-host stdio deploy.

## Decide placement first

Ask only what is not already stated:

1. **Where it runs:** this operator machine, or a dedicated always-on box
   (same LAN or remote). OS may be Windows, macOS, or Linux.
2. **Who consumes it:** public HTTPS Connector (example: Grok app / Grok
   Bot), local agent/TUI/IDE on this machine or another on the LAN, other
   harnesses, or more than one.
3. **Local stdio (new deploy on this machine only):** yes or no. Default
   **no** unless they already asked. If yes: launcher + harness
   `command`/`args` for the **same tools** as HTTP. Skip launcher and
   stdio wiring if they decline or the vendor will not run on this machine.
4. **Tunnel:** if a shared auth gateway + hostname already exist, **add a
   path** (sibling). If none exist, create gateway + hostname (first MCP).
   Do not start a second tunnel on a reserved domain (a second agent
   steals it).
5. **New vs migrate:** new stack, or flip one existing **public** path
   onto another host while other gateway routes stay put.

### If migrating a public path

Inspect the **current** operator machine before changing anything. State
findings to the user (do not guess):

- **Local MCP (same tools):** harness config with `command`/`args` (stdio)
  or loopback HTTP to this MCP.
- **Broader-than-MCP local use:** agent/scripts talking to the vendor
  outside that tool list (local daemons, CLIs, host files). Do not
  implement or preserve that in this skill.

Then state the **default** migrate plan and wait for objection:

- Public gateway route will point at the dedicated box.
- If a **local MCP** existed: at the end, that harness entry becomes
  `url = "http://<box>:<port>/mcp"` (same tools over LAN). Stdio (or
  loopback HTTP) for this MCP is no longer the local connection.
- If **broader-than-MCP** local use existed: it will **stop** when the
  public path and default cleanup run. Rebuilding it is out of scope.
- **Keep local stack:** they may interrupt and keep the entire local
  support stack (processes, autostart, harness MCP entries, and any
  broader local use) **unchanged**. This skill then only deploys on the
  box and flips the **public** route. Record `keep-local-stack` and skip
  Phase 6 cleanup and Phase 7 local-harness edits. Two vendor processes
  (old machine + box) may contend for the same refresh grant; say so
  once; do not block if they still want it.

## Architecture

```
Public client (Connector-style UI)
        |
 HTTPS hostname
        |
 shared auth gateway (OAuth PKCE + API key)
        |
 vendor MCP HTTP (mcp-proxy wrapping stdio, or proven native --http)

Optional, same tools, vendor on this machine: local agent --stdio--> vendor
Optional, same tools: local agent -- LAN/loopback HTTP --> vendor /mcp
```

Rules:

- Public clients never talk to the vendor MCP process directly on the
  internet.
- One public hostname, many path prefixes (`/thing-mcp`, `/other-mcp`, …).
- Each MCP gets its own loopback or LAN port, OAuth client id, and prefix.
- Vendor OAuth (Google, Atlassian, …) is **service auth**. Gateway OAuth +
  API key is **caller auth**. Both may exist; they are not substitutes.
- Pin the vendor package version. Tokens live on the host that runs that
  MCP.

## Phase 1 — Choose the implementation

Prefer community servers with: current API (not a sunset predecessor),
broader tool catalog, recent commits/releases, npm/pypi/cargo presence,
wrapable stdio (HTTP can be added with `mcp-proxy`), tests/CI, OSI license.

If two implementations overlap, install both only when the user wants
breadth **and** write a short consumer skill that names tools (do not
leave the agent to guess which server to call).

Reject: unmaintained clones of a more active repo, APIs that shutdown soon,
enterprise-only products when the user wants personal data, or anything
that requires pasting refresh tokens into prompts.

Record: package name + pinned version, transports, token path, default bind
address, OAuth callback port. Pin the MCP runtime the vendor was written
for (do not float a major that renamed the stdio/HTTP API).

Default public HTTP: wrap vendor stdio with `mcp-proxy --streamEndpoint /mcp`.
Use the vendor's native streamable HTTP only after `initialize` and
`tools/list` succeed on that HTTP endpoint.

## Phase 2 — Security review (plan and source)

Complete [references/security-review.md](references/security-review.md)
on the published source **before** install, then again on the **pinned
installed tree** before exposing a port. Stop and tell the user if the
vendor binds `0.0.0.0` with no auth, returns tokens from tools, shells
out unsafely, or has no license.

Then review the **planned deployment**: gateway in front of public HTTP, secrets
not in git, bind addresses, no extra host firewall/iptables changes
without asking, no compose merge into unrelated stacks, no privileged or
host-network containers unless the service truly needs them.

This phase reviews code, architecture, and the plan. It does **not**
replace Phase 8. Installs change to meet the box (wrappers, env files,
compose interpolation, cookie/token handling, ports). Those diffs are
why the as-built pass exists.

## Phase 3 — Install the vendor MCP

- Dedicated directory for this MCP (compose project, repo, or service
  folder). Never add it to an unrelated compose file (home automation,
  databases, etc.).
- Env file + vendor config/token dir, mode `600`.
- HTTP via `mcp-proxy --streamEndpoint /mcp` wrapping stdio, **or** native
  streamable HTTP only if already proven in Phase 1.
- Stdio launcher (load `.env`, exec vendor) **only** if placement chose
  local stdio **and** this install is on the operator machine. Omit it
  otherwise.
- Default bind **loopback**. Bind all interfaces only when another machine
  on the LAN must reach it (migrate, or a local agent on another LAN
  host). After the tunnel lives on the same host as the MCP, return to
  loopback unless a local agent still needs LAN HTTP.
- Do not publish the vendor's other local ports (sidecars, daemons) on
  all interfaces unless the user asked for LAN access to **that**
  protocol. The MCP port is not those ports.
- If the vendor client only skips TLS verify for `localhost`, keep it on
  `127.0.0.1` (share a network namespace with a sidecar if needed).
- Pick a **free** port. Do not reuse SSH, RDP, the existing gateway, the
  tunnel admin API, or other documented service ports.
- Vendor loopback OAuth callback must not collide with an existing local
  HTTP port; set an explicit redirect URI if the default is taken.
- Headless: disable vendor auto-browser re-auth on the server. Consent on
  an operator desktop, then copy tokens **once** to the server. Prefer the
  vendor's documented non-GUI secret store in containers; persist it on a
  volume. Pin the native binary flavor that runs on **that** CPU.

## Phase 4 — Shared gateway (OAuth + API key)

**First MCP (no gateway yet):** create the gateway from
[references/auth-gateway.md](references/auth-gateway.md), then the tunnel
in Phase 5.

**Sibling (gateway already public):** add a route only. Do not recreate
the gateway, the API key, or the tunnel.

Minimum for every MCP:

- Path prefix → `{host, port}` (host defaults to loopback; set to the
  dedicated-box LAN IP only for flipped routes).
- OAuth 2.1 authorization code + PKCE S256, well-known AS metadata,
  per-resource protected-resource metadata, client id unique to this MCP
  (Connector UIs often have a client-id field; do not reuse another MCP's).
- Allowed redirects: loopback plus **that harness's** documented OAuth
  callback hosts (example: Grok `grok.com`, `x.ai`, and subdomains).
- JSON, 401, and proxied MCP/SSE responses include CORS and expose
  `WWW-Authenticate` and `mcp-session-id` (browser PKCE). OPTIONS alone
  is not enough.
- `/health` is a **gateway-local** always-on 200 (do not proxy it to a
  migratable MCP) so a tunnel watchdog can probe the public hostname.

The API key is the master secret. It lives in a mode-`600` file next to
the gateway. One key for every prefix. Unauthenticated `POST /<prefix>`
is 401. A caller proves they have the key by pasting it on the gateway
consent page, which mints an OAuth code. After that, the client holds
issued tokens. The key does not leave the gateway host. See
[references/auth-gateway.md](references/auth-gateway.md).

After a **route or host-env** change: restart the **gateway** only. Leave
the tunnel running. Restart the tunnel only if it is down (reboot, crash)
or the hostname is moving — see
[references/keep-alive-and-tunnels.md](references/keep-alive-and-tunnels.md).
Force-kill drops in-memory tokens; they must reload from the token file.
Do not tell the user OAuth was wiped unless that file is gone.

## Phase 5 — Keep-alive, reboot, tunnel

Follow [references/keep-alive-and-tunnels.md](references/keep-alive-and-tunnels.md)
(first MCP vs sibling autostart is there).

Process must outlive the agent session and, if possible, a GUI login:
Docker `restart: unless-stopped`, systemd (linger for user units),
LaunchDaemon, or a Windows task at **startup** (not a child of the agent
shell). Never `Start-Process` / `&` / foreground SSH a long-running
server from the agent.

Prefer a **stable** public hostname (ngrok reserved domain or Cloudflare
named tunnel). Random/quick tunnels require rewriting the saved URL and
updating every Connector after each restart.

Watchdog: probe the **public** `/health` (or documented health), not
merely that `ngrok`/`cloudflared` exists. If the URL is dynamic and
changed, persist it and tell the user to re-register the Connector.

## Phase 6 — Migrate a public path (optional)

Only if placement chose migrate. Move **one** gateway path at a time.

1. Run the MCP on the box, publish the **same port**, reachable from the
   gateway host.
2. Set that route's host env on the gateway; restart gateway only (tunnel
   stays unless it is down).
3. **Cleanup — skip entirely when `keep-local-stack`:**
   Stop the old **public** HTTP wrapper for this MCP. Remove its autostart
   and make start-the-stack scripts no-op as in
   [references/keep-alive-and-tunnels.md](references/keep-alive-and-tunnels.md)
   (Dedicated box vs operator PC). Kill leftover agent-spawned stdio
   **children** for this MCP only when the harness entry is being switched
   off stdio (default migrate with a local MCP). Do not stop unrelated
   services on either machine.
4. **Local harness (Phase 7):**
   - Default, local MCP existed: point it at
     `url = "http://<box>:<port>/mcp"`. Do not add stdio on the box
     unless they asked for a **new** on-box stdio deploy.
   - `keep-local-stack`: do not edit harness MCP config, do not stop
     local stdio/HTTP, do not remove local autostart.
5. Verify the flipped **public** path **and** every other gateway path
   **and** unrelated services on the box. Reconnect only harnesses this
   skill is allowed to change.

Do not start the public tunnel on the box until **every** MCP host on the
gateway is that box. One reserved domain.

## Phase 7 — Connect each chosen consumer

Wire **only** the consumers from the placement questions. After writing a
harness config, apply **that** product's reconnect (reload MCP servers,
new session, IDE restart, or re-add Connector). Do not reload a harness
that is not a consumer, and do not change local config when
`keep-local-stack`.

**Local agent / TUI / IDE** (only if chosen, and not `keep-local-stack`):

- **Fresh deploy on this machine** and they asked for stdio:
  `command` + `args` to the launcher on this host (same tools as HTTP).
- Otherwise (including default migrate of a former local MCP, or an
  agent on another LAN machine):
  `url = "http://<host>:<port>/mcp"`.
- Then reconnect that harness so old stdio children are not left running
  when the entry is no longer stdio.

If a **managed** Connector and a **user** MCP both exist, call the user
one. Distinguish by server name and by a tool that reports token/config
path (container `/data/...` vs a managed host).

**Public Connector** (only if chosen; example: Grok app / Grok Bot):

- URL: `https://<stable-host>/<path-prefix>`
- OAuth client id: the gateway client id for this MCP (public client, no
  secret)
- Never put the gateway API key in the URL. Never put it in Connector
  headers (`Authorization: Bearer …` or `x-api-key`). Stuffing the key
  there copies the master secret off the box into the Connector config,
  where it does not expire.

The connect card / OAuth PKCE flow is required. The user pastes the API
key on the **gateway consent page** (the auth prompt in the browser).
That page is the gateway host. The client then stores minted
access/refresh tokens, not the key. Point at the key file on the gateway
host if they need to look it up. Do not paste the key into chat. Sibling
MCPs share that same key; do not mint a second one. If a Connector
already existed at the old URL, they re-auth in that UI.

**Other agents** (only if chosen): that product's MCP config (stdio
command only for a fresh on-host stdio deploy they asked for; else HTTP
URL or Connector). Reconnect the way that harness documents. Same rule:
do not put the gateway API key in a remote Connector URL or headers.

Prove it on each wired consumer: `initialize` / `tools/list`, then one
real read tool. Confirm other custom MCPs still answer. Confirm the
tunnel process is still the intended one.

### Follow-on (fresh on-host stdio only)

If this run **created** local stdio on the vendor host, tell the user this
skill stopped at the MCP tool list. A local agent on that host can also
use a shell against vendor CLIs, local daemons, or files. That is a
**separate** task if they want it. Do not start it here. Do not offer it
after a migrate (broader local use was either kept intact or called out
as disabled).

## Phase 8 — Final as-built security review

Required after Phase 7. Phase 2 passing does not skip this.

Re-read [references/security-review.md](references/security-review.md)
**As-built (after implementation)** against the live stack, not the plan:
running compose/unit files, container inspect, actual secret files and
modes, wrappers added during install, gateway route as committed,
unauthenticated probe, log leak check, sibling services still up.

List **drift** from the Phase 2 plan (anything added, renamed, unescaped,
interpolating, or bound differently to make it work). Report a verdict:
**GO**, **GO-with-fixes**, or **NO-GO**. Do not silently fork the vendor
to “fix” findings unless the user asked. Do not paste secrets into chat.

The install is not done until this pass is reported.

## Windows / macOS / Linux specifics

Detach from the agent:

| OS | Long-running process |
|---|---|
| Windows | WMI `Win32_Process.Create` (or a task that runs `wscript` → hidden PowerShell). Not `Start-Process` from the agent. |
| Linux | `docker compose up -d` or systemd. Not a session-owned `&`. |
| macOS | `launchctl` bootstrap of a daemon/agent plist. |

Reboot:

| OS | Mechanism |
|---|---|
| Windows | Task Scheduler at startup (prefer “whether user is logged on or not” for headless HTTP). Daily watchdog as backup. |
| Linux | Docker `unless-stopped` (engine starts at boot) or systemd `Restart=always`. `loginctl enable-linger` if a user unit must run without login. |
| macOS | LaunchDaemon (machine, no login) preferred for HTTP; LaunchAgent only if it must see the user session. |

Vendor GUI consent still needs a desktop once; copy tokens to the
headless host afterward.

## Do not

- Put secrets, IPs, or hostnames from this skill's examples into the
  user's files as if they were theirs.
- Treat stdio as a second API or implement broader host access in this
  skill.
- On migrate, change or tear down the local stack when `keep-local-stack`.
- `docker compose down` / prune / restart Docker / reboot a shared box
  without being asked.
- Change host firewall, iptables, or daemon.json without review.
- Merge this stack into another product's compose file.
- Expose vendor HTTP on the internet without the gateway.
- Start a second ngrok/cloudflared on a hostname that is already reserved.
- Claim the Connector works after only a local stdio smoke test.
- Recreate the gateway or tunnel when adding a sibling MCP.
- Paste live tokens, keys, or webhook URLs into chat or into this skill.
- Skip Phase 8 because Phase 2 already reviewed the vendor source or the
  plan.
- Stuff the gateway API key into Connector `headers` (`Authorization:
  Bearer` or `x-api-key`). That copies the master key off the box into
  Connector config with no expiry. Use OAuth client id plus the connect
  card instead.
- Ask the user to paste the API key into chat. The paste belongs on the
  gateway consent page.
