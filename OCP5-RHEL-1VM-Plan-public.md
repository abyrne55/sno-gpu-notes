---
last-updated: 2026-10-21
status: complete
type: plan
---

> **Public-safe copy:** hostnames, IP addresses, MAC addresses, PCI addresses, UUIDs, user identifiers, local paths, and private links have been anonymized. Commands and network examples are reference material and are not directly runnable.


> [!info] Plan completed
> Milestones 1, 2, and 3 were accepted on 2026-10-20 21:06 UTC, 2026-10-21 00:44 UTC, and 2026-10-21 17:17 UTC.
>
> The test installed classic OCP `5.0.0-rc.3` on one GPU_HOST KVM/libvirt SNO, passed one Intel Arc Pro B70 PF through to the in-tree `xe` driver, and installed GBO `0.4.0` with DRA `0.11.0`, XPU Manager `2.2.0`, and NFD `0.19.0`. A DRA claim exposed the guest GPU to the pinned SYCL workload: 50/50 iterations passed; the unclaimed control saw no GPU and exited 2.
>
> The [[Notes - RHEL+1VM Approach to OCPv5 Classic]] records accepted state, evidence paths, cleanup, and known limitations. This result covers one PF and one classic SNO on rc.3; it is not product certification or validation of other GPU/release combinations, CNV/KubeVirt GPU use, or claim persistence across reboot.

Use this procedure to build or repeat the classic OCP `5.0.0-rc.3` single-node test on GPU_HOST, pass through one B70 PF, and validate GPU compute through DRA.

This remains the canonical rebuild and repeat procedure. The completed run does not authorize reinstallation or changes to the retained lab. Before a future repeat, refresh live host, VM, GPU, cluster, network, and guard state, then follow the milestones below in dependency order. Use the [[Notes - RHEL+1VM Approach to OCPv5 Classic]] for the accepted run and retained-state record.

## Decisions and scope

- Approved 2026-10-16: a **fresh VM-specific ISO generated on GPU_HOST**, dedicated `openshift-install-v5rc3`, cluster `REDACTED_HOSTNAME`, VM `SNO_VM`, and separate NAT network `REDACTED_IP`, subject to collision checks.
- Use Agent-based installation, `platform: none`, one control-plane node, zero separate workers and OVN-Kubernetes. API servers and etcd run inside the SNO, as on e3; no hosted control plane.
- Give the cluster ordinary NAT egress and workstation API/SSH tunnels through GPU_HOST.
- Publish the accepted image from EXISTING_VM_B to Quay **before stopping EXISTING_VM_B**; then stop EXISTING_VM_A, followed by EXISTING_VM_B. Both stay off for the rest of this phase, on success or failure. Preserve their definitions, disks/NVRAM, networks, registry data, guard and records.
- Assign host PF **`REDACTED_BDF`**, the first validated card, after ownership/group checks. Leave `REDACTED_BDF` unassigned on host VFIO. The guest uses in-tree `xe` and `dra.manageBinding=false`.
- Reserve CNV/KubeVirt GPU testing for later bare-metal OCP 5 work. HPP/LVMS, SR-IOV VFs, inference, out-of-tree drivers, firmware changes and GPU recovery are also outside this phase. GBO may create an unused VFIO DeviceClass.

## Evidence and compatibility limits

### Read-only planning inventory, 2026-10-16

| Area | Observed state |
|---|---|
| GPU_HOST | `REDACTED_HOSTNAME`; RHEL 10.2; Xeon w9-3475X; 72 logical CPUs; 124 GiB RAM; XFS `/home`, about 3.4 TiB free |
| Existing VMs | EXISTING_VM_B and EXISTING_VM_A running, persistent, autostart disabled. EXISTING_VM_B: 4 vCPU / 12 GiB, UUID `REDACTED_UUID`. EXISTING_VM_A: 24 vCPU / 64 GiB, UUID `REDACTED_UUID` |
| EXISTING_VM_A health | OCP 4.22.15, Kubernetes 1.35.6, RHCOS 9.8; one Ready node; all 34 ClusterOperators healthy; master MCP Updated, not Updating/Degraded |
| GPUs | Both `8086:e223` PFs use host `vfio-pci`; `REDACTED_BDF` is alone in group 24, `REDACTED_BDF` alone in group 16. Both attached to EXISTING_VM_A |
| Selected PF | `REDACTED_BDF`: 32 GiB BAR 2, reset methods `flr bus`, zero enabled VFs |
| Guard | `existing-lab-guard.service` active/enabled; approved-source, rules and C4-marker checks passed. Hook permits unrelated domains provided they do not reference reserved private networks/taps |
| Existing networks | `default` NAT `REDACTED_IP`; `isolated-example-net` isolated `REDACTED_IP`; `example-uplink` NAT `REDACTED_IP` |
| EXISTING_VM_B | RHEL 9.8, `REDACTED_IP`; Podman 5.8.2/Python 3 available, Skopeo absent; workload manifest verified by HTTPS |
| Quay destination | Anonymous tag enumeration succeeded. Existing tag: `neo-26.35.39758.10-dpcpp-2026.1.1` |

Refresh the host, VM and GPU inventory before execution.

### Tools ready on GPU_HOST and exact release

Verified on GPU_HOST after preparation, 2026-10-16:

| Tool | Path and verified version |
|---|---|
| Installer | `/path/to/lab-data`, `5.0.0-rc.3` |
| OCP client | `/path/to/lab-data`, `5.0.0-rc.3`; SHA-256 `a1db1bb8ae76f601a3bb6d27646a3d6836a0cda9c23381703d8e2b0daaab51e6` |
| Butane | `/usr/bin/butane`, `0.27.0`; installed RPM `butane-0.27.0-1.el10_2.x86_64` from `beaker-AppStream`, with no additional packages |

Copied binaries are executable and checksum-verified. GPU_HOST also has OCP 4.22.15 `oc`, Podman, Skopeo, libvirt/virt-install, XML validation, qemu-img, coreos-installer, xorriso and Python. Run Helm 4 from the workstation.

The dedicated installer reports:

```text
5.0.0-rc.3
commit b6f119779c7ce529a94c7b8956a057343675f212
quay.io/openshift-release-dev/ocp-release@sha256:29da60c98a5347be6e412b4176b5c27e1ca6336158178d77afc9dc5dbd1ed922
architecture amd64
```

Binary SHA-256: `f94206f0b9c50db346c0eaa798aae127b19fb125ea4550352304cc3c2cae4228`.

The [rc.3 release page](https://amd64.ocp.releases.ci.openshift.org/releasetag/5.0.0-rc.3) agrees and lists Kubernetes `1.36.3`, CoreOS base `10.2.20260910-2`. Installer boot media is `10.2.20260423-0`, labeled `rhcos-4.22`; record the installed OS/kernel separately.

Public OCP 5 documentation returned HTTP 404. Installation assumptions were checked against the tools/artifacts and [exact installer source](https://github.com/openshift/installer/tree/b6f119779c7ce529a94c7b8956a057343675f212/docs/user/agent), also available at `/path/to/lab-data`.

### Completed test result and remaining limits

- e3 had **no GPU**; its rc.3 smoke test validated operator installation only.
- On the tested GPU_HOST guest, rc.3 RHCOS initialized the B70 with in-tree `xe`, loaded firmware, and exposed a render node. The DRA `resource.k8s.io/v1` claim allocated that device and the workload passed. This is observed lab compatibility, not certification of the Intel/RHCOS combination. [Intel's Xe table](https://dgpu-docs.intel.com/overview/supported-hardware/xe-driver-gpus.html) and [RHEL 10 guidance](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/10/html/configuring_and_managing_linux_virtual_machines/managing-gpu-devices-in-virtual-machines) remain relevant support references.
- DRA `0.11.0` was tested on Kubernetes `v1.36.3`; DRA `0.12.0` was not tested. GBO `0.4.0` defaults to `0.12.0`, so the repeat procedure pins the accepted `0.11.0` image digest. Its logs can misreport the version.
- Cert-manager `1.20.0` supported the GBO webhooks. The later `1.20.1` Manual InstallPlan was intentionally left unapproved. KMM was unnecessary for this in-tree-driver policy; keep `kernelModule` unset.
- The run covered one PF, one SNO node, and one sequential pod workload. It did not test claim persistence across reboot, CNV/KubeVirt GPU access, multiple GPUs, SR-IOV, inference, or out-of-tree drivers; those remain separate experiments.

## Execution workspace and fixed inputs

| Item | Planned value |
|---|---|
| GPU_HOST workspace | `/path/to/lab-data` (`RUN_ROOT`) |
| Build | `$RUN_ROOT/build/example-sno-<UTC-run-id>/` (`BUILD_DIR`), new and private |
| Durable execution record | [[Notes - RHEL+1VM Approach to OCPv5 Classic]]; accepted state, essential identities, evidence paths, milestone verdicts and repeat limits |
| Secrets/scratch | `$RUN_ROOT/secrets/` and `$RUN_ROOT/.tmp/`; mode 0700 directories / 0600 secret files |
| New VM storage | `/var/lib/libvirt/images/ocp5-classic/`; new standalone `SNO_VM.qcow2`, 300 GiB virtual capacity, plus new ISO and independent NVRAM |
| VM | `SNO_VM`; 24 vCPU, 64 GiB RAM; host-passthrough CPU; Q35/UEFI; manual start, no autostart |
| New network | `example-nat`, bridge `virbr-ocp5`, `REDACTED_IP`, NAT |
| Node | `REDACTED_HOSTNAME`, `REDACTED_IP`, MAC `REDACTED_MAC`; single virtio NIC; `/dev/vda` |
| API and apps DNS | `REDACTED_HOSTNAME`, `REDACTED_HOSTNAME`, `REDACTED_HOSTNAME` → `REDACTED_IP` inside the VM network |
| Access | Workstation ports `17443` → guest `6443`, `13222` → guest `22`; context `example-sno` |
| Accepted workload tag | `quay.io/example-namespace/validation-image:ocp5-rc3-GPU_HOST-matmul-20261016-1cec2ced159c`; immutable digest `sha256:1cec2ced159c8a3081fb4c9e70aacc7e7f5b9828662f226e57eba6d57f2b095f` |

Resolve `<UTC-run-id>` once. Check names/UUIDs, bridge/tap/MAC, IP/routes, disk/NVRAM/build paths and workstation ports for collisions, including corporate/VPN and pod/service networks. Update conflicting new inputs before generation; never repurpose existing resources.

Run GPU_HOST SSH **outside the sandbox** using alias `GPU_HOST`. Sandbox/VPN failures do not establish cluster health. From GPU_HOST, access EXISTING_VM_B with its existing pinned key:

```bash
ssh -o BatchMode=yes -o StrictHostKeyChecking=yes \
  -o GlobalKnownHostsFile=/dev/null \
  -o UserKnownHostsFile=/path/to/lab-data \
  -o IdentitiesOnly=yes -i /path/to/lab-data \
  root@REDACTED_IP
```

Use the existing new workspace for task-owned files. Keep temporary files in its ignored `./.tmp/` and remove them when finished. Preserve unrelated edits; no install-config repository checkout is required for the normal path.

## Milestone 1 — Publish the workload and prepare a recoverable transition

Execution corrections, 2026-10-20: use pod CIDR `REDACTED_IP` after collision checks found the original `REDACTED_IP` overlaps the workstation VPN route. the lab owner approved replacing the unavailable Skopeo pin with the independently inspected official linux/amd64 Skopeo 1.22.3 manifest used below, then resuming Milestone 1. Retain the failed pull evidence; see [[Notes - RHEL+1VM Approach to OCPv5 Classic]] and the protected GPU_HOST build evidence.

### 1. Refresh and retain the current state

C10 was accepted **2026-10-15 14:55 UTC** after two sequential GPU runs, including a guarded VM restart. Old EXISTING_VM_A mappings were host `REDACTED_BDF` → guest `REDACTED_BDF` / `renderD128` and host `REDACTED_BDF` → guest `REDACTED_BDF` / `renderD129`. Preserve its XPU `All/adminAccess` claim; validation compute claims were removed. Its cert-manager/KMM versions were 1.20.1/2.7.1; the GPU/runtime baseline is retained below.

EXISTING_VM_A's allowed EXISTING_VM_B services are DNS/NTP, registry `:8443` and private registry `REDACTED_IP:5000`, with no internet path. TCP 8090 is closed; no host listener on 5000/8090 is expected. EXISTING_VM_B requires `/srv/mirrors`, active `mirror-distribution.service` / `mirror-distribution-guard.service`, and private registry TLS `/v2/` HTTP 200.

Before mutation, capture inactive domain XML, disk/NVRAM paths, network XML, autostart state and guard identities. With `sudo -n /usr/REDACTED_HOSTNAME/sbin/existing-lab-start-check`, require `--verify-approved-sources`, `--check-rules`, `--check-c4-pass`, `--check-network isolated-example-net`, `--check-network example-uplink`, `--check-domain EXISTING_VM_A` and `--check-domain EXISTING_VM_B` to pass. No pending transition/probe exception should exist. Preserve approvals/markers; investigate mismatches through the targeted fallback.

Use the dedicated existing kubeconfig/context to capture EXISTING_VM_A health:

```bash
oc --kubeconfig /path/to/lab-data \
  --context existing-lab-admin get clusterversion,nodes,co,mcp
```

Capture PF groups/peers, `lspci -nnk/-vv`, BARs, reset methods, VFs, driver overrides and QEMU ownership. Confirm ASPEED/BMC recovery access. VFIO binding alone does not mean a device is free.

### 2. Verify or publish the accepted workload image

The accepted image is already published at the immutable digest above. Before a repeat, verify that the Quay pullspec still resolves to this digest and reuse it if present; do not republish or start EXISTING_VM_B just to repeat the compute test. If the image is absent or the digest differs, stop and arrange a separately authorized publication step. For a fresh initial run with the image still available on EXISTING_VM_B, use the procedure below, preserve its manifest digest, and never overwrite an existing tag.

1. Immediately before a new copy, inspect the destination tag. Reuse it only if its digest matches the accepted image; if absent, choose a fresh collision-checked UTC suffix. If it exists with a different digest, stop and choose a different tag. Use one publisher.
2. Authenticate **on EXISTING_VM_B** using GPU_HOST `/path/to/lab-data`, an observed `podman login` command with username/password flags. Parse that form strictly on GPU_HOST with `shlex`/argument parsing, no shell evaluation, and stream credentials in memory through pinned SSH. A private EXISTING_VM_B helper reads them from stdin and runs `podman login --authfile /root/ocp5-publish/.tmp/quay-auth.json --username <in-memory-user> --password-stdin quay.io`. Use 0700/0600 task paths; never print/trace credentials or put the password in process arguments. If the helper form differs, stop and adapt the parser privately. Keep its contents unchanged until cleanup.
3. EXISTING_VM_B lacks Skopeo. Use the pinned linux/amd64 tool container below with read-only auth/CA mounts. Keep TLS verification and SELinux enforcing; do not relabel system trust files. The verified execution correction uses an explicit source certificate directory: copy only the accepted public CA from GPU_HOST `/path/to/lab-data` (SHA-256 `0262f912944e61c398ab231367c43d422d697a2f7aca6768cde526a2e22c92dd`) to EXISTING_VM_B's task-owned `/root/ocp5-publish/.tmp/ca-diagnosis/certs/ca.crt`; verify the copy, then mount it read-only as shown. Read-only inspection with this CA passed after the original bundle-only mount failed source validation.
4. On EXISTING_VM_B, with `PUBLISH_TAG` set to the collision-checked tag suffix, the copy is:

```bash
podman run --rm --network host \
  -v /root/ocp5-publish/.tmp:/auth:ro,Z \
  -v /root/ocp5-publish/.tmp/ca-diagnosis/certs:/certs:ro,Z \
  -v /etc/pki/ca-trust/extracted/pem/tls-ca-bundle.pem:/trust/ca-bundle.pem:ro \
  -e SSL_CERT_FILE=/trust/ca-bundle.pem \
  quay.io/skopeo/stable@sha256:26e3dc7cb7446837b000e9aeff5918df21662485f15861398cfd89cc7f54ee2e \
  copy --src-cert-dir /certs --src-tls-verify=true --dest-tls-verify=true \
  --preserve-digests --dest-authfile /auth/quay-auth.json \
  docker://REDACTED_HOSTNAME:5000/c10/ubuntu-neo-matmul@sha256:1cec2ced159c8a3081fb4c9e70aacc7e7f5b9828662f226e57eba6d57f2b095f \
  "docker://quay.io/example-namespace/validation-image:${PUBLISH_TAG}"
```

5. Independently inspect the destination from GPU_HOST/workstation. Require the source manifest digest in the command, config digest `sha256:45a83030ade06dc14a3c6c2373c9b12a8717e162ef9099fab6a42169dbc0940b`, linux/amd64 and matching layer descriptors. Stop on mismatch. Anonymous pulls worked during planning; if that changes, resolve scoped read-only access before stopping EXISTING_VM_B.
6. Record tag, immutable pullspec, digests and completion time. Remove task auth/parser scratch and, if introduced and unused, the tool image; never prune shared storage. Keep push credentials out of the SNO.
7. Delete GPU_HOST's `/path/to/lab-data` once no longer needed.

### 3. Prepare the new network and ISO

Define `example-nat` using this libvirt NAT/DNS (link removed in public copy) input:

```xml
<network xmlns:dnsmasq='REDACTED_URL>
  <name>example-nat</name>
  <forward mode='nat'/>
  <bridge name='virbr-ocp5' stp='on' delay='0'/>
  <domain name='REDACTED_HOSTNAME' localOnly='yes'/>
  <dns>
    <host ip='REDACTED_IP'>
      <hostname>REDACTED_HOSTNAME</hostname>
      <hostname>REDACTED_HOSTNAME</hostname>
      <hostname>REDACTED_HOSTNAME</hostname>
    </host>
  </dns>
  <ip address='REDACTED_IP' netmask='REDACTED_IP'>
    <dhcp>
      <host mac='REDACTED_MAC' name='REDACTED_HOSTNAME' ip='REDACTED_IP'/>
    </dhcp>
  </ip>
  <dnsmasq:options>
    <dnsmasq:option value='address=/REDACTED_HOSTNAME/REDACTED_IP'/>
  </dnsmasq:options>
</network>
```

Start it manually. Check dnsmasq configuration, A/PTR/apps wildcard answers via `REDACTED_IP`, upstream resolution and prompt negative/AAAA responses. Keep host resolvers unchanged. After libvirt adds NAT rules, recheck the old guard; never flush nftables, disable the guard, change global forwarding or use old private attachment points.

Reuse the staged installer and client; verify their versions/checksums above before execution. Use explicit paths and clear inherited release/RHCOS overrides. Never substitute unversioned `openshift-install`.

Create private `install-config.yaml` below. Populate `pullSecret` from GPU_HOST `/root/mirror-registry/pull-secret.txt` after privately verifying access to the exact release, and `sshKey` from the workstation access key. Preserve that credential file; exclude secrets/populated configs/raw installer logs from output. Do not use EXISTING_VM_A's mirror-only or Quay push credentials.

```yaml
apiVersion: v1
baseDomain: REDACTED_HOSTNAME
metadata:
  name: example-sno
controlPlane:
  architecture: amd64
  name: master
  replicas: 1
compute:
- name: worker
  replicas: 0
networking:
  networkType: OVNKubernetes
  machineNetwork:
  - cidr: REDACTED_IP
  clusterNetwork:
  - cidr: REDACTED_IP
    hostPrefix: 23
  serviceNetwork:
  - REDACTED_IP
platform:
  none: {}
# Add pullSecret and sshKey privately before generation.
```

Use the DHCP reservation without NMState; put the NIC on the first PCIe port and confirm its name at boot. AgentConfig:

```yaml
apiVersion: v1beta1
kind: AgentConfig
metadata:
  name: example-sno
rendezvousIP: REDACTED_IP
minimalISO: false
additionalNTPSources:
- REDACTED_HOSTNAME
- REDACTED_HOSTNAME
hosts:
- hostname: REDACTED_HOSTNAME
  role: master
  interfaces:
  - name: enp1s0
    macAddress: "REDACTED_MAC"
  rootDeviceHints:
    deviceName: /dev/vda
```

Use the normal CoreOS disk layout. Save `$RUN_ROOT/config/92-serial-console.bu` below; Butane 0.27.0 already compiled it to Ignition 3.5.0 during preparation:

```yaml
variant: openshift
version: 4.21.0
metadata:
  labels:
    machineconfiguration.openshift.io/role: master
  name: 92-serial-console
openshift:
  kernel_arguments:
  - console=ttyS0,115200n8
```

Retain mode-protected input copies outside the consumed build assets. Generate and prepare serial boot:

```bash
mkdir -p "$BUILD_DIR/openshift"
/usr/bin/butane --strict "$RUN_ROOT/config/92-serial-console.bu" \
  -o "$BUILD_DIR/openshift/92-serial-console.yaml"
"$RUN_ROOT/tools/openshift-install-v5rc3" agent create image --dir "$BUILD_DIR"
coreos-installer iso kargs modify \
  -a console=tty0 -a console=ttyS0,115200n8 \
  "$BUILD_DIR/agent.x86_64.iso"
```

Hash the finished ISO and verify its staged copy. Inspect embedded configuration without exposing secrets: require the new identity/MAC/IP, release, `/dev/vda`, classic topology, serial manifest and `rootfs.img`, with no e3/HPP/disconnected inputs. Preserve matching auth assets and label only new image paths under the existing SELinux storage policy.

**Milestone 1 pass:** collision-free new inputs; exact rc.3 installer/ISO identity; Quay image verified independently; old lab healthy and recoverable; old guard unchanged. Finish publication before proceeding to either guest shutdown.

## Milestone 2 — Stop the old lab and install the GPU SNO

### 4. Shut down EXISTING_VM_A, then EXISTING_VM_B; preserve everything

On GPU_HOST:

```bash
sudo -n virsh shutdown EXISTING_VM_A
# Poll domstate until it is exactly "shut off"; allow a graceful SNO shutdown.
sudo -n virsh domstate EXISTING_VM_A
# Only after EXISTING_VM_A is off:
sudo -n virsh shutdown EXISTING_VM_B
sudo -n virsh domstate EXISTING_VM_B
```

Allow ten minutes per guest; stop if graceful shutdown fails. No forced shutdown, managed-save, undefine or disk replacement. Capture stopped XML/NVRAM identities and disk metadata; preserve EXISTING_VM_A's two-PF definition and:

- `/var/lib/libvirt/images/disconnected/{EXISTING_VM_A,EXISTING_VM_B,EXISTING_VM_B-content}.qcow2`, old ISOs and UEFI variables.
- `/path/to/lab-data`, `/path/to/lab-data`, EXISTING_VM_B `/srv/mirrors/`, all registry data/TLS material and build/execution evidence.
- `isolated-example-net`, `example-uplink`, `default`, guard policy/checker/hooks/service, approved XML and C4 records.
- `/path/to/lab-data`, an older pre-GPU checkpoint; retain it unchanged.

With both VMs off, recheck `REDACTED_BDF`: singleton group, no VFs, expected BAR/reset state, host VFIO and **no QEMU/VFIO/iommufd owner**. Check active XML and process/file ownership, not only legacy `/dev/vfio/<group>`. Stop on a changed/busy device; never detach/reset a live owner.

### 5. Define a separate VM and boot the approved ISO

Create a new standalone 300 GiB qcow2 disk, with no backing file. Render a new domain XML with virt-install in print/dry-run mode, then validate it before `virsh define`. Use the layout table and these settings:

- New UUID/NVRAM/disk/ISO paths; one approved-MAC virtio NIC on `example-nat`, no old private network/tap references.
- One managed VFIO hostdev, `REDACTED_BDF`.
- Q35 `pc-q35-rhel10.2.0`, host-passthrough CPU, serial console/PCIe root ports; `pcie-root` controller `pcihole64=134217728 KiB` (128 GiB). No address cap, BAR shrinking or virtual IOMMU.
- RHEL 10 guest osinfo metadata if available; install only from the fresh Agent ISO.
- Headless/no video, disk-first/CD-ROM fallback, reboot restart, no autostart. UEFI loader `/usr/share/edk2/ovmf/OVMF_CODE.fd`, template `/usr/share/edk2/ovmf/OVMF_VARS.fd`, raw format; disable Secure Boot/enrolled keys. Verify firmware paths and use independent NVRAM.

Required selected-PF fragment:

```xml
<hostdev mode='subsystem' type='pci' managed='yes'>
  <driver name='vfio'/>
  <source>
    <address domain='0x0000' bus='0x18' slot='0x00' function='0x0'/>
  </source>
</hostdev>
```

Validate XML (`virt-xml-validate`), paths/labels and NVRAM independence. The existing hook allows this unrelated domain/network without guard changes. Start `SNO_VM` using `virsh start`; retain serial/QEMU diagnostics.

At live-ISO boot, verify NIC/MAC/DHCP/IP/default route, API/apps DNS, NTP, registry HTTPS egress and selected PCI device. Preserve the disconnected guard while diagnosing networking.

Monitor with the dedicated installer and matching build:

```bash
"$RUN_ROOT/tools/openshift-install-v5rc3" agent wait-for bootstrap-complete --dir "$BUILD_DIR"
"$RUN_ROOT/tools/openshift-install-v5rc3" agent wait-for install-complete --dir "$BUILD_DIR"
```

Keep durable private waiter logs and actual exit status. If installation leaves the VM off after writing disk, verify completion/definition before starting it once from disk. Verify installed-rootdisk boot, then detach its ISO to prevent reinstallation.

### 6. Establish access and prove the platform/GPU baseline

Copy the new kubeconfig to workstation `.tmp/example-sno.kubeconfig`, mode 0600, context `example-sno`. Retain its CA; set server `REDACTED_URL` and `tls-server-name: REDACTED_HOSTNAME`. Preserve existing contexts and TLS verification.

Run the tunnel outside the sandbox, bound to loopback:

```bash
ssh -N -o ExitOnForwardFailure=yes -o ServerAliveInterval=30 \
  -L REDACTED_IP:17443:REDACTED_IP:6443 \
  -L REDACTED_IP:13222:REDACTED_IP:22 GPU_HOST
```

Obtain the guest host key through trusted serial/management access. For `ssh -p 13222 core@REDACTED_IP`, use the matching workstation key, `IdentitiesOnly=yes`, `StrictHostKeyChecking=yes`, `HostKeyAlias=REDACTED_HOSTNAME` and relative `UserKnownHostsFile=.tmp/example-sno-known_hosts`. Preserve other known-host entries. Browser/OAuth access needs additional DNS/forwarding and is outside this access setup.

Use the dedicated rc.3 client for every new-cluster command. On the workstation:

```bash
OC5=/path/to/lab-data
"$OC5" --kubeconfig .tmp/example-sno.kubeconfig --context example-sno get clusterversion,nodes,co,mcp
"$OC5" --kubeconfig .tmp/example-sno.kubeconfig --context example-sno get --raw=/readyz
```

For GPU_HOST-local checks, use `$RUN_ROOT/tools/oc-v5rc3` and the new build's kubeconfig/context. Keep the existing OCP 4 client/context for EXISTING_VM_A.

Require completed `5.0.0-rc.3` at the recorded digest, one Ready Kubernetes `v1.36.3` node, all required ClusterOperators Available/not Progressing/not Degraded, and master MCP Updated/not Updating/not Degraded. Report the actual operator count. Verify node-local etcd/control-plane workloads and record RHCOS/kernel/CRI-O/firmware versions. No dynamic storage is needed; the internal registry may remain Removed under platform-none defaults.

Capture guest `lspci -nnk`, `uname -r`, `modinfo xe`, PCI driver and DRM sysfs paths, `/dev/dri/by-path/` and kernel/firmware messages. Require one B70 PF on **guest `xe`** with a render node. Record host `REDACTED_BDF` → observed guest BDF → render node; rediscover rather than reuse EXISTING_VM_A's mapping.

Stop on missing E223 support/firmware, force-probe requirements, hangs or VFIO/MMIO errors. Preserve evidence; kernel replacement, `force_probe`, out-of-tree drivers and firmware changes require a revised plan.

With EXISTING_VM_B off, run an ordinary pod using the **Quay digest**, `imagePullPolicy: Always`, `command: ["/bin/true"]`. Require pull/start/completion; capture events/imageID, accounting for runtime manifest/config ID differences. The fresh SNO has no inherited image cache. Verify ordinary egress with no EXISTING_VM_B mirror/CA/host-entry dependency.

**Milestone 2 pass:** classic rc.3 SNO healthy; workstation API/SSH work; one host VFIO PF maps to guest `xe`/render node; Quay image pulls while EXISTING_VM_B and EXISTING_VM_A remain off; old guard and retained definitions remain intact.

## Milestone 3 — Install GBO and validate DRA compute

### 7. Install only the required operator stack

Use workstation Helm 4+, separate releases `gpu-operator` and `gpu-policy`, namespace `intel-gpu-base-operator`. The baseline is GBO 0.4.0, DRA 0.11.0, XPU Manager 2.2.0 and bundled NFD 0.19.0. Use the published archives below without dependency updates.

Query PackageManifest `openshift-cert-manager-operator` in `openshift-marketplace`. The accepted candidate was **1.20.0**, channel `stable-v1`, source `redhat-operators`. Verify live CSV/bundle constraints and AllNamespaces support before a repeat. Install in `cert-manager-operator`, OperatorGroup `targetNamespaces: []`, Manual Subscription with the observed 1.20.0 `startingCSV`; approve only its intended InstallPlan. Require CSV Succeeded, operator/controller/webhook/cainjector ready, `cert-manager.io/v1` and healthy TLS. Stop if unavailable/incompatible. Use GBO's self-signed Issuer without ACME/API certificate changes.

Fetch on the workstation into a task-only directory:

```bash
CHART_DIR="$PWD/.tmp/example-sno"
mkdir -p "$CHART_DIR"
export HELM_CACHE_HOME="$CHART_DIR/cache" HELM_CONFIG_HOME="$CHART_DIR/config" HELM_DATA_HOME="$CHART_DIR/data"
helm pull oci://ghcr.io/intel/intel-gpu-base-operator-chart --version 0.4.0 --destination "$CHART_DIR"
helm pull oci://ghcr.io/intel/intel-gpu-base-operator-policy-chart --version 0.4.0 --destination "$CHART_DIR"
```

Require both the OCI manifest digest reported by Helm and the SHA-256 of each downloaded archive to match:

| Archive in `$CHART_DIR` | OCI manifest digest | Archive SHA-256 |
|---|---|---|
| `intel-gpu-base-operator-chart-0.4.0.tgz` | `sha256:cd0fbe9e152764f7aae592497d18d326c96d12f208d6358642b1f5739e0a94ea` | `1368e317afefa3b109899a9b4f190eedb62bca05944bcff0393ed3f0e29174c3` |
| `intel-gpu-base-operator-policy-chart-0.4.0.tgz` | `sha256:f03b8857422cb3d9626e6792dc66fcbbfeeff10bed186a6bb5f314d343184930` | `42f16882b8f02abc3347d4846a79743279c868829380ae4fc6028695069f62ab` |

Confirm packaged NFD is 0.19.0. Save these complete values as `$CHART_DIR/operator-values.yaml` and `$CHART_DIR/policy-values.yaml`; their image pins use public registries and retain the accepted runtime versions:

```yaml
# operator-values.yaml
createNamespace: false
openshift:
  enabled: true
operator:
  image:
    repository: ghcr.io/intel/intel-gpu-base-operator@sha256:c7486ca76ee1ca10055146a38cc94dff46d0943f52a9a8335a08a79afce97c70
nfd:
  install: true
  image:
    repository: registry.k8s.io/nfd/node-feature-discovery
    tag: v0.19.0@sha256:2fa1c99ad09bdf2c8ad97706a4ad2fd548c84d5ecd70ba32a6152c667b96c4d2
kueue:
  install: false
```

```yaml
# policy-values.yaml
resourceRegistration: dra
useNFDLabeling: true
resourceMonitoring: true
enableKueue: false
dra:
  manageBinding: false
  image: ghcr.io/intel/intel-resource-drivers-for-kubernetes/intel-gpu-resource-driver@sha256:49f38fcbee4f98d748b537b1a728a12f39e56fd208c86dc3fa32ab2162c21197
xpu:
  image: ghcr.io/intel/xpumanager/xpumd@sha256:fdefec8e5d93f76a1c7b4dda56001b7f3c2c15eae29bfbbc8971b900e6e74ae8
```

Save/apply this as `$CHART_DIR/gbo-access.yaml` before installing the operator; it grants privileged SCC only to the NFD worker:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: intel-gpu-base-operator
  labels:
    app.kubernetes.io/name: intel-gpu-base-operator
    resource.kubernetes.io/admin-access: "true"
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: gpu-operator-nfd-worker-privileged
  namespace: intel-gpu-base-operator
roleRef: {apiGroup: rbac.authorization.k8s.io, kind: ClusterRole, name: "system:openshift:scc:privileged"}
subjects:
- {kind: ServiceAccount, name: gpu-operator-nfd-worker, namespace: intel-gpu-base-operator}
```

Render both archives with these values and check images/API versions and GBO-created SCC subjects. Keep `kernelModule` unset and install only GBO's NFD/DRA stack. Apply access resources and install the operator:

```bash
"$OC5" --kubeconfig .tmp/example-sno.kubeconfig --context example-sno apply -f "$CHART_DIR/gbo-access.yaml"
helm --kubeconfig .tmp/example-sno.kubeconfig --kube-context example-sno \
  upgrade --install gpu-operator "$CHART_DIR/intel-gpu-base-operator-chart-0.4.0.tgz" \
  -n intel-gpu-base-operator -f "$CHART_DIR/operator-values.yaml" --wait --timeout 20m
```

Require manager/NFD readiness, Certificate/Issuer readiness and populated admission webhook CA bundles before installing the policy:

```bash
helm --kubeconfig .tmp/example-sno.kubeconfig --kube-context example-sno \
  upgrade --install gpu-policy "$CHART_DIR/intel-gpu-base-operator-policy-chart-0.4.0.tgz" \
  -n intel-gpu-base-operator -f "$CHART_DIR/policy-values.yaml" --wait --timeout 20m
```

Require `ClusterPolicy/gpu-dra`, NFD GPU labels/rule, healthy GBO/webhook, Ready DRA/XPU workloads and a `gpu.intel.com` ResourceSlice containing the PF. Validate DeviceClass `gpu.intel.com` and `resource.k8s.io/v1` schemas. Zero desired DRA pods or inconsistent policy status fails readiness; capture evidence. Distinguish XPU's monitoring claim from compute allocation.

### 8. Reuse the validated workload and prove actual allocation

The pinned image's `/usr/REDACTED_HOSTNAME/bin/run-probe` runs precompiled `/opt/neo-test/probe`: 50 deterministic 2048² SYCL matmul iterations, finite-value checks, error threshold `< 0.1`. Runtime GPU-kernel JIT is allowed; no rebuild, runtime downloads or C++ compilation. Use these manifests.

Create namespace `gpu-dra-validation` and service account `matmul-runner`. The image runs as root and was admitted with SCC `anyuid`/UID 0 in the accepted run; no `anyuid` RoleBinding was created. The tunnel kubeconfig used for the read-only closeout identifies as `system:admin`, while an authorization check for `matmul-runner` returned `no`. Therefore the accepted run proves GPU compute, but not least-privilege admission by a non-admin deployer. If that path is required on a repeat, grant only this account namespace-scoped `anyuid` and verify the intended deployer path. Use no privileged SCC, hostPath or manual device mounts.

Save the claim and pod separately. Replace `<observed-guest-BDF>` and `<observed-node-hostname-label>` from live inventory; verify the claim schema before applying:

```yaml
# claim.yaml
apiVersion: resource.k8s.io/v1
kind: ResourceClaim
metadata: {name: gpu-pf, namespace: gpu-dra-validation}
spec:
  devices:
    requests:
    - name: gpu
      exactly:
        deviceClassName: gpu.intel.com
        allocationMode: ExactCount
        count: 1
        selectors:
        - cel:
            expression: device.attributes["resource.kubernetes.io"].pciBusID == "<observed-guest-BDF>"
```

```yaml
# pod.yaml
apiVersion: v1
kind: Pod
metadata: {name: matmul-pf, namespace: gpu-dra-validation}
spec:
  serviceAccountName: matmul-runner
  restartPolicy: Never
  nodeSelector:
    kubernetes.io/hostname: "<observed-node-hostname-label>"
  resourceClaims:
  - {name: gpu, resourceClaimName: gpu-pf}
  containers:
  - name: matmul
    image: quay.io/example-namespace/validation-image@sha256:1cec2ced159c8a3081fb4c9e70aacc7e7f5b9828662f226e57eba6d57f2b095f
    imagePullPolicy: Always
    env:
    - {name: EXPECTED_BDF, value: "<observed-guest-BDF>"}
    - {name: ONEAPI_DEVICE_SELECTOR, value: "level_zero:gpu"}
    resources:
      claims:
      - {name: gpu}
```

Apply claim, then pod, using the new-cluster context. Retain the claim after completion; `EXPECTED_BDF` is diagnostic, not allocation proof. After the positive run, run a copy named `unclaimed-gpu-control`, removing `spec.resourceClaims`, container `resources.claims` and `EXPECTED_BDF`; retain image/account/node/entrypoint.

Capture, before cleanup:

1. Claim YAML/allocation: driver, pool/device, node restrictions, request/consumer, resolving to ResourceSlice `resource.kubernetes.io/pciBusID`.
2. Host PF → libvirt guest address → guest sysfs/render node → pod device. All must match the allocation; selectors use guest BDFs.
3. Pod spec/events, SCC/node, image identity, injection, logs and exit code. Package baseline: `icpx 2026.1.1.20260724`, IGC 2.41.5, NEO/ocloc 26.35.39758.10, gmmlib 22.10.0, Level Zero 1.32.0.
4. **Positive:** 50/50 iterations, zero failures, worst difference `0.000`, exit 0. Record memory bytes/MiB against OCP 4's `34,242,297,856` / `32,656 MiB`; investigate differences.
5. **Control:** zero render nodes, expected no-GPU error, exit 2. Unexpected access fails isolation/allocation.
6. After capture, delete validation pods/compute claim and verify consumer/allocation release. Preserve XPU's monitoring claim; remove task SCC binding/account/namespace. Repeat allocation only if preparation/release remains uncertain.

ResourceSlice health/memory fields or enumeration alone cannot settle acceptance. Require allocation-to-device-to-compute evidence; record API/CDI/driver/scheduler failures instead of bypassing DRA with privileged device access.

**Milestone 3 / acceptance:** exact rc.3 cluster remains healthy; required cert-manager/GBO/NFD/DRA/XPU components are healthy; one real claim maps to the guest GPU/render node; the existing workload passes and the unclaimed control fails as expected; compute resources cleanly release; EXISTING_VM_B/EXISTING_VM_A remain off; old guard/configuration/artifacts remain preserved. Leave the accepted new VM running with autostart disabled.

## Failure handling

Stop on failed compatibility/health/digest/pull gates, unexpected GPU ownership/peers, unsafe references or guard changes. Capture the failing layer: VFIO/boot, guest kernel/firmware, cluster/API, operator/webhook, DRA, userspace or calculation. Distinguish access/VPN failure from cluster failure.

Preserve claim/pod/kernel/QEMU evidence and the new VM's disk/ISO/XML and current state for diagnosis. Record the blocker and next diagnostic step. EXISTING_VM_B and EXISTING_VM_A remain off after their planned shutdown; returning to OCP 4 is outside this phase.

Host changes are limited to the prepared tools and new libvirt resources. Reboot, firmware/driver changes or uplink/resolver reconfiguration need revised scope/recovery instructions.

## Repeat and closeout

The [[Notes - RHEL+1VM Approach to OCPv5 Classic]] contains accepted results, the build identity, protected evidence paths, retained lab state, and recovery notes. For a repeat, create a new run directory and identifier, refresh the host/cluster/guard preflight, and follow Milestones 1–3 above. Keep secrets and raw installer logs private. Treat historical acceptance as a reference, not proof of current state. This procedure covers one PF and pod/DRA compute; claim reboot persistence and the other excluded GPU workflows require separate tests.

## Targeted fallback sources

Open only the relevant section when the plan is missing a necessary detail or a live check contradicts it. Ordinary execution requires no source-document reading.

| Discrepancy | Fallback |
|---|---|
| Old lab identity or preflight guard checks differ | GPU_HOST `/path/to/lab-data`, heading `C10 accepted — 2026-10-15 14:55 UTC`; [guard diagnostics](../disconnected/rhel-2vm/guard/README.md); [[Plan - RHEL+2VM Approach to Disconnected OCP]]. Inspect read-only; these sources do not authorize restarting EXISTING_VM_B/EXISTING_VM_A. |
| Workload/image/allocation differs from embedded inputs | [workload lock](../disconnected/rhel-2vm/compute/workload-content-lock.yaml), [accepted manifests](../disconnected/rhel-2vm/compute/dra-sequential-final-20261015/), [[Disconnected OCP GPU validation results]] |
| Chart hash, values, SCC or reconciled images differ | [Helm evidence](../disconnected/rhel-2vm/operators/HELM-INPUTS.md), `disconnected/rhel-2vm/operators/{gpu-operator-values.yaml.in,gpu-policy-values.yaml,namespace-and-nfd-scc.yaml,runtime-images.lock.tsv}`; `/path/to/lab-data` at `v0.4.0` |
| rc.3 installer/catalog behavior differs | Exact installer commit and release links above; [[OCPv5 testing initiative]]; `/path/to/lab-data` as a reference only |
| Libvirt boot/MMIO/access differs | [[Deploying SNO LabEnv with Day-2 Worker and BMG Passthrough]] and the vendor links above; preserve this plan's RHEL 10, rc.3 and single-PF scope |
