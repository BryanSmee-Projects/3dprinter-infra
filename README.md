# 3dprinter-infra

GitOps manifests for 3D-printer infrastructure, deployed via Argo CD.

## Structure

```
argocd/            Argo CD Application definitions
  manyfold.yaml    Manyfold (Helm chart) + plain DB/cache + HTTPRoute (multi-source)
apps/              Kubernetes manifests layered on top of the chart
  manyfold/
    postgres.yaml     PostgreSQL (official image) StatefulSet + Service
    redis.yaml        Redis (official image) Deployment + Service
    httproute.yaml    Gateway API HTTPRoute (HTTP only)
    sealedsecret.yaml Bitnami SealedSecret -> manyfold-secrets (you generate; see below)
```

## Manyfold

[Manyfold](https://manyfold.app) — self-hosted digital asset manager for 3D print files.

The app is deployed via the community Helm chart
[`jeffresc/manyfold`](https://artifacthub.io/packages/helm/jeffresc/manyfold)
(`https://charts.jeffresc.dev`, chart `1.0.3`).

The chart's bundled **Bitnami** PostgreSQL/Valkey subcharts are **disabled**:
Broadcom now ships only a subscription-gated, FIPS-hardened `latest` tag for
Bitnami images, which breaks the chart's default md5 database auth (exit-1
crashloop). Instead we run **plain official `postgres` and `redis` images** as
our own manifests — closer to the reference docker-compose and not subject to
Bitnami's licensing/FIPS changes.

The Argo CD Application is therefore **multi-source**:
1. the chart — the Manyfold app only (`postgresql.enabled=false`, `valkey.enabled=false`);
2. this repo (`apps/manyfold/`) — PostgreSQL, Redis, the `HTTPRoute`, and the SealedSecret.

- **Image:** `ghcr.io/manyfold3d/manyfold` (chart appVersion)
- **App Service:** `manyfold:3214` (pinned via `fullnameOverride`)
- **Database:** `manyfold-postgres:5432` (`postgres:16-alpine`, 5Gi PVC)
- **Cache/queue:** `manyfold-redis:6379` (`redis:7-alpine`, no persistence)
- **Ingress:** `HTTPRoute` on `manyfold.smee.ovh`, attached to
  `traefik/main-gateway`, **HTTP only** — HTTPS is terminated by a reverse proxy
  in front of the cluster Gateway.
- **Model library:** 20Gi PVC mounted at `/manyfold-library`.

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

## Secrets — SealedSecret (`manyfold-secrets`)

Sensitive values live in a Kubernetes Secret named `manyfold-secrets` in the
`manyfold` namespace, holding three keys: `secret-key-base`, `postgres-password`
(shared by the Postgres server and Manyfold's `DATABASE_PASSWORD`), and
`oidc-client-secret`. The Secret is created from a **SealedSecret**, which is
safe to commit to git.

Generate it (requires the `sealed-secrets` controller in-cluster and `kubeseal`):

```sh
# 1. Build the plaintext Secret locally — DO NOT COMMIT this file.
kubectl create secret generic manyfold-secrets \
  --namespace manyfold \
  --from-literal=secret-key-base="$(openssl rand -hex 64)" \
  --from-literal=postgres-password="$(openssl rand -base64 24)" \
  --from-literal=oidc-client-secret="<paste the secret from PocketID>" \
  --dry-run=client -o yaml > /tmp/manyfold-secret.plain.yaml

# 2. Seal it into the repo. Adjust --controller-name/--controller-namespace to
#    match your sealed-secrets install (defaults shown).
kubeseal --format yaml \
  --controller-name sealed-secrets \
  --controller-namespace kube-system \
  < /tmp/manyfold-secret.plain.yaml \
  > apps/manyfold/sealedsecret.yaml

# 3. Destroy the plaintext, then commit the sealed version.
shred -u /tmp/manyfold-secret.plain.yaml   # or: rm
git add apps/manyfold/sealedsecret.yaml && git commit -m "Add manyfold sealed secret"
```

> The SealedSecret is scoped to namespace `manyfold` + name `manyfold-secrets`
> by default (strict scope) — keep those exact values or it won't decrypt.

### Still to fill in (non-secret, in `argocd/manyfold.yaml`)

| Value | Placeholder |
|-------|-------------|
| `OIDC_ISSUER` | `https://pocketid.smee.ovh` — your PocketID URL |
| `OIDC_CLIENT_ID` | `CHANGE_ME_oidc_client_id` |

## Deploy

```sh
# after the SealedSecret is committed and Argo CD can see this repo
kubectl apply -f argocd/manyfold.yaml
```
