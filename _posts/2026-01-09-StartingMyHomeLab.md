---
title: "Homelab Part 1: Getting started..."
date: 2026-01-09
categories: [Blogging, Homelab]
tags: [homelab, proxmox, kubernetes, k8s, dns, vpn, haos]

description: First post in a series about setting up a 'homelab'
---
First ever post - let's see how this goes...

## Introduction
Over the past few months I've gotten caught up in the world of [homelab](https://old.reddit.com/r/homelab), an online community of people who like tinkering with hardware and run a variety of software on their home network. But before we get into that, I'll have to start at the beginning...


For the longest time I've primarily been an Apple laptop person, but my trusty MacBook Air finally stopped working, so I had a choice. I could fork over quite a bit of money for a new or refurbished MacBook Pro or I could finally get around to finally learning Linux. After scouring various subreddits and forums I landed on a trusty Lenovo ThinkPad; I installed Ubuntu 24.04 LTS and have officially fallen in love with Linux.


I'm not completely unfamiliar with Linux. As a Software Engineer in the Infrastructure organization at Affirm I, of course, had extensive exposure to machines running various Linux distributions. I'd never run a Linux distribution on my personal machine nor had I ever set up a server from a scratch - my experience was just from the cloud :cloud:.


After a few weeks of getting used to Ubuntu and far too many times messing up `NetworkManager`, among other things. I decided it was time for me to start running some of these services I'd heard people discuss on r/homelab. I saw quite a few posts from folks talking about `HA` and found that really confusing... "High Availability what"?? Soon found out they're talking about [Home Assistant](https://www.home-assistant.io/).


My apartment had a few Wi-Fi connected outlets I used for scheduling my espresso machine on and off as well as my lights, but I wasn't crazy about the fact these required different iOS apps and no doubt they communicated data back to the vendor's cloud. I wanted to isolate my home network.

## Enter Proxmox Virtual Environment!
From what I'd read on r/homelab it seemed like Proxmox VE was a solid place to start, but before I could start running services like Home Assistant I needed a host. In recent years I've made a concentrated effort to buy used whenever possible so I turned to eBay for my hardware.

Not one to jump into a hobby anything but feet first I decided I *needed* to run a Proxmox cluster, so I ended up buying 5 mini PCs off eBay. I've learned quite a bit about hardware since purchasing those PCs several months ago, but they'll suffice for my needs.

```
Dell OptiPlex 7050 Micro
* 4 x Intel(R) Core i5-6500T CPU
* 16 GiB RAM (upgraded from 8 GiB)
* 128 GiB SATA SSD
* 500 GB NVMe SSD (upgraded; originally came without)
* 1 gigabit NIC
```

I configured 3 of them in a Proxmox cluster with a shared ZFS pool. They are so named: `campari-pve`, `cynar-pve`, and `fernet-branca-pve`. If it wasn't obvious they're named after famous [amaro](https://en.wikipedia.org/wiki/Amaro_(liqueur). These hosts are connected to my LAN via a modest managed TP-Link switch.

This cluster isn't quite set up how I envisioned it to be; it's not HA and it's not currently set up for replication. I've decided (and will discuss in a future blog post) that I'd rather spend that time and energy when I ultimately migrate this setup to Kubernetes - more to come. 

Right now, the services I'm running are as follows: 

| Host              | Service           | Deployment | Notes                     |
|-------------------|-------------------|------------|---------------------------|
| campari-pve       | Home Assistant OS | VM         | smart home                |
|                   | Talos Linux OS    | VM         | worker node               |
| cynar-pve         | AdGuard Home      | LXC        | DNS & ad/tracker blocking |
|                   | Caddy             | LXC        | reverse proxy             |
|                   | Wireguard         | LXC        | VPN                       |
|                   | Talos Linux OS    | VM         | worker node               |
| fernet-branca-pve | Jellyfin          | LXC        | media server              |


### Why these services?
As I mentioned earlier I was inspired to start down this journey because I wanted to "isolate my network" and setting up Home Assistant has certainly achieved that, but I also wanted to learn. Admittedly, my familiarity and knowledge of this area (virtualization, networking, various protocols, router configuration) was limited at the beginning, but I've learned a ton along the way. 

After I got Home Assistant up I began to wonder how I might access it when outside of my LAN and I also wanted to be able to reach it on my laptop without having to remember its IP address. So that problem led me to Wireguard and Caddy. 

I decided to buy a cheap domain, configured a simple DDNS script on the AdGuard LXC, and created subdomains for all of my services so now I have:

```
homeassistant.{{myDomain}}.com
wireguard.{{myDomain}}.com
jellyfin.{{myDomain}}.com
nas.{{myDomain}}.com
robovac.{{myDoamin}}.com
dns.{{myDomain}}.com
```

Caddy easily takes care of creating and renewing TLS certificates for all these site's with API keys from my domain registrar, porkbun. And AdGuard Home handles the DNS re-writes. I only registered `wireguard.{{myDomain}}.com` with porkbun, so that I only needed to expose `51820` over UDP for wireguard. 

## What's next...

Now that I've played around with Proxmox quite a bit and explored what services are possible to run at home I think it's time to apply some of my professional skills and experience towards a HA Kubernetes cluster. I'm thinking Talos. To be continued...

