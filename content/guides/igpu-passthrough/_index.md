---
title: "Intel iGPU Split Passthrough"
description: "This guide enables hardware-accelerated GPU passthrough from Proxmox VE hosts through Kubernetes VMs to containers, allowing applications like Jellyfin and Plex to utilize Intel integrated GPUs while maintaining Proxmox console functionality."
summary: "Share the GPU Love: iGPU Split Passthrough to Kubernetes Containers"
tags: ["guide", "gpu", "virtualization", "kubernetes", "intel", "proxmox", "k3s", "k8s", "talos"]
---

{{< lead >}}
Share the GPU Love: Intel iGPU Split Passthrough to Kubernetes Containers
{{< /lead >}}

Transform your virtualization stack with Intel iGPU GVT-g split passthrough! Route GPU power from [Proxmox VE](https://www.proxmox.com/) hosts through Kubernetes VMs straight to your containers – hello hardware-accelerated [Jellyfin](https://jellyfin.org/) and [Plex](https://plex.tv/)! Bonus: your Proxmox console stays functional. Just don't expect to train your next AI model with this setup – leave those workloads to the big GPUs.

---