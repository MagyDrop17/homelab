# Phase 2 — k3s in HA mode with kube-vip

## Goal of this phase

Take the three Pis that Phase 1 left in a hardened, idempotent state and turn
them into a single highly-available Kubernetes cluster — three control planes
with embedded etcd, fronted by a floating virtual IP so any single node can die
without breaking `kubectl`. By the end of this phase, running
`kubectl get nodes` from WSL must show three `Ready` nodes via the VIP, and
rebooting any one node must not interrupt the API.

This is the phase where "three Linux boxes" becomes "one Kubernetes cluster".

## What we built

- **k3s v1.35** installed in **server** mode on all three Pis (no dedicated
  workers — every Pi is an equal control-plane peer that also runs workloads;
  this is called **stacked topology**).
- **Embedded etcd** as the cluster datastore, replacing k3s's default SQLite.
  etcd is a Raft consensus database, which is what makes the control plane
  HA in the first place.
- **kube-vip v1.1.1** deployed as a DaemonSet on the three control planes,
  providing a Layer-2 (ARP) virtual IP at `192.168.1.240` that fronts the
  Kubernetes API server.
- **Traefik and servicelb disabled** at install time. Both are k3s defaults
  but we'll be replacing them with ingress-nginx and MetalLB in Phase 3.
- A kubeconfig fetched back to the WSL workstation at `~/.kube/homelab.yaml`,
  rewritten so its `server:` line points at the VIP rather than `127.0.0.1`.

## New concepts this phase teaches

These are the ideas I want to actually understand, not just paste-and-go past.
Each one matters in later phases too.

### Control plane vs worker, and why we run "stacked"

In Kubernetes, the **control plane** is the brain — API server, scheduler,
controller-manager, etcd. The **kubelet** is the per-node agent that actually
runs containers. A "server" node in k3s runs all of those. An "agent" node
runs only the kubelet and kube-proxy and connects to a server.

For a 3-node homelab there's no benefit to dedicated workers: that would mean
sacrificing one of three precious Pis to be control-plane-only, leaving only
two for actual workloads. So all three are servers, and every Pi runs both
the control plane *and* user workloads. This is called **stacked topology**
and it's the right call below ~5 nodes.

### Embedded etcd and the Raft majority rule

Single-node k3s uses SQLite as the datastore. That's fine until the node
dies, at which point your cluster state dies with it. HA requires a
distributed datastore, and k3s embeds **etcd** for that purpose.

etcd uses the **Raft consensus algorithm**. Raft requires a *majority* of
nodes to agree before any write commits. With 3 nodes, the majority is 2,
so you can lose 1 and the remaining 2 still form quorum. With 2 nodes the
majority is also 2, so losing either breaks quorum — which is why you always
want an *odd* number of etcd peers. Three is the minimum that buys real HA;
five lets you survive two simultaneous failures but is overkill at this scale.

### The cluster-init / join asymmetry

A Raft cluster has to be bootstrapped. Exactly one node creates the initial
cluster of size 1, then the other peers join it one at a time. This means
the install procedure is asymmetric:

- **rp-node-1** runs k3s with `cluster-init: true` — it creates a fresh
  embedded etcd cluster with just itself in it.
- **rp-node-2** then runs k3s with `server: https://192.168.1.230:6443`,
  pointing at node 1, and joins as a Raft peer.
- **rp-node-3** does the same.

The asymmetry only exists at install time. Once all three have joined, they
are equal peers and node 1 has no special status — you could shut it down
permanently and the cluster would carry on with nodes 2 and 3.

This is why our Ansible inventory splits `cluster` into two child groups,
`k3s_init` (just rp-node-1) and `k3s_join` (rp-node-2 and rp-node-3), and
why our `k3s.yml` play runs them in two separate plays with `serial: 1` on
the join group.

### kube-vip: a floating IP for the API server

With three API servers on three different IPs, you have a problem: every
client (kubectl, every joining node, every controller) needs *one* address
to talk to. If you hardcode `192.168.1.230` and that Pi dies, every client
breaks even though nodes 2 and 3 are perfectly healthy.

The fix is a **virtual IP** — a fourth IP, `192.168.1.240`, that doesn't
belong to any specific Pi but "floats" to whichever Pi is currently elected
leader. If that Pi dies, the VIP migrates to another within a few seconds.

**kube-vip** is the program that does this. It runs as a DaemonSet on the
control-plane nodes (one pod per node, three pods total), and the pods elect
a leader among themselves using a Kubernetes Lease object. The leader sends
out **gratuitous ARP** packets on the LAN saying "hi, IP `192.168.1.240` is
at MAC address `xx:xx:xx:...`" — the MAC of whichever Pi currently holds
it. The switch updates its ARP tables, and traffic to `.240` flows to that
Pi until the next election.

This is "ARP mode" or "Layer 2 mode", and it's the right choice for a flat
home LAN. The alternative is BGP, which needs a router that speaks BGP.

We configure kube-vip with `cp_enable: true` (act as a control-plane VIP)
and `svc_enable: false` (do NOT also act as a service-type-LoadBalancer
provider — that's MetalLB's job in Phase 3). One tool, one job.

### The auto-manifests directory (chicken-and-egg solver)

There's a circular dependency hiding in this design: kube-vip is itself a
Kubernetes workload, but the cluster it's supposed to provide a VIP for
doesn't exist yet when we install node 1. How does it get applied?

k3s solves this with a magic directory:

```
/var/lib/rancher/k3s/server/manifests/
```

Anything dropped in there gets auto-applied by k3s as soon as the API
server comes up. Our Ansible role drops the kube-vip RBAC + DaemonSet YAML
there *before* starting k3s on node 1, k3s installs itself, sees the
manifest, and brings up kube-vip as part of bootstrap. By the time node 2
tries to join via the VIP, it's already alive.

This trick comes back in Phase 4 when Flux bootstraps itself.

### TLS SANs

The k3s API server presents a TLS certificate. By default that cert is
only valid for the node's own IP — so when `kubectl` connects via
`https://192.168.1.240:6443`, it would get a TLS error because `.240`
isn't on the cert.

The fix is **Subject Alternative Names**: extra hostnames/IPs that get
baked into the cert at issuance time. Our `config.yaml` lists every node
IP and the VIP under `tls-san:`, so the cert is valid for any of them.
You'd never want this in production for a real public cert, but for an
internal cluster CA it's exactly the right thing.

### Why we disable Traefik and servicelb at install time

k3s ships batteries-included: a Traefik ingress controller, a klipper-lb
service-load-balancer, etc. We're going to install **ingress-nginx** and
**MetalLB** in Phase 3, so we want Traefik and servicelb *off* from day
one — otherwise we'd fight two ingress controllers and two load-balancer
implementations later. We disable them in `config.yaml`:

```yaml
disable:
  - traefik
  - servicelb
```

Future-me will be grateful.

## Repository layout added in this phase

```
ansible/
├── ansible.cfg                 # now has vault_password_file = ~/.ansible-vault-pass
├── inventory/
│   └── hosts.yml               # cluster split into k3s_init + k3s_join children
├── group_vars/
│   └── cluster.yml             # k3s_channel, kube_vip_version, apiserver_vip, vault-encrypted k3s_token
├── k3s.yml                     # new top-level play for Phase 2
├── site.yml                    # Phase 1 — unchanged
└── roles/
    ├── common/                 # Phase 1
    └── k3s/                    # NEW
        ├── defaults/main.yml
        ├── tasks/main.yml
        └── templates/
            ├── config.yaml.j2
            └── kube-vip-daemonset.yaml.j2
```

## Configuration decisions worth recording

- **k3s channel: `v1.35`** (pinned, not `stable`). Channel pinning gets us
  automatic patch updates within the v1.35 line but never an unexpected
  jump to v1.36. The actually-installed version when this phase ran was
  `v1.35.3+k3s1`.
- **kube-vip image: `ghcr.io/kube-vip/kube-vip:v1.1.1`** (pinned, not
  `latest`). Pinning images is a habit that prevents 3am surprises later.
- **VIP: `192.168.1.240/32`** on `eth0`. Reserved outside the router's
  DHCP range so nothing else will ever try to claim it.
- **kube-vip mode: ARP (Layer 2), control-plane only.** `vip_arp=true`,
  `cp_enable=true`, `svc_enable=false`. MetalLB will handle service
  LoadBalancers in Phase 3.
- **Lease timing: aggressive (5/3/1 seconds).** `vip_leaseduration=5`,
  `vip_renewdeadline=3`, `vip_retryperiod=1`. This gives ~5–10 second
  failover when a node dies. The defaults (15/10/2) are safer for noisy
  networks; on a clean home LAN the aggressive timing is fine and makes
  the failover demo more dramatic.
- **Stacked topology, no agents.** All three nodes are servers.
- **Traefik and servicelb disabled.** Replaced by ingress-nginx and
  MetalLB in Phase 3.
- **Token storage: ansible-vault.** The k3s join token is encrypted at
  rest in `group_vars/cluster.yml` with ansible-vault. The vault password
  itself lives in `~/.ansible-vault-pass` on the workstation only — never
  committed.

## Running the playbook

From `~/homelab/ansible` on the WSL workstation:

```bash
# Pre-flight
ansible-playbook k3s.yml --syntax-check
ansible-playbook k3s.yml -K --check --diff   # expect failure on the systemd
                                              # task in check mode — that's normal

# Real run
ansible-playbook k3s.yml -K
```

Total time: 5–10 minutes on Pi 5 hardware. The slowest tasks are the
curl-pipe-sh download of the k3s installer (1–3 minutes per node) and the
first systemd start of the k3s service.

## Convergence proof — what "done" looked like

### 1. `kubectl get nodes` from WSL via the VIP

```
NAME        STATUS   ROLES                AGE    VERSION        INTERNAL-IP
rp-node-1   Ready    control-plane,etcd   2m8s   v1.35.3+k3s1   192.168.1.230
rp-node-2   Ready    control-plane,etcd   83s    v1.35.3+k3s1   192.168.1.231
rp-node-3   Ready    control-plane,etcd   37s    v1.35.3+k3s1   192.168.1.232
```

Three Ready, all `control-plane,etcd`, all on the same k3s patch version.
The age stagger (2m / 83s / 37s) shows the `serial: 1` join order working
as designed.

### 2. The VIP answers

```bash
$ ping -c 3 192.168.1.240
64 bytes from 192.168.1.240: icmp_seq=1 ttl=63 time=1.57 ms
64 bytes from 192.168.1.240: icmp_seq=2 ttl=63 time=1.19 ms
```

### 3. The API server answers via the VIP with valid TLS

```bash
$ kubectl get --raw /livez
ok
$ kubectl get --raw /readyz
ok
```

(`curl -k https://192.168.1.240:6443/livez` returns a `Status` object with
code 401 — that's also healthy, it just means the API server is alive and
correctly rejecting an unauthenticated request. The `kubectl get --raw`
form uses the kubeconfig credentials so it returns the real `ok`.)

### 4. kube-vip is running on all three nodes

```bash
$ kubectl -n kube-system get pods -l app.kubernetes.io/name=kube-vip-ds -o wide
NAME                READY   STATUS    NODE
kube-vip-ds-2r2rp   1/1     Running   rp-node-1
kube-vip-ds-8v4jt   1/1     Running   rp-node-2
kube-vip-ds-f4mcb   1/1     Running   rp-node-3
```

One pod per control-plane node, all `Running`. One of them holds the
leader lease at any given moment; the others stand by.

### 5. Idempotency: re-running the playbook is a no-op

```
PLAY RECAP ********************************************************************
rp-node-1   : ok=12  changed=0  unreachable=0  failed=0  skipped=1
rp-node-2   : ok=6   changed=0  unreachable=0  failed=0  skipped=3
rp-node-3   : ok=6   changed=0  unreachable=0  failed=0  skipped=3
```

`changed=0` everywhere. The `skipped` count on the join nodes is correct —
they correctly skip the kube-vip manifest drop (which is `when:
inventory_hostname in groups['k3s_init']`) and the install task (because
the binary already exists, gated by `creates: /usr/local/bin/k3s`).

### 6. Failover proof

Find the current VIP holder (the WSL `ip neigh` trick doesn't work because
WSL2 doesn't sit on the LAN's L2 segment — its packets get NAT'd through
the Windows host, so it never builds a neighbor entry for `192.168.1.240`):

```bash
# Method A — ask the cluster who holds the lease
kubectl -n kube-system get lease plndr-cp-lock \
  -o jsonpath='{.spec.holderIdentity}{"\n"}'

# Method B — find which Pi has the VIP bound on eth0
for ip in 192.168.1.230 192.168.1.231 192.168.1.232; do
  echo "=== $ip ==="
  ssh dark@$ip "ip -4 addr show eth0 | grep 192.168.1.240"
done
```

Then SSH into the leader and reboot it:

```bash
ssh dark@rp-node-X sudo reboot
```

Watch the failover from WSL:

```bash
watch -n 1 '
  echo "=== ping ==="; ping -c 1 -W 1 192.168.1.240;
  echo; echo "=== nodes ==="; kubectl get nodes;
  echo; echo "=== holder ===";
  kubectl -n kube-system get lease plndr-cp-lock \
    -o jsonpath="{.spec.holderIdentity}{\"\n\"}"
'
```

Expected behaviour: 5–10 second ping gap while a new leader is elected and
starts answering for `.240`; the `holder` field flips to a different node;
the rebooted node goes `NotReady` for ~60 seconds and then comes back. No
manual intervention. That's HA.

## Things that look like errors but aren't

A small troubleshooting reference for future-me.

- **`curl -k https://192.168.1.240:6443/livez` returns `code: 401`** — fine.
  It's an unauthenticated request being correctly rejected by a working
  API server. Use `kubectl get --raw /livez` to hit it with credentials.
- **`ip neigh show 192.168.1.240` is empty in WSL** — fine, WSL2 quirk.
  WSL doesn't see LAN ARP. Use `kubectl -n kube-system get lease
  plndr-cp-lock` instead, or SSH into a Pi and run `ip neigh` from there.
- **The dry-run (`--check`) playbook fails on `Ensure k3s service is
  enabled and running`** — fine. Check mode fakes the install but systemd
  is real, so it can't find a service that wasn't really installed. The
  failure proves nothing else upstream is broken; the real run will work.
- **`skipped=2` or `skipped=3` on join nodes during real runs** — correct.
  The kube-vip manifest task is gated to `k3s_init` only.

## What's now possible

With Phase 2 done, the cluster can:

- Schedule pods on any of three nodes
- Survive a single node failure with no client-visible API outage
- Be talked to from WSL via a stable, version-controlled kubeconfig
- Be torn down and rebuilt from scratch with a single Ansible command

What it *cannot* yet do:

- Expose any service to the LAN beyond a NodePort hack — there's no
  LoadBalancer implementation. **Phase 3: MetalLB.**
- Route HTTP traffic by hostname — there's no ingress controller.
  **Phase 3: ingress-nginx.**
- Issue TLS certificates automatically. **Phase 3: cert-manager.**
- Provide persistent storage to pods — `local-path-provisioner` is the
  only StorageClass and it's not replicated. **Phase 3: Longhorn.**

## Concepts to take forward

- Server vs agent, stacked topology, and why we picked all-servers.
- Embedded etcd, Raft, and the majority rule — and why odd numbers matter.
- The cluster-init / join asymmetry, and why it only exists at install time.
- TLS SANs and why API certs need to know all their future names ahead of
  time.
- The `/var/lib/rancher/k3s/server/manifests/` auto-apply directory as a
  chicken-and-egg solver. (Returning in Phase 4 with Flux.)
- Layer-2 ARP VIPs and why they're the right tool for a flat home LAN.
- Disabling default components at install time to avoid fighting them later.
- ansible-vault for secrets at rest in a Git-tracked repo.
- Idempotency as a *property of the playbook*, not just an aspiration.
