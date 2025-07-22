---
title: "Intel iGPU Split Passthrough"
description: "This guide enables hardware-accelerated GPU passthrough from Proxmox VE hosts through Kubernetes VMs to containers, allowing applications like Jellyfin and Plex to utilize Intel integrated GPUs while maintaining Proxmox console functionality."
summary: "Share the GPU Love: iGPU Split Passthrough to Kubernetes Containers"
tags: ["guide", "gpu", "virtualization", "kubernetes", "intel", "proxmox", "k3s", "k8s", "talos"]
---

{{< lead >}}
Share the GPU Love: Intel iGPU Split Passthrough to Kubernetes Containers
{{< /lead >}}

Let's face it: GPUs are too valueable to leave idle. This guide shows you how to squeeze every drop of performance from your Intel iGPU by sharing it across your entire virtualization stack – from Proxmox host to Kubernetes pods. Your media servers will thank you, your power bill will love you, and yes, you'll still have a console to admire your work. Just don't expect to train your next AI model with this setup – leave those workloads to the big GPUs.

---