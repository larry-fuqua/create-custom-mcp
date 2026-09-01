# Keep-alive and tunnels

## Detach from the agent

The MCP HTTP process, gateway, and tunnel must not be children of the
Grok/agent shell. Agent-owned `Start-Process`, background `&`, and
session `ssh` children die or get stolen.

| OS | Start detached | Survive reboot |
|---|---|---|
| Windows | WMI `Win32_Process.Create` with hidden window, or `wscript` launching hidden PowerShell | Task Scheduler **at startup**; “run whether user is logged on or not” when a stored user credential is acceptable. Daily watchdog as backup, not a 5-minute poll. |
| Linux | `docker compose up -d` or `systemctl --user start` / system unit | Compose `restart: unless-stopped` (Docker starts at boot) or systemd `Restart=always`. For user units: `loginctl enable-linger <user>`. |
| macOS | `launchctl bootout/bootstrap` | LaunchDaemon (no GUI login) for HTTP; LaunchAgent only if the process needs a logged-in user. |

Pid/unit files and logs live next to that service’s directory. Stop
scripts must not kill the tunnel when restarting a single MCP.

## Tunnel flavors

Prefer stable names so Connectors do not rot.

| Flavor | Hostname stability | After reboot |
|---|---|---|
| ngrok reserved/static domain | Stable `--url https://<assigned>.ngrok-free.app` (or paid domain) | Same command; no Connector edit if auth token + domain persist |
| ngrok random URL | Changes every process | Parse local API / logs, save `public-url.txt`, tell user to update Connector |
| Cloudflare named tunnel | Stable CNAME you control | Same `cloudflared tunnel run`; DNS already set |
| Cloudflare quick tunnel | Random `*.trycloudflare.com` | Same as random ngrok: persist URL, re-register |

One reserved domain globally. A second agent on another machine **steals**
it. Keep the tunnel on the gateway host until **all** upstreams have moved,
then cut over in one step (stop old, start new with the same `--url`).

**When to restart the tunnel vs the gateway**

| Event | Restart |
|---|---|
| New sibling route or route host-env flip | Gateway only. Tunnel still points at the gateway. |
| Tunnel process gone (reboot, crash, sleep) | Tunnel. Gateway too if it did not come back. |
| Moving the reserved hostname to another machine | Stop the old tunnel, start the new one with the same `--url`. |

**First MCP** registers tunnel + gateway + this MCP autostart.
**Sibling** registers only this MCP's autostart.

Watchdog: HTTPS GET `https://<host>/health` (include the tunnel vendor's
skip-interstitial header if a free hostname injects a browser warning).
Success is HTTP 200 with the expected body, not “process listed.” If the
public URL **changed**, write it and notify — that is domain-to-endpoint
reregistration.

Machines that must stay reachable overnight: do not leave AC sleep-on-idle
enabled without the user agreeing.

## Dedicated box vs operator PC

- Operator PC: gateway + tunnel + some MCPs. Fine for a workstation that
  stays powered.
- Dedicated box: MCP compose/units only at first; gateway/tunnel stay on
  the operator PC; routes use LAN host. Later, move gateway then tunnel.
- When an MCP's **public** path leaves a machine and the user did **not**
  keep the local support stack, **remove** that machine's autostart for
  the moved public wrapper (scheduled tasks, login items). Do not leave
  disabled leftovers. If they kept the local stack, leave autostart and
  local processes alone.
- Old “start the whole stack” scripts must no-op the **tunnel** once every
  gateway route host is the dedicated box (a second agent steals a
  reserved domain).
- Do not bind the box’s existing product ports. Do not compose-down
  unrelated projects. Do not reboot the box to “apply” an MCP.

## Health order after a change

1. Upstream listens on the intended address/port.
2. Gateway `/health` local.
3. Public `/health` through the existing tunnel.
4. Authenticated `initialize` on the new prefix.
5. Sibling prefixes still `initialize`.
6. Unrelated services on the box still answer (whatever the user already
   runs there).
