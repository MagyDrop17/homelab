# Phase 4 — Flux bootstrap

In this phase we install **FluxCD** into the cluster and give it a Git
repo to watch. From this point on, the cluster's desired state lives in
Git: anything you want running in the cluster you commit to the repo,
and Flux makes reality match. We stop running `kubectl apply` for app
deployments. We start writing YAML, committing it, and watching Flux
pull it in.

The repo Flux watches is the same `MagyDrop17/homelab` GitHub repo we
created in Phase 0 — that was the whole point of putting it on GitHub
early. Phase 5 will migrate this Git source to self-hosted Forgejo, but
GitHub is the bootstrap source today because Forgejo doesn't exist yet.

---

## Concepts

### What GitOps actually means in practice

The shift is subtle but important. Without GitOps, your workflow is:

    you → kubectl apply → cluster

The cluster's state is whatever the last person who ran `kubectl` made
it. There's no record of *why* it looks this way, no easy rollback, no
audit trail, and if two people run conflicting commands the last one
wins silently.

With GitOps:

    you → git commit → git push → (Flux pulls) → cluster

The cluster's state is *defined* by what's in Git. The cluster is
"eventually consistent" with the repo — Flux reconciles every few
minutes (default: 1 minute for the source, 10 minutes for Kustomize
applies). If someone manually edits a resource in the cluster, Flux
notices the drift on the next reconcile and overwrites it. The repo is
the single source of truth.

Practical consequences:

- **Rollback is `git revert`.** Bad deploy? Revert the commit, push,
  Flux undoes it.
- **Code review applies to infrastructure.** PRs to the homelab repo
  get reviewed (by you, or future-you) before they land.
- **You stop logging into the cluster to "fix things."** If you find
  yourself reaching for `kubectl edit`, that's a smell — change the
  YAML in Git instead.
- **Disaster recovery is `flux bootstrap` against a new cluster.**
  Re-point Flux at the same repo and the new cluster reconverges.

### Pull-based, not push-based

Some CI/CD tools push to the cluster (Jenkins, GitHub Actions running
`kubectl apply`). Flux is the opposite: it runs *inside* the cluster
and pulls from Git. The cluster only needs outbound access to GitHub,
not inbound access from a CI runner. For a homelab behind NAT this is
exactly the right shape — no ingress, no exposed kubeconfig, no shared
credentials between CI and prod.

### The Flux controllers

Flux is not one process; it's a small set of controllers that each do
one job, all installed into the `flux-system` namespace:

- **source-controller** — fetches Git repos, Helm repos, and OCI
  artifacts. Caches them, exposes them to the other controllers as
  internal HTTP URLs.
- **kustomize-controller** — takes a `Kustomization` resource (which
  points at a path inside a source) and applies the rendered manifests
  to the cluster.
- **helm-controller** — takes a `HelmRelease` resource and reconciles
  Helm charts.
- **notification-controller** — sends events outward (Slack, webhooks)
  and receives events inward (GitHub webhooks for instant reconcile).

There are also `image-reflector-controller` and
`image-automation-controller` for "watch a container registry, bump
the tag in Git when a new image appears." We won't install those yet —
they're a Phase 6+ concern.

### The two CRDs that matter today

Flux installs a lot of CRDs, but the two you'll use constantly are:

**`GitRepository`** — "here is a Git repo, here is the branch, check
it every N seconds." Bootstrap creates one of these pointing at
`MagyDrop17/homelab` on `main`. You'll rarely create more.

**`Kustomization`** — "take the manifests at this path inside that
GitRepository, render them with Kustomize, and apply them to the
cluster." This is the workhorse. Almost everything you deploy will be
behind a `Kustomization`.

(Note the lowercase-k `Kustomization` is Flux's CRD. There's also a
plain Kustomize file called `kustomization.yaml` — same word, different
thing. Confusing, sorry.)

### What `flux bootstrap` actually does

The bootstrap command is doing four things in sequence, and it helps
to know them so you can debug if it gets stuck halfway:

1. **Clones your repo locally** (in a temp dir).
2. **Generates the Flux controller manifests** and writes them to
   `clusters/rp-cluster/flux-system/gotk-components.yaml`. ("gotk" =
   "GitOps Toolkit", Flux's internal name.) Also writes a
   `gotk-sync.yaml` containing the `GitRepository` and `Kustomization`
   resources that point Flux back at this same folder. Plus a
   `kustomization.yaml` that ties them together.
3. **Commits and pushes** those files to your repo.
4. **Applies the manifests directly** to the cluster (this is the only
   non-GitOps step in the whole workflow — bootstrap has to run
   imperatively because there's no Flux yet to pull). Once the
   controllers are up, they read the `gotk-sync.yaml` from the repo
   and from now on Flux manages itself: any future change to the
   `flux-system/` folder gets reconciled by Flux.

That self-management is what makes Flux upgrades trivial later: bump
the version in `gotk-components.yaml`, commit, push, Flux upgrades
itself.

### Why `clusters/rp-cluster/` and the app-of-apps shape

Convention: one folder per cluster under `clusters/`. You only have
one cluster today, but the pattern scales — if you add a second
cluster (a cloud one, a separate test cluster), it gets its own
folder and its own bootstrap into the same repo.

Inside `clusters/rp-cluster/` you'll grow this layout over time:

    clusters/rp-cluster/
    ├── flux-system/          ← created by bootstrap, manages Flux itself
    ├── infrastructure.yaml   ← Kustomization → ../../infrastructure/
    └── apps.yaml             ← Kustomization → ../../apps/

The "app-of-apps" pattern means: bootstrap's root Kustomization watches
`clusters/rp-cluster/`, finds `infrastructure.yaml` and `apps.yaml`,
and each of those is itself a `Kustomization` pointing at a folder
full of more manifests. So Flux's reconciliation cascades:

    flux-system → infrastructure.yaml → infrastructure/* (Forgejo, Keycloak, ...)
                → apps.yaml           → apps/*           (workloads)

You add a new app by dropping its manifests into `apps/<name>/` and
referencing it from a top-level `kustomization.yaml`. No new
`flux bootstrap` ever needed.

We're not going to deploy any actual app in this phase — just bootstrap
Flux and put the empty scaffold in place. First real app (Forgejo)
comes in Phase 5.

---

## Pre-flight

From your WSL workstation, before you start:

```bash
# Cluster reachable?
kubectl get nodes
# Should show 3 nodes Ready.

# No existing flux-system namespace?
kubectl get ns flux-system
# Should say "not found". If it exists from an earlier attempt, delete
# it first: kubectl delete ns flux-system --wait

# Repo clean and pushed?
cd ~/homelab
git status
# Should say "nothing to commit, working tree clean" and
# "Your branch is up to date with 'origin/main'".

# Flux CLI version sane?
flux --version
# 2.x is fine.
```

If any of those fails, fix it before bootstrapping. A half-bootstrapped
cluster is annoying to clean up.

---

## Step 1 — Create a GitHub Personal Access Token

Flux needs to push to your repo (to commit `clusters/rp-cluster/flux-system/`)
and pull from it. The simplest auth is a fine-grained PAT.

1. GitHub → Settings → Developer settings → Personal access tokens →
   Fine-grained tokens → Generate new token.
2. Name: `flux-bootstrap-rp-cluster`.
3. Expiration: 90 days is reasonable. (You'll rotate this.)
4. Repository access: Only select repositories → `MagyDrop17/homelab`.
5. Repository permissions:
   - **Contents:** Read and write
   - **Metadata:** Read-only (this is auto-selected)
6. Generate, copy the token (`github_pat_...`).

Export it in your shell, along with your username:

```bash
export GITHUB_TOKEN=github_pat_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
export GITHUB_USER=MagyDrop17
```

These only need to live in this shell session. They are not committed
anywhere. After bootstrap, Flux stores its own copy as a Secret in the
cluster (`flux-system/flux-system`).

---

## Step 2 — Bootstrap

```bash
flux bootstrap github \
  --owner=$GITHUB_USER \
  --repository=homelab \
  --branch=main \
  --path=clusters/rp-cluster \
  --personal \
  --token-auth
```

Flag-by-flag, since this is the only time we run it:

- `--owner` / `--repository` — which repo to bootstrap into.
- `--branch=main` — which branch Flux watches and commits to.
- `--path=clusters/rp-cluster` — the folder inside the repo where
  Flux's own manifests live, and the folder Flux watches by default.
- `--personal` — the repo is under a user account, not an org. (If you
  ever move to an org, drop this flag.)
- `--token-auth` — use the PAT for all Git operations. The alternative
  is `--ssh` which generates a deploy key; PAT is simpler the first
  time.

What you'll see, roughly in order:

    ► connecting to github.com
    ► cloning branch "main" from Git repository
    ► generating component manifests
    ► committing component manifests
    ► pushing component manifests
    ► installing components in "flux-system" namespace
    ► reconciled components
    ► determining if source secret "flux-system/flux-system" exists
    ► generating source secret
    ► applying source secret
    ► generating sync manifests
    ► committing sync manifests
    ► pushing sync manifests
    ► applying sync manifests
    ► waiting for Kustomization "flux-system/flux-system" to be reconciled
    ► bootstrap finished

If it hangs at "waiting for Kustomization to be reconciled," open
another terminal and check controller logs:

```bash
kubectl -n flux-system get pods
kubectl -n flux-system logs deploy/source-controller
kubectl -n flux-system logs deploy/kustomize-controller
```

The most common first-time issue is the PAT not having write access
to the repo — bootstrap commits files, so read-only doesn't cut it.

---

## Step 3 — Verify

```bash
flux check
# Validates that all controllers are healthy and at the expected
# version. Should print a list of green checks.

flux get sources git
# Should show "flux-system" GitRepository, READY=True, with the latest
# commit SHA from main.

flux get kustomizations
# Should show "flux-system" Kustomization, READY=True.

kubectl -n flux-system get pods
# Four controllers, all Running:
#   source-controller
#   kustomize-controller
#   helm-controller
#   notification-controller
```

Then pull the bootstrap commits down to your local clone — Flux made
commits to your repo from the bootstrap machine and they're not in
your local checkout yet:

```bash
cd ~/homelab
git pull
git log --oneline -5
```

You should see commits authored by `flux <bot@flux.local>` adding
`clusters/rp-cluster/flux-system/`.

---

## Step 4 — Prove the loop works

Before scaffolding apps, do a tiny end-to-end test: commit something
to the repo and verify Flux applies it.

Create a throwaway namespace manifest:

```bash
mkdir -p clusters/rp-cluster
cat > clusters/rp-cluster/test-namespace.yaml <<'EOF'
apiVersion: v1
kind: Namespace
metadata:
  name: flux-test
  labels:
    purpose: bootstrap-verification
EOF

git add clusters/rp-cluster/test-namespace.yaml
git commit -m "test: verify Flux reconciliation"
git push
```

Within a minute or so:

```bash
kubectl get ns flux-test
# NAME        STATUS   AGE
# flux-test   Active   30s

flux get kustomizations
# flux-system should show the new commit SHA in the REVISION column.
```

If you want to watch it happen, in one terminal run:

```bash
flux reconcile source git flux-system
flux reconcile kustomization flux-system
```

`flux reconcile` is the "don't wait, do it now" button — useful while
testing, you won't reach for it day-to-day.

Now clean up:

```bash
git rm clusters/rp-cluster/test-namespace.yaml
git commit -m "test: remove verification namespace"
git push
```

After Flux reconciles, the namespace should be **deleted** from the
cluster. This is "garbage collection" — Flux tracks what it created
and removes anything that disappears from Git. Verify:

```bash
kubectl get ns flux-test
# Error from server (NotFound)
```

If both halves of that test passed, the GitOps loop works end-to-end.

---

## Step 5 — Scaffold the app-of-apps layout

Create the empty top-level folders and Kustomizations so Phase 5 is a
matter of dropping files into `infrastructure/forgejo/`, not designing
the structure under pressure.

```bash
cd ~/homelab
mkdir -p infrastructure apps clusters/rp-cluster

# Top-level Kustomization that points at infrastructure/
cat > clusters/rp-cluster/infrastructure.yaml <<'EOF'
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: infrastructure
  namespace: flux-system
spec:
  interval: 10m
  path: ./infrastructure
  prune: true
  sourceRef:
    kind: GitRepository
    name: flux-system
  wait: true
  timeout: 5m
EOF

# Top-level Kustomization that points at apps/
cat > clusters/rp-cluster/apps.yaml <<'EOF'
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: apps
  namespace: flux-system
spec:
  interval: 10m
  path: ./apps
  prune: true
  sourceRef:
    kind: GitRepository
    name: flux-system
  wait: true
  dependsOn:
    - name: infrastructure
  timeout: 5m
EOF

# Empty kustomization manifests in the target folders, so Flux has
# something valid to apply (an empty resource list is valid).
cat > infrastructure/kustomization.yaml <<'EOF'
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources: []
EOF

cat > apps/kustomization.yaml <<'EOF'
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources: []
EOF

git add clusters/rp-cluster/infrastructure.yaml \
        clusters/rp-cluster/apps.yaml \
        infrastructure/kustomization.yaml \
        apps/kustomization.yaml
git commit -m "flux: scaffold app-of-apps layout"
git push
```

Notes on what's in those Kustomizations:

- `interval: 10m` — re-reconcile every 10 minutes even if Git hasn't
  changed. Catches manual drift.
- `prune: true` — turn on garbage collection (the thing you just
  proved works in Step 4).
- `wait: true` — wait for the applied resources to become healthy
  before reporting success.
- `dependsOn` on `apps` — apps shouldn't try to deploy before
  infrastructure (Forgejo, ingress routes, etc.) is ready.

After the push, verify the new Kustomizations registered:

```bash
flux get kustomizations
# NAME             REVISION    READY
# apps             main@sha:... True
# flux-system      main@sha:... True
# infrastructure   main@sha:... True
```

Three READY=True rows means the scaffold is in place. They're applying
nothing right now (empty resource lists), which is exactly what we
want.

---

## Step 6 — Update the README

Edit `~/homelab/README.md` to mark Phase 4 done:

    ## Phases

    0. ✅ Workstation prep (WSL2 + toolchain)
    1. ✅ OS hardening with Ansible
    2. ✅ k3s install (HA)
    3. ✅ Platform layer (MetalLB, ingress, cert-manager, Longhorn)
    4. ✅ Flux bootstrap
    5. ⏳ Forgejo + migrate Flux source
    6. Identity, VPN, apps
    7. Observability + backups

Commit and push.

---

## Decisions made in this phase

- Flux bootstrap source: **`github.com/MagyDrop17/homelab`** on
  `main`, path `clusters/rp-cluster/`.
- Auth: GitHub PAT, fine-grained, scoped to this single repo. To be
  rotated every ~90 days. (Phase 5 will switch to a Forgejo deploy
  key when we migrate.)
- Cluster name (Flux's perspective): `rp-cluster`.
- App-of-apps layout: `infrastructure/` for platform-adjacent
  services, `apps/` for workloads, with `apps` depending on
  `infrastructure`.
- Reconcile intervals: 1 minute for the source, 10 minutes for the
  Kustomizations. Defaults are fine for a homelab.

## What Phase 5 will do

- Deploy **Forgejo** as the first real Flux-managed app. Manifests
  land in `infrastructure/forgejo/`. Flux pulls, applies, Forgejo
  comes up at a hostname behind ingress-nginx with a cert from
  cert-manager.
- Mirror the homelab repo from GitHub into Forgejo.
- Re-run `flux bootstrap` (or edit the `GitRepository` directly)
  pointing at Forgejo's URL with a Forgejo deploy key. From then on
  GitHub is just a backup mirror; Flux's source of truth is internal.
- Decide what to do about the GitHub PAT (probably: revoke it once
  the migration is done and verified).
