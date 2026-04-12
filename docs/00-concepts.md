# Core concepts

Cross-cutting Kubernetes concepts that aren't tied to a single phase.
When something here gets referenced from a phase note, link back to it.

## The control plane

The "control plane" is the brain of a Kubernetes cluster. It's the set of
processes that decide what should be running where. The main components:

- **API server** — the front door. Every `kubectl` command, every
  controller, every node talks to it. It's the only component that talks
  directly to etcd.
- **Scheduler** — decides which node a new pod runs on, based on resource
  requests, affinities, taints, etc.
- **Controller manager** — runs the reconciliation loops that make reality
  match desired state ("I want 3 replicas, I see 2, start one").
- **etcd** — the database that stores the entire cluster state: every
  object, every secret, every config. If you back up etcd, you back up
  the cluster.

Worker components (run on every node, including control plane nodes in a
"stacked" topology):

- **kubelet** — the agent that actually starts and stops containers on
  the node, talking to the container runtime.
- **kube-proxy** — network glue that implements Service routing rules.

## HA control plane and Raft

In a single-control-plane setup, all of the brain lives on one node. If
it dies, the workers keep running whatever they were running, but
nothing new can be scheduled, no config can change, and `kubectl` stops
working.

In a 3-control-plane setup, k3s replaces its default SQLite backend with
**embedded etcd**, a distributed database that uses a consensus
algorithm called **Raft**. Raft requires a *majority* of nodes to agree
before any write is committed. This is why you want an odd number:

- 1 node: tolerates 0 failures
- 3 nodes: tolerates 1 failure (2 out of 3 still form a majority)
- 5 nodes: tolerates 2 failures
- 2 nodes: tolerates 0 failures *and* is worse than 1, because losing
  either one breaks the majority

Three control planes is the standard small-cluster choice.

In a "stacked" topology (what we're doing), every node is both control
plane *and* worker. This is normal and recommended for small clusters.
Larger production clusters separate the two roles so control-plane load
is isolated from workload load.

## The virtual IP (VIP)

With 3 API servers, `kubectl` needs *one* address to talk to, not three.
A **virtual IP** is an address that floats between the control plane
nodes — if the node currently holding it dies, another node takes over
the IP within seconds. We use **kube-vip** in ARP mode for this: it
sends gratuitous ARP packets on the LAN announcing "I own this IP",
which is enough for any device on the same subnet to find it. The
router doesn't need to know about the VIP at all.

Our VIP: `192.168.1.240`.

### Why .240 specifically

The Adamo router doesn't allow excluding an IP from the DHCP pool. To
minimize collision risk, we picked an address near the top of the /24,
far from where the router starts allocating leases (low end). As an
extra safety measure, .240 is also reserved as a static lease in the
router GUI bound to one of the Pi MAC addresses, so the router will
never lease it out. The Pi itself ignores the reservation; kube-vip
claims .240 independently via ARP.

## Idempotency

A core idea that shows up in Ansible, Kubernetes, and Terraform alike:
running the same operation multiple times produces the same result as
running it once. You don't write "create this user" — you write "this
user should exist", and the tool checks and only acts if reality
doesn't match. Every config tool we use in this project is built on
this principle, and it's the reason GitOps works at all.
