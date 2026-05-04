# ngrok-hono-auth

A minimal example of delegating authentication entirely to ngrok, with zero auth logic in the app.

## The pattern

Two traffic policy rules do all the work:

**1. `oauth` action** — ngrok intercepts matching requests, redirects unauthenticated users to your OAuth provider, and stores the verified identity in `actions.ngrok.oauth.identity`.

**2. `add-headers` action** — a subsequent rule reads that identity and injects it into the forwarded request:

```yaml
- type: add-headers
  config:
    headers:
      X-Forwarded-User-Email: "${actions.ngrok.oauth.identity.email}"
      X-Forwarded-User-Name: "${actions.ngrok.oauth.identity.name}"
```

Your app reads those two headers and trusts them. That's the whole auth layer.

## Why this is safe

ngrok's agent makes an **outbound** connection to ngrok's infrastructure. Your app binds to `localhost` only — there is no open inbound port. The only way to reach the app is through the ngrok tunnel, which enforces the traffic policy before any request arrives. There is no direct path an attacker could use to forge the identity headers.

This is different from a reverse proxy setup where you'd need a separate firewall rule to block direct access to the app port. With ngrok, the closed-port guarantee is structural.

## Setup

**1. Install dependencies**

```bash
bun install
```

**2. Configure ngrok**

Copy `ngrok.example.yml` to `~/.config/ngrok/ngrok.yml` and fill in your authtoken. You can get one at [dashboard.ngrok.com](https://dashboard.ngrok.com).

**3. Start the app**

```bash
bun run dev
```

**4. Start the tunnel**

```bash
ngrok http 3000 --traffic-policy-file traffic-policy.yml
```

`traffic-policy.yml` is committed to this repo — it's the interesting part. If you need to pass a config file (e.g. for a named tunnel or authtoken on a shared machine), add `--config ~/.config/ngrok/ngrok.yml`.

Visit the ngrok URL. You'll be redirected to GitHub to log in. After that, `GET /me` returns your identity.

## Routes

| Route | Auth required | Description |
|-------|--------------|-------------|
| `GET /` | No | Public |
| `GET /me` | Yes | Returns `{ email, name }` from ngrok headers |
| `GET /private` | Yes | Greeting with your name |

## Extending this

**Persist users** — on first login, upsert the email into a `users` table. The first user can be auto-promoted to admin (see `src/index.ts`).

**Protect only some routes** — apply `requireAuth()` selectively, or add a path expression to the traffic policy so ngrok only enforces login on `/app/*` while leaving `/` public.

**Different providers** — swap `google` for `github`, `microsoft`, `gitlab`, or `facebook` in `traffic-policy.yml`. No code changes needed.

**Production / always-on** — to survive reboots, embed the traffic policy inline in your `ngrok.yml` under `tunnels.app.traffic_policy`, then run ngrok as a service:

```bash
sudo ngrok service install --config ~/.config/ngrok/ngrok.yml
sudo ngrok service start
```
