# ngrok-hono-auth

A small Hono app where ngrok handles OAuth at the edge and the app reads two headers it can trust. About 30 lines of TypeScript and 30 lines of YAML, end to end.

The repo demonstrates one specific pattern. If it suits the way you build, take it. If it doesn't, no problem.

## The front-door pattern

ngrok is the front door. The app sits behind an internal ngrok endpoint that's only routable from inside your ngrok account. Public traffic only reaches the app through a public cloud endpoint that runs a Traffic Policy: OAuth, identity propagation, then forward to the internal endpoint.

```
[ public ] → [ cloud endpoint + Traffic Policy ] → [ internal endpoint ] → [ Hono app ]
```

The shape is the same whether the app is one process on your laptop or a fleet of pods in Kubernetes. Scaling out only changes how many things sit behind the internal endpoint URL; the gateway side stays put.

## What's in the app

The app doesn't run any OAuth code: no callbacks, no session store, no token refresh, no PKCE, no provider SDK, no users-with-credentials table. It does have about ten lines of middleware that read `X-Forwarded-User-Email` and `X-Forwarded-User-Name` and gate routes. ngrok adds those headers after a successful OAuth login at the gateway.

## The Traffic Policy

Three rules sit in `traffic-policy.yml`. The `oauth` action intercepts matching requests, redirects unauthenticated users to your provider, and stores the verified identity in `actions.ngrok.oauth.identity`. The `add-headers` action injects that identity into the forwarded request. The `forward-internal` action sends the request to the internal endpoint that backs the app:

```yaml
- type: forward-internal
  config:
    url: https://hono-app.internal
```

The app reads the headers and trusts them.

## The trust boundary

The app trusts `X-Forwarded-User-Email` because the only path to reach it is through the cloud endpoint that runs the policy. The internal endpoint isn't publicly addressable; the agent that backs it makes an outbound connection to ngrok. There's no public address on the app itself, so there's no second path to forge headers on.

If you give the app a public address some other way (also bound to `0.0.0.0` on a host with a public NIC, behind a public Kubernetes ingress, mapped through a public load balancer), the headers become forgeable at that path. The point of the front-door pattern is that the app has no public address at all.

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

There's no logout. The OAuth session lives in a cookie on ngrok's gateway, and there's no app-side endpoint to clear it. The workarounds are closing the tab, clearing cookies, or setting a short session TTL in the policy. Honestly: who logs out of anything anymore? If a real logout button matters to you, this isn't the right pattern.

It also doesn't cover fine-grained RBAC, MFA enrollment, or anything that mutates identity. Those belong with an IdP, regardless of what's in front.

## Setup

**1. Install dependencies**

```bash
bun install
```

**2. Claim a free static domain**

Visit [dashboard.ngrok.com/domains](https://dashboard.ngrok.com/domains) and create a free static domain. It'll look like `something-something-1234.ngrok.app`. One time only.

**3. Set credentials**

Grab your authtoken and an API key from [dashboard.ngrok.com](https://dashboard.ngrok.com). The agent uses the authtoken; the API call uses the API key.

```bash
export NGROK_AUTHTOKEN=<YOUR_AUTHTOKEN>
export NGROK_API_KEY=<YOUR_API_KEY>
```

Copy `ngrok.example.yml` to `~/.config/ngrok/ngrok.yml` and paste the same authtoken into the file. The agent reads it from there at startup.

**4. Create the cloud endpoint**

This is the gateway. One time only.

```bash
ngrok api endpoints create \
  --api-key $NGROK_API_KEY \
  --type cloud \
  --bindings public \
  --url https://<YOUR_DOMAIN> \
  --traffic-policy-file traffic-policy.yml
```

**5. Start the app**

```bash
bun run dev
```

**6. Start the agent**

```bash
ngrok start --all
```

This brings up the internal endpoint at `hono-app.internal`, which the cloud endpoint forwards to. Visit your static domain, log in with Google, and `GET /me` returns your identity.

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

**Push policy edits.** The cloud endpoint holds its own copy of the policy from when you created it. After editing `traffic-policy.yml`, push the new version with `ngrok api endpoints update <ENDPOINT_ID> --traffic-policy-file traffic-policy.yml`. Find the ID with `ngrok api endpoints list`.

**Run the agent always-on.** The cloud endpoint is already permanent. To keep the agent running across reboots, install it as a system service:

```bash
sudo ngrok service install --config ~/.config/ngrok/ngrok.yml
sudo ngrok service start
```

**Graduate to multi-instance or Kubernetes.** Stop running the agent on your laptop. Run it as a sidecar or deployment with the same `ngrok.yml` and the same internal endpoint URL. Multiple agents serving the same internal endpoint URL pool automatically. The cloud endpoint, the Traffic Policy, and the app code don't change.
