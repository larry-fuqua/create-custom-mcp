# Security review

Run this three times: (1) on published source before install, (2) on the
pinned installed tree before exposing a port, (3) as-built on the live stack
after consumers are wired. Phase 2 of the skill is (1)+(2). Phase 8 is (3).
Installs change to meet the box; (3) exists because of that drift. Report
findings to the user; do not “fix” a vendor by silently forking unless they
asked.

## Community / vendor code

Read README, tool list, auth docs, and the HTTP/stdio entrypoint on the
published source. After pin-install, re-check the installed entrypoint
(what will actually run).

Search the tree (and lockfile) for:

- `0.0.0.0`, `::`, `INADDR_ANY` binds
- tools or logs that echo `access_token`, `refresh_token`, `client_secret`,
  `Authorization`
- `child_process`, `exec(`, `eval(`, `Function(` on user-controlled strings
- writes outside a documented config/token directory
- unexpected network destinations besides the documented API
- missing or non-OSI license

Check:

- Tokens on disk with restrictive permissions; never returned by tools
- Privacy/redaction modes if the API is sensitive (health, mail, location)
- Vendor’s own warning: HTTP is loopback-only / no caller RBAC
- Dependency freshness vs abandoned packages
- Pin exact version (`pkg@1.2.3`), not floating tags
- Scoped API tokens: the grant may be a subset of what the UI showed
  (higher privilege often does not include read/write). Verify with a
  live call; recreate the token if scopes cannot be edited.
- Native vendor HTTP: confirm `initialize` and `tools/list` work. If
  they do not, wrap stdio with `mcp-proxy` instead.

Sensitive/restricted OAuth scopes (health, mail, …): personal-use and
unpublished “testing” apps may expire refresh tokens on a short timer.
Publishing the OAuth **consent screen** (not a store listing) is often
enough for a single-user desktop client; full Google verification is not
required for personal use. Do not silently request write/clinical scopes.

## Deployment

- Public path always through the auth gateway. Probe unauthenticated
  `POST /<prefix>` and expect 401 + `WWW-Authenticate` resource metadata.
- Gateway listens on loopback; only the tunnel binds the public hostname.
- LAN `0.0.0.0` is a hybrid-period exception. Treat that port as trusted
  LAN only; close it or bind loopback after the tunnel colocates.
- `.env`, token JSON, API keys: gitignored, `600`, not in image layers
  (mount or env_file at run).
- One OAuth grant → one refresher unless the user kept a local stack
  beside a migrated public path (warn once). After a public-path move,
  remove the old machine's autostart for that wrapper only when they did
  not keep the local stack (do not leave disabled leftovers).
- No extra capabilities, privileged mode, or `network_mode: host` unless
  the service cannot work otherwise (and never steal host ports owned by
  an existing product).
- Do not enable or rewrite host firewall, Docker daemon config, or
  default iptables to “make the MCP work.”
- Operator SSH: batch mode, short commands, no orphaned `ssh` to the box.
- Final pass: `tools/list` does not include debug dump-secret tools;
  health endpoints do not leak tokens.

## As-built (after implementation)

Run after the MCP is installed, the gateway route exists, keep-alive is on,
and at least one consumer has done `initialize` / `tools/list` plus one real
read. Re-read **what is running**, not the Phase 2 plan.

### Live stack

- Compose/unit/task files as they exist on disk now (ports, `restart`,
  `user`, `privileged`, `cap_add`, `network_mode`, `env_file`, `environment`,
  volumes). Bind must still be loopback unless LAN was an explicit exception.
- `docker inspect` / service status: health, user, mounts, published ports
  (not `0.0.0.0` for vendor HTTP), no extra capabilities.
- Secret files: key **names** only in notes, mode `600`, gitignored, not in
  image layers. Confirm compose/systemd is not interpolating `$` inside JWTs
  or cookies (project `.env` auto-load is a common footgun). If it is,
  secrets belong in a file compose does not parse for substitution.
- Wrappers added during install (persist-on-refresh, cookie unescape,
  extra entrypoints): read them. Confirm they do not log secrets and that
  token write-back stays mode `600` on the bind mount.
- Operator paste path (browser cookies, copy-as-cURL): platform escaping
  (`^` on Windows cmd, etc.) must be reversed on disk; say so if it was.

### Gateway and blast radius

- Unauthenticated `POST /<prefix>` still 401. Do not put the API key in
  the report.
- Gateway still loopback; only the existing tunnel is public. No second
  tunnel on the reserved hostname.
- Container/process logs: cookies, `Authorization`, JWT leak yes/no
  (redact if you have to look).
- Sibling MCPs and unrelated services on the box still up. Did not
  `compose down` a shared stack.
- `tools/list` still has no dump-secret tool you would call from a
  transcript (flag vendor session-paste tools as residual risk).

### Drift and verdict

List every material difference from the Phase 2 plan (wrappers, env
filenames, bind, image tag, cookie/token shape, interpolation, extra
packages). Then one verdict: **GO**, **GO-with-fixes**, or **NO-GO**.
High/med/low on each finding. The install is not done until this is
reported. Do not paste secrets into chat.
