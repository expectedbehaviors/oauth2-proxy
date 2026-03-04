# OAuth2-Proxy Helm Chart

Baseline for [OAuth2-Proxy](https://github.com/oauth2-proxy/oauth2-proxy) with OIDC-first defaults and 1Password-backed secrets. Uses upstream chart plus [expectedbehaviors/OnePasswordItem-helm](https://github.com/expectedbehaviors/OnePasswordItem-helm). No sensitive values in defaults.

## Subcharts

| Subchart | Source | Values prefix | Description |
|----------|--------|---------------|-------------|
| **oauth2** (oauth2-proxy) | [oauth2-proxy/manifests](https://github.com/oauth2-proxy/manifests) | `oauth2.*` | Upstream OAuth2-Proxy: config, ingress, deployment, autoscaling. |
| **onepassworditem** | [expectedbehaviors/OnePasswordItem-helm](https://github.com/expectedbehaviors/OnePasswordItem-helm) | `onepassworditem.*` | Syncs 1Password item into a Secret for client-id, client-secret, cookie-secret. |

All inputs: **`oauth2.config`** (configFile, existingSecret), **`oauth2.ingress`**, **`oauth2.extraEnv`**, **`oauth2.deploymentAnnotations`**, **`onepassworditem.enabled`**, **`onepassworditem.items`**. Defaults: see `values.yaml`.

## Configuration reference (all inputs)

Every value accepted by this chart is documented below. Values are grouped by subchart. Source: upstream [oauth2-proxy/manifests](https://github.com/oauth2-proxy/manifests) (as `oauth2.*`) and [expectedbehaviors/OnePasswordItem-helm](https://github.com/expectedbehaviors/OnePasswordItem-helm) (as `onepassworditem.*`).

### Subchart: oauth2 (upstream oauth2-proxy)

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `oauth2.global.imageRegistry` | string | `""` | Global image registry. |
| `oauth2.global.imagePullSecrets` | list | `[]` | Image pull secrets. |
| `oauth2.config.annotations` | object | `{}` | Config annotations. |
| `oauth2.config.clientID` | string | `"XXXXXXX"` | OAuth client ID (prefer `existingSecret`). |
| `oauth2.config.clientSecret` | string | `"XXXXXXXX"` | OAuth client secret (prefer `existingSecret`). |
| `oauth2.config.cookieSecret` | string | — | Cookie secret (prefer `existingSecret`). |
| `oauth2.config.requiredSecretKeys` | list | `[client-id, client-secret, cookie-secret]` | Secret keys to expose as env. |
| `oauth2.config.existingSecret` | string | — | Use existing Secret for OAuth credentials (recommended; use onepassworditem to create it). |
| `oauth2.config.cookieName` | string | `""` | Cookie name (defaults to release name). |
| `oauth2.config.configFile` | string | — | Full oauth2_proxy config file (provider, redirect_url, cookie_domains, etc.). |
| `oauth2.config.existingConfig` | string | — | Use existing ConfigMap for config. |
| `oauth2.extraArgs` | object | `{}` | Extra container args. |
| `oauth2.extraEnv` | list | `[]` | Extra env vars (e.g. `OAUTH2_PROXY_GITHUB_USERS`). |
| `oauth2.image.registry` | string | `""` | Image registry. |
| `oauth2.image.repository` | string | `"oauth2-proxy/oauth2-proxy"` | Image repository. |
| `oauth2.image.tag` | string | `""` | Image tag (defaults to appVersion). |
| `oauth2.image.pullPolicy` | string | `"IfNotPresent"` | Pull policy. |
| `oauth2.service.type` | string | `"ClusterIP"` | Service type. |
| `oauth2.service.portNumber` | number | `80` | Service port (4180 for upstream default). |
| `oauth2.ingress.enabled` | bool | `false` | Enable Ingress. |
| `oauth2.ingress.hosts` | list | — | Hosts (e.g. `[auth.example.com]`). |
| `oauth2.ingress.path` | string | `"/"` | Path (e.g. `/oauth2`). |
| `oauth2.ingress.pathType` | string | `"ImplementationSpecific"` | Path type (e.g. `Prefix`). |
| `oauth2.ingress.annotations` | object | `{}` | Ingress annotations. |
| `oauth2.ingress.tls` | list | `[]` | TLS (secretName, hosts). |
| `oauth2.deploymentAnnotations` | object | `{}` | Annotations on Deployment (e.g. Reloader). |
| `oauth2.replicaCount` | int | `1` | Replicas. |
| `oauth2.autoscaling.enabled` | bool | `false` | Enable HPA. |
| `oauth2.autoscaling.minReplicas` | int | `1` | HPA min replicas. |
| `oauth2.autoscaling.maxReplicas` | int | `10` | HPA max replicas. |
| `oauth2.autoscaling.targetCPUUtilizationPercentage` | int | `80` | HPA target CPU. |
| `oauth2.resources` | object | `{}` | Container resources. |
| `oauth2.podSecurityContext` | object | `{}` | Pod security context. |
| `oauth2.securityContext` | object | — | Container security context (runAsNonRoot, readOnlyRootFilesystem, etc.). |
| `oauth2.livenessProbe` | object | — | Liveness probe. |
| `oauth2.readinessProbe` | object | — | Readiness probe. |
| `oauth2.extraVolumes` | list | `[]` | Extra volumes. |
| `oauth2.extraVolumeMounts` | list | `[]` | Extra volume mounts. |
| `oauth2.podDisruptionBudget.enabled` | bool | `true` | Enable PDB. |
| `oauth2.metrics.enabled` | bool | `true` | Enable Prometheus metrics. |
| `oauth2.sessionStorage.type` | string | `"cookie"` | Session storage (cookie or redis). |

### Subchart: onepassworditem

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `onepassworditem.enabled` | bool | `true` | Create OnePasswordItem resources; set `false` if you supply the secret another way. |
| `onepassworditem.defaultVault` | string | `""` | Default vault (id or title) for items that don't set `vault`. |
| `onepassworditem.items` | list | `[]` | List of `{ item, name, type }`; optional per-item `vault`, `namespace`, `annotations`, `labels`. `name` must match `oauth2.config.existingSecret`. |

## Chart contents

- **OAuth2-Proxy:** Upstream subchart; config via `oauth2.config.configFile` and `oauth2.extraEnv`; credentials from existing secret (e.g. `oauth2-proxy-secret`).
- **Secrets:** onepassworditem subchart syncs a 1Password item into a Kubernetes Secret (client-id, client-secret, cookie-secret). Reference it with `config.existingSecret`.
- **Ingress:** Host and path (e.g. `auth.example.com`, `/oauth2`); typically global auth is disabled on this ingress so the login flow is not protected by itself.
- **Reloader:** Annotate the deployment so [Stakater Reloader](https://github.com/stakater/Reloader) restarts the pod when the credentials secret changes.

## Features

- **External auth** — Exposes `/oauth2/auth`, `/oauth2/start`, `/oauth2/callback` for ingress-nginx `global-auth-url` / `global-auth-signin`.
- **OIDC-first provider model** — Baseline now uses `provider = "oidc"` with `oidc_issuer_url`; supports standards-compliant IdPs directly, and Plex via an OIDC bridge.
- **Cookie security** — HttpOnly, Secure, SameSite, CSRF, refresh interval; set cookie domain for your domain (e.g. `.example.com`).

## Prerequisites

- **1Password Connect** (optional): if you use `onepassworditem.enabled: true`. Secrets are created in the release namespace. Set `onepassworditem.items[]` with `{ item, name, type }`; the default example uses `existingSecret` for the OAuth client secret.

## Requirements

- NGINX Ingress Controller with `global-auth-url` / `global-auth-signin` pointing at this service (e.g. `https://auth.example.com/oauth2/auth` and `.../oauth2/start`).
- TLS for the auth host (ingress controller default cert or per-host cert).
- OIDC client with callback URL matching your config (e.g. `https://auth.example.com/oauth2/callback`).
- 1Password item for credentials (client-id, client-secret, cookie-secret) — see **Plex-backed OIDC + 1Password setup** below.
- For Plex login specifically: a Plex-to-OIDC bridge that exposes standard OIDC discovery, auth, token, userinfo, and JWKS endpoints.

## Plex-backed OIDC + 1Password setup

Plex does not currently expose a standards-compliant OIDC provider endpoint directly. To use Plex identity with oauth2-proxy (or apps that require OIDC), run Plex behind an OIDC bridge and point consumers to that bridge issuer.

1. **Create or deploy an OIDC bridge for Plex**
   - The bridge must expose standard OIDC endpoints, including:
     - `/.well-known/openid-configuration`
     - authorization endpoint
     - token endpoint
     - userinfo endpoint
     - JWKS endpoint
   - Example output issuer: `https://plex-oidc.example.com`.
2. **Create an OIDC client in the bridge**
   - Set callback/redirect URL to match chart config:
     - `https://auth.example.com/oauth2/callback`
   - Copy the client ID and client secret from the bridge.
3. **Generate a cookie secret**
   - `openssl rand -base64 32 | tr -d '\n'`
4. **Create the 1Password item**
   - Create an item (for example in `Kubernetes`) with fields mapped to:
     - `client-id`
     - `client-secret`
     - `cookie-secret`
   - Set `onepassworditem.items[].item` to that item path and `name` to the same secret used by `oauth2.config.existingSecret`.
5. **Set chart values**
   - `provider = "oidc"`
   - `oidc_issuer_url = "https://plex-oidc.example.com"`
   - `redirect_url = "https://auth.example.com/oauth2/callback"`
   - Optionally set `scope`/claims handling in `oauth2.extraEnv` depending on bridge behavior.

### What must exist before deploy

| Resource to create | Why it is required | Example |
|---|---|---|
| Plex OIDC bridge service | Plex is not a native OIDC issuer for oauth2-proxy consumers | `https://plex-oidc.example.com` |
| OIDC client registration in bridge | oauth2-proxy needs `client-id` and `client-secret` | callback `https://auth.example.com/oauth2/callback` |
| 1Password item fields | onepassworditem syncs these into K8s Secret keys | `client-id`, `client-secret`, `cookie-secret` |
| Kubernetes Secret target name | oauth2-proxy reads credentials from existing secret | `oauth2-proxy-secret` |

### Feasibility matrix for other OIDC use-cases

| Target service | Direct Plex as OIDC provider | Plex via OIDC bridge | Notes |
|---|---|---|---|
| oauth2-proxy (this chart) | No | Yes | Works when bridge exposes standard OIDC endpoints and claims used by oauth2-proxy. |
| Portainer | No | Yes | Portainer requires OIDC/OAuth endpoints (auth, token, user data/JWKS). |
| Argo CD | No | Yes | Argo CD OIDC (direct or via Dex) needs a standards-compliant issuer and callback mapping. |
| Harbor | No | Yes | Harbor requires OIDC discovery and commonly `openid profile email` (+ groups if RBAC by group). |

`Direct Plex` is marked `No` because this workflow depends on standards OIDC metadata and token validation endpoints. If Plex adds first-party OIDC issuer support later, the matrix can be updated.

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
| **Allowlist** | `extraEnv` e.g. `OAUTH2_PROXY_EMAIL_DOMAINS`, group filters, or issuer-specific claim checks. |

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
