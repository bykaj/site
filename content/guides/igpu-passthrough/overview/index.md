---
title: "Overview & Prerequisites"
date: '2025-05-26'
weight: 1
draft: false
description: "This guide enables hardware-accelerated GPU passthrough from Proxmox VE hosts through Kubernetes VMs to containers, allowing applications like Jellyfin and Plex to utilize Intel integrated GPUs while maintaining Proxmox console functionality."
slug: "overview"
tags: ["gpu", "virtualization", "kubernetes", "intel", "proxmox", "k3s", "k8s", "talos"]
series: ["Intel iGPU Split Passthrough"]
series_order: 1
showDate: true
showAuthor: true
showReadingTime: true
---
## Overview
Let's face it: GPUs are too valuable to leave idle. This guide shows you how to squeeze every drop of performance from your Intel iGPU by sharing it across your entire virtualization stack – from [Proxmox VE](https://www.proxmox.com/) host to Kubernetes pods. Your media servers will thank you, your power bill will love you, and yes, you'll still have a console to admire your work. Just don't expect to train your next AI model with this setup – leave those workloads to the big GPUs.

What we're trying to accomplish:
{{< mermaid >}}
graph LR;
    A[Proxmox VE Host] --> |Passthrough| B[Kubernetes Node]
    B --> |Passthrough| C[Pod]
    C --> |Request| D[Container]
{{< /mermaid >}}

I've tested this guide across Proxmox VE 8.4.1 hosts powered by **Intel Core i5-8500T** (Coffee Lake) and **Intel Core i7-6700K** (Skylake) processors. The Kubernetes side covers both [K3s](https://k3s.io/) nodes running on Ubuntu 24.04 LTS (Noble) [cloud images](https://cloud-images.ubuntu.com) and [K8s](https://kubernetes.io) nodes on [Talos 1.10](https://www.talos.dev).

## Prerequisites
### CPU Generation
First things first – not all Intel CPUs can play this game. GVT-g Split Passthrough is exclusive to **Intel Core generations 5 to 10**. Supported CPU families:
- Broadwell (5<sup>th</sup> gen)
- Skylake (6<sup>th</sup> gen)
- Kaby Lake (7<sup>th</sup> gen)
- Coffee Lake (8<sup>th</sup> gen)
- Comet Lake (10<sup>th</sup> gen)

**12<sup>th</sup> gen and beyond?** You're in luck! Forget this guide and explore vGPU/SR-IOV – it splits your GPU into 7 devices!
**11<sup>th</sup> gen (Tiger Lake)?** The forgotten generation. Neither GVT-g nor SR-IOV will work for you. Sorry.

### Secure Boot
Hardware passthrough and Secure Boot don't play nice together – you'll need to disable Secure Boot in the virtual machine for this to work. I'll show you how in this guide. However, if you're running an OS that requires Secure Boot (like certain Talos images), you're facing a tough choice: reinstall with a non-Secure Boot image and rebuild your cluster from scratch.

### BIOS Virtualization Settings
Before anything else works, you'll need to enable the virtualization trinity in your BIOS:
- VT-d (Intel Virtualization for Directed I/O)
- IOMMU (Input-Output Memory Management Unit)
- VT-x (Intel Virtualization Technology)
- And any other virtualization-related option you can find

You can do this in [Step 4]({{< ref "configuring-proxmox#step-4-reboot" >}}) where you have to reboot your host anyway.

### Disable VM Autostart (Optional)
To make your life easier, disable the autostart for the Kubernetes node VMs in Proxmox VE so they stay in a stopped state after the host reboot. Re-enable it when you're finished with this guide.

Ok, with that out of the way, let's get started.
