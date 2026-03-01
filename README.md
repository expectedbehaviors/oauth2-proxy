# OAuth2-Proxy Helm Chart

Baseline for [OAuth2-Proxy](https://github.com/oauth2-proxy/oauth2-proxy) with GitHub OAuth and 1Password-backed secrets. No sensitive values in defaults.

## Usage

Override in your values (or ArgoCD):

- **redirect_url** — Set to your auth host (e.g. `https://auth.yourdomain.com/oauth2/callback`).
- **cookie_domains**, **whitelist_domains** — Your domain (e.g. `.yourdomain.com`).
- **ingress.hosts** — Your auth hostname.
- **OAUTH2_PROXY_GITHUB_USERS** — Add as `extraEnv` if you want user allowlist.
- **onepassworditem.items[].item** — Your 1Password item path for the OAuth client secret.

## Install

```bash
helm dependency update
helm install oauth2-proxy . -f my-values.yaml -n edge --create-namespace
```
