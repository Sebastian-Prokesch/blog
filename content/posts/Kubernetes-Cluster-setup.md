---
title: Kubernetes Cluster Setup
date: 2024-08-31
tags:
  - home-lab
  - kubernetes
  - tailscale
categories:
  - kubernetes
author: Sebastian
showToc: true
TocOpen: false
draft: false
hidemeta: false
comments: false
disableHLJS: false
disableShare: false
hideSummary: false
searchHidden: true
ShowReadingTime: true
ShowBreadCrumbs: true
ShowPostNavLinks: true
ShowWordCount: true
ShowRssButtonInSectionTermList: true
UseHugoToc: true
cover:
  hidden: true
editPost:
  appendFilePath: true
---
## Overview Kubernetes components

On my little home lab host, I run a small Kubernetes cluster mainly to tinker around with. However, in the last months, I have started migrating some productive applications to it. The cluster should be able to serve the public internet without exposing my home IP address, which also changes every 24 hours, and I was not particularly eager to play around with DynDNS. For that, I found a pretty elegant solution with an exposed node in the Hetzner cloud that handles all ingress and Tailscale as peer-to-peer VPN to connect all nodes and functions as the interface for the cluster CNI.
But first, a rundown of the nodes that the cluster is composed of.  
### Local components VM 
The main workload is on my [home lab server, which I described here](/posts/homelab-setup-2024/). Two control nodes and two worker nodes are currently running. The CPU on the control nodes might be too high, but I over-provisioned the proxmox node, and all the VMs are sharing the CPU resources. Currently, I don't run in any CPU limits. Should that happen, I will either upgrade the server or start looking into expanding Proxmox to a multi-node setup 😛. 

> control nodes specs:
> CPU: 4
> RAM: 4 GBi
> worker nodes specs: 
> CPU: 8 
> RAM: 8 GBi
### Local components RaspberryPi
Two Raspberry Pis are also part of the Cluster: a Raspberry Pi 4 as the control node and a Raspberry Pi 5 as the worker node. Both Raspberry Pis have 8 GBi of RAM. RKE2, the Kubernetes distro I'm using, has supported ARM64 since 1.27. 
### Remote Hetzner VM
Lastly for my "ingress" node I run the cheapest [Hetzner VPS](https://www.hetzner.com/de/cloud/) a two core 4 GBi machine. 
## Network using tailscale
I have a dedicated VLAN for all the Kubernetes nodes I run in my home network. However, to connect the VPS, I installed tailscale on all of them and configured the CNI. RKE2 comes with support for multiple CNI. The default one is canal. I deployed this canal configuration on all nodes, setting the interface to use `tailscale0`. 

```yaml
---
apiVersion: helm.cattle.io/v1
kind: HelmChartConfig
metadata:
	name: rke2-canal
	namespace: kube-system
spec:
	valuesContent: |-
		flannel:
		iface: "tailscale0"
	calico:
		vethuMTU: 1180
```
Tailscale tries to connect over all interfaces it finds on a machine having a lot of interfaces created by the CNI that can lead to some problems. To force tailscale to only use the main NIC I found this easy to setup systemd units, [tailgate by fernandoenzo](https://github.com/fernandoenzo/tailgate). This unit let's you configure a whitelist of interface tailscale should use. 
## Storage provider
For storage, I run TrueNAS on the home lab server. To provide persistent storage for the Cluster, I use [democatic-csi](https://github.com/democratic-csi/democratic-csi). This CSI can use the TrueNAS API to automatically provision and mount Storage for PVCs created on the cluster.
## Future changes
Overall, the current setup works quite well. Building it, I learned a great deal about tailscale, network, and operating a Kubernetes Cluster and RKE2 in particular. 
I didn't discuss how I manage the nodes using Ansible or how I deploy applications on the cluster; that will come in future posts.  