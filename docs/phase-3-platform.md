# Phase 3 — Platform layer (MetalLB, ingress-nginx, cert-manager, Longhorn)

## Goal of this phase

Take the bare HA k3s cluster from Phase 2 and turn it into a *platform* —
something that can actually run useful applications with predictable
networking, hostname-based routing, automatic TLS, and replicated persistent
storage. By the end of this phase, deploying a new app will be a single
Ingress + Deployment + Service, and all four cross-cutting concerns (IP
assignment, HTTP routing, TLS, storage) are handled automatically.

This is the phase where the cluster stops feeling like "three Pis running k3s"
and starts feeling like "a thing I deploy apps on."

## What we built

Four components, installed in order via Helm driven by Ansible.

- **MetalLB 0.14.9** in Layer-2 mode, handing out LoadBalancer IPs from the
  pool `192.168.1.241-250`. This is what makes `Service type: LoadBalancer`
  actually work on bare metal.
- **ingress-nginx 4.11.3** pinned to `192.168.1.241` (the first IP in the
  MetalLB pool) via a MetalLB annotation. HTTP/HTTPS entry point for
  everything in the cluster, routing by `Host:` header.
- **cert-manager v1.16.2** with a self-signed internal CA (`ca-issuer`
  ClusterIssuer) valid for 10 years, issuing 1-year app certificates via
  ECDSA 256. Graduating to Let's Encrypt later is a one-line config change.
- **Longhorn 1.7.2** with 3 replicas per volume, data locality best-effort,
  data path on the NVMe at `/var/lib/rancher/longhorn`, UI exposed via
  Ingress with HTTP Basic Auth and a proper TLS cert from our CA.

And as a side effect:
- The `longhorn` StorageClass is now cluster default; `local-path` is not.
- A new top-level play, `platform.yml`, provisions all four components
  idempotently.
- A new role, `longhorn_prereqs`, runs OS-level prep on the Pis
  (open-iscsi, kernel module, data directory).

## New concepts this phase teaches

### Why bare-metal Kubernetes needs MetalLB at all

`Service type: LoadBalancer` is a Kubernetes abstraction. On a cloud provider
(AWS, GCP, Azure), creating one calls out to the provider's API and
provisions an actual cloud load balancer. On bare metal there's nobody to
call, so without a tool to fill the gap, every LoadBalancer service sits in
`<pending>` forever. **MetalLB is the tool that fills the gap.** It watches
for LoadBalancer services, picks an IP from a pool we give it, assigns that
IP to the service, and announces it on the LAN so traffic actually reaches
the cluster.

### Layer-2 mode vs BGP mode

MetalLB supports two announcement modes. **Layer-2 (ARP)** is what we're
using — one "speaker" pod per node, elected leader per IP, sends gratuitous
ARP packets to claim the IP on the LAN. Same mechanism kube-vip uses for
the API VIP, which is why we can have both running side by side without
conflict (we explicitly disabled kube-vip's service mode with
`svc_enable=false` back in Phase 2).

**BGP mode** announces IPs to a BGP-capable router, which gives you real
load-balancing across all nodes instead of failover. For a flat home LAN
without a BGP router, L2 is the right choice. L2 is not really
load-balancing — one node owns each IP at any given moment — but it gives
you fast failover, which is all a homelab needs.

### Ingress controller vs Ingress resource

Kubernetes uses the word "ingress" for two completely different things:

- An **Ingress controller** is a running program (a Deployment + a Service).
  It's the thing that actually receives HTTP traffic and routes it. One per
  cluster. Implementations: ingress-nginx, Traefik, HAProxy, Istio, others.
- An **Ingress resource** is a YAML object per application. It's config
  that says "route requests for this hostname/path to this Service". It
  doesn't run anything on its own — it's data the controller reads.

One controller, many Ingress resources. When you deploy an app, you create
an Ingress resource for it; the already-running ingress controller picks it
up within seconds and starts routing. This is also why cert-manager can
auto-issue certs just from Ingress annotations — it watches the same
Ingress objects.

### Why ingress-nginx as a Deployment, not a DaemonSet

ingress-nginx's Helm chart defaults to DaemonSet-with-hostNetwork, meaning
each node directly owns ports 80/443 on its host IP. That's a valid pattern
but it conflicts with MetalLB (MetalLB wants to own the LoadBalancer IP, not
share ports with host-network pods). Running ingress-nginx as a regular
Deployment behind a LoadBalancer Service is cleaner: ingress-nginx doesn't
need host networking, MetalLB owns the IP, kube-proxy routes traffic to
whichever replica. Two components, clean separation, no shared state.

### Pinning a LoadBalancer IP with MetalLB

MetalLB supports `metallb.universe.tf/loadBalancerIPs` as a Service
annotation, which tells it "give me this specific IP, not just any free one
from the pool". We pin ingress-nginx to `192.168.1.241` so it's always the
same IP in DNS and in our head. The rest of the pool (`.242-.250`) is free
for future services.

### The cert-manager bootstrap chicken-and-egg

A self-signed internal CA in cert-manager needs a three-object bootstrap,
because a CA has to be signed by *something*:

1. A **`selfSigned` ClusterIssuer** — a built-in cert-manager type that
   signs certs with their own key. Useless for real apps (no trust chain),
   but perfect for signing a root CA cert. Its only job is to bootstrap
   step 2.
2. A **`Certificate` with `isCA: true`** — this is the actual root CA cert,
   signed by the selfSigned issuer above. Its private key is stored in a
   Secret, which is the basis of the chain of trust.
3. A **`ca` ClusterIssuer** that references the Secret from step 2 — this
   is the issuer you actually *use* from day-to-day. Every app cert from
   here on is signed by this.

The selfSigned issuer and the root Certificate both exist forever but you
never touch them again. From Phase 3 onward, every app that wants TLS just
adds `cert-manager.io/cluster-issuer: ca-issuer` to its Ingress.

### Public CAs vs private CAs are the same thing, technically

A Certificate Authority is just a keypair. Public CAs (Let's Encrypt,
DigiCert) are technically identical to the internal CA we built — the only
difference is that their root certs are pre-installed in every browser's
trust store. You can achieve the same effect locally by installing your own
CA cert into each device's trust store, which we did on the WSL workstation
with `update-ca-certificates`. Browsers and other devices on your LAN will
need the same treatment once you start using them.

### The Ingress → cert-manager → ingress-nginx → TLS pipeline

Once all four components are up, deploying a TLS-protected app is a single
YAML:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp
  annotations:
    cert-manager.io/cluster-issuer: ca-issuer
spec:
  ingressClassName: nginx
  tls:
    - hosts: [myapp.nebula.cat]
      secretName: myapp-tls
  rules:
    - host: myapp.nebula.cat
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: myapp
                port:
                  number: 80
```

Four things then happen automatically, in order:
1. ingress-nginx sees the Ingress and starts listening for the hostname.
2. cert-manager sees the annotation + `tls:` block and creates a
   `Certificate` resource for `myapp.nebula.cat`.
3. cert-manager asks `ca-issuer` to sign it, writes the result to
   `myapp-tls` Secret.
4. ingress-nginx picks up the Secret (name matches) and starts serving TLS.

No manual cert management, ever. This is the moment the project stops
feeling like glued-together YAML and starts feeling like a platform.

### Why Longhorn and not local-path

k3s ships with `local-path-provisioner`, which writes PVC data to the node
where the pod is currently scheduled. That's fast and simple, but it pins
the pod to one node forever — if the node reboots, the pod can't move,
because its data isn't anywhere else. For a 3-node cluster with HA goals,
that's unusable. **Longhorn** stores each volume as 3 replicas synchronously
mirrored across all three nodes. Lose a node, reschedule the pod elsewhere,
the data is already there. This is the piece that makes stateful workloads
actually survive failure.

### Why Longhorn needs OS-level prep on the Pis

Unlike the other three components (pure in-cluster Helm installs), Longhorn
reaches down to the host to do its job. It presents volumes to pods as
iSCSI targets — the pod thinks it's mounting a remote SAN LUN, even though
the target is on the same machine. This requires:

- **`open-iscsi`** on every node, for `iscsid` to run and handle the iSCSI
  protocol.
- **The `iscsi_tcp` kernel module** loaded. Present in Ubuntu's default
  kernel but has to be loaded explicitly (done via `/etc/modules-load.d`).
- **No `multipath-tools`**. multipath-tools is a SAN-world package that
  tries to deduplicate multi-path storage, and if it's running it races
  Longhorn for volume control. Symptom: volumes attach then detach a few
  seconds later. Fix: uninstall it.
- **A writable data directory on the NVMe**. Longhorn writes its replicas
  to `/var/lib/longhorn` by default; we redirect that to
  `/var/lib/rancher/longhorn` so it lands on the NVMe instead of the SD
  card. Phase 1 already mounted `/var/lib/rancher` on the NVMe, so it's
  just a subdirectory.

All of this is handled by the new `longhorn_prereqs` role, which runs
against the `cluster` group in the first play of `platform.yml`. It also
has a paranoid `findmnt` check that fails loudly if the data directory
somehow ended up on the SD card — catches the "I re-imaged a Pi and
forgot to re-run Phase 1" mistake before it costs you an SD card.

### Two-play `platform.yml`

Because `longhorn_prereqs` targets the Pis (needs SSH + sudo) but the
actual Helm installs target localhost (talks to the cluster via kubeconfig),
`platform.yml` has two plays:

```yaml
- hosts: cluster
  become: true
  roles: [longhorn_prereqs]

- hosts: localhost
  connection: local
  roles: [metallb, ingress_nginx, cert_manager, longhorn]
```

Ansible runs plays in order, so Pi-side prep completes before in-cluster
installs start. Idempotency holds across both plays.

### Why `group_vars/all.yml` is different from `group_vars/cluster.yml`

Hit this head-on in a debugging session. Variables in `group_vars/cluster.yml`
are only loaded when running plays that target the `cluster` group (or its
children `k3s_init` / `k3s_join`). Phase 2 is fine because `site.yml` and
`k3s.yml` both target `cluster`. Phase 3a is fine because it also targets
`cluster`. But Phase 3b runs with `hosts: localhost`, which is *not* a
member of `cluster`, so `group_vars/cluster.yml` is never loaded for it.

**`group_vars/all.yml`** is the fix — it's loaded for every host, including
localhost. Any variable that Phase 3b (or future Phase 4's Flux) needs,
especially vault-encrypted secrets, belongs in `all.yml`, not `cluster.yml`.

This is a Phase 2-era blind spot that only became visible when we added
the first secret that needed to be read from a localhost play
(`longhorn_ui_basic_auth_password`).

### StorageClass default swap

After Longhorn installs, the `longhorn` StorageClass is marked default by
the chart — but `local-path` (from k3s's own provisioner) is *also* still
marked default. Having two defaults is worse than having none: Kubernetes
rejects PVCs without an explicit class because it can't decide. Fix:
`state: patched` on the `k8s` Ansible module to strip the default annotation
from `local-path`, leaving only `longhorn` as the cluster default.
Idempotent — re-running the playbook reasserts the desired state if
anything ever drifts.

### HTTP Basic Auth via ingress-nginx annotations

The Longhorn UI has no built-in authentication, which is fine for a
team-sharing-with-SSO deployment but not for "deploy into homelab and hope".
Until Keycloak exists (Phase 6), we front the UI with HTTP Basic Auth at
the ingress layer:

1. Generate an htpasswd line with bcrypt: `htpasswd -nbB user password`.
2. Base64-encode it and store it in a Secret with a single key `auth`.
3. Reference the Secret from the Ingress via three annotations:
   `auth-type: basic`, `auth-secret: <secret-name>`, `auth-realm: ...`.

ingress-nginx intercepts every request, checks the Authorization header
against the bcrypt hash, and either forwards or returns 401. Same pattern
works for any internal tool that doesn't have its own auth story.

## Repository layout added in this phase

```
ansible/
├── platform.yml                    # NEW — Phase 3 top-level play (2 plays)
├── group_vars/
│   ├── all.yml                     # NEW — vars visible to localhost plays
│   └── cluster.yml                 # Phase 2, still used for cluster-only vars
└── roles/
    ├── common/                     # Phase 1
    ├── k3s/                        # Phase 2
    ├── longhorn_prereqs/           # NEW — Pi-side OS prep
    │   ├── defaults/main.yml
    │   └── tasks/main.yml
    ├── metallb/                    # NEW
    │   ├── defaults/main.yml
    │   ├── tasks/main.yml
    │   └── templates/ipaddresspool.yaml.j2
    ├── ingress_nginx/              # NEW
    │   ├── defaults/main.yml
    │   └── tasks/main.yml
    ├── cert_manager/               # NEW
    │   ├── defaults/main.yml
    │   ├── tasks/main.yml
    │   └── templates/bootstrap-ca.yaml.j2
    └── longhorn/                   # NEW
        ├── defaults/main.yml
        ├── tasks/main.yml
        └── templates/ui-ingress.yaml.j2
```

## Configuration decisions worth recording

- **MetalLB pool: `192.168.1.241-250`**, L2 mode, `avoidBuggyIPs: true`.
  10 IPs, carved from the same `/24` as the nodes, outside the router DHCP
  range.
- **ingress-nginx: Deployment with 2 replicas**, pinned to `.241`,
  `use-forwarded-headers: true`, `proxy-body-size: 50m`, default
  IngressClass, metrics disabled (no Prometheus yet).
- **cert-manager: 10-year root CA**, 1-year app certs, ECDSA 256,
  `crds.enabled: true`, prometheus disabled.
- **Longhorn: 3 replicas always**, hard anti-affinity (no two replicas on
  the same node — `replicaSoftAntiAffinity: false`), data path
  `/var/lib/rancher/longhorn`, `defaultDataLocality: best-effort`,
  `upgradeChecker: false`, `service.ui.type: ClusterIP` (UI is only
  exposed via our custom Ingress).
- **Longhorn UI: HTTP Basic Auth + TLS from ca-issuer**, hostname
  `longhorn.nebula.cat`, user `admin`, password vault-encrypted in
  `group_vars/all.yml`.
- **Helm chart versions pinned**: MetalLB 0.14.9, ingress-nginx 4.11.3,
  cert-manager v1.16.2, Longhorn 1.7.2. Same "reproducible installs"
  discipline as k3s and kube-vip in Phase 2.
- **Secret storage**: `longhorn_ui_basic_auth_password` in
  `group_vars/all.yml` (vault-encrypted). `k3s_token` stays in
  `group_vars/cluster.yml` — both encrypted, just in different files based
  on which plays need them.

## Running the playbook

From `~/homelab/ansible`:

```bash
# Once only: htpasswd dependency for the Longhorn auth task
sudo apt install apache2-utils

# Real run (both plays)
ansible-playbook platform.yml -K
```

Timing: OS prep on the Pis is 1-2 minutes in parallel. MetalLB,
ingress-nginx, and cert-manager install in ~2-4 minutes each. Longhorn is
the slow one — 10-15 minutes for the first run because it pulls a dozen
container images onto each of three Pis. Total first-time run: 20-30
minutes. Subsequent runs are seconds (idempotent).

## Convergence proof — what "done" looked like

### 1. StorageClass defaults correctly swapped

```
NAME                 PROVISIONER             DEFAULT
local-path           rancher.io/local-path   false
longhorn (default)   driver.longhorn.io      true
```

### 2. Longhorn sees all three nodes as schedulable

```bash
kubectl -n longhorn-system get nodes.longhorn.io
```

Three rows, all `READY: True`, `ALLOWSCHEDULING: true`.

### 3. Dynamic provisioning works end-to-end

Created a 1Gi PVC without specifying a StorageClass; it bound to a
Longhorn-backed PV within seconds. Verified three replicas were placed on
three different nodes by querying the `replicas.longhorn.io` CRD. PVC
deleted cleanly, replicas cleaned up.

### 4. The UI Ingress works end-to-end with TLS and auth

With the homelab CA installed on the WSL workstation
(`update-ca-certificates`), no `-k` flag needed on curl:

```bash
# Without auth — proves basic auth is enforced
curl --resolve longhorn.nebula.cat:443:192.168.1.241 \
  https://longhorn.nebula.cat/ -o /dev/null -s -w "HTTP %{http_code}\n"
# HTTP 401

# With auth — proves the whole chain works
curl --resolve longhorn.nebula.cat:443:192.168.1.241 \
  -u admin:<redacted> \
  https://longhorn.nebula.cat/ -o /dev/null -s -w "HTTP %{http_code}\n"
# HTTP 200
```

The 401 → 200 transition proves:
- DNS resolution (faked via `--resolve`) routed to `.241`
- MetalLB delivered the packet to an ingress-nginx pod
- ingress-nginx matched the `Host:` header to the longhorn-ui Ingress
- ingress-nginx enforced basic auth via the Secret
- ingress-nginx served the TLS cert from `longhorn-ui-tls`
- cert-manager had issued that cert, signed by `ca-issuer`
- `ca-issuer` is backed by the root CA we bootstrapped
- WSL trusted the CA because we installed the root cert
- the backend (`longhorn-frontend` Service) responded

Eight distinct layers, one curl. This is the proof that Phase 3 is a
platform, not a pile.

### 5. Idempotency: re-running the playbook is a no-op

```
PLAY RECAP
localhost   : ok=32  changed=0  ...
rp-node-1   : ok=9   changed=0  ...
rp-node-2   : ok=9   changed=0  ...
rp-node-3   : ok=9   changed=0  ...
```

## Things that looked like errors but weren't (troubleshooting reference)

- **`--check` dry-run fails on the Helm install** with "repo not found".
  Check mode reports `changed` on the `helm_repository` task but doesn't
  actually add the repo, so the subsequent install can't find it. Skip
  check mode for platform.yml; go straight to real runs.
- **`group_vars/cluster.yml` variables invisible to a `hosts: localhost`
  play**. That's group-var scoping — `cluster.yml` is only loaded for
  members of the `cluster` group, and localhost isn't one. Fix: put vars
  that need to be visible from localhost plays in `group_vars/all.yml`.
- **`ip neigh show 192.168.1.240` empty in WSL** (Phase 2 holdover). WSL2
  doesn't sit on the LAN's L2 segment, so it never sees ARP replies for
  LAN IPs. Not a kube-vip or MetalLB bug. Use `kubectl` queries or SSH to
  the Pis directly to verify ARP state.
- **`curl: Could not resolve host`** with a `nebula.cat` hostname. The CA
  install only fixes TLS trust, not DNS. Use curl's `--resolve` flag to
  bypass DNS entirely when testing, or add entries to `/etc/hosts`, or
  eventually set up real DNS (router-side or Pi-hole). Not a cluster bug.
- **`kubectl get nodes.longhorn.io` returns "no resources found in
  default namespace"**. Longhorn's node CRDs live in `longhorn-system`.
  Use `-n longhorn-system`.
- **Two StorageClasses marked default after Longhorn install**. The chart
  marks `longhorn` as default, but `local-path` is still default from k3s.
  Fixed by the `state: patched` task in the `longhorn` role.

## What's now possible

With Phase 3 done, the cluster can:

- Expose any Service to the LAN with a real IP (MetalLB)
- Route HTTP by hostname with a single entry point (ingress-nginx)
- Automatically issue TLS certs for any Ingress (cert-manager + CA)
- Provide replicated persistent storage that survives node failure (Longhorn)
- Host internal admin UIs with TLS + HTTP Basic Auth (the pattern used for
  the Longhorn UI)

What it *cannot* yet do:

- **Deploy apps declaratively from Git.** Everything so far is
  `ansible-playbook` runs. **Phase 4: Flux** sets up the watch-Git-and-apply
  loop that makes the cluster react to commits automatically.
- **Host its own source of truth.** The repo lives on GitHub. **Phase 5**
  deploys Forgejo on the cluster and migrates Flux to watch it.
- **Authenticate real users.** HTTP Basic Auth is a stopgap; **Phase 6**
  replaces it with Keycloak + PrivacyIDEA.
- **Survive accidental deletion.** No backups yet. **Phase 7** adds
  Velero + Prometheus + Grafana + Loki.

## Concepts to take forward

- Helm as Kubernetes' package manager, and why we drive it from Ansible
  instead of by hand (reproducibility, version control, idempotency).
- The distinction between an Ingress controller (the running program) and
  an Ingress resource (the per-app YAML) — and why every later layer
  (cert-manager, external-dns if you ever add it) hangs off the same
  Ingress object.
- How LoadBalancer, Ingress, and StorageClass are all **provider
  abstractions** — Kubernetes defines the contract, bare metal needs you
  to bring the implementation (MetalLB, ingress-nginx, Longhorn).
- The bootstrap pattern for self-referential CAs: a throwaway issuer to
  sign the real issuer. Same idiom appears in any system that needs to
  sign its own roots.
- Public CAs and private CAs are the same mechanism; the difference is
  pre-installed trust. Installing your own CA on each device once turns
  your homelab into a "public CA" for your LAN.
- The Phase 2 chicken-and-egg manifest pattern (auto-apply directory for
  kube-vip) generalizes: Longhorn's own UI Ingress is another instance
  of the platform-bootstraps-itself shape.
- **`group_vars/all.yml` vs `group_vars/<group>.yml`**: scoping is by
  inventory group, not by convenience. If a play targets localhost,
  cluster-group vars are invisible to it. Put shared/localhost vars in
  `all.yml`.
- Idempotency still holds across 4 Helm charts, 1 OS-prep role, and a
  vault secret. The playbook as desired-state declaration scales.
