# lfx26-t3-test-cluster — GitOps repo (Argo CD + Kubeflow on OKE)

Test cluster: <https://cloud.oracle.com/containers/clusters/ocid1.cluster.oc1.iad.aaaaaaaaejqldnhqdjkzd366aey7prmf65i7hk75cnhwuchwpcpr53ata63a/details?region=us-ashburn-1>

This repo is the **single source of truth** for what runs on the cluster.
Argo CD (in-cluster, namespace `argocd`, manual sync everywhere) compares the
cluster against `main` and a human approves every apply.

## Layout

```text
argocd/
  root.yaml            # app-of-apps: watches argocd/apps/ (apply this ONCE by hand)
  projects/kubeflow.yaml  # AppProject guardrail: allowed repo + namespaces + cluster kinds
  apps/                # one Application CR per component (root manages these)
platform/              # per-component kustomizations; upstream = kubeflow/community-distribution
                       # pinned via remote bases ?ref=26.03.1 (change ref here to upgrade)
apps/hello/            # phase 2 practice app (nginx)
docs/                  # setup notes per phase
```

## Sync waves (order matters — wait for green before the next wave)

| Wave | Apps | Why this order |
| ---- | ---- | -------------- |
| 0 | `cert-manager` | webhook certs for everything else |
| 1 | `istio` | service mesh; sidecar injection webhook |
| 2 | `dex`, `oauth2-proxy` | auth stack (needs istio) |
| 3 | `kubeflow-core` | kubeflow ns, roles, Gateway/auth policies |
| 4 | `pipelines`, `dashboard`, `notebooks` | the platform itself |
| 5 | `user-namespace` | example Profile (profile-controller must exist first) |

## Phase 3 bootstrap (run once)

```bash
# 0. always: confirm you are on the TEST cluster
kubectl config current-context

# 1. after reviewing + pushing this repo to main:
kubectl apply -f argocd/projects/kubeflow.yaml
kubectl apply -f argocd/root.yaml     # the LAST manual kubectl apply

# 2. In the Argo CD UI: sync `root` (creates all child apps),
#    then sync children wave by wave (0 → 5), waiting for Healthy between waves.
```

⚠️ **Capacity**: full wave 4 needs more than 2 × 1.8 allocatable vCPU — scale the
node pool (e.g. 2 × 4 OCPU or 3 nodes) before syncing wave 4, or expect Pending pods.

## Access (no LoadBalancer on purpose)

```bash
# Argo CD UI
kubectl port-forward svc/argocd-server -n argocd 8080:443     # https://localhost:8080

# Kubeflow dashboard (after wave 4)
kubectl port-forward svc/istio-ingressgateway -n istio-system 8081:80   # http://localhost:8081
# default login user@example.com / 12341234 — CHANGE IT (dex config) before any real exposure
```

## Runbook

- **Change anything**: edit files → PR → merge to `main` → review diff in Argo CD → Sync.
- **Rollback**: `git revert <commit>` → push → Sync. Git log = deployment history.
- **Add a component** (e.g. Katib later): add `platform/<name>/kustomization.yaml`
  (remote base pinned to the release tag) + `argocd/apps/<name>.yaml` → sync `root` → sync the new app.
- **Remove a component**: delete its two files → sync `root` **with Prune ticked**
  (child app deletion) — prune is off by default so this is always a deliberate act.
- **Upgrade Kubeflow**: bump every `?ref=` in `platform/*` on a branch, open a PR,
  point a throwaway Application at the branch to preview diffs, then merge + sync.
- **Sync fails "not permitted in project kubeflow"**: the AppProject whitelist is doing
  its job — review the named kind, add it to `argocd/projects/kubeflow.yaml`, push, re-sync.
- **Pod stuck `ImageInspectError` "short name mode is enforcing"**: an image name without
  a registry host — patch it to a fully-qualified name (e.g. `docker.io/library/…`) in a
  kustomize patch under `platform/<component>/`.

## Prod notes (phase 5, later)

Same repo + apps against the prod cluster; keep manual sync + no prune; verify the
kube-context OCID against the prod OCID **every session**; rotate Dex + Argo CD admin
credentials; decide UI exposure (port-forward vs LB + TLS + SSO) before install.
