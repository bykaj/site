---
title: 'Intel iGPU Split Passthrough'
date: '2025-05-26'
weight: 1
draft: false
description: "Transform your virtualization stack with Intel iGPU GVT-g split passthrough! Route GPU power from Proxmox VE hosts through Kubernetes VMs straight to your containers – hello hardware-accelerated Jellyfin and Plex! Bonus: your Proxmox console stays functional. Just don't expect to train your next AI model with this setup – leave those workloads to the big GPUs."
tags: ["gpu", "virtualization", "kubernetes", "intel", "proxmox", "k3s", "k8s"]
showDate: true
---
## Overview
Transform your virtualization stack with Intel iGPU GVT-g split passthrough! Route GPU power from [Proxmox VE](https://www.proxmox.com/) hosts through Kubernetes VMs straight to your containers – hello hardware-accelerated [Jellyfin](https://jellyfin.org/) and [Plex](https://plex.tv/)! Bonus: your Proxmox console stays functional. Just don't expect to train your next AI model with this setup – leave those workloads to the big GPUs.

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
- And any other virtualization-related option you'll can find

You can do this in [Step 4](#step-4-reboot) where you have to reboot your host anyway.

### Disable VM Autostart (Optional)
To make your life easier, disable the autostart for the Kubernetes node VMs in Proxmox VE so they stay in a stopped state after the host reboot. Re-enable it when you're finished with this guide.

Ok, with that out of the way, let's get started.

## Configuring the Proxmox hosts
Access your Proxmox host via SSH or the web GUI's Shell option.

### Step 1: Enable IOMMU and Intel GVT-g
Edit the GRUB configuration file `/etc/default/grub` and update the `GRUB_CMDLINE_LINUX_DEFAULT` variable to:
```bash
GRUB_CMDLINE_LINUX_DEFAULT="intel_iommu=on iommu=pt i915.enable_gvt=1 pcie_acs_override=downstream,multifunction"
```
This is why you need this: `intel_iommu=on iommu=pt i915.enable_gvt=1` enables the core trio of IOMMU, passthrough, and GVT-g (mediated devices) functionality. The `pcie_acs_override=downstream,multifunction` parameter does the heavy lifting by forcing hardware into separate IOMMU groups. No more dealing with device clusters – you get individual control over each piece of hardware, plus the sweet benefit of a stable, non-crashing host.ß

Apply these new boot parameters by running:
```bash
sudo proxmox-boot-tool refresh
```

### Step 2: Load Required Kernel Modules
Those parameters won't work their magic without the right kernel modules loaded first. Add these to `/etc/modules`:
```bash
# Modules required for PCI passthrough
vfio
vfio_iommu_type1
vfio_pci
vfio_virqfd

# Modules required for Intel GVT-g Split
kvmgt
vfio_mdev
i915
```

### Step 3: Update Boot Image
Regenerate the boot image to include the new modules:
```bash
sudo update-initramfs -u -k all
```

### Step 4: Reboot
{{< alert icon="triangle-exclamation" >}}
**WARNING**<br />Make sure you drain your Kubernetes nodes on the Proxmox host before rebooting!
{{< /alert >}}

Restart your Proxmox host for all changes to take effect:
```bash
sudo reboot
```
While rebooting, make sure you enabled all virtualization options in your BIOS like described [here](#bios-virtualization-settings).

### Step 5: Verify Configuration After Reboot
After reboot, verify the devices are properly separated into different IOMMU groups. Replace `<node>` with your actual PVE node name:
```bash
sudo pvesh get /nodes/<node>/hardware/pci --pci-class-blacklist ""
```

Your output should look similar to this:
```
┌──────────┬────────┬──────────────┬────────────┬────────┬──────────────────────────────────────
│ class    │ device │ id           │ iommugroup │ vendor │ device_name
╞══════════╪════════╪══════════════╪════════════╪════════╪══════════════════════════════════════
...
│ 0x020000 │ 0x154d │ 0000:01:00.0 │         10 │ 0x8086 │ Ethernet 10G 2P X520 Adapter
├──────────┼────────┼──────────────┼────────────┼────────┼──────────────────────────────────────
│ 0x020000 │ 0x154d │ 0000:01:00.1 │         11 │ 0x8086 │ Ethernet 10G 2P X520 Adapter
├──────────┼────────┼──────────────┼────────────┼────────┼──────────────────────────────────────
│ 0x030000 │ 0x3e92 │ 0000:00:02.0 │          0 │ 0x8086 │ CoffeeLake-S GT2 [UHD Graphics 630]
├──────────┼────────┼──────────────┼────────────┼────────┼──────────────────────────────────────
│ 0x040380 │ 0xa348 │ 0000:00:1f.3 │          9 │ 0x8086 │ Cannon Lake PCH cAVS
...
```
Take note of the GPU ID (which is `0000:02:00.0` in this example), as you'll use it in the next step.

### Step 6: Confirm GVT-g Support
Verify that GVT-g split is working and mediated devices are available. Replace `<gpu id>` with your GPU's ID from the previous step:
 ```bash
 sudo ls /sys/bus/pci/devices/<gpu id>/mdev_supported_types
```

You should see the following output:
```
i915-GVTg_V5_4  i915-GVTg_V5_8
```

### Step 7: Configure Additional Hosts
If you have multiple Proxmox VE hosts with GPUs, repeat these steps on each host.

{{< alert icon="lightbulb" >}}
**TIP**<br />If you have a Proxmox VE cluster, you can create a resource mapping (`Datacenter → Resource Mappings`). This mapping can then be added to your VM hardware instead of a directly connected raw device.
![Resource Mappings in the Datacenter view](img/igpu-passthrough-01.png "Resource Mappings in the Datacenter view")
{{< /alert >}}

## Adding the GPU to the Kubernetes Nodes
Time to add the GPU! For this your Kubernetes node VM needs to be powered off first. Hot-plugging isn't an option here. If you disabled the autostart option, it should still be powered off after the last host reboot.

### Step 1: Configure GPU Passthrough in Proxmox
Add the GPU via a mediated device (`MDev Type`) to your Kubernetes node VM using either a resource mapping or as a raw device:
![Adding the device to the VM](img/igpu-passthrough-02.png "Adding the device to the VM")

![Overview of the VM configuration](img/igpu-passthrough-03.png "Overview of the VM configuration")

**Important:** Ensure that `vIOMMU` in the Machine configuration is set to `Default (None)`.

### Step 2: Disable Secure Boot
Two ways to do this.

#### Method 1: EFI Disk Replacement
Simply remove the current EFI disk and add a new one, ensuring `Pre-Enroll keys` is disabled:
![EFI disk with the Pre-Enroll keys option disabled](img/igpu-passthrough-04.png "EFI disk with the Pre-Enroll keys option disabled")

#### Method 2: BIOS Configuration
Alternatively, disable Secure Boot through the VM's BIOS:
1. Start the VM and quickly open the console
2. Press `Esc` repeatedly during boot to enter the BIOS
3. Navigate to `Device Manager → Secure Boot` and disable it (multiple confirmations required)
4. Save settings and reboot

### Step 3: Install GPU Drivers
Your approach depends on your OS.

#### For Talos Systens
Add the official `i915` system extension to your installation. Extension management is beyond this guide's scope, but here's what you need:
```yaml
customization:
  systemExtensions:
    officialExtensions:
	  - siderolabs/i915
```

#### For Minimal or Cloud-Based Installations
SSH into your node and install the GPU drivers, reboot afterwards:
```bash
sudo apt install linux-generic -y
sudo reboot
```
#### For Other Distributions
Most full distributions include these drivers by default. Verify that `linux-generic` is installed.

### Step 4: Verify GPU Passthrough
Let's confirm your GPU made it through.

#### Verification on Talos
```bash
talosctl -n <node-name> ls /dev/dri		# Use your actual node name
```
Expected output:
```bash
NODE          NAME
<node-name>   .
<node-name>   by-path
<node-name>   card0						# Your GPU!
<node-name>   renderD128				# The render device
```

#### Verification on Other Systems
Log into your node and execute:
```bash
ls /dev/dri
```
Should show:
```bash
by-path  card0  renderD128				# card0 is your GPU!
```

### Step 5: Configure Additional Nodes
Rinse and repeat! Apply these steps to every Kubernetes node that needs GPU support.


## K3s
More information:
- https://github.com/UntouchedWagons/K3S-Intel
- https://www.reddit.com/r/selfhosted/comments/121vb07/plex_on_kubernetes_with_intel_igpu_passthrough/

Tag the agent nodes that have GPUs. The tag is used by the Intel Device Plugin Operator. Change `<node-name>`  to your K3s Agent Node name.
```bash
kubectl label nodes <node-name> intel.feature.node.kubernetes.io/gpu=true
```

Install the [Intel Device Plugins](https://github.com/intel/helm-charts/). Add the Helm repo and update:
```bash
helm repo add intel https://intel.github.io/helm-charts/
helm repo update
```

Install the Operator:
```bash
helm install --namespace=intel-device-plugins --create-namespace device-plugin-operator intel/intel-device-plugins-operator
```

Make a `values.yaml` file with the following content:
```yaml
sharedDevNum: 1       # Number of pods that are allowed to use the GPU
nodeFeatureRule: true
```

Install the GPU device plugin for the Operator with the created `values.yaml`:
```bash
helm install --namespace=intel-device-plugins --create-namespace gpu-device-plugin intel/intel-device-plugins-gpu -f values.yaml
```

You can now request the GPU in a container with the following configuration override:
```yaml
resources:
  limits:
	gpu.intel.com/i915: "1"
  requests:
	gpu.intel.com/i915: "1"
```

(Optional) If some but not all of your nodes have an Intel GPU you'll want to set up a node selector in your deployment:
```yaml
nodeSelector:
  intel.feature.node.kubernetes.io/gpu: "true"
```

You can check if all the steps went successful by going into the container shell and execute:
```bash
ls /dev/dri
```

You should see the following result:
```
by-path  card0  renderD128
```

## Applications

> [!note]
> Note that the 6th Gen Core with HD 5xx iGPUs lacks 10-bit support, it's best to choose 7th Gen and newer processors, which usually have HD / UHD 6xx series iGPUs.

### Jellyfin
See this guide about the encoding and decoding capabilities of your iGPU: https://jellyfin.org/docs/general/post-install/transcoding/hardware-acceleration/intel

### Plex

### Step 8: Test GPU Functionality (Optional)
To verify GPU transcoding capabilities, install the Intel GPU tools:
```bash
sudo apt install intel-gpu-tools -y
```

When an application (like Plex) is using the GPU you can check the GPU transcoding status with:
```bash
intel_gpu_top
```
