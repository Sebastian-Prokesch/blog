---
title: "Hybrid IaC: Proxmox + Hetzner"
date: 2026-01-18
tags:
  - home-lab
  - terraform
  - proxmox
  - hetzner
  - kubernetes
categories:
  - home lab
author: Sebastian
showToc: true
TocOpen: false
draft: false
hidemeta: false
comments: false
disableShare: false
disableHLJS: false
hideSummary: false
searchHidden: true
ShowReadingTime: true
ShowBreadCrumbs: true
ShowPostNavLinks: true
ShowWordCount: true
ShowRssButtonInSectionTermList: true
UseHugoToc: true
summary: "A pragmatic way to provision a “hybrid” homelab: local Proxmox VMs plus one (or more) Hetzner Cloud nodes acting as public ingress, with clean boundaries and a safe handoff into configuration management."
cover:
  image: "cover-images/Hybrid-IaC-Proxmox-Hetzner.png"
  alt: "Hybrid IaC: Proxmox + Hetzner"
  hidden: false
editPost:
  appendFilePath: true
---

## Hybrid homelab foundations: Proxmox baseline, Hetzner Cloud ingress

Hybrid here means: local capacity for the “always on” baseline, plus optional cloud capacity you can scale up and down. The main reason I keep an edge node in the cloud is simple: it can act as the ingress node with a stable public IP, so I don’t have to expose my home connection to the public internet.

My cluster is split across two worlds:

- **Local compute** on Proxmox (where the bulk of workloads run)
- **Hetzner Cloud worker nodes** acting as the public ingress layer (stable public IP(s), without exposing my home IP)

The goal is not “maximum automation”. The goal is a workflow where I can:

- create/replace nodes repeatably
- have a clear handoff into day-2 operations (upgrades, config changes, joining a cluster)

This post focuses on **Terraform boundaries**, a **Proxmox template workflow that is safe to clone**, and a simple **Terraform → inventory** handoff that doesn’t turn into “Terraform does everything”.

## Boundaries: what Terraform owns (and what it shouldn’t)

Terraform is great at **creating** and **wiring** infrastructure objects. It’s less great at:

- ongoing drift management inside machines
- orchestrating multi-step node lifecycle operations (drain, replace, join, validate)
- operating long-lived software stacks (RKE2 upgrades, kernel tuning, etc.)

So I like the following boundary:

- **Terraform**: networks, VMs, cloud instances, DNS records (if needed), firewall rules, static IPs, and the “inventory contract” (outputs).
- **Config management (Ansible, scripts, etc.)**: OS config, package installs, Kubernetes installation/upgrade, node labeling/taints, joining/removing nodes, and application bootstrap.

The rest of this post stays on the Terraform side: provisioning infrastructure and producing a clean inventory handoff. The actual node configuration and Kubernetes/RKE2 operations (Ansible and day-2 management) will be covered separately.

## Proxmox: build a “clone-safe” Ubuntu cloud-init template

To make VM provisioning reliable, I start with a Proxmox VM template built from an official Ubuntu cloud image. The template is cloud-init enabled, and I use a vendor snippet to install and start the QEMU guest agent on first boot. This approach keeps the base image clean and makes clones behave predictably.

This “template-first” approach is also the practical baseline for Proxmox automation. The alternative would be creating empty VMs and installing the OS from an ISO, which is harder to make repeatable at scale.

For Proxmox provisioning, a common choice is the `Telmate/proxmox` provider (docs: [Telmate Proxmox provider](https://registry.terraform.io/providers/Telmate/proxmox/latest/docs)).

### Use Proxmox snippets as vendor cloud-init config

Instead of installing packages into the image, keep the image pristine and use cloud-init itself to run first-boot commands.

In my setup, I keep a Proxmox snippet like:

- `vendor.yaml` (stored under Proxmox snippets, e.g. `/var/lib/vz/snippets/vendor.yaml`)
- and attach it to the VM/template via `qm set ... --cicustom vendor=...`

Example vendor config (minimal):

```yaml
#cloud-config
runcmd:
  - apt update
  - apt install -y qemu-guest-agent
  - systemctl start qemu-guest-agent
```

### Template creation (high-level workflow)

My template creation process looks like this:

1. Download Ubuntu cloud image (e.g. Ubuntu Noble)
2. Create a VM, import the disk, attach cloud-init drive
3. Set serial console and basic networking
4. Attach `vendor.yaml` snippet
5. Enable QEMU guest agent
6. Convert VM to a Proxmox template

I keep the actual `qm ...` script as a repeatable “make template” job on the Proxmox host so I can rebuild templates without guessing. Since I use Ubuntu LTS for most VMs, I typically only rebuild the template occasionally (for example on a major LTS jump).

## Hetzner cloud workers: keep the public edge simple

The Hetzner edge exists for one reason: **stable public IP(s)** that can handle ingress without exposing my home IP or dealing with dynamic DNS.

Terraform should own the boring-but-important bits here:

- instance creation
- SSH access (via keys)
- firewall rules / security groups
- (optional) reserved IP / floating IP if you use it

Everything else (installing Kubernetes components, joining the cluster, node hardening) belongs to config management.

## Repo layout: “stacks” first, modules second

In theory, I like the clean “modules + envs” layout. In practice, I find it more useful to group Terraform configuration by “what it provisions” (Kubernetes nodes, edge nodes, subnet router, GitLab, etc.) and wire it together from a central `main.tf`, producing a **single state** for the homelab.

One example structure for a Terraform project like this looks like:

```text
gitlab-runner/
gitlab-vm/
hetzner-edge/
kubernetes-cluster/
subnet-router/
main.tf
variables.tf
output/
```

If I were refactoring over time, I’d keep the “stack folders” but slowly extract shared bits into `modules/` once duplication starts to hurt.

## The handoff: Terraform renders an Ansible inventory (INI)

In my setup, Terraform produces an **Ansible inventory in INI format** directly.

High-level flow:

```text
terraform apply
terraform output -raw ansible_inventory > inventory/hosts.ini
ansible-playbook -i inventory/hosts.ini site.yml
```

This still follows the same boundary: Terraform owns the **inventory contract** (what hosts exist + how to reach them), while config management owns what happens inside the machines.

### How to model this in Terraform (conceptual)

The pattern is: build a list/map of nodes in Terraform, render an INI template, expose it as a string output.

```hcl
output "ansible_inventory" {
  value = templatefile("./${path.module}/templates/k8s.tpl", {
    k8s_masters       = local.k8s_masters
    k8s_local_workers = local.k8s_local_workers
    k8s_remote_workers = local.k8s_remote_workers
  })
}
```

Even if you don’t use modules yet, having this one output makes your Terraform stacks usable by ops tooling without manual glue.

### Example INI inventory template (what I actually use)

This is the template shape I like because it is explicit, minimal, and maps directly to RKE2 roles:

```hcl
[masters]
%{ for instance in k8s_masters ~}
${instance.name}   ansible_host=${instance.default_ipv4_address}    rke2_type=server
%{ endfor ~}

[local_workers]
%{ for instance in k8s_local_workers ~}
${instance.name}   ansible_host=${instance.default_ipv4_address}    rke2_type=agent
%{ endfor ~}

[remote_workers]
%{ for instance in k8s_remote_workers ~}
${instance.name}   ansible_host=${instance.default_ipv4_address}    rke2_type=agent
%{ endfor ~}

[workers:children]
local_workers
remote_workers

[k8s_cluster:children]
masters
workers
```

Notes:

- I split workers into **`local_workers`** and **`remote_workers`** so I can keep “common worker” logic shared, while still targeting edge-specific config (ingress/firewall/routing) to the remote nodes.
- **`workers:children`** keeps a convenient “all workers” group.
- **`k8s_cluster:children`** is the “run on everything” umbrella.
- `rke2_type` is a simple variable the playbook can use to decide whether to install a **server** or an **agent**.
- For multi-arch clusters, I set the architecture **per node** when needed. For example, on ARM nodes I add `rke2_architecture=arm64` to that host line. The default in my playbooks is `amd64`, which is why I can omit it on x86 nodes.

Example host line for an ARM64 worker:

```text
pi-worker-01 ansible_host=192.168.100.50 rke2_type=agent rke2_architecture=arm64
```
- `ansible_host` points to `default_ipv4_address` in my case. The important part is that your inventory consistently uses the interface you actually manage the machines over.

