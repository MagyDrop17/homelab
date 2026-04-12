# Phase 1 — OS preparation with Ansible

Goal: turn 3 fresh Ubuntu 24.04 installs into k8s-ready nodes,
reproducibly, with one command.

## Why Ansible

A configuration management tool. You write YAML describing the desired
state of a machine ("hostname X, package Y installed, file Z present")
and Ansible SSHes in and makes reality match. It's idempotent by
design: running the same playbook ten times yields the same result as
running it once. This is the same principle Kubernetes uses for
workloads, so learning Ansible properly also teaches the GitOps
mindset.

## Core concepts

**Inventory** — the list of machines Ansible manages, grouped and with
per-host or per-group variables. Ours lives at
`ansible/inventory/hosts.yml` and defines a group called `cluster`
containing the 3 Pis.

**Playbook** — the entry point. A YAML file that says "for these
hosts, apply these roles." Ours is `ansible/site.yml`.

**Role** — a folder bundling related tasks with a conventional layout
(`tasks/`, `handlers/`, `templates/`, etc.). Roles let one playbook
stay short by delegating each concern to its own folder.

**Task** — a single unit of work calling an Ansible module (`apt`,
`copy`, `hostname`, `lineinfile`...). The module is responsible for
idempotency: the `apt` module checks if the package is already
installed before installing.

**Handlers** — tasks that only run when "notified" by another task
that actually changed something. Example: edit `sshd_config` →
notify `restart ssh`. If the edit was a no-op, ssh doesn't get
restarted.

**Variables** — Ansible has built-ins (`inventory_hostname`, `groups`,
`hostvars`) plus anything you define in `group_vars/`, `host_vars/`,
or inline. Templates use Jinja2 syntax (`{{ var }}`, `{% for %}`).

## The sudo model

The `dark` user on each Pi has sudo *with* a password. We keep it that
way: passwordless sudo is a real security downgrade, and Ansible
handles password sudo cleanly via `--ask-become-pass` (or `-K`),
which prompts once at the start of a run and reuses the password for
every `become: true` task.

So every playbook run looks like:

    ansible-playbook site.yml -K

Tasks that need root have `become: true` set explicitly.

## ansible.cfg

Lives at `ansible/ansible.cfg`. Key settings:

- `inventory` — default inventory path so we don't need `-i`
- `host_key_checking = False` — skip the SSH host-key prompt (LAN only)
- `stdout_callback = yaml` — readable multi-line output
- `pipelining = True` — faster SSH execution
- `retry_files_enabled = False` — no `.retry` files cluttering the repo

## Inventory

`ansible/inventory/hosts.yml` defines the `cluster` group with 3 hosts
and group-level vars (`ansible_user: dark`).

The Ansible name (`rp-node-1`) and the IP (`192.168.1.230`) come from
this file. Every other role references these via `inventory_hostname`
and `groups['cluster']` — single source of truth.

## Roles in this phase

1. **common** — apt update/upgrade and baseline packages.
2. **hostname** — set each Pi's hostname and populate `/etc/hosts`
   with all cluster nodes from inventory.
3. *(more added as we go: ssh-hardening, swap, cgroups, time-sync,
   nvme, firewall, auto-upgrades)*

## Useful Ansible commands

    # Connectivity check
    ansible cluster -m ping

    # Dry run with diffs
    ansible-playbook site.yml --check --diff -K

    # Real run
    ansible-playbook site.yml -K

    # Run only specific tagged tasks
    ansible-playbook site.yml -K --tags packages

    # Run against a single host
    ansible-playbook site.yml -K --limit rp-node-1

## The idempotency check

After every playbook change, run it twice in a row. The second run
must show `changed=0` for all hosts. If anything reports `changed` on
the second run, the role is non-deterministic and needs fixing before
moving on. This is non-negotiable — non-idempotent playbooks cause
real outages later.

## Gotcha: `become` and `--check` mode

`--check` mode runs Ansible without actually modifying the target — it
reports what *would* change. This means tasks that lack `become: true`
will appear to succeed in `--check` even though they would fail
permission-wise on a real run, because the modules never actually try
to acquire locks or write files.

**Always follow `--check` with a real run.** Don't trust a clean dry
run alone.

We avoid this whole class of bug by setting `become: true` once at the
play level in `site.yml`, which makes every task in every role run
with sudo unless explicitly overridden:

    - name: Prepare cluster nodes
      hosts: cluster
      become: true
      gather_facts: true
      roles:
        - common
        - hostname

For provisioning playbooks where almost everything is privileged, this
is the standard pattern.
