# argocd-applications

**Purpose:** Holds all workload manifests and Argo CD `Application` objects, organized by **app → env → cluster**.
**Pair repo:** [argocd-controlplane](https://github.com/Fusekla/argocd-controlplane) installs/configures Argo CD and applies the single Root Application that points here.

## 📂 Repository layout

```bash
argocd-applications/
├─ README.md
├─ LICENSE
├─ docs/
│  └─ progress.md                 # change log / notes
├─ projects/
│  └─ platform.yaml               # Argo CD AppProject (guardrails)
├─ apps/                          # application source-of-truth (Kustomize)
│  └─ guestbook/
│     ├─ base/                    # generic manifests (no env assumptions)
│     │  ├─ kustomization.yaml
│     │  ├─ deployment.yaml
│     │  └─ service.yaml
│     └─ overlays/                # env-specific tweaks (namespace, patches)
│        ├─ local/
│        │  ├─ kustomization.yaml
│        │  └─ namespace.yaml
│        ├─ dev/
│        ├─ stage/
│        └─ prod/
├─ envs/                          # which apps/overlays belong to an env
│  ├─ local/
│  │  ├─ apps/
│  │  │  └─ guestbook.yaml        # Argo CD Application → apps/guestbook/overlays/local
│  │  └─ kustomization.yaml       # aggregates env Applications
│  ├─ dev/
│  ├─ stage/
│  └─ prod/
└─ clusters/                      # app-of-apps entry per cluster
   └─ local/
      ├─ kustomization.yaml       # usually includes ../../envs/local (+ cluster extras)
      └─ root.yaml                # optional root-in-repo; Root App lives in controlplane repo

templates/                         # optional scaffolds for new apps
└─ kustomize-deployment/
   ├─ kustomization.yaml
   └─ deployment.yaml
```

## Separation of concerns

- `apps/` → Workload manifests. Keep `base/` pure (no namespaces/hostnames). Put env diffs in `overlays/<env>/`
- `envs/` → Argo `Application` CRDs per environment (one per app). `kustomization.yaml` bundles the env
- `clusters/` → App-of-apps entry for each cluster. Lets a cluster "follow" an env bundle (and any cluster-only extras)
- `projects/` → Argo `AppProject` policies (source repos, destinations, allowed kinds)

## How this repo binds to argocd-controlplane

In [argocd-controlplane/clusters/root/root-app.yaml](https://github.com/Fusekla/argocd-controlplane/blob/main/clusters/root/root-app.yaml), the **Root Application** points to this repo:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: root-local
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/Fusekla/argocd-applications
    targetRevision: main
    path: clusters/local
    directory:
      recurse: true
  destination:
    server: https://kubernetes.default.svc
    namespace: argocd
  syncPolicy: {}   # manual sync; prune disabled
```

The Root App is **applied from the controlplane repo** after Argo CD is installed there.

## 🚀 Quick start (local cluster example)

1. Populate app base & overlay
- `apps/guestbook/base/{deployment.yaml,service.yaml}`
- `apps/guestbook/overlays/local/{kustomization.yaml,namespace.yaml}` (must include `../../base`)
2. Create the env Application
- `envs/local/apps/guestbook.yaml` (Argo `Application` → `apps/guestbook/overlays/local`)
- `envs/local/kustomization.yaml` includes `apps/guestbook.yaml`
3. Cluster entry
- `clusters/local/kustomization.yaml` includes `../../envs/local`
- (Optional) `clusters/local/root.yaml` if you also want an in-repo root
4. From `argocd-controlplane`
- `kubectl apply -k clusters/local` (install Argo CD)
- `kubectl apply -f root/root-app.yaml` (Root App → this repo)
- In Argo UI: open **root-local** and **Sync** (manual, prune off)

## Compile checks (run before PRs)

```bash
# Workload manifests
kustomize build apps/guestbook/base
kustomize build apps/guestbook/overlays/local

# Env bundle (renders Argo Applications)
kustomize build envs/local

# Cluster bundle (renders root + env bundle)
kustomize build clusters/local
```

If any build fails with “no resources specified”, ensure each `kustomization.yaml` lists at least one resource

## Conventions

- **Naming (suggested):**
  - Apps (Argo): `app-<name>-<env>` (unique per Argo instance)
  - Root (cluster): `root-<cluster>`
- **Labels:** apply Kubernetes recommended labels in base (`app.kubernetes.io/name`, `part-of`, etc.)
- **Namespaces:** set in overlays (or env), not in base. Commit a `namespace.yaml` for auditability
- **Sync policy:** conservative at first — `syncPolicy: {}` (manual, prune disabled)
- **Sync waves** (later): use `argocd.argoproj.io/sync-wave` for ordering (e.g., CRDs: `-1`, namespaces: `-1`, apps: `0/1`)

## Adding a new app (5 steps)

1. **Scaffold app:** `apps/<app>/base/` with minimal `deployment.yaml`, `service.yaml`, ``.
2. **Create overlay:** `apps/<app>/overlays/<env>/kustomization.yaml` with:
- `namespace: <ns>`, `resources: ["../../base"]`, optional patches
3. **Env Application:** `envs/<env>/apps/<app>.yaml` (Argo Appl`ication → the overlay path)
4. **Bundle env:** add the new file to `envs/<env>/kustomization.yaml`
5. **Compile & PR:** run the build checks above; open a PR

## AppProject (policies)

- `projects/platform.yaml` defines:
  - `sourceRepos:` allow this repo (and others if needed)
  - `destinations:` typically server: `https://kubernetes.default.svc`, `namespace: '*'`
  - Whitelists for cluster/namespace resources (tighten over time)
- Point `spec.project` of Applications to this project when ready

## CI (optional but recommended)

- Add a lightweight CI to render key paths:
  - `kustomize build apps/**/base`
  - `kustomize build apps/**/overlays/*`
  - `kustomize build envs/*`
  - `kustomize build clusters/*`
- Fail the job on any render errors to catch drift early

## Troubleshooting

- **Root App shows no children:** verify `repoURL`, `targetRevision`, `path: clusters/<cluster>`, and `directory.recurse: true`
- **App stuck OutOfSync:** expected with manual sync; press **Sync** in UI (or CLI)
- **Namespace issues:** ensure `namespace.yaml` is committed in overlay and `destination.namespace` matches
- **Missing CRDs:** if an app needs CRDs, deploy the CRDs with **earlier sync wave** (e.g., `-1`) and sync order
- **RBAC/permissions:** adjust in `argocd-controlplane` (that repo owns `argocd-rbac-cm`)

## Relationship to `argocd-controlplane`

- Controlplane installs Argo CD and applies the **Root Application** that points to `clusters/<cluster>` here
- This repo then defines everything Argo CD should deploy (by env and app)
- Controlplane = *platform plumbing*. Applications repo = *workloads topology*