# Phase 5 — Forgejo + migrate Flux source

This phase has three acts:

1. **Deploy Forgejo** as the first real Flux-managed application.
2. **Mirror the homelab repo** from GitHub into Forgejo.
3. **Migrate Flux's Git source** from GitHub to Forgejo, so the
   cluster's source of truth is self-hosted.

By the end, the cluster manages its own Git server, which hosts the
repo that tells the cluster what to run. That's a satisfying loop to
close.

---

## Concepts

### Why Forgejo and not Gitea

Forgejo is a hard fork of Gitea that happened because of governance
concerns (Gitea was transferred to a for-profit company). Forgejo is
community-governed, has the same features, the same UI, the same API,
and the same Docker images (just re-tagged). It uses ~300–500 MB RAM
on ARM64 — very comfortable on a Pi with 16 GB. All Gitea docs and
community knowledge transfer directly.

### HelmRelease: Flux's way of deploying Helm charts

In Phase 3 you installed things with Ansible running `helm install`.
That was fine for bootstrapping the platform (Flux didn't exist yet).
Now that Flux is running, the GitOps-correct way to deploy a Helm
chart is to declare a **`HelmRelease`** in Git and let Flux manage it.

A HelmRelease is a Flux CRD that says: "install this Helm chart, from
this repo, at this version, with these values." Flux's helm-controller
watches it, runs the equivalent of `helm install` (or `helm upgrade`)
inside the cluster, and keeps the release in sync with what's in Git.
If you change a value in the HelmRelease and push, Flux upgrades the
release. If you delete the HelmRelease from Git, Flux uninstalls it.

To use a HelmRelease, you also need a **`HelmRepository`** — a source
that tells Flux where to find the chart (the OCI registry or HTTP
index URL). Think of it as `apt-add-repository` before `apt install`.

### DNS: the `.homelab.local` problem

The cluster is private. Forgejo's Ingress will serve traffic on
`git.homelab.local` via ingress-nginx on `192.168.1.241`. But your
WSL workstation, your browser, and `git` itself need to resolve
`git.homelab.local` → `192.168.1.241`, and there's no DNS server on
your LAN that knows about that name.

The simplest solution: **`/etc/hosts`** on your WSL workstation. One
line, works immediately, no extra infrastructure. The downside is
that every new hostname you add to the cluster (Keycloak, Grafana,
NetBird dashboard, ...) needs another hosts entry on every device
that wants to reach it. That's acceptable for now — you're the only
user and you access everything from one machine. In Phase 6 or 7 you
can deploy CoreDNS as a LAN resolver or set up Pi-hole if the hosts
file gets unwieldy.

For Git operations specifically (push, pull, clone), only the machine
running `git` needs DNS. That's your WSL workstation. The cluster
itself doesn't need to resolve `git.homelab.local` because Flux talks
to the Forgejo Service by its in-cluster DNS name
(`forgejo-http.forgejo.svc.cluster.local`), not by the Ingress
hostname.

### SSH vs HTTPS for Flux → Forgejo

When Flux watches GitHub, we used a PAT over HTTPS. For Forgejo we'll
switch to **SSH with a deploy key**. Reasons:

- A deploy key is scoped to one repo (read-only or read-write), which
  is the principle of least privilege.
- It's a keypair you generate and store as a Secret in the cluster —
  no PAT to rotate, no third-party auth dependency.
- It works over SSH port 22 (or whatever Forgejo's SSH port is),
  which is internal-only.

The flow: generate an SSH keypair, add the public key to the Forgejo
repo as a deploy key, store the private key as a Kubernetes Secret
that Flux reads.

---

## Pre-flight

```bash
# Cluster healthy?
kubectl get nodes
flux get kustomizations
# All three (flux-system, infrastructure, apps) should be READY=True.

# Platform components from Phase 3 — quick sanity check:
kubectl -n metallb-system get pods        # MetalLB running
kubectl -n ingress-nginx get pods         # ingress-nginx running
kubectl -n cert-manager get pods          # cert-manager running
kubectl get clusterissuer ca-issuer       # CA issuer ready

# Repo clean?
cd ~/homelab && git status
```

If any of those are unhealthy, fix them before continuing — Forgejo
depends on all four platform components.

---

## Step 1 — DNS: add hosts entry

On your **WSL workstation** (not on the Pis):

```bash
echo "192.168.1.241 git.homelab.local" | sudo tee -a /etc/hosts
```

Verify:

```bash
ping -c 1 git.homelab.local
```

Should resolve to `192.168.1.241`. It won't get a ping reply (that's
fine — ingress-nginx doesn't respond to ICMP), but the DNS resolution
is what matters. If `ping` says "Name or service not known," the
hosts entry didn't take.

**Note for future services:** every new `*.homelab.local` hostname you
add to the cluster will need a line in this file. Since they all
point at the same ingress-nginx IP (`192.168.1.241`), you can also
do one line with multiple names:

    192.168.1.241 git.homelab.local keycloak.homelab.local grafana.homelab.local

---

## Step 2 — Create the Forgejo Flux manifests

This is the first real use of the app-of-apps scaffold. All Forgejo
manifests go into `infrastructure/forgejo/`.

### Directory layout

```
infrastructure/
├── kustomization.yaml      ← update to include forgejo/
└── forgejo/
    ├── kustomization.yaml
    ├── namespace.yaml
    ├── helmrepository.yaml
    ├── helmrelease.yaml
    └── ingress.yaml
```

### `infrastructure/forgejo/namespace.yaml`

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: forgejo
```

### `infrastructure/forgejo/helmrepository.yaml`

```yaml
apiVersion: source.toolkit.fluxcd.io/v1
kind: HelmRepository
metadata:
  name: forgejo
  namespace: flux-system
spec:
  type: oci
  interval: 60m
  url: oci://code.forgejo.org/forgejo-helm
```

This tells Flux's source-controller: "there's a Helm chart at this
OCI registry, check it every 60 minutes for new versions." OCI
(rather than a classic Helm HTTP index) is how the Forgejo project
distributes their chart.

### `infrastructure/forgejo/helmrelease.yaml`

```yaml
apiVersion: helm.toolkit.fluxcd.io/v2
kind: HelmRelease
metadata:
  name: forgejo
  namespace: forgejo
spec:
  interval: 30m
  chart:
    spec:
      chart: forgejo
      version: "11.0.x"
      sourceRef:
        kind: HelmRepository
        name: forgejo
        namespace: flux-system
  values:
    image:
      rootless: true

    service:
      http:
        type: ClusterIP
        port: 3000
      ssh:
        type: LoadBalancer
        port: 22

    ingress:
      enabled: true
      className: nginx
      annotations:
        cert-manager.io/cluster-issuer: ca-issuer
      hosts:
        - host: git.homelab.local
          paths:
            - path: /
              pathType: Prefix
      tls:
        - secretName: forgejo-tls
          hosts:
            - git.homelab.local

    persistence:
      enabled: true
      storageClass: longhorn
      size: 10Gi

    gitea:
      config:
        server:
          DOMAIN: git.homelab.local
          ROOT_URL: https://git.homelab.local
          SSH_DOMAIN: git.homelab.local
          SSH_PORT: 22
          LFS_START_SERVER: true
        database:
          DB_TYPE: sqlite3
        service:
          DISABLE_REGISTRATION: false
        session:
          PROVIDER: memory
        cache:
          ADAPTER: memory
        queue:
          TYPE: level

    postgresql:
      enabled: false

    postgresql-ha:
      enabled: false

    redis:
      enabled: false

    redis-cluster:
      enabled: false
```

Key decisions in those values, explained:

- **`rootless: true`** — runs the Forgejo process as a non-root user
  inside the container. More secure, no practical downsides.
- **SQLite instead of Postgres** — for a single-user homelab Git
  server, SQLite is plenty and avoids deploying a whole Postgres
  instance. If you ever need multi-user with high concurrency, switch
  to Postgres later.
- **Redis/cache/queue disabled** — the in-memory and level-db
  alternatives are fine for a low-traffic instance and save ~200 MB
  RAM.
- **SSH service as LoadBalancer** — MetalLB will give Forgejo's SSH a
  dedicated IP (probably `.242`, the next free one in your pool). This
  lets you `git clone git@192.168.1.242:user/repo.git` or, once you
  add a DNS entry, `git@git.homelab.local:user/repo.git`.
- **10 Gi PVC on Longhorn** — Forgejo stores repos, LFS objects, and
  config here. Longhorn replicates it across NVMe drives. 10 Gi is
  generous for a homelab; expand later with `kubectl edit pvc` if
  needed (Longhorn supports online expansion).
- **Ingress with TLS** — cert-manager sees the
  `cert-manager.io/cluster-issuer: ca-issuer` annotation, issues a
  cert signed by your self-signed CA, stores it in the `forgejo-tls`
  Secret. ingress-nginx serves it. Your browser will show a warning
  (untrusted CA) unless you install the root CA cert — we'll do that
  later.
- **`version: "11.0.x"`** — pins to the 11.0 series but allows patch
  updates. Verify the latest chart version before applying; this may
  need adjusting.

### `infrastructure/forgejo/kustomization.yaml`

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - namespace.yaml
  - helmrepository.yaml
  - helmrelease.yaml
```

Note: we don't list `ingress.yaml` separately because the Helm
chart creates the Ingress for us via the `ingress:` values block.

### Update `infrastructure/kustomization.yaml`

Change the empty resources list to include the forgejo folder:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - forgejo
```

---

## Step 3 — Commit and let Flux deploy

```bash
cd ~/homelab
git add infrastructure/
git commit -m "infrastructure: deploy forgejo via helm"
git push

flux reconcile source git flux-system
flux reconcile kustomization flux-system
flux reconcile kustomization infrastructure
```

Then watch it come up:

```bash
# Flux sees the HelmRelease?
flux get helmreleases -n forgejo

# Pods coming up?
kubectl -n forgejo get pods -w
```

The Helm chart will create: a Deployment (the Forgejo server), a
PVC (Longhorn volume), two Services (HTTP + SSH), and an Ingress.
First startup takes 2–5 minutes because the container image needs
to pull and the PVC needs to provision.

Wait until the pod is `1/1 Running` and the HelmRelease shows
`READY=True` before continuing.

**Troubleshooting if it's stuck:**

```bash
# HelmRelease status (most useful)
flux get helmreleases -n forgejo

# Events on the HelmRelease
kubectl -n forgejo describe helmrelease forgejo

# Pod logs
kubectl -n forgejo logs deploy/forgejo -f

# PVC bound?
kubectl -n forgejo get pvc
```

Common issues: chart version not found (check exact version
available), PVC stuck in Pending (Longhorn issue), image pull
failure (ARM64 image availability).

---

## Step 4 — Verify Forgejo is accessible

From your WSL workstation:

```bash
# HTTP redirect to HTTPS?
curl -I http://git.homelab.local

# HTTPS (skip cert verification since self-signed CA)?
curl -kI https://git.homelab.local
```

You should get a `200 OK` or a `302` redirect to the setup/login
page. If you get `503 Service Temporarily Unavailable`, the pod
isn't ready yet or the Ingress backend isn't wired correctly.

Open `https://git.homelab.local` in your browser (Brave). You'll get
a certificate warning — expected, since your browser doesn't trust
the homelab CA. Accept it and you should see the Forgejo landing
page.

**First-time setup:**

1. Register the first user — this user becomes the admin.
2. Pick a username and password. Write them down.
3. After registration, go to Site Administration (top-right menu →
   Site Administration) and confirm you have admin access.

---

## Step 5 — Create the homelab repo in Forgejo

In the Forgejo web UI:

1. Click `+` → New Repository.
2. Name: `homelab`
3. Visibility: Private (it's internal-only anyway, but good habit).
4. Do NOT initialize with README — we're going to push an existing
   repo.
5. Create.

Forgejo will show you the "push an existing repository" instructions.
We'll do exactly that, but first we need to handle SSH auth.

---

## Step 6 — Push the homelab repo to Forgejo

### Get Forgejo's SSH IP

```bash
kubectl -n forgejo get svc
```

Find the SSH service — it should have a LoadBalancer EXTERNAL-IP
(something like `192.168.1.242`). Note this IP.

Add it to `/etc/hosts`:

```bash
# If you haven't already covered git.homelab.local for SSH:
# The SSH service has its own IP from MetalLB, but git operations
# over SSH use the hostname. Add an entry if the SSH IP differs
# from the ingress IP, or use the IP directly in the remote URL.
```

### Add Forgejo as a second remote

```bash
cd ~/homelab

# Add the Forgejo remote using HTTPS for the initial push
# (SSH setup comes next, for Flux)
git remote add forgejo https://git.homelab.local/YOUR_USERNAME/homelab.git

# Push (use -k to skip TLS verification for self-signed cert)
git -c http.sslVerify=false push forgejo main
```

Git will prompt for your Forgejo username and password. After the
push, refresh the Forgejo web UI — you should see all your files.

---

## Step 7 — Generate SSH keypair for Flux

This key will be Flux's credential for reading from Forgejo. It never
leaves the cluster.

```bash
ssh-keygen -t ed25519 -f ~/flux-forgejo-key -N "" -C "flux-readonly"
```

This creates `~/flux-forgejo-key` (private) and
`~/flux-forgejo-key.pub` (public).

### Add the public key as a deploy key in Forgejo

1. In Forgejo, go to the `homelab` repo → Settings → Deploy Keys.
2. Title: `flux-readonly`
3. Key: paste the contents of `~/flux-forgejo-key.pub`
4. Leave "Allow Write Access" **unchecked** — Flux only needs to
   read. (Unlike the GitHub bootstrap which needed write to commit
   `flux-system/`, the migration doesn't need write access because
   `flux-system/` already exists.)
5. Add Key.

### Store the private key as a Kubernetes Secret

```bash
kubectl create secret generic forgejo-ssh-key \
  --namespace=flux-system \
  --from-file=identity=~/flux-forgejo-key \
  --from-file=known_hosts=<(ssh-keyscan -p 22 FORGEJO_SSH_IP 2>/dev/null)
```

Replace `FORGEJO_SSH_IP` with the actual SSH LoadBalancer IP from
Step 6.

**What this does:** creates a Secret in `flux-system` containing the
private key (`identity`) and the SSH host fingerprint
(`known_hosts`). Flux's source-controller will mount this Secret
when it connects to Forgejo over SSH.

After the migration is verified, delete the local private key:

```bash
rm ~/flux-forgejo-key ~/flux-forgejo-key.pub
```

---

## Step 8 — Migrate Flux's GitRepository to Forgejo

This is the critical step. We're changing where Flux looks for the
repo — from GitHub to Forgejo.

Edit `clusters/rp-cluster/flux-system/gotk-sync.yaml`. Find the
`GitRepository` resource and update it:

**Before (GitHub, HTTPS + PAT):**

```yaml
apiVersion: source.toolkit.fluxcd.io/v1
kind: GitRepository
metadata:
  name: flux-system
  namespace: flux-system
spec:
  interval: 1m0s
  ref:
    branch: main
  secretRef:
    name: flux-system
  url: https://github.com/MagyDrop17/homelab.git
```

**After (Forgejo, SSH + deploy key):**

```yaml
apiVersion: source.toolkit.fluxcd.io/v1
kind: GitRepository
metadata:
  name: flux-system
  namespace: flux-system
spec:
  interval: 1m0s
  ref:
    branch: main
  secretRef:
    name: forgejo-ssh-key
  url: ssh://git@FORGEJO_SSH_IP/YOUR_USERNAME/homelab.git
```

Replace `FORGEJO_SSH_IP` with the SSH service's LoadBalancer IP and
`YOUR_USERNAME` with your Forgejo username.

**Important:** commit this change to **both** remotes — push to
GitHub first (so the current Flux picks it up), then to Forgejo (so
the new source has it too):

```bash
cd ~/homelab
git add clusters/rp-cluster/flux-system/gotk-sync.yaml
git commit -m "flux: migrate source from GitHub to Forgejo"
git push origin main
git -c http.sslVerify=false push forgejo main
```

Then trigger reconciliation:

```bash
flux reconcile source git flux-system
```

Watch what happens:

```bash
flux get sources git
```

The URL should now show the Forgejo SSH URL, and READY should be
True with a fresh revision. If it shows an error, check:

```bash
kubectl -n flux-system describe gitrepository flux-system
kubectl -n flux-system logs deploy/source-controller | tail -30
```

Common issues: wrong SSH IP, known_hosts mismatch, deploy key not
added to the repo, Secret name typo.

---

## Step 9 — Verify the full loop via Forgejo

The ultimate test: make a change, push it **only to Forgejo** (not
GitHub), and confirm Flux picks it up.

```bash
cd ~/homelab

# Make a trivial change
echo "# Flux source: Forgejo (self-hosted)" >> README.md
git add README.md
git commit -m "test: verify Flux reads from Forgejo"

# Push ONLY to Forgejo
git -c http.sslVerify=false push forgejo main

# Do NOT push to GitHub
```

```bash
flux reconcile source git flux-system
flux get sources git
flux get kustomizations
```

If the revision updates and all Kustomizations reconcile, **Flux is
now reading from Forgejo.** The migration is complete.

Push the same commit to GitHub to keep it in sync as a backup:

```bash
git push origin main
```

---

## Step 10 — Clean up

### Revoke the GitHub PAT

GitHub → Settings → Developer settings → Personal access tokens →
Fine-grained tokens → `flux-bootstrap-rp-cluster` → Delete.

The old `flux-system` Secret in the cluster (which held the GitHub
PAT) is no longer referenced by the GitRepository. You can delete it:

```bash
kubectl -n flux-system delete secret flux-system
```

### Set Forgejo as the default push remote

```bash
cd ~/homelab
git remote set-url --push origin \
  https://git.homelab.local/YOUR_USERNAME/homelab.git

# Or if you prefer, make forgejo the default:
# git remote rename origin github
# git remote rename forgejo origin
```

Your workflow going forward: push to Forgejo (the source of truth),
optionally push to GitHub (as a public mirror / backup).

### Disable Forgejo public registration

Now that your admin account exists, lock registration:

Edit `infrastructure/forgejo/helmrelease.yaml`, change:

```yaml
        service:
          DISABLE_REGISTRATION: true
```

Commit, push to Forgejo, Flux applies the change, Forgejo restarts
with registration closed.

---

## Decisions made in this phase

- **Forgejo** as the Git server (over Gitea, GitLab). ~300 MB RAM,
  ARM64-native, community-governed.
- **SQLite** for Forgejo's database. No external Postgres needed for
  a single-user instance.
- **Redis/cache/queue disabled.** In-memory and level-db are fine at
  this scale.
- **SSH deploy key (ed25519, read-only)** for Flux → Forgejo auth.
  More secure than a PAT, no rotation needed.
- **`git.homelab.local`** as the hostname. Resolved via `/etc/hosts`
  on the workstation.
- **Self-signed TLS** via `ca-issuer`. Browser will warn; acceptable
  for internal use.
- **GitHub retained as a backup remote.** Optional public mirror.

## What Phase 6 will do

- Deploy **Keycloak** (identity provider) with a Postgres backend on
  Longhorn.
- Wire **PrivacyIDEA** into Keycloak for MFA.
- Deploy **NetBird** for VPN access to the cluster from outside the
  LAN.
- Start deploying workload apps through `apps/`.
