---
title: "Configuring Kubernetes"
date: '2025-05-26'
weight: 4
draft: false
description: "This guide enables hardware-accelerated GPU passthrough from Proxmox VE hosts through Kubernetes VMs to containers, allowing applications like Jellyfin and Plex to utilize Intel integrated GPUs while maintaining Proxmox console functionality."
slug: "configuring-kubernetes"
tags: ["gpu", "kubernetes", "intel", "k3s", "k8s"]
series: ["Intel iGPU Split Passthrough"]
series_order: 4
showDate: true
showAuthor: true
showReadingTime: true
---
You have everything in place to make Kubernetes GPU-aware! For that you'll be using the [Intel Device Plugins for Kubernetes](https://github.com/intel/intel-device-plugins-for-kubernetes).

## Configuration
### Step 1: Label GPU-Enabled Nodes
First, label each GPU-equipped node so the Intel Device Plugin Operator knows where to work its magic (replace `<node-name>` with your own):
```bash
kubectl label nodes <node-name> intel.feature.node.kubernetes.io/gpu=true
```

### Step 2: Install Intel Device Plugin Components
{{< alert icon="circle-info" >}}
**NOTE**<br />**Update (July 2025):** When I first documented this process, I was living the manual life – `helm install` this, `kubectl apply` that. Since then, I've migrated to FluxCD for GitOps-based deployments. For those interested in the automated approach, my [home-ops repository](https://github.com/bykaj/home-ops/tree/main/kubernetes/apps/system/intel-device-plugin) shows how to deploy Intel Device Plugins declaratively.
<br />**Update (October 2025):** And... it's gone. The Intel Device Plugins in my repo I mean. I migrated to the [Intel Resource Drivers](https://github.com/intel/intel-resource-drivers-for-kubernetes) (in beta!) by using Dynamic Resource Allocation (DRA) and a Container Device Interface (CDI) for Kubernetes. This [commit](https://github.com/bykaj/home-ops/commit/be4131c5dc3432f4124101e2af1bd7cfcdc457ee) outlines the changes.
{{< /alert >}}

Grab the Intel Helm charts and update:
```bash
helm repo add intel https://intel.github.io/helm-charts/
helm repo update
```

Deploy the Device Plugin Operator:
```bash
helm install --namespace=system intel-device-plugins-operator intel/intel-device-plugins-operator
```

### Step 3: Configure the GPU Device Plugin
Create `values.yaml` to define your sharing strategy:
```yaml
name: i915
sharedDevNum: 1         # Maximum pods per GPU
nodeFeatureRule: false  # Disable automatic node feature discovery
```

Deploy the GPU plugin with your created config:
```bash
helm install --namespace=system intel-device-plugins-gpu intel/intel-device-plugins-gpu -f values.yaml
```

## Using GPU Resources
### Resource Requests
You can now configure pods to request GPU resources through Helm charts or Deployment manifests:
```yaml
resources:
  limits:
    gpu.intel.com/i915: "1"
  requests:
    gpu.intel.com/i915: "1"
```

### Node Selection
Got a mixed cluster? Force GPU workloads to GPU enabled nodes:
```yaml
nodeSelector:
  intel.feature.node.kubernetes.io/gpu: "true"
```

## Final Check
When you deployed your first application with a resource request for a GPU, verify within the container by executing:
```bash
ls /dev/dri
```

Sweet success looks (again) like this:
```
by-path  card0  renderD128
```
**Congratulations!** You've just pulled off the triple axel of virtualization: GPU passthrough from host to VM to container. Your media stack is now hardware-accelerated, your streams are smooth, and somewhere, a CPU is breathing a sigh of relief. Happy transcoding!
