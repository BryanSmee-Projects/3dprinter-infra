# 3dprinter-infra

GitOps manifests for 3D-printer infrastructure, deployed via Argo CD.

## Structure

```
argocd/            Argo CD Application definitions (app-of-apps entrypoints)
  spoolman.yaml
apps/              Kubernetes manifests for each app
  spoolman/        Spoolman filament tracker (Kustomize)
```

## Spoolman

[Spoolman](https://github.com/Donkie/Spoolman) filament/spool manager.

- **Image:** `ghcr.io/donkie/spoolman:latest`
- **Container port:** `8000` (HTTP)
- **Data:** persisted on a PVC mounted at `/home/app/.local/share/spoolman`
- **Ingress:** Gateway API `HTTPRoute`, **HTTP only** — HTTPS is terminated by a
  reverse proxy in front of the cluster Gateway.

### Values to review before applying

| File | Field | Default | Notes |
|------|-------|---------|-------|
| `apps/spoolman/httproute.yaml` | `parentRefs` | `gateway/gateway`, `sectionName: http` | Point at your cluster's Gateway + HTTP listener. |
| `apps/spoolman/httproute.yaml` | `hostnames` | `spoolman.local` | Hostname the reverse proxy forwards. |
| `apps/spoolman/pvc.yaml` | `storageClassName` | cluster default | Uncomment to pin a class. |
| `argocd/spoolman.yaml` | `source.repoURL` / `targetRevision` | `main` | Adjust if the default branch differs. |

Apply the Argo CD Application once and it will sync the rest:

```sh
kubectl apply -f argocd/spoolman.yaml
```
