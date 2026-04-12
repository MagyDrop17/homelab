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

## Roles added in this batch

### `ssh-hardening`
Disables password auth, root login, X11 forwarding; sets `MaxAuthTries 3`;
forces pubkey auth. Uses `validate:` with `sshd -t` so a broken config can
never be written. Removes `/etc/ssh/sshd_config.d/50-cloud-init.conf`,
which Ubuntu's cloud-init creates to re-enable password auth — a classic
silent override of the main config. Restarts sshd via a handler so re-runs
don't bounce the service.

**Safety:** only run after confirming SSH key auth works. After this
role runs, password SSH login is permanently disabled.

### `swap`
Kubernetes requires swap to be off. Disables swap at runtime, removes
the entry from `/etc/fstab`, and deletes `/swap.img`. Uses `changed_when:`
on the `swapoff` command so re-runs report `ok` instead of always
`changed` (the `command` module has no idea what your command does, so
you have to tell it).

### `cgroups`
The Pi gotcha. Raspberry Pi kernels disable the `memory` and `cpuset`
cgroup controllers by default, and Kubernetes refuses to run without
them. Fix: add `cgroup_memory=1`, `cgroup_enable=memory`, and
`cgroup_enable=cpuset` to `/boot/firmware/cmdline.txt` (the kernel
command line) and reboot. Without this, k3s fails to start with errors
about missing cgroup controllers.

The role reads the current cmdline, only appends parameters that
aren't already there, and triggers a reboot handler if anything
changed. The reboot handler uses the `reboot` module, which waits for
the host to come back via a `test_command` before letting the playbook
continue.

## Concept: handlers

A handler is a task that runs only if "notified" by another task that
actually made a change. Pattern:

    - name: Edit some config
      lineinfile:
        ...
      notify: restart service

    handlers:
      - name: restart service
        service:
          name: foo
          state: restarted

If the `lineinfile` task is `ok` (file already correct), the handler
doesn't fire. If it's `changed`, the handler runs once at the end of
the play (not immediately — handlers run after all tasks finish, so
if multiple tasks notify the same handler, it only fires once).

This is how you avoid restarting services unnecessarily: the restart
only happens when the config actually changed.

## Concept: `validate:` for config-file edits

Modules that write files (`copy`, `template`, `lineinfile`,
`blockinfile`) accept a `validate:` parameter. After writing the new
content to a temp file, Ansible runs the validate command with `%s`
replaced by the temp path. If the command fails, the original file is
left untouched and the task fails.

Use this whenever the target tool has a syntax-check mode:
- `sshd -t -f %s` for sshd_config
- `nginx -t -c %s` for nginx
- `visudo -c -f %s` for sudoers
- `named-checkconf %s` for bind

It turns "I just deployed a broken config to 50 servers" into
"playbook failed, nothing changed."

## Concept: `changed_when:`

The `command` and `shell` modules always report `changed: true` because
Ansible can't introspect arbitrary commands to know whether they did
anything. To preserve idempotency, you tell Ansible *when* the task
should count as a change:

    - name: Disable swap
      command: swapoff -a
      when: ansible_swaptotal_mb > 0
      changed_when: ansible_swaptotal_mb > 0

Now re-runs report `ok` because there's no swap to disable.

## Gotcha: cloud-init and /etc/hosts on Ubuntu Pi images

Ubuntu Server images flashed with Raspberry Pi Imager bake user-data
into the SD card with `manage_etc_hosts: true`. This makes cloud-init
re-render `/etc/hosts` from a template **on every boot** (not just
first boot, despite common belief), wiping out any local edits.

Adding `manage_etc_hosts: false` in a drop-in does *not* fix this,
because cloud-init reads its config from a cached merged copy in
`/var/lib/cloud/instance/cloud-config.txt`, which was generated on
first boot from the user-data and is not refreshed.

The fix is two-pronged:

1. **Disable the `update_etc_hosts` module entirely** via a drop-in
   that overrides `cloud_init_modules`. Cloud-init drop-ins REPLACE
   list values rather than merging, so the drop-in must contain the
   full module list from `/etc/cloud/cloud.cfg` minus
   `update_etc_hosts`. If Ubuntu changes the default list in a
   future release, this drop-in must be updated.

2. **Run `cloud-init clean --logs`** to clear the instance cache so
   the override takes effect. This regenerates SSH host keys on next
   boot, which means you'll get a host key warning the first time
   you SSH to each Pi after applying this fix. Resolve with:

       ssh-keygen -R 192.168.1.230

   Then reconnect and accept the new key.

Diagnostic commands used during this debug session:

    # What's cloud-init's cached config say?
    sudo cat /var/lib/cloud/instance/cloud-config.txt | grep manage_etc_hosts

    # What did Pi Imager bake into user-data?
    sudo cat /var/lib/cloud/instance/user-data.txt

    # What modules does cloud-init run on boot?
    sudo grep -A 30 cloud_init_modules /etc/cloud/cloud.cfg

    # Has cloud-init logged anything about hosts this boot?
    sudo journalctl -b | grep -iE 'cloud.init.*hosts|update_etc_hosts'

## Lesson: debugging is observation, not guessing

Two failed fixes preceded the one that worked. Each was based on a
plausible theory ("a later drop-in is overriding ours", "the filename
sort order is wrong") that turned out to match a different problem
than the one we had. The actual fix only became clear after running
diagnostic commands that revealed:

- `manage_etc_hosts: true` was in the cached merged config, not in any
  drop-in we could see
- Pi Imager's user-data was the original source
- `update_etc_hosts` runs on every boot, not just first boot

Next time something behaves unexpectedly: **look first, hypothesize
second**. The 5 minutes spent reading actual state beats 30 minutes
of trying plausible-sounding fixes.
