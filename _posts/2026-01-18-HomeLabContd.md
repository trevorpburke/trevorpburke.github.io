---
title: "Homelab Part 2: Bootstrapping Talos k8s cluster"
date: 2026-01-21
categories: [Blogging, Homelab]
tags: [homelab, kubernetes, k8s, talos, cka]

description: Second post in a series about setting up a 'homelab'
---
Over the last few months I've been studying for the [Certified Kubernetes Administrator exam](https://training.linuxfoundation.org/certification/certified-kubernetes-administrator-cka/). I have quite a bit of experience with Kubernetes, but not so much with cluster management and the underlying storage and networking foundation. To prepare for the exam I've enrolled in the `Certified Kubernetes Administrator (CKA) With Practice Tests` Udemy course (shout out San Francisco Public Library for the free Udemy account!). As I've progressed through the course I've found that it might be helpful to actually apply some of these lessons in a semi real world environment. And this thinking has led me to Talos Linux.

## Talos Linux

Talos Linux is a minimal Linux distribution to run Kubernetes. It removes some of the headaches with managing OS-level dependencies if you were to deploy Kubernetes on Ubuntu/Debian/RHEL nodes. With Talos you can focus just on Kubernetes and its requirements. For my bare metal, mini business PC application it seemed like an ideal fit to create a High Availability Kubernetes cluster. 

## Bootstrapping Talos and Initial Setup

To refresh, I purchased several Dell Optiplex 7050 Micro PCs off eBay several months ago to build a Proxmox VE cluster. 3/5 of those machines have been repurposed as worker nodes for my Talos cluster. One of the nodes continues to run Proxmox - mainly to separate my DNS server and VPN server from k8s. It also runs a Home Assistant OS VM. For the control plane nodes, I found a good deal on a lot of three HP EliteDesk 800 G3 mini PCs - they all have 8GiB of RAM and a 250GB SSD.

Setting up this cluster began with creating images for these machines via the [Talos Image Factory](https://factory.talos.dev/). I selected bare-metal hardware type, amd64 architecture, and did not choose any system extensions. This was the first of a few mistakes I made in the initial OS installation.

I knew that I'd like to port my Jellyfin media server from Proxmox to this Talos Kubernetes cluster and I knew that I'd like to continue hardware accelerated transcoding. So I should have installed both `siderolabs/i915` and `siderolabs/intel-ucode` extensions when setting up the nodes. Also, I knew that I'd like to try out [Longhorn](https://longhorn.io/) for its distributed block storage capabilities and its secondary storage backup functionality. So I should have also included `siderolabs/iscsi-tools`, as well. For good measure it also seemed like including `siderolabs/util-linux-tools` would be a good call too, so that nodes would have common command line tools like `cat`, `dd`, `fstrim`, and others.

Thankfully, installing these system extensions after installation is pretty dang simple. As with all other aspects of Talos management I turned to `talosctl` to update my nodes. But before running any commands I needed to generate a new Talos image, with the necessary extensions, to create a new schematic ID via the image factory. With that I could run: 

```shell
# '.204' is one of my worker nodes
talosctl upgrade --nodes 192.168.50.204 --image factory.talos.dev/metal-installer/$SCHEMATIC_ID:v1.12.1```
```

I ran the above command on one of my worker nodes first then one of my control plane nodes, then I combined the remaining nodes into one command to "upgrade" them with the new extensions. 

### Machine Configuration

I won't spend too much time on how to install Talos as I believe the [official documentation](https://docs.siderolabs.com/talos/v1.12/getting-started/getting-started) covers this area quite well, but I will cover some of the nuances I ran into that weren't totally clear to me. 

I knew that I wanted to set up a High Availability cluster using Talos, but I don't think setting up and managing a separate load balancer was what I was looking for. So, I was happy to learn about more about Talos [Virtual (shared) IP](https://docs.siderolabs.com/talos/v1.11/networking/vip). The way this works is best explained directly from the horse's mouth:

> What happens is that the controlplane machines vie for control of the shared IP address using etcd elections. There can be only one owner of the IP address at any given time. If that owner disappears or becomes non-responsive, another owner will be chosen, and it will take up the IP address.

For me, I decided to update my router's DHCP range to `192.168.50.2` to `192.168.50.199` so that I could reserve `192.168.50.200` to `192.168.50.255` for my Talos nodes, Talos VIP, and MetalLB (more on that later).

| IP Address                    | Hostname        | Function                |
|-------------------------------|-----------------|-------------------------|
| 192.168.50.200                | n/a             | VIP                     |
| 192.168.50.201                | control-plane-1 | control plane node      |
| 192.168.50.202                | control-plane-2 | control plane node      |
| 192.168.50.203                | control-plane-3 | control plane node      |
| 192.168.50.204                | worker-node-1   | worker node             |
| 192.168.50.205                | worker-node-2   | worker node             |
| 192.168.50.206                | worker-node-3   | worker node             |
| 192.168.50.207-192.168.50.255 | n/a             | MetalLB IP Address Pool |

I'd figured out how I wanted to handle IP addresses and IP address ranges for my cluster and its services. So, the next step was to actually update the nodes accordingly. I accomplished this with two main files in addition to `controlplane.yaml` and `worker.yaml`, which are created when you first run `talosctl gen config`.

This brings me to multi-doc configurations, which is a new-ish feature in Talos v1.12. The Sidero Labs folks actually put out a pretty [helpful video](https://www.youtube.com/watch?v=0lrFpqi4bOI) about this topic that's worth checking out. But I'll share how I applied it to my usecase here. 

For each node, I created a `$nodeName-multidoc.yaml` file and a `$nodeName-patch.yaml` file. For control plane nodes, `$control-plane-1-multidoc.yaml` file looked like this:


```yaml
apiVersion: v1alpha1
kind: HostnameConfig
hostname: control-plane-1
---
apiVersion: v1alpha1
kind: Layer2VIPConfig
name: 192.168.50.200
link: eno1
---
apiVersion: v1alpha1
kind: LinkConfig
name: eno1
up: true
addresses:
  - address: 192.168.50.201/24
routes:
  - gateway: 192.168.50.1
```

Here I've specified a hostname, configured `192.168.50.200` as the VIP, and instructed Talos to use `192.168.50.201` for this node's IP address. When initially configuring Talos nodes in "maintenance mode" my router's DHCP server assigned each node a random IP address. So, to actually apply this and `$nodeName-patch.yaml` I would run something like: 

```shell
# After apply, `control-plane-1` will be assigned `192.168.50.201`
talosctl apply -f controlplane.yaml -i -n 192.168.50.156 -p '@control-plane-1-multidoc.yaml' -p '@control-plane-1-patch.yaml'
```

The `$nodeName-multidoc.yaml` looked the same for worker nodes, but had the `Layer2VIPConfig` omitted. And the `$nodeName-patch.yaml` file simply specified where to install Talos OS. For example: `control-plane-1-patch.yaml`

```yaml
machine:
  install:
    disk: /dev/sdb
```

And, finally I enabled metrics by following [this guide](https://docs.siderolabs.com/kubernetes-guides/monitoring-and-observability/deploy-metrics-server). The TL;DR for this is to update both `controlplane.yaml` and `worker.yaml` with the following prior to installation.

```yaml
machine:
  kubelet:
    extraArgs:
      rotate-server-certificates: true
cluster:
  extraManifests:
    - https://raw.githubusercontent.com/alex1989hu/kubelet-serving-cert-approver/main/deploy/standalone-install.yaml
    - https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

This may be obvious to some, but I was confused why `metrics-server-*` pod was stuck in Pending state. After I finished setting up worker nodes (started with control plane) the deployment was able to schedule pods. The `metrics-server` deployment does not have any tolerations to schedule on nodes with `NoSchedule` taints.


### Conclusion
Alright, that about covers the steps I took to get my 6 node Talos cluster up and running. In the next post I'll write about MetalLB, ArgoCD, and what else I plan to run on this cluster. I may also share a bit of the story around the physical setup. I've got a AC/DC power supply in the mail and I recently finished setting up a DeskPi RackMate for all these mini PCs. Once that's cleaned up a bit I'll share more. 