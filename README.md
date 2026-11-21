# lab-argo-cd

Source of truth for installing [Argo CD](https://argo-cd.readthedocs.io/) in the homelab at **https://argo.joeckr.dev**.

`chart/` is a thin wrapper around the upstream [`argo-cd`](https://github.com/argoproj/argo-helm/tree/main/charts/argo-cd) Helm chart plus a Gateway API `HTTPRoute`.

| Concern | Approach |
|---|---|
| Talos Linux | No host access needed; everything runs as ordinary unprivileged pods |
| Pod Security Standard | All components run as non-root, `drop: ALL`, no privilege escalation, `RuntimeDefault` seccomp, read-only root FS. Label the namespace `pod-security.kubernetes.io/enforce: restricted` |
| Storage | Redis HA persists to a Longhorn PVC (`storageClass: longhorn`) |
| Ingress | Traefik Kubernetes Gateway (`HTTPRoute` → `argocd-server:80`); server runs with `server.insecure=true` as TLS ends at Cloudflare |
| Public access | Cloudflare Tunnel → Traefik |
| Auth | Zitadel OIDC with PKCE (no client secret); local `admin` disabled |
| Git | Forgejo repositories and webhooks |

## Prerequisites

- Traefik installed with Gateway API enabled and an existing `Gateway` (set `httpRoute.gateway.*` in `chart/values.yaml`).
- Longhorn with a `longhorn` StorageClass.
- Cloudflare Tunnel with a public hostname `argo.joeckr.dev` pointing at the Traefik service (e.g. `http://traefik.traefik.svc:80`).
- A Zitadel **User Agent / Web** application using the **PKCE** auth method with:
  - Redirect URI: `https://argo.joeckr.dev/auth/callback`
  - Post-logout URI: `https://argo.joeckr.dev`
  - Roles/groups claim included in the ID token (Zitadel: enable "Add user roles to ID token" and map the role `argocd-admins`).
  Then set `issuer` and `clientID` in `argo-cd.configs.cm.oidc.config`.

## Install

```sh
kubectl create namespace argocd
kubectl label namespace argocd pod-security.kubernetes.io/enforce=restricted

mise run helm-i   # helm dependency build + helm upgrade --install argocd chart/ -n argocd
```

Other tasks: `mise run helm-l` (lint), `mise run helm-t` (render to `rendered.yaml`), `mise run helm-u` (uninstall).

## Forgejo

Secrets are never committed. Create them out-of-band:

```sh
# Repository credentials (use a read-only Forgejo token)
kubectl -n argocd create secret generic forgejo-repo \
  --from-literal=type=git \
  --from-literal=url=https://forgejo.joeckr.dev \
  --from-literal=username=argocd \
  --from-literal=password=<token>
kubectl -n argocd label secret forgejo-repo argocd.argoproj.io/secret-type=repo-creds
```

For push-triggered syncs, add a Forgejo webhook (Gogs type) to `https://argo.joeckr.dev/api/webhook` and store the shared secret under the `webhook.gogs.secret` key of the `argocd-secret` Secret.

## Notes

- `redis-ha` runs with a single replica to suit a small cluster; raise `replicas` for HA.
- With local admin disabled, make sure your Zitadel user carries the `argocd-admins` role before the first login, otherwise you only get `role:readonly`. Break-glass: temporarily set `admin.enabled: true`.
- Run `helm dependency build chart/` after bumping the dependency in `chart/Chart.yaml`.
- Not yet rendered or tested against a live cluster (no Helm in the authoring environment); validate with `mise run helm-l` and `mise run helm-t`.
