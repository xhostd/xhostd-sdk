---
name: xhostd-domains
description: >-
  Puts an xhostd project on the user's own domain or a raw TCP port, and adds
  visitor sign-in to the user's app. Starts when the user wants a custom domain
  attached or its DNS records checked, a public TCP port for a protocol that
  HTTPS cannot carry, or Google sign-in for the visitors of an app hosted on
  xhostd. Holds custom domains, exposed ports, and the identity cookie.
---

# xhostd custom domains, raw TCP ports, and visitor sign-in

Changing a domain and exposing a port are write operations: perform them when the user asks for that
operation on that environment. Never expose a port without the user's explicit direction.

## Custom domains

`add_custom_domain` (`project`, `environment`, `domain`, optional `service`, the public service it
points at) returns DNS instructions in the `instructions` field: relay that text to the user
verbatim. After they create the records, call `add_custom_domain` again with the same arguments and
`verify=true`. HTTPS is automatic once the domain is verified. `list_custom_domains` and
`remove_custom_domain` are also available, and each domain names its `service`. Limit 5 per
environment.

A verify or a removal does not restart the app: the container's `XHOSTD_DOMAINS` changes at the
environment's next deploy. When the reply carries a `notice`, relay it and offer to deploy the
environment.

## Public raw-TCP endpoints

When the environment's HTTPS URL can't carry the protocol (a database protocol, a message broker, a
game server, a custom binary protocol), `expose_port` (`project`, `environment`, optional
`allow_cidrs`, optional `service`) returns a public `host:port` that forwards raw TCP into the
container. `allow_cidrs` is a source-address allowlist; omitting it means the whole internet can
connect. The container side is fixed: listen on `0.0.0.0:$XHOSTD_FORWARD_PORT`, which every
non-`static` container receives.

The current plan decides whether public TCP endpoints are available (`port_forwarding` in
`get_stats` with `view="overview"`), and the project's port-forwarding toggle must also be on. The
toggle is a protected action that a person turns on in the console; no tool flips it.
`list_exposed_ports` shows the project's endpoints, and `unexpose_port` releases one. xhostd adds
no TLS and authenticates nothing on that port: the app's own handling of each connection is its
only access control. The raw-TCP recipe,
<https://docs.xhostd.com/guides/recipes-port-forwarding.md>, holds the raw text of a worked service.

## Google sign-in for the user's app

No configuration and no MCP tool. `/xhost-auth/*` works on every deployed environment. After Google
sign-in the gateway sets a signed identity cookie `__Host-xhost_id` (an RS256 JWT) on the
environment's host; the app verifies it against the JWKS at
`https://auth.xhostd.com/xhost-auth/jwks` and gates its own routes.

**xhostd does identity only, never edge gatekeeping — nothing is blocked at the edge, so every route
stays public (anonymous visitors get `200`) until your app verifies the cookie and enforces access
itself in code.**

Send signed-out users to `/xhost-auth/login?return_to=<path>`, logout via
`/xhost-auth/logout?return_to=/`; SPA/JS-only apps call `GET /xhost-auth/whoami`.
**`__Host-xhost_id` is a reserved cookie name — never set or read it as a raw value; always verify
it (pin `RS256`, check `iss` and `exp`, and check `aud` against the request's host — a custom
domain's cookie carries that domain).** The sign-in recipe,
<https://docs.xhostd.com/guides/recipes-oauth.md>, holds the raw text of a verifier for each
stack.

