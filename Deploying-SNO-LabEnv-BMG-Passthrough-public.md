> **Public-safe copy:** hostnames, IP addresses, MAC addresses, PCI addresses, UUIDs, user identifiers, local paths, and private links have been anonymized. Commands and network examples are reference material and are not directly runnable.

> **OCP version:** 4.22.0-ec.5 (pre-release as of May 2026)
>
> **Goal:** Deploy a single-node OpenShift cluster inside a libvirt/KVM virtual machine on an existing RHEL bare-metal host, preserving the host OS.
>
> **Host:** `REDACTED_HOSTNAME` -- currently RHEL 9.6. This guide covers both RHEL 9 and RHEL 10, noting differences where they exist.
>
> **Progress (as of 2026-05-21):** [[SNO]] cluster install complete (Steps 0-8 done). Worker node joined to cluster using RHCOS qcow2 image. GPU passthrough complete -- [[BMG|Battlemage]] dGPU visible inside worker VM with `xe` driver loaded and `/dev/dri/renderD128` present. Three host/VM changes required: IOMMU DMA mode, `maxphysaddr limit=42`, remove virtio-vga.

## Overview

The Agent-Based Installer (ABI) generates a self-contained bootable ISO that contains everything needed to deploy an OpenShift cluster -- no external SaaS, no bootstrap node, no internet required. You create two YAML files, generate the ISO, boot a VM from it, and wait.

This guide targets a **Single-Node OpenShift (SNO)** topology: one node acting as both control plane and worker. Worker VMs can be added day-2.

## Prerequisites

### Host machine

- RHEL 9.6+ or RHEL 10 bare-metal host with SSH access (in this guide: `REDACTED_HOSTNAME`)
- CPU with hardware virtualization (Intel VT-x / AMD-V)
- libvirt, QEMU-KVM, and virt-install installed:

  ```text
  sudo dnf install qemu-kvm libvirt virt-install
  sudo systemctl enable --now libvirtd
  ```

  > **Note on `RHEL_HOST`:** **Done (2026-05-20).** Repos fixed (`sed` to `latest-RHEL-9.6.0`), SAP/SAPHANA/buildroot repos disabled. `qemu-kvm 9.1.0`, `libvirt 10.10.0`, `virt-install 5.0.0` installed. `libvirtd` enabled and running. All packages updated. `/var/lib/libvirt/images/` created with `virt_image_t` SELinux context.

### Disable IPv6 on the host

Lab networks often advertise IPv6 router advertisements but don't actually route IPv6 traffic. This causes NTP sync failures in VMs (chrony tries IPv6 first and times out) and general DNS/connectivity issues. Disable IPv6 on the host's external interface:

```bash
# Replace enp129s0 with your host's external interface name
nmcli connection modify enp129s0 ipv6.method disabled
nmcli connection up enp129s0
```

Verify `ping google.com` uses IPv4 and succeeds.

### SNO VM resource requirements

| Resource | Minimum (docs) | SNO node | Worker node |
| -------- | -------------- | -------- | ----------- |
| vCPU     | 8              | 10       | 10          |
| RAM      | 16 GB          | 28 GB    | 16 GB       |
| Disk     | 120 GB         | 120 GB   | 120 GB      |

> **`RHEL_HOST` has 20 CPUs and 62 GB RAM.** An SNO VM (10 vCPU, 28 GB) plus a worker VM (10 vCPU, 16 GB) totals 20 vCPU (fully committed) and 44 GB RAM (18 GB headroom for the host). The worker VM RAM is kept at 16 GB to leave room for VFIO memory locking overhead when GPU passthrough is enabled (see PCI passthrough section).

### Disk layout

The host's 476 GB NVMe is fully allocated to LVM with no free extents:

| LV         | Size   | Free   |
| ---------- | ------ | ------ |
| `/` (root) | 70 GB  | 66 GB  |
| `/home`    | 374 GB | 372 GB |
| swap       | 31 GB  | --     |

Two 120 GB qcow2 VM images won't fit on root. Store them under `/home` instead:

```bash
sudo mkdir -p /var/lib/libvirt/images
```

The `virt-install` commands in this guide use `/var/lib/libvirt/images/` for disk paths.

### Accounts and artifacts

- A Red Hat account with an active OpenShift subscription
- A pull secret (link removed in public copy) from the Red Hat Hybrid Cloud Console
- An SSH keypair (the public key will be baked into the RHCOS node for `core` user access)

### DNS and DHCP

For platform `none` (which is what we use for KVM-hosted clusters), you are responsible for DNS. OpenShift requires the following records (for a cluster named `sno-cluster` with base domain `example.com`):

| Record                            | Value                                     |
| --------------------------------- | ----------------------------------------- |
| `api.sno-cluster.example.com`     | A record pointing to the VM's IP          |
| `api-int.sno-cluster.example.com` | A record pointing to the VM's IP          |
| `*.apps.sno-cluster.example.com`  | Wildcard A record pointing to the VM's IP |

For SNO, all three records point to the same IP. The wildcard `*.apps` entry is essential -- it routes all OpenShift routes (console, OAuth, user apps) to the ingress controller on the node. A plain `/etc/hosts` file cannot handle wildcards, so you need a real DNS server.

The easiest approach for a lab is to embed DNS and DHCP directly in a custom libvirt network (see Step 0 below), which uses dnsmasq under the hood. This gives you DHCP reservations, A records, and wildcard CNAME support all in one place, scoped to the VM network.

## Step 0: Set up the libvirt network with DNS and DHCP

Rather than using the default libvirt network, create a dedicated network that provides DHCP (with a static reservation for the SNO node) and DNS (including the required wildcard `*.apps` entry). Libvirt drives dnsmasq under the hood, so this is all configured in a single network XML file.

Save the following as `ocp-net.xml`:

```xml
<network xmlns:dnsmasq="REDACTED_URL">
  <name>ocp</name>
  <forward mode='nat'>
    <nat>
      <port start='1024' end='65535'/>
    </nat>
  </forward>
  <bridge name='virbr-ocp' stp='on' delay='0'/>
  <ip family='ipv4' address='REDACTED_IP' netmask='REDACTED_IP'>
    <dhcp>
      <range start='REDACTED_IP' end='REDACTED_IP'/>
      <host mac="REDACTED_MAC" name="sno-node.sno-cluster.example.com" ip="REDACTED_IP"/>
    </dhcp>
  </ip>
  <dns>
    <host ip="REDACTED_IP">
      <hostname>api.sno-cluster.example.com</hostname>
      <hostname>api-int.sno-cluster.example.com</hostname>
    </host>
  </dns>
  <dnsmasq:options>
    <!-- Wildcard *.apps route via CNAME to a host-record -->
    <dnsmasq:option value="host-record=lb.sno-cluster.example.com,REDACTED_IP"/>
    <dnsmasq:option value="cname=*.apps.sno-cluster.example.com,lb.sno-cluster.example.com"/>
    <!-- Make dnsmasq authoritative for the cluster domain to prevent 5s -->
    <!-- timeouts on AAAA queries that would otherwise go unanswered.   -->
    <!-- Trailing comma on auth-server means "no interface restriction." -->
    <dnsmasq:option value="auth-server=sno-cluster.example.com,"/>
    <dnsmasq:option value="auth-zone=sno-cluster.example.com"/>
  </dnsmasq:options>
</network>
```

This gives you:
- **DHCP** with a static lease: MAC `REDACTED_MAC` always gets `REDACTED_IP`.
- **`api` and `api-int` A records** via the `<dns><host>` block.
- **`*.apps` wildcard** via a dnsmasq CNAME pointing to a host-record.
- **NAT forwarding** so the VM can reach the internet to pull release images.

Create and start the network:

```bash
sudo virsh net-define ocp-net.xml
sudo virsh net-start ocp
sudo virsh net-autostart ocp
```

### Making the host resolve cluster DNS

The libvirt dnsmasq only listens on the bridge interface (`virbr-ocp` at `REDACTED_IP`), so by default your host cannot resolve `api.sno-cluster.example.com`. To fix this, tell NetworkManager to forward the cluster's domain to the libvirt dnsmasq:

```bash
sudo tee /etc/NetworkManager/dnsmasq.d/ocp-sno.conf <<EOF
server=/sno-cluster.example.com/REDACTED_IP
EOF

# If NetworkManager isn't already using dnsmasq as its DNS backend:
sudo tee /etc/NetworkManager/conf.d/00-dns-dnsmasq.conf <<EOF
[main]
dns=dnsmasq
EOF

sudo systemctl restart NetworkManager
```

Verify:

```bash
dig api.sno-cluster.example.com
dig console-openshift-console.apps.sno-cluster.example.com
```

Both should resolve to `REDACTED_IP`.

## Step 1: Download the installer and CLI tools

Since 4.22 is pre-release, pull the installer from the candidate-4.22 mirror (link removed in public copy):

```bash
MIRROR=REDACTED_URL

# Download the 4.22 candidate installer
curl -LO ${MIRROR}/openshift-install-linux.tar.gz
tar xzf openshift-install-linux.tar.gz
sudo mv openshift-install /usr/REDACTED_HOSTNAME/bin/

# Download the oc CLI
curl -LO ${MIRROR}/openshift-client-linux.tar.gz
tar xzf openshift-client-linux.tar.gz
sudo mv oc kubectl /usr/REDACTED_HOSTNAME/bin/
```

Verify:

```bash
openshift-install version
```

## Step 2: Prepare the configuration directory

```bash
mkdir ~/sno-install
```

## Step 3: Create install-config.yaml

Create `~/sno-install/install-config.yaml`:

```yaml
apiVersion: v1
baseDomain: example.com
compute:
- architecture: amd64
  name: worker
  replicas: 0
controlPlane:
  architecture: amd64
  name: master
  replicas: 1
metadata:
  name: sno-cluster
networking:
  clusterNetwork:
  - cidr: REDACTED_IP
    hostPrefix: 23
  machineNetwork:
  - cidr: REDACTED_IP
  networkType: OVNKubernetes
  serviceNetwork:
  - REDACTED_IP
platform:
  none: {}
fips: false
pullSecret: '<paste your pull secret here>'
sshKey: '<paste your SSH public key here>'
```

Key points:
- `compute.replicas: 0` and `controlPlane.replicas: 1` define a SNO topology.
- `platform: none` is required for KVM-hosted clusters. This means you are responsible for DNS and load balancing (for SNO, the node is the load balancer).
- `machineNetwork.cidr` must match the libvirt network your VM will be on. The `ocp` network created in Step 0 uses `REDACTED_IP`.
- Do not set `machineNetwork.cidr` to `REDACTED_IP` -- it conflicts with Podman's default bridge.

## Step 4: Create agent-config.yaml

Create `~/sno-install/agent-config.yaml`. The example below uses a static IP; for DHCP, you can omit the `networkConfig` block and just set `rendezvousIP` to the IP the DHCP server will assign.

### Static IP example

```yaml
apiVersion: v1beta1
kind: AgentConfig
metadata:
  name: sno-cluster
rendezvousIP: REDACTED_IP
hosts:
- hostname: sno-node
  role: master
  interfaces:
  - name: enp1s0
    macAddress: REDACTED_MAC
  rootDeviceHints:
    deviceName: /dev/vda
  networkConfig:
    interfaces:
    - name: enp1s0
      type: ethernet
      state: up
      mac-address: REDACTED_MAC
      ipv4:
        enabled: true
        address:
        - ip: REDACTED_IP
          prefix-length: 24
        dhcp: false
      ipv6:
        enabled: false
    dns-resolver:
      config:
        server:
        - REDACTED_IP
    routes:
      config:
      - destination: REDACTED_IP
        next-hop-address: REDACTED_IP
        next-hop-interface: enp1s0
        table-id: 254
```

### DHCP example

If you set up the libvirt network with a DHCP reservation as in Step 0 (MAC `REDACTED_MAC` -> `REDACTED_IP`), you can omit the `networkConfig` block entirely. The VM will get its IP, gateway, and DNS from the libvirt dnsmasq:

```yaml
apiVersion: v1beta1
kind: AgentConfig
metadata:
  name: sno-cluster
rendezvousIP: REDACTED_IP
hosts:
- hostname: sno-node
  role: master
  interfaces:
  - name: enp1s0
    macAddress: REDACTED_MAC
```

The DHCP reservation in the network XML guarantees the VM always gets `REDACTED_IP`, so `rendezvousIP` will match even though the address is assigned dynamically. `rootDeviceHints` is omitted here because a single-disk VM only has `/dev/vda` -- no disambiguation needed.

Key points:
- `rendezvousIP` is the IP of the node that runs the bootstrap/assisted-service. For SNO there is only one node, so this is its IP.
- `macAddress` must match the MAC address of the VM's network interface. You choose this value now and use the same one when creating the VM with `virt-install`.
- The `networkConfig` block uses NMState (link removed in public copy) format.
- `rootDeviceHints` is optional -- only needed when the host has multiple disks and you need to specify which one gets RHCOS.
- The interface name `enp1s0` assumes a single virtio NIC on the default Q35 machine type. If you use a different machine type or add multiple NICs, the name may differ. Check with `virsh domiflist sno-node` and match by MAC address.

## Step 5: Generate the agent ISO

```bash
openshift-install --dir ~/sno-install agent create image
```

This consumes both YAML files and produces `~/sno-install/agent.x86_64.iso`. The YAML files are deleted after ISO generation, so back them up first if you want to keep them.

## Step 6: Create and boot the VM

Copy the ISO to the libvirt images directory and create the VM:

```bash
sudo cp ~/sno-install/agent.x86_64.iso /var/lib/libvirt/images/

sudo virt-install \
  --name sno-node \
  --ram 28672 \
  --vcpus 10 \
  --cpu host \
  --disk path=/var/lib/libvirt/images/sno-node.qcow2,size=120,bus=virtio \
  --cdrom /var/lib/libvirt/images/agent.x86_64.iso \
  --network network=ocp,mac=REDACTED_MAC \
  --os-variant rhel9-unknown \
  --boot hd,cdrom \
  --noautoconsole
```

After creating the VM, set autostart so it survives host reboots:

```bash
sudo virsh autostart sno-node
```

Key points:
- The `mac=` sub-option on `--network` must match the `macAddress` in your `agent-config.yaml` (and the DHCP reservation in the network XML).
- `--cpu host` uses the host-model CPU mode, which selects the closest named CPU model matching the host. This is sufficient for OpenShift. Use `--cpu host-passthrough` if you need exact CPU passthrough (e.g., for nested virtualization).
- `--cdrom` handles the initial boot from the agent ISO. `--boot hd,cdrom` sets the post-install boot priority: hard disk first (where RHCOS gets installed), then CDROM as fallback. On first boot the empty disk is skipped and the ISO boots; after RHCOS is installed and the VM reboots, it boots from disk.
- `--os-variant rhel9-unknown` describes the guest OS (RHCOS is based on RHEL 9), not the host. On RHEL 9.6 you can also use `rhel9.6`; verify available variants with `osinfo-query os | grep rhel9`.
- `--noautoconsole` returns control to your shell immediately while the VM continues running in the background. The agent ISO handles installation and reboots autonomously -- no manual intervention needed.
- **Important:** After the agent writes RHCOS to disk, the VM shuts down (rather than rebooting). This is a `virt-install --cdrom` quirk. If the VM is in `shut off` state after the "Writing image to disk" / "Rebooting" stage, start it manually with `virsh start sno-node`.

To watch the VM console:

```bash
sudo virsh console sno-node
```

## Step 7: Monitor installation progress

From the host (or any machine with the `openshift-install` binary and access to the install directory):

```bash
# Wait for bootstrap to complete (rendezvous host reboots after this)
openshift-install --dir ~/sno-install agent wait-for bootstrap-complete \
  --log-level=info

# Wait for full install to complete
openshift-install --dir ~/sno-install agent wait-for install-complete
```

Expected output on success:

```text
INFO Cluster is installed
INFO Install complete!
INFO To access the cluster as the system:admin user when using 'oc', run
INFO   export KUBECONFIG=$HOME/sno-install/auth/kubeconfig
INFO Access the OpenShift web-console here: https://console-openshift-console.apps.sno-cluster.example.com
```

The full installation typically takes 30-45 minutes.

## Step 8: Access the cluster

```bash
export KUBECONFIG=~/sno-install/auth/kubeconfig

oc get nodes
oc get clusterversion
```

The `kubeadmin` password is in `~/sno-install/auth/kubeadmin-password`.

## Troubleshooting

### Installation stalls or fails

SSH into the node and gather logs:

```bash
ssh core@REDACTED_IP

# From the node:
sudo journalctl -u assisted-service -f
sudo journalctl -u bootkube -f
```

Or from the host, gather an agent diagnostic bundle:

```bash
ssh core@REDACTED_IP agent-gather -O >agent-gather.tar.xz
```

After bootstrap completes and nodes reboot, you can also use:

```bash
openshift-install --dir ~/sno-install agent wait-for install-complete --log-level=debug
```

### DNS resolution failures

The agent console TUI (visible on the VM console) will show red icons for failed checks. The most common failure is DNS. Verify resolution from the host:

```bash
dig @REDACTED_IP api.sno-cluster.example.com
dig @REDACTED_IP console-openshift-console.apps.sno-cluster.example.com
```

And from within the VM (SSH as `core@REDACTED_IP` once it's booted):

```bash
dig api.sno-cluster.example.com
```

If the libvirt network was defined correctly in Step 0, the VM gets `REDACTED_IP` as its DNS server via DHCP, so cluster DNS should work automatically within the VM.

### Release image pull failures

If the VM cannot reach `quay.io` to pull the release image, verify that:
1. The VM has internet access via the `ocp` libvirt network (which NATs through the host)
2. No firewall rules are blocking outbound HTTPS

## Adding worker nodes (day-2)

After the SNO cluster is running, you can add worker VMs. The approach below uses a pre-built RHCOS qcow2 image (faster and more reliable than booting the live ISO, which can stall during ignition processing).

### Extract the worker ignition

```bash
oc extract -n openshift-machine-api secret/worker-user-data-managed --keys=userData --to=- > /var/lib/libvirt/images/worker.ign
```

### Download the RHCOS qcow2 image

```bash
curl -L -o /var/lib/libvirt/images/rhcos-4.22.0-ec.5-x86_64-qemu.x86_64.qcow2.gz \
  REDACTED_URL
gunzip /var/lib/libvirt/images/rhcos-4.22.0-ec.5-x86_64-qemu.x86_64.qcow2.gz
```

### Create the worker disk as a qcow2 overlay

This keeps the base RHCOS image reusable for additional workers:

```bash
qemu-img create -f qcow2 \
  -b /var/lib/libvirt/images/rhcos-4.22.0-ec.5-x86_64-qemu.x86_64.qcow2 \
  -F qcow2 \
  /var/lib/libvirt/images/ocp-worker-0.qcow2 120G
```

### Create the worker VM (without GPU)

Do **not** include `--host-device` at this stage. OVMF firmware hangs when trying to initialize the Battlemage GPU during boot (it lacks a compatible UEFI GOP driver). Install the worker first, then add the GPU post-install (see PCI passthrough Step 6).

```bash
IGNITION_CONFIG="/var/lib/libvirt/images/worker.ign"

sudo virt-install \
  --name ocp-worker-0 \
  --ram 16384 \
  --vcpus 10 \
  --cpu host-passthrough \
  --import \
  --disk path=/var/lib/libvirt/images/ocp-worker-0.qcow2,bus=virtio \
  --network network=ocp,mac=REDACTED_MAC \
  --boot uefi \
  --os-variant rhel9-unknown \
  --qemu-commandline="-fw_cfg name=opt/com.coreos/config,file=${IGNITION_CONFIG}" \
  --noautoconsole
```

Key differences from the SNO VM:
- `--import` instead of `--cdrom` -- boots directly from the pre-installed qcow2 disk, no ISO needed.
- `--cpu host-passthrough` is required for GPU passthrough workloads.
- The `--qemu-commandline` fw_cfg passes the worker ignition to RHCOS on first boot. Ignition fetches the full config from the cluster's Machine Config Server (`api-int:22623`), sets up the kubelet, and the node registers with the cluster.

### Approve CSRs

After 2-3 minutes, the worker's kubelet will register with the cluster and submit certificate signing requests. There are two rounds:

```bash
# Watch for pending CSRs
oc get csr

# Approve each pending CSR (typically two: client cert, then server cert)
oc adm certificate approve <csr-name>
```

After approving both CSRs, the worker should transition to `Ready` within a few minutes:

```bash
oc get nodes
# worker-0   Ready    worker   ...
```

## PCI passthrough: Intel BMG GPU to a worker VM

This section covers passing through the Intel Battlemage discrete GPU (`REDACTED_BDF`, device ID `8086:e223`) from the bare-metal host to a KVM-based OpenShift worker VM.

### Known issue: QEMU 10.0 IGD misidentification (RHEL 10 only)

RHEL 10.0 ships `qemu-kvm 10.0.0`, which has a [bug](https://lists.nongnu.org/archive/html/qemu-devel/2025-02/msg04902.html) where Intel **discrete** GPUs (Arc, Battlemage) are misidentified as integrated graphics (IGD). This causes the VM to fail with:

```text
IGD device REDACTED_BDF is unsupported in legacy mode, try SandyBridge or newer
```

The fix was merged in QEMU 10.1. Check whether your `qemu-kvm` package includes the backport:

```bash
rpm -q --changelog qemu-kvm | grep -i -e IGD -e "discrete" -e "legacy mode"
```

If the fix is absent, file a bug with Red Hat to backport the QEMU 10.1 IGD patches to the RHEL 10 `qemu-kvm` package. **This bug does not affect RHEL 9** (qemu-kvm ~9.x).

### Known issue: xe driver on the host (RHEL 9.6, kernel 5.14)

On RHEL 9.6, the `xe` kernel module in kernel 5.14.0-687+ **does** include the Battlemage device ID `e223` in its PCI ID table (backported in-tree). However, the host kernel's xe driver may not fully support Battlemage initialization, and allowing it to bind would interfere with VFIO passthrough. Blacklist `xe` and `i915` on the host (see Step 3).

Inside the worker VM, the same RHCOS 9.8 kernel (5.14.0-687+) successfully loads xe, initializes GuC/HuC/DMC firmware, and exposes `/dev/dri/renderD128` for GPU compute workloads.

### Known issue: dual-VGA fbcon deadlock

When both virtio-vga (the emulated display) and a passthrough GPU are present, the guest kernel's `fbcon` layer deadlocks on DRM modeset locks. This manifests as `hung_task` timeouts on systemd, agetty, and kworkers ~2 minutes after boot. The fix is to remove the virtio-vga device (`virt-xml --remove-device --video all`) so only the passthrough GPU is present. See Step 6.

### Step 1: Verify IOMMU is enabled -- DONE

On `RHEL_HOST`, IOMMU is **already enabled via platform opt-in** (Arrow Lake forces it on). No kernel parameter changes are needed:

```bash
dmesg | grep -i -e DMAR -e IOMMU
# Should show: "DMAR: Intel-IOMMU force enabled due to platform opt in"
```

If your host does not show IOMMU as enabled, add kernel parameters to `/etc/default/grub`:

```bash
# Append to GRUB_CMDLINE_LINUX:
#   intel_iommu=on  -- enables VT-d
#   iommu=pt        -- passthrough mode: only VFIO-bound devices get IOMMU translation
sudo sed -i 's/\(GRUB_CMDLINE_LINUX="[^"]*\)/\1 intel_iommu=on iommu=pt/' /etc/default/grub
sudo grub2-mkconfig -o /boot/grub2/grub.cfg
sudo reboot
```

### Step 2: Check the IOMMU group -- DONE

All devices in the same IOMMU group must be passed through together. Check what shares a group with the GPU:

```bash
IOMMU_GROUP=$(basename "$(readlink /sys/bus/pci/devices/REDACTED_BDF/iommu_group)")
echo "GPU is in IOMMU Group: $IOMMU_GROUP"

for dev in /sys/kernel/iommu_groups/$IOMMU_GROUP/devices/*; do
    lspci -nns "$(basename "$dev")"
done
```

On `RHEL_HOST`, the GPU is **alone in IOMMU group 18** -- no audio function, no companion devices. This is the ideal case for passthrough.

Intel discrete GPUs sometimes have an associated audio function (e.g., `REDACTED_BDF` for HDMI/DP audio) in the same group. If so, both devices must be passed through. Unrelated devices in the same group are a problem -- see the [ArchWiki PCI passthrough guide](https://wiki.archlinux.org/title/PCI_passthrough_via_OVMF#Ensuring_that_the_groups_are_valid) for options.

### Step 3: Blacklist GPU drivers on the host

On RHEL 9.6 (kernel 5.14), the `xe` module does not support Battlemage (`e223`), but it's still good practice to blacklist it so no host driver interferes with passthrough. On RHEL 10 (kernel 6.12+) this is essential since `xe` would otherwise claim the device.

```bash
sudo tee /etc/modprobe.d/blacklist-xe.conf <<EOF
blacklist xe
blacklist i915
EOF
```

> **Note:** The host kernel args `i915.force_probe=!7d67` and `xe.force_probe=7d67` (if present) target the **iGPU** (`7d67`), not the dGPU (`e223`). They do not interfere with dGPU passthrough.

We rely on libvirt's `managed='yes'` mode (the default) rather than boot-time `vfio-pci` binding. Libvirt automatically binds `vfio-pci` when the VM starts and unbinds it when the VM stops.

### Step 4: Add DHCP reservation for the worker VM

Add a second host entry to the libvirt network XML for the worker VM. Edit the running network:

```bash
sudo virsh net-edit ocp
```

Add a new `<host>` line inside the `<dhcp>` block:

```xml
<host mac="REDACTED_MAC" name="worker-0.sno-cluster.example.com" ip="REDACTED_IP"/>
```

Also add a DNS entry in the `<dns>` block:

```xml
<host ip="REDACTED_IP">
  <hostname>worker-0.sno-cluster.example.com</hostname>
</host>
```

Restart the network for changes to take effect:

```bash
sudo virsh net-destroy ocp
sudo virsh net-start ocp
```

### Step 5: Switch the IOMMU group to DMA mode

On `RHEL_HOST`, IOMMU group 18 defaults to `identity` mode (1:1 address mapping). VFIO needs `DMA` mode to create the IOVA mappings required for GPU passthrough:

```bash
echo DMA > /sys/kernel/iommu_groups/18/type
cat /sys/kernel/iommu_groups/18/type
```

> **Note:** This is volatile -- it resets to `identity` on host reboot. Add to a startup script if needed.

### Step 6: Configure the VM and attach the GPU

The worker must be installed and joined to the cluster **without** the GPU first (see "Adding worker nodes" section above). Once the worker is joined and `Ready`, shut it down and configure GPU passthrough.

Two VM XML changes are required:

1. **`maxphysaddr limit=42`** -- keeps guest physical addresses within the host IOMMU's 42-bit MGAW (Maximum Guest Address Width). Without this, QEMU places GPU BARs at ~3.5 TB, which exceeds the IOMMU's address space and causes `VFIO_MAP_DMA failed: Invalid argument`.

2. **Remove the virtio-vga video device** -- having two VGA devices (virtio-vga + the passthrough GPU) causes a `fbcon` / `drm_modeset_lock` deadlock in the guest kernel, leading to `hung_task` panics. The passthrough GPU provides display output if needed.

```bash
virsh shutdown ocp-worker-0

virt-xml ocp-worker-0 --add-device --hostdev pci_REDACTED_BDF
virt-xml ocp-worker-0 --remove-device --video all
virt-xml ocp-worker-0 --edit --xml xpath.create=./cpu/maxphysaddr \
  --xml xpath.set=./cpu/maxphysaddr/@mode=passthrough \
  --xml xpath.set=./cpu/maxphysaddr/@limit=42

virsh start ocp-worker-0
```

Verify no VFIO errors:

```bash
tail -5 /var/log/libvirt/qemu/ocp-worker-0.log | grep VFIO
```

If you see `VFIO_MAP_DMA failed`:
- `Invalid argument` at IOVAs above ~3 TB -- check `maxphysaddr limit=42` is set
- `Invalid argument` at any IOVA -- check IOMMU group type is `DMA`, not `identity`

If the VM boots but hangs (`hung_task` timeouts in guest dmesg):
- Two VGA devices present -- remove virtio-vga with `virt-xml --remove-device --video all`

### Verifying the GPU inside the worker VM

Once the worker VM is running and joined to the cluster, SSH in and confirm the GPU is visible:

```bash
ssh core@REDACTED_IP

lspci | grep -i vga
# REDACTED_BDF VGA compatible controller: Intel Corporation Battlemage G31 [Intel Graphics]

lspci -ks REDACTED_BDF
# Kernel driver in use: xe

ls /dev/dri/
# card0  card1  renderD128
```

The PCI address inside the VM will differ from the host (e.g., `REDACTED_BDF` instead of `REDACTED_BDF`) -- this is normal. The `xe` driver loads automatically on RHCOS 9.8 (kernel 5.14.0-687+) which has in-tree Battlemage support (device ID `e223` is in `INTEL_BMG_IDS` in `include/drm/intel/pciids.h`). The `renderD128` device node is what userspace GPU compute frameworks (Level Zero, oneAPI) use to access the GPU.

## Official documentation

- [Agent-Based Installer overview (OCP 4.21)](https://docs.redhat.com/en/documentation/openshift_container_platform/4.21/html/installing_an_on-premise_cluster_with_the_agent-based_installer/preparing-to-install-with-agent-based-installer)
- [Installing a cluster with the ABI (OCP 4.21)](https://docs.redhat.com/en/documentation/openshift_container_platform/4.21/html/installing_an_on-premise_cluster_with_the_agent-based_installer/installing-with-agent-based-installer)
- [Sample install-config.yaml for bare metal (OCP 4.21)](https://docs.redhat.com/en/documentation/openshift_container_platform/4.21/html/installing_an_on-premise_cluster_with_the_agent-based_installer/preparing-to-install-with-agent-based-installer#sample-install-config-yaml-for-bare-metal)
- [Adding worker nodes to SNO clusters](https://docs.redhat.com/en/documentation/openshift_container_platform/4.21/html/nodes/worker-nodes-for-single-node-openshift-clusters)
- OCP 4.22 candidate client mirror (link removed in public copy)
- OCP 4.22 release status (link removed in public copy)
- [Red Hat blog: Meet the Agent-Based Installer](https://www.redhat.com/en/blog/meet-the-new-agent-based-openshift-installer-1)
- [RHEL 9: Managing virtual devices / PCI passthrough](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/9/html/configuring_and_managing_virtualization/managing-virtual-devices_configuring-and-managing-virtualization)
- [RHEL 10: Attaching host devices to VMs (PCI passthrough)](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/10/html/configuring_and_managing_linux_virtual_machines/attaching-host-devices-to-virtual-machines)
- [ArchWiki: PCI passthrough via OVMF](https://wiki.archlinux.org/title/PCI_passthrough_via_OVMF)
- [Intel dGPU virtualization docs](https://dgpu-docs.intel.com/driver/client/virtualized.html)
- [QEMU 10.1 IGD fix (patch discussion)](https://lists.nongnu.org/archive/html/qemu-devel/2025-02/msg04902.html)
