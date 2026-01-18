---
title: Homelab Setup 2024
date: 2024-08-15
tags:
  - home-lab
  - kubernetes
  - proxmox
  - tailscale
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
summary: Descriping my homelab setup, the hardware used, the main components and a short introduction into the Software stack.
cover:
  hidden: true
editPost:
  appendFilePath: true
---

## Why my home lab

The goal of this post is to describe my home lab setup, primarily for myself, so that I have a reference point to compare all the changes that will surely happen over the next months and years.

Operating this home lab is mainly about having a system to tinker with and try new technologies, which is the same reason most of you have a home lab. But I also consider some services "production" systems, like the local DNS servers I'm running or the NAS where my backups and other large file sets are stored. Energy cost is also a consideration. That's why I don't have multiple compute nodes and a separate NAS box. Everything is put into a single tower case that's sound-damped and fits nicely in a corner next to my working Desk. 

## Hardware of the lab
I got the server over a year ago, and some of the components were not the latest at that time already. I've chosen these components mostly because they balance price and performance well. 
The case has a lot of space for 3.5 HDD disks, and the sound-damping, together with three be quiet! 120mm fans keep the temperatures at a reasonable level without causing any noise from it. 
The CPU is from the last AM4 generation and is an eight-core version hitting the 65-watt TDP target. The integrated GPU was planned for use as an encoding accelerator, but forwarding it into a VM was more challenging than I anticipated. 
All Storage I use for my NAS is attached to the HBA, which is forwarded as a PCI device to the NAS VM. 
### Server components
* Case: [Nanoxia Deep Silence 8 Pro](https://amzn.to/4cB1Y22)
* Board: [AsRock Rack X470D4U](https://amzn.to/4dOJYT6)
* CPU: [AMD Ryzen 7 5700G](https://amzn.to/3SU1wFo)
* RAM: 4x Samsung 32GB DDR4-2933 ECC
* Power Supply: [be quiet! 600W System Power 9](https://amzn.to/4csgAkd)
* HBA: [SATA/SAS-Controller, LSI 2308](https://amzn.to/3WTOmcF)
* SSD: 2x [1TB WD Blue SN550](https://amzn.to/3Mbi5bV)
* HDD: 4x [Toshiba 8TB N300](https://amzn.to/3Mg0iQI)

## Hypervisor and NAS
I went with [Proxmox](https://www.proxmox.com/en/) as a Hypervisor platform. It's easy to install and use. The web UI may look dated, but it's clearly structured. The hardware support is excellent. You can use everything in the community version for free. Proxmox is widely used in home lab community, and you can find a lot of tutorials about it, for example the [install tutorial by Jeff from Craft Computing](https://youtu.be/sZcOlW-DwrU?si=fC0IYMJYf_Ju9FzM).

I currently operate 6 VMs and 2 LXCs. One of the LXCs is a Docker host running [Portainer](https://www.portainer.io/) and a minecraft Bedrock server. The other LXC acting as Tailscale subnet router, more on that later in the Kubernetes-tailscale section.
One VM is a TrueNAS install with the forwarded HBA, acting as my primary storage appliance. Currently, it consists of two Storage Pools on the Raid-Z pool, with the four spinning rust 8TB HDDs giving me around 24 TB of storage capacity and a second Pool composed of two 1 TB SSDs I had lying around in a mirror configuration. 
The second VM is a [GitLab](https://about.gitlab.com) installation I use for local code storage and as a private docker registry.  
The other four VMs are part of my Kubernetes Cluster: two Control-Plane Nodes acting as API-Server and ETCD-Host, as well as two Worker nodes. 
## Kubernetes Cluster
My Kubernetes Cluster setup is a bit more complex. I use [RKE2](https://docs.rke2.io) mainly the Clusters I have to deal with in my day job use this as the Kubernetes distribution.  
The Cluster consists of the VMs on my Proxmox host and also of two RaspberyPi nodes and a VM on the [Hetzner Cloud](https://www.hetzner.com/cloud/). One PI 4  acts as the third Control Plane, and recently, I added a Pi 5 as an additional worker node. This allows me to build ARM64 software and Docker images. The Hetzner-cx22 worker node comes with a fixed IP address, which makes it easy to point DNS records to it and set up the cluster ingress to listen on this IP. There is no need for any DyDNS setups and exposing my changing IP address. 
### Tailscale
To connect the K8s sitting in my private IP network to the "public" node, I use Tailscale as the peer-to-peer VPN software. It creates a Tailscale interface on each node the CNI binds to and uses it for all in-cluster traffic. A dedicated LXC acts as a sub-net router, giving the remote node access to the NAS on the same local network. 

This is a high-level overview of my current home lab system. In future posts, I will go into further detail on different aspects of this setup. Software that I'm using, tools to manage these systems and more.
