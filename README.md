# ngrok-hono-auth

OAuth at the ngrok edge. Two trusted headers in the app. No auth plumbing.

## The claim

You skip the *plumbing*: no OAuth callbacks, no session store, no token refresh, no PKCE, no provider SDKs, no user table for credentials. You still write *enforcement* — about ten lines of middleware that reads two headers and gates routes.

Full app: ~30 lines of Hono. Full auth config: ~20 lines of YAML.

## How it works

Two traffic policy rules:

**1. `oauth` action** — ngrok intercepts matching requests, redirects unauthenticated users to your provider, stores the verified identity in `actions.ngrok.oauth.identity`.

**2. `add-headers` action** — injects that identity into the forwarded request:

```yaml
- type: add-headers
  config:
    headers:
      X-Forwarded-User-Email: "${actions.ngrok.oauth.identity.email}"
      X-Forwarded-User-Name: "${actions.ngrok.oauth.identity.name}"
```

The app reads those headers and trusts them. That's the whole auth layer.

## The trust boundary — read this

The app trusts `X-Forwarded-User-Email` because of one structural fact: **the only way to reach the app is through the ngrok tunnel.**

The agent makes an *outbound* connection to ngrok's edge. The app binds to `localhost` only — no inbound port. No second path means no way to forge identity headers.

This property is what makes the pattern safe. The moment your upstream is reachable some other way — also bound to `0.0.0.0`, behind a public Kubernetes ingress, shared on a VPC — anything that can reach it can spoof the headers. So when you take this beyond `bun run dev`:

- **Single-host deploys**: keep the same shape. Agent and app on the same machine, app on `127.0.0.1`, ngrok installed as a system service. Identical trust model, always on. Scales further than you'd think.
- **Multi-instance / Kubernetes**: use [internal endpoints](https://ngrok.com/docs/universal-gateway/internal-endpoints/) so the upstream is only routable from inside ngrok's network. Same property, different topology.
- **What not to do**: leave the upstream addressable at a public URL or behind a public ingress. The headers become forgeable.

## What about lock-in?

`X-Forwarded-User-Email` is a convention, not an ngrok invention — oauth2-proxy and Cloudflare Access emit the same shape. If you ever leave ngrok, drop oauth2-proxy in front. The app doesn't change.

## Authz lives in the policy too

Once login is at the edge, coarse authz can live there alongside it. Allowlist a domain:

```yaml
- expressions:
    - "actions.ngrok.oauth.identity.email != ''"
    - "!actions.ngrok.oauth.identity.email.endsWith('@yourcompany.com')"
  actions:
    - type: deny
      config:
        status_code: 403
```

Now the policy file is your auth surface: login, identity propagation, and who's allowed in. Per-resource authz still lives in the app, where it belongs.

## What this doesn't give you

Mostly, logout. ngrok holds the OAuth session in a cookie on its edge — there's no app-side endpoint to clear it. Honestly: who logs out of anything anymore? Close the tab, clear cookies, or set a short session TTL on the policy. If a real logout button is non-negotiable, this pattern isn't for you — or you accept writing the one piece of real auth code that gets you there.

You also won't get fine-grained RBAC, MFA enrollment, or anything that mutates identity. Those belong to a real IdP, with or without ngrok in front.

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

`traffic-policy.yml` is committed — it's the interesting part. Visit the ngrok URL. You'll be redirected to Google to log in. After that, `GET /me` returns your identity.

## Routes

| Route | Auth required | Description |
|-------|--------------|-------------|
| `GET /` | No | Public |
| `GET /me` | Yes | Returns `{ email, name }` from ngrok headers |
| `GET /private` | Yes | Greeting with your name |

## Extending this

- **Persist users**: on first login, upsert the email into a `users` table. The first user can be auto-promoted to admin.
- **Protect only some routes**: tighten the path expression in `traffic-policy.yml` so ngrok only enforces login on `/app/*` and leaves marketing routes public.
- **Different providers**: swap `google` for `github`, `microsoft`, `gitlab`, or `facebook` in `traffic-policy.yml`. No code changes.
- **Production, always-on**: embed the traffic policy inline under `tunnels.app.traffic_policy` in `ngrok.yml`, then run ngrok as a service:

  ```bash
  sudo ngrok service install --config ~/.config/ngrok/ngrok.yml
  sudo ngrok service start
  ```
