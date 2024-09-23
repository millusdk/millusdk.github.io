---
layout: post
title:  "Homelab setup"
date:   2024-09-22 10:04:15 +0200
categories: homelab
---

As I wrote in my previous post, I have a homelab setup at home (surprise).
In this post I will go through the hardware and the software that make up my systems.

My homelab currently consists of three physical servers, one of which is a NAS, and the two others being used as hosts for virtual machines and containers.

## NAS
For my NAS I use a custom built server based around a A2SDi-8C-HLN4F board from Supermicro. This board consists of a Intel Atom C3758 CPU with four cores and eight threads, plenty for a small home NAS. For memory it runs 16 GB of unregistered ECC memory.

For software on my NAS I have chosen TrueNAS Core, as it is a stable system, and has robust support for the ZFS filesystem, that I intend to run, protecting my data from failure.

### Storage
For storage I have gone with four 4 TB drives, and a single 6 TB drive. I have chosen NAS drives from different vendors, and I have purchased them over a longer time from different stores, with the intent of ensuring that they are from different batches, and that way hopefully spread out when they fail, allowing me to replace them one by one. The 6 TB drive is the last one I purchased, and allows me to expand the storage slightly at a later point if I replace all 4 TB drives with similar 6 TB or larger drives.

I have configured the drives as a RAIDZ2 data vdev, protecting me from up to two drive failures, and giving me around 10GB of usable space in the pool.

## Virtual Machine hosts
For my virtual machine hosts, I am running a pair of Intel NUCs. More specifically the NUC11TNHi50L00. These computers feature an Intel Core i5-1135G7 CPU with 4 cores and 8 threads. That should give it plenty of power to run the workloads that I intend to run on it. I have populated each of them with 32 GB of memory, again, this should be plenty for the workloads I intend.

I have specifically chosen NUCs from the 11th generation, as Proxmox, which I intend to run as the hypervisor, as there have been some issues running Proxmox on the newer generations of Intel CPUs with LITTLE.big architecture.

### Hypervisor
As the hypervisor for the virtual machine servers, I have chosen Proxmox. I have chosen Proxmox because it is relatively easy to setup, supports both full virtual machines and LXC containers, supports clustered setup, and has a GUI that makes it realtively easy to work with.

Since I have two servers, I have set them up in a two-node cluster. Running only two nodes means that I can't have High Availability for the services that I run, however, given that this is for a homelab and not a production setup, that is not really a big problem.

## Services
As far as services go, I currently run a very limited set of services on the Proxmox nodes. The services I am currently running are:
- Pi-hole
- Plex

In terms of pi-hole, each node runs its own pi-hole instance, which is set as the primary DNS server for all LXC containers on that node, with the pi-hole instance on the other node set as the secondary. This means that all containers should be able to reach at least one DNS server at all times, given that there is no reason for me to restart the nodes simultaneously.

I use Plex to store all my media files, such as movies and TV-series. Allowing me and the rest of the family to stream these to the various screens we have in the house, be it TV's, tablets of phones. Since I don't have a ton of storage on the NUCs, all the media is stored on the NAS, with the relevant folders mounted on the host and mapped into the container, granting Plex access to the files.