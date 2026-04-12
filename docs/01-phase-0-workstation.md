# Phase 0 — Workstation prep

Goal: turn a Windows 11 machine into a usable Linux control station for
managing the cluster, with all the CLI tools we'll need in later phases.

## Why WSL2 and not native Windows?

Ansible doesn't run natively on Windows (it's Python + SSH and assumes a
POSIX environment). kubectl, helm, and flux all *do* have Windows
builds, but every tutorial, every Helm chart, and every error message
on the internet assumes you're on Linux. WSL2 gives us a real Ubuntu
kernel running alongside Windows with near-native performance, full
filesystem access in both directions, and one consistent shell
environment. It's the right tool.

## Step 1 — Install WSL2

In **PowerShell as Administrator**:

    wsl --install -d Ubuntu-24.04

This enables the WSL Windows feature, enables Virtual Machine Platform,
downloads the WSL2 kernel, sets WSL2 as default, and installs Ubuntu
24.04 — all in one command. A reboot is usually required partway
through; reopen PowerShell after rebooting and the install resumes.

When Ubuntu finishes, it asks for a UNIX username and password. This
account is independent from your Windows user. The password is for
`sudo` inside WSL.

Verify you're on WSL2 (not WSL1) from PowerShell:

    wsl -l -v

`Ubuntu-24.04` should show `VERSION 2`. WSL1 has networking quirks that
break things later — if you see a 1, fix it before continuing.

## Step 2 — Update the package index

Inside the WSL Ubuntu shell:

    sudo apt update && sudo apt upgrade -y

Standard hygiene. WSL Ubuntu images can be a few weeks behind on
release.

## Step 3 — Install basics

    sudo apt install -y git curl wget vim ca-certificates gnupg \
        lsb-release software-properties-common python3-pip pipx

Nothing exotic: `git` for version control, `curl`/`wget` for fetching
installers, `ca-certificates`/`gnupg`/`lsb-release` because nearly every
"add a third-party apt repo" guide needs them, `pipx` for installing
Ansible.

## Step 4 — Install Ansible (via pipx)

    pipx install --include-deps ansible
    pipx ensurepath

Then **close and reopen** the WSL terminal so the PATH update takes
effect, and verify:

    ansible --version

**Why pipx, not `apt install ansible`?** Ubuntu's apt version of Ansible
lags behind by months. pipx installs Ansible into its own isolated
Python environment, gives you the latest version, and keeps it
upgradable independently of system Python. This is the modern
recommended install method.

**What is Ansible?** A configuration management tool. You write YAML
files (called "playbooks") describing the desired state of a machine —
"this hostname, this package installed, this service running, this
file present" — and Ansible SSHes into the target machines and makes
reality match. It's idempotent: run it ten times, the result is the
same as running it once. We'll use it in Phase 1 to provision the 3
Pis from this WSL machine.

## Step 5 — Install kubectl

    curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
    sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
    rm kubectl
    kubectl version --client

The nested `curl` fetches the current stable Kubernetes version string,
then downloads the matching kubectl binary.

**What is kubectl?** *The* CLI for talking to a Kubernetes API server.
Every operation against the cluster — list pods, create a deployment,
read logs, exec into a container — goes through it. It reads
`~/.kube/config` to know where the cluster is and how to authenticate.
We'll populate that file in Phase 2 after k3s is running.

**Why manual install instead of apt?** The official k8s apt repo pins
you to a specific minor version (1.30, 1.31...) and you have to add a
new repo each time k8s releases a new minor. Manual install is simpler
for a homelab.

## Step 6 — Install Helm

    curl -fsSL https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
    helm version

**What is Helm?** The Kubernetes package manager. A "Helm chart" is a
bundle of templated YAML manifests with configurable values — instead
of writing 400 lines of YAML by hand to deploy Forgejo, you reference
a chart and override the few values you care about. Even though we're
going GitOps with Flux, Flux *uses* Helm under the hood (it has a
`HelmRelease` resource type), and you'll occasionally want `helm`
locally for inspecting charts and templating them out.

**On `curl | bash`:** you're trusting the script to do what it claims.
For official projects (Helm, kubectl, Flux) this is the documented
install method and is generally fine, but it's worth knowing what
you're agreeing to. If you want to be cautious, download the script
first, read it, then run it.

## Step 7 — Install Flux CLI

    curl -s https://fluxcd.io/install.sh | sudo bash
    flux --version

**What is Flux?** The GitOps controller we picked. The CLI you just
installed runs on your laptop and is mostly used for *bootstrapping*
("here's a Git repo, install yourself into the cluster and start
watching it") and *debugging* ("why hasn't this app reconciled?"). The
actual Flux *controllers* run as pods inside the cluster, installed in
Phase 4. For now the CLI just sits ready.

## Step 8 — Sanity check

    ansible --version && kubectl version --client && helm version && flux --version

All four should print versions cleanly.

## Step 9 — Create the homelab repo folder

    mkdir -p ~/homelab/docs
    cd ~/homelab

**Important:** keep the repo on the **WSL filesystem** (`~/homelab`), not
on `/mnt/c/Users/...`. Files on the Windows side are dramatically
slower to access from WSL and Ansible/git operations will feel
sluggish. To edit from VS Code on Windows, install the "WSL"
extension — it opens WSL folders natively.

We'll `git init` this folder in Phase 1 when we have something worth
committing.

## Decisions made in this phase

- WSL2 + Ubuntu 24.04 as the control station.
- Toolchain installed: ansible, kubectl, helm, flux.
- VIP for the cluster API: **192.168.1.240**.
- Repo location: **`~/homelab`** inside WSL.
