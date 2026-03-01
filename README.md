# OAuth2-Proxy Helm Chart

Baseline for [OAuth2-Proxy](https://github.com/oauth2-proxy/oauth2-proxy) with GitHub OAuth and 1Password-backed secrets. Uses upstream chart plus [expectedbehaviors/OnePasswordItem-helm](https://github.com/expectedbehaviors/OnePasswordItem-helm). No sensitive values in defaults.

## Chart contents

- **OAuth2-Proxy:** Upstream subchart; config via `oauth2.config.configFile` and `oauth2.extraEnv`; credentials from existing secret (e.g. `oauth2-proxy-secret`).
- **Secrets:** onepassworditem subchart syncs a 1Password item into a Kubernetes Secret (client-id, client-secret, cookie-secret). Reference it with `config.existingSecret`.
- **Ingress:** Host and path (e.g. `auth.example.com`, `/oauth2`); typically global auth is disabled on this ingress so the login flow is not protected by itself.
- **Reloader:** Annotate the deployment so [Stakater Reloader](https://github.com/stakater/Reloader) restarts the pod when the credentials secret changes.

## Features

- **External auth** — Exposes `/oauth2/auth`, `/oauth2/start`, `/oauth2/callback` for ingress-nginx `global-auth-url` / `global-auth-signin`.
- **GitHub OAuth** — Configure for GitHub (or other providers via config); allowlist via `OAUTH2_PROXY_GITHUB_USERS` or `OAUTH2_PROXY_EMAIL_DOMAINS`.
- **Cookie security** — HttpOnly, Secure, SameSite, CSRF, refresh interval; set cookie domain for your domain (e.g. `.example.com`).

## Prerequisites

- **1Password Connect** (optional): if you use `onepassworditem.enabled: true`. Secrets are created in the release namespace. Set `onepassworditem.items[]` with `{ item, name, type }`; the default example uses `existingSecret` for the OAuth client secret.

## Requirements

- NGINX Ingress Controller with `global-auth-url` / `global-auth-signin` pointing at this service (e.g. `https://auth.example.com/oauth2/auth` and `.../oauth2/start`).
- TLS for the auth host (ingress controller default cert or per-host cert).
- GitHub OAuth App with callback URL matching your config (e.g. `https://auth.example.com/oauth2/callback`).
- 1Password item for credentials (client-id, client-secret, cookie-secret) — see **GitHub OAuth + 1Password setup** below.

## GitHub OAuth + 1Password setup

1. **Create a GitHub OAuth App** (GitHub → Settings → Developer settings → OAuth Apps). Set **Authorization callback URL** to match `redirect_url` in the chart config (e.g. `https://auth.example.com/oauth2/callback`). Copy Client ID and Client secret.
2. **Generate a cookie secret:** e.g. `openssl rand -base64 32 | tr -d '\n'`. Use the raw output.
3. **1Password item:** Create an item (e.g. under `Kubernetes`) with three fields that map to the Kubernetes Secret keys: **`client-id`**, **`client-secret`**, **`cookie-secret`**. Set `onepassworditem.items[].item` to the 1Password path and `name` to the Secret name used in `config.existingSecret`.
4. **Chart config:** Set `config.existingSecret` to the Secret name created by onepassworditem. Ensure `redirect_url` in the configFile matches the GitHub OAuth App callback URL exactly.

## Usage

Override in your values (or Argo CD):

- **redirect_url** — Set to your auth host (e.g. `https://auth.example.com/oauth2/callback`).
- **cookie_domains**, **whitelist_domains** — Your domain (e.g. `.example.com`).
- **ingress.hosts** — Your auth hostname (e.g. `auth.example.com`).
- **onepassworditem.items[].item** — Your 1Password item path (e.g. `vaults/Kubernetes/items/oauth2-proxy`).

## Configuration (key values)

| Area | What to set |
|------|-------------|
| **Provider / URLs** | `config.configFile`: `provider`, `redirect_url`, `proxy_prefix`, `cookie_domains`, `whitelist_domains`. |
| **Secrets** | `config.existingSecret` — name of Secret created by onepassworditem (e.g. `oauth2-proxy-secret`). |
| **Cookie** | In configFile: `cookie_secure`, `cookie_httponly`, `cookie_expire`, `cookie_refresh`. |
| **Ingress** | `ingress.hosts`, `ingress.path`, `ingress.annotations`. |
| **Allowlist** | `extraEnv` e.g. `OAUTH2_PROXY_GITHUB_USERS` or `OAUTH2_PROXY_EMAIL_DOMAINS`. |

**Pitfalls:** Keep `show_debug_on_error = false` in production. Reloader annotation `secret.reloader.stakater.com/reload` on the deployment ensures the pod restarts when the credentials secret changes.

## Values (onepassworditem)

| Key | Description |
|-----|-------------|
| `onepassworditem.enabled` | If `true` (default), the subchart creates OnePasswordItem resources. Set `false` if you supply the secret another way. |
| `onepassworditem.items` | List of `{ item, name, type }`. `name` must match `config.existingSecret`. |

## Install

**From this repo:**

```bash
helm dependency update
helm install oauth2-proxy . -f my-values.yaml -n edge --create-namespace
```

**From Helm repo (expectedbehaviors):**

```bash
helm repo add expectedbehaviors https://expectedbehaviors.github.io/oauth2-proxy
helm install oauth2-proxy expectedbehaviors/oauth2-proxy -f my-values.yaml -n edge --create-namespace
```

## Argo CD

Deploy as an Argo CD Application (Helm, from this repo). Changes to values in git are applied on sync. Upgrade the oauth2-proxy dependency by bumping the version in Chart.yaml and running `helm dependency update`.

## Template filenames

This chart wraps the upstream OAuth2-Proxy chart and adds onepassworditem; template filenames follow the upstream and the expectedbehaviors OnePasswordItem subchart.
