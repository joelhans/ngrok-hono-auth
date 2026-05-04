# ngrok-hono-auth

A small Hono app where ngrok handles OAuth at the edge and the app reads two headers it can trust. About 30 lines of TypeScript and 20 lines of YAML, end to end.

The repo exists to demonstrate one specific pattern. If it suits the way you build, take it. If it doesn't, no problem.

## What's in here

The app doesn't run any OAuth code: no callbacks, no session store, no token refresh, no PKCE, no provider SDK, no users-with-credentials table. It does have about ten lines of middleware that read `X-Forwarded-User-Email` and `X-Forwarded-User-Name` and gate routes. ngrok adds those headers after a successful OAuth login at the edge.

## How it works

The Traffic Policy file has two rules. The first uses the `oauth` action to intercept matching requests, redirect unauthenticated users to your provider, and store the verified identity in `actions.ngrok.oauth.identity`. The second uses `add-headers` to inject that identity into the forwarded request:

```yaml
- type: add-headers
  config:
    headers:
      X-Forwarded-User-Email: "${actions.ngrok.oauth.identity.email}"
      X-Forwarded-User-Name: "${actions.ngrok.oauth.identity.name}"
```

The app reads those headers and trusts them.

## The trust boundary

Trusting `X-Forwarded-User-Email` rests on one structural fact: the only way to reach the app is through the ngrok tunnel. The agent makes an outbound connection to ngrok's edge, and the app binds to `localhost` only. There's no inbound port, so there's no other path to forge the headers on.

If the upstream becomes reachable some other way (also bound to `0.0.0.0`, behind a public Kubernetes ingress, shared on a VPC), anything that can reach it can spoof the headers. Two deployment shapes preserve the property:

- **Single-host.** Agent and app on the same machine, app on `127.0.0.1`, ngrok installed as a system service.
- **Multi-instance or Kubernetes.** Use [internal endpoints](https://ngrok.com/docs/universal-gateway/internal-endpoints/) so the upstream is only routable from inside ngrok's network.

Leaving the upstream addressable at a public URL or behind a public ingress breaks the property and makes the headers forgeable.

## A note on portability

The `X-Forwarded-User-Email` header name is a convention shared with oauth2-proxy and Cloudflare Access. Swapping ngrok for one of those in front of the same app wouldn't require app changes.

## Authz in the same file

Coarse authz can live in the same Traffic Policy file as login. A domain allowlist:

```yaml
- expressions:
    - "actions.ngrok.oauth.identity.email != ''"
    - "!actions.ngrok.oauth.identity.email.endsWith('@yourcompany.com')"
  actions:
    - type: deny
      config:
        status_code: 403
```

Per-resource authz still belongs in the app.

## What this doesn't do

There's no logout. The OAuth session lives in a cookie on ngrok's edge, and there's no app-side endpoint to clear it. The workarounds are closing the tab, clearing cookies, or setting a short session TTL in the policy. Honestly: who logs out of anything anymore? If a real logout button matters to you, this isn't the right pattern.

It also doesn't cover fine-grained RBAC, MFA enrollment, or anything that mutates identity. Those belong with an IdP, regardless of what's in front.

## Setup

**1. Install dependencies**

```bash
bun install
```

**2. Configure ngrok**

Copy `ngrok.example.yml` to `~/.config/ngrok/ngrok.yml` and fill in your authtoken from [dashboard.ngrok.com](https://dashboard.ngrok.com).

**3. Start the app**

```bash
bun run dev
```

**4. Start the tunnel**

```bash
ngrok http 3000 --traffic-policy-file traffic-policy.yml
```

`traffic-policy.yml` is committed because it's the interesting part. Visit the ngrok URL. You'll be redirected to Google to log in. After that, `GET /me` returns your identity.

## Routes

| Route | Auth required | Description |
|-------|--------------|-------------|
| `GET /` | No | Public |
| `GET /me` | Yes | Returns `{ email, name }` from ngrok headers |
| `GET /private` | Yes | Greeting with your name |

## Extending this

**Persist users.** On first login, upsert the email into a `users` table. The first user can be auto-promoted to admin.

**Protect only some routes.** Tighten the path expression in `traffic-policy.yml` so ngrok only enforces login on `/app/*` and leaves marketing routes public.

**Switch providers.** Swap `google` for `github`, `microsoft`, `gitlab`, or `facebook` in `traffic-policy.yml`. No code changes.

**Run it always-on.** Embed the Traffic Policy inline under `tunnels.app.traffic_policy` in `ngrok.yml`, then run ngrok as a service:

```bash
sudo ngrok service install --config ~/.config/ngrok/ngrok.yml
sudo ngrok service start
```
