---
title: "Tailscale Subnet Router"
date: 2026-01-28
tags:
  - home-lab
  - tailscale
  - proxmox
  - networking
  - security
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
summary: "Make hybrid networking boring: route securely from your Tailscale tailnet into a dedicated homelab VLAN using a Proxmox LXC subnet router, with a least-privilege ACL model, clear failure modes, and a validation checklist."
cover:
  image: "cover-images/Tailscale-Subnet-Router.png"
  alt: "Tailscale Subnet Router"
  hidden: false
editPost:
  appendFilePath: true
---

## Introduction: make “hybrid networking boring”

Most of my Kubernetes cluster runs **at home**, VMs on a **Proxmox hypervisor** plus a couple of **Raspberry Pis**, all sitting in a **dedicated homelab VLAN**.

At the same time, I also have “remote” nodes (for example a Hetzner worker that handles public ingress) and devices (laptop/phone) that must reach services inside that home VLAN — without exposing my home IP address to the public internet.

This post shows a pragmatic pattern that keeps things understandable:

- Put **one dedicated subnet router** in your tailnet (a Proxmox **LXC**, unprivileged)
- Advertise only the needed **home subnets**
- Control access with **Tailscale ACLs** (least privilege)
- Know the **failure modes** and how to validate/troubleshoot

## Architecture overview

The key idea is simple: a subnet-router node sits in your tailnet and provides routed access into your home VLAN.

Architecture at a glance:

- Tailnet peers (remote nodes/devices) connect to the **subnet-router LXC** over Tailscale.
- The subnet router forwards traffic into the **home VLAN** for the advertised CIDRs.
- Home VLAN services remain private; access is controlled via **ACLs** + **home firewalling**.

### What the subnet router does (and doesn’t)

- **Does**: route traffic from tailnet peers to specific home VLAN CIDRs
- **Doesn’t**: replace a firewall, an ingress controller, or identity-based access control

## Design decisions (security boundaries)

### Why a dedicated router node

Advertising routes from “random” nodes works, until it doesn’t: it blurs security boundaries and makes it hard to reason about what can reach your LAN.

A dedicated subnet-router node gives you:

- A single, explicit choke point
- A stable place to apply hardening, logging, and change control
- A clear ACL target (tags)

### Provisioning the LXC via Terraform (Telmate/Proxmox)

In my setup, I provision this subnet-router container via Terraform using the `Telmate/proxmox` provider. This keeps the router reproducible: if the container is gone or needs rebuilding, I can recreate it like any other piece of infrastructure.

A minimal module can look like this (placeholders; adapt to your Proxmox environment, template, and networking):

```hcl
terraform {
  required_providers {
    proxmox = {
      source = "Telmate/proxmox"
    }
  }
}

variable "target_node" { type = string }
variable "hostname" { type = string }
variable "lan_bridge" { type = string }   # e.g. "vmbr0"
variable "lan_ip_cidr" { type = string }  # e.g. "192.168.100.10/24"
variable "lan_gw" { type = string }       # e.g. "192.168.100.1"

resource "proxmox_lxc" "subnet_router" {
  target_node  = var.target_node
  hostname     = var.hostname
  ostemplate   = "local:vztmpl/debian-12-standard_12.2-1_amd64.tar.zst"
  unprivileged = true

  cores  = 1
  memory = 512
  swap   = 0

  rootfs {
    storage = "local-lvm"
    size    = "4G"
  }

  network {
    name   = "eth0"
    bridge = var.lan_bridge
    ip     = var.lan_ip_cidr
    gw     = var.lan_gw
  }
}
```

I keep the *software configuration* (installing Tailscale, sysctls, hardening) out of Terraform and handle it with configuration management or a simple bootstrap script. That mirrors the same “Terraform provisions, CM operates” boundary used elsewhere in this homelab.

### Give the container TUN access

Tailscale needs a TUN device. In Proxmox LXC terms, that means binding `/dev/net/tun` into the container and allowing the character device.

On Proxmox, this is typically configured in the container’s config (most commonly: `/etc/pve/lxc/<CTID>.conf`). The exact knobs vary by Proxmox/LXC version, but the intent is:

- allow the TUN character device
- bind-mount `/dev/net/tun` into the container

If `tailscaled` fails to start and logs mention TUN permissions, revisit this step.

Example config snippet (common on recent Proxmox versions; adjust to your setup):

```ini
# allow TUN character device
lxc.cgroup2.devices.allow: c 10:200 rwm

# bind-mount /dev/net/tun into the container
lxc.mount.entry: /dev/net/tun dev/net/tun none bind,create=file
```

### Enable IP forwarding

Enable forwarding inside the container (at least IPv4):

- `net.ipv4.ip_forward = 1`

If you advertise IPv6 routes later, also enable the IPv6 forwarding sysctls and update ACLs accordingly.

Example:

```bash
echo 'net.ipv4.ip_forward=1' | sudo tee /etc/sysctl.d/99-tailscale-subnet-router.conf
sudo sysctl --system
```

## Install and configure Tailscale subnet routing

Install Tailscale in the container and bring it up with:

- advertised routes for your home VLAN subnet(s)
- an advertised tag so you can write ACLs against it

Example (placeholders):

```bash
sudo tailscale up \
  --advertise-routes=192.168.100.0/24 \
  --advertise-tags=tag:subnet-router
```

Then, in the Tailscale admin console:

- approve the advertised routes
- verify they’re enabled

## ACL strategy (least privilege)

The pattern:

- tag the router: `tag:subnet-router`
- only allow approved identities/groups to reach the home VLAN CIDRs they need

At minimum you’ll want:

- `tagOwners` for `tag:subnet-router`
- ACL rules that allow a limited set of sources to reach a limited set of destinations

In practice, I prefer to model this as a dedicated group for **Kubernetes nodes** (for example cloud workers), so you can grant them access to home subnets without also granting it to every device in your tailnet.

`group:k8s-remote-nodes` contains only your remote Kubernetes nodes (Hetzner workers, etc.). Those nodes can talk to the home VLAN, but your other tailnet devices remain separate unless explicitly allowed.

Example ACL skeleton (minimal; adapt groups, CIDRs, and ports):

```json
{
  "tagOwners": {
    "tag:subnet-router": ["group:admins"]
  },
  "acls": [
    {
      "action": "accept",
      "src": ["group:k8s-remote-nodes"],
      "dst": [
        "192.168.100.0/24:*"
      ]
    }
  ]
}
```

If you want to be stricter, narrow the destination to specific ports (for example `:443`, `:22`, `:5432`) and create a separate rule per service boundary.

## Validation checklist

On the subnet-router LXC:

- `tailscale status` shows it connected
- the routes appear as advertised

On a remote node (or laptop) that should reach the home VLAN:

- confirm route selection (`ip route` and/or `ip rule`, depending on OS)
- `ping` a LAN IP (if permitted)
- `curl` or `nc` a specific service/port you expect to be reachable


## Troubleshooting: “works on the node, fails in pods” (Hetzner worker catch)

This one is subtle and easy to miss.

On some Linux hosts, Tailscale installs routes into a **policy routing table** (for example, table `52`) and relies on `ip rule` ordering. On a Kubernetes worker, you can end up with a situation where:

- traffic to the home VLAN works fine **from the node itself**
- the same traffic from **pods** does **not** follow the Tailscale policy route, and instead goes out via the default gateway (for example, the Hetzner network)

### Confirm the symptoms

Check whether Tailscale is using a separate routing table:

```bash
ip route show table 52
```

And inspect rules:

```bash
ip rule show
```

### Fix: add an explicit destination rule for the home subnet

Add a rule that forces traffic *to your home VLAN CIDR* to consult Tailscale’s routing table:

```bash
sudo ip rule add to 192.168.100.0/24 pref 5000 lookup 52
```

### Make it persistent: systemd override for `tailscaled`

Create:

- `/etc/systemd/system/tailscaled.service.d/override.conf`

With:

```ini
[Service]
ExecStartPost=/usr/bin/bash -c 'ip rule add to 192.168.100.0/24 pref 5000 lookup 52'
```

Then reload systemd and restart Tailscale:

```bash
sudo systemctl daemon-reload
sudo systemctl restart tailscaled
```

Reference: `https://mrkaran.dev/posts/travel-tailscale/`.

## Wrap-up

This pattern keeps the networking story simple:

- home VLAN stays private
- remote nodes/devices get routed access only where needed
- ACLs remain the primary control plane, not ad-hoc firewall rules

One more “defense in depth” note: I still enforce additional firewall rules in the home network (for example on the VLAN gateway/firewall) to treat traffic coming *from outside the home* as untrusted by default — even if it arrives over Tailscale. ACLs decide who can route; the home firewall can still enforce what those peers may do once they reach the VLAN.

