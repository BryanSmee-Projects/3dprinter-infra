# 3dprinter-infra

GitOps manifests for 3D-printer infrastructure, deployed via Argo CD.

## Structure

```
argocd/            Argo CD Application definitions
  manyfold.yaml    Manyfold (Helm chart) + HTTPRoute, multi-source app
apps/              Extra Kubernetes manifests layered on top of the charts
  manyfold/
    httproute.yaml Gateway API HTTPRoute (HTTP only)
```

## Manyfold

[Manyfold](https://manyfold.app) — self-hosted digital asset manager for 3D print files.

Deployed via the community Helm chart
[`jeffresc/manyfold`](https://artifacthub.io/packages/helm/jeffresc/manyfold)
(`https://charts.jeffresc.dev`, chart `1.0.3`). The chart bundles PostgreSQL and
Valkey (Redis) as subcharts, replacing the `postgres` and `redis` services from
the reference docker-compose.

Because the chart only ships an `Ingress` (not Gateway API), the Argo CD
Application is **multi-source**: the chart provides the app + DB + Redis, and a
second source (this repo, `apps/manyfold/`) provides the Gateway API
`HTTPRoute`.

- **Image:** `ghcr.io/manyfold3d/manyfold` (chart appVersion)
- **Service:** `manyfold:3214` (pinned via `fullnameOverride`)
- **Ingress:** `HTTPRoute` on `manyfold.smee.ovh`, attached to
  `traefik/main-gateway`, **HTTP only** — HTTPS is terminated by a reverse proxy
  in front of the cluster Gateway.
- **Storage:** 20Gi model-library PVC (`/manyfold-library`) + the PostgreSQL
  subchart's own PVC.

### Reverse proxy requirement

Manyfold runs with `HTTPS_ONLY=enabled` so it emits `https://` URLs and secure
cookies. The upstream reverse proxy **must** forward `X-Forwarded-Proto: https`,
or the app will redirect-loop.

### OIDC (PocketID)

Single sign-on is pre-wired for [PocketID](https://pocket-id.org) via
`MULTIUSER=enabled` and the `OIDC_*` env vars. In PocketID, create an OIDC
client and register this callback URL:

```
https://manyfold.smee.ovh/users/auth/openid_connect/callback
```

Manyfold requests fixed scopes `openid email profile`.

### ⚠️ Placeholder values to replace before applying

All in `argocd/manyfold.yaml` (committed to git in plaintext — consider moving
to a Secret / sealed-secrets):

| Value | Placeholder |
|-------|-------------|
| `SECRET_KEY_BASE` | `CHANGE_ME_long_random_string` — `openssl rand -hex 64` |
| PostgreSQL password | `CHANGE_ME_db_password` |
| `OIDC_ISSUER` | `https://pocketid.smee.ovh` — your PocketID URL |
| `OIDC_CLIENT_ID` | `CHANGE_ME_oidc_client_id` |
| `OIDC_CLIENT_SECRET` | `CHANGE_ME_oidc_client_secret` |

Apply the Argo CD Application once and it syncs the rest:

```sh
kubectl apply -f argocd/manyfold.yaml
```
