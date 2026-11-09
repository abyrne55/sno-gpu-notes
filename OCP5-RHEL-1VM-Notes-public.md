> **Public-safe copy:** hostnames, IP addresses, MAC addresses, PCI addresses, UUIDs, user identifiers, local paths, and private links have been anonymized. Commands and network examples are reference material and are not directly runnable.

# OCP 5 lab notebook

## Completed run — Milestones 1–3 accepted

**Milestone 1 accepted 2026-10-20 21:06 UTC; Milestone 2 accepted 2026-10-21 00:44 UTC; Milestone 3 accepted 2026-10-21 17:17 UTC.** This notebook is the accepted execution record for [[Plan - RHEL+1VM Approach to OCPv5 Classic]]. The plan retains the complete rebuild and repeat procedure.

The tested stack was OCP `5.0.0-rc.3` / Kubernetes `v1.36.3`, one Intel Arc Pro B70 PF passed through to guest `xe`, GBO `0.4.0`, DRA `0.11.0`, XPU Manager `2.2.0`, and NFD `0.19.0`. The exact-count DRA claim selected ResourceSlice device `0000-05-00-0-0xe223`, guest BDF `REDACTED_BDF`, and `renderD128`; the injected container device was render node 226:128. The pinned SYCL matmul passed 50/50 iterations with zero failures and worst difference `0.000`, reporting 34,242,297,856 bytes / 32,656 MiB, matching the OCP 4 reference. The unclaimed control saw zero render nodes and exited 2. This is a one-PF, one-SNO rc.3 result, not a certification or broader compatibility guarantee.

### Retained lab state — read-only check 2026-10-21 20:20 UTC

The API `/readyz` returned `ok`; the node was Ready, all 34 ClusterOperators were Available/not Progressing/not Degraded, and the master MCP was Updated 1/1. ClusterVersion was Available, not Failing or Progressing; `RetrievedUpdates=False/VersionNotFound` on `stable-5.0` remained an update-discovery condition. Cert-manager 1.20.0, GBO, NFD, DRA, and XPU workloads were Ready; DRA and XPU reported 1/1. No validation namespace or compute claim remained; the XPU `adminAccess` monitoring claim remained allocated.

On GPU_HOST, `SNO_VM` was running with autostart disabled and its ISO detached; EXISTING_VM_A and EXISTING_VM_B were shut off. `existing-lab-guard.service` was active/enabled, and all seven guard checks passed. The host PF `REDACTED_BDF` remained bound to `vfio-pci` and was the VM's only GPU hostdev; `REDACTED_BDF` remained unassigned on host `vfio-pci`.

### M3 evidence and qualifications

Protected evidence is under `/path/to/lab-data`: `m3-preflight-20261021T1657Z.txt`, `m3-gpu-ownership-20261021T1710Z.txt`, `m3-gpu-qemu-ownership-20261021T1711Z.txt`, `m3-pf54-final-20261021T1718Z.txt`, `m3-preinstall-catalog-20261021T1704Z.txt`, `m3-cert-manager-installplan-20261021T1708Z.txt`, `m3-guest-gpu-20261021T1712Z.txt`, `m3-compute-validation-20261021T1715Z.txt`, and `m3-final-acceptance-20261021T1717Z.txt`. Pinned Helm archives, values, NFD SCC access manifest, render outputs, `chart-inputs.lock.txt`, and the positive/control manifests are also retained there. Raw logs and secrets remain private.

- The accepted run pinned DRA `0.11.0`; the GBO chart default `0.12.0` and cert-manager `1.20.1` were not accepted/tested. The stable-v1 `1.20.1` Manual InstallPlan remains unapproved by design.
- The workload pod ran as UID 0 under SCC `anyuid`, with no task `anyuid` RoleBinding. The tunnel kubeconfig used during closeout authenticated as `system:admin`; a post-cleanup authorization check for `matmul-runner` returned `no`. This proves the GPU compute path, but does not establish a non-admin/least-privilege admission path. If needed on a repeat, bind `anyuid` only to `matmul-runner` in the validation namespace and verify that deployer path.
- A read-only check after M3 found that around 18:21 UTC the kube-apiserver pod was replaced while the GBO controller and NFD worker restarted eight times. Their previous logs show connections refused to the Kubernetes service at `REDACTED_IP:443`; the GBO manager lost leader election and NFD could not update its NodeFeature. At 20:20 UTC the components and API were healthy again. The cause was not established. NFD logs also reported permission errors reading generic host feature paths, although feature discovery completed and the GPU label was present.
- Claim reboot persistence was not tested. CNV/KubeVirt GPU use, multiple PFs, SR-IOV, inference, out-of-tree drivers, and firmware changes are outside this run.

### Rebuild and repeat

Use the linked plan's Milestones 1–3 for the exact installer, ISO, VM, image, Helm chart digests and values, claim/pod/control manifests, readiness gates, and cleanup. Before a future repeat, refresh host, VM, GPU, cluster, network, and guard state and assign a new run identifier. The accepted state here is historical evidence, not proof of current state or authorization to reinstall this VM.

### Accepted state at M2 completion

M2 snapshot; values below are historical accepted observations at 00:44 UTC, not a new health assertion.

| Item                | Accepted state                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                 |
| ------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| VM                  | `SNO_VM`, UUID `REDACTED_UUID`, running, persistent, autostart disabled; disk `/var/lib/libvirt/images/ocp5-classic/SNO_VM.qcow2`, no backing file; NVRAM `/var/lib/libvirt/qemu/nvram/SNO_VM_VARS.fd`; installed root `/dev/vda4`; ISO ejected from live and persistent XML                                                                                                                                                                                                                                                   |
| Network             | `example-nat`, `REDACTED_IP`, guest `REDACTED_IP`, MAC `REDACTED_MAC`; NAT active, autostart disabled                                                                                                                                                                                                                                                                                                                                                                                                                          |
| Cluster             | `REDACTED_HOSTNAME`, completed `5.0.0-rc.3`, release `quay.io/openshift-release-dev/ocp-release@sha256:29da60c98a5347be6e412b4176b5c27e1ca6336158178d77afc9dc5dbd1ed922`; 1 Ready node `REDACTED_HOSTNAME`, Kubernetes `v1.36.3`; 34/34 ClusterOperators healthy; master MCP Updated 1/1; worker pool Updated 0/0                                                                                                                                                                                                              |
| Guest OS            | RHCOS `10.2.20260910-2`; kernel `6.12.0-211.54.1.el10_2.x86_64`; CRI-O `1.36.5-202609081227.p2.g224b14a.assembly.stream.el10`                                                                                                                                                                                                                                                                                                                                                                                                  |
| Client              | Use `/path/to/lab-data`, version `5.0.0-rc.3`, SHA-256 `a1db1bb8ae76f601a3bb6d27646a3d6836a0cda9c23381703d8e2b0daaab51e6`; do not use an unversioned `oc` for this cluster.                                                                                                                                                                                                                                                                                                                                                    |
| GPU                 | Host PF `REDACTED_BDF` (`8086:e223`, `vfio-pci`, zero VFs) is the sole hostdev; PF `REDACTED_BDF` remains unassigned on host `vfio-pci`. Guest BDF `REDACTED_BDF` uses `xe`; render node `renderD128` via `/dev/dri/by-path/pci-REDACTED_BDF-render`; GuC `70.65.0`, HuC `8.2.10`; virtual BIOS `edk2-20251114-5.el10_2.2`                                                                                                                                                                                                     |
| Old lab/guard       | EXISTING_VM_A and EXISTING_VM_B remain `shut off`; UUIDs, definitions, disks/NVRAM and networks are retained. `existing-lab-guard.service` active/enabled; all seven guard checks pass. Do not start EXISTING_VM_A/EXISTING_VM_B.                                                                                                                                                                                                                                                                                              |
| Workstation access  | API tunnel `REDACTED_IP:17443 → REDACTED_IP:6443`; SSH tunnel `REDACTED_IP:13222 → :22`; kubeconfig `.tmp/example-sno.kubeconfig` mode 0600, context `example-sno`, API `REDACTED_URL`, TLS server name `REDACTED_HOSTNAME`, CA retained, TLS verification enabled. At 00:49 UTC both loopback listeners were present and `/readyz` returned `ok`; recheck/restart tunnels before future access. Strict guest SSH uses `.tmp/example-sno-known_hosts`, alias `REDACTED_HOSTNAME`, fingerprint `REDACTED_HOST_KEY_FINGERPRINT`. |
| M2 image-pull proof | `default/example-sno` pulled the previously uncached image with `imagePullPolicy: Always`, `/bin/true`, exit 0, and `imageID` equal to `quay.io/example-namespace/validation-image@sha256:1cec2ced159c8a3081fb4c9e70aacc7e7f5b9828662f226e57eba6d57f2b095f`. EXISTING_VM_A/EXISTING_VM_B were off.                                                                                                                                                                                                                             |

Active etcd, API server, controller-manager, and scheduler pods were Ready with zero restarts. Historical installer pods include failed attempts followed by successful retry pods; all active control-plane pods and operators were healthy. No IDMS/ITMS/ICSP, additional trusted registry CA, or `quay.io` hosts entry is configured.

### M2 recovery notes for a future repeat

- When EXISTING_VM_A stopped, kernel logs showed libvirt releasing VFIO and probing both PFs with `xe`; both `driver_override` values were null. After confirming zero VFs, singleton IOMMU groups, no QEMU/DRM users, disconnected BMG outputs, and ASPEED host display, the user authorized a reversible per-BDF override/unbind/bind to `vfio-pci` for `REDACTED_BDF` and `REDACTED_BDF`. No global driver/firmware setting changed. Before attaching a PF on a future repeat, recheck hostdev ownership and guest `xe`; do not assume bindings from this record if VM/host state changes.
- The first bootstrap waiter timed out at its built-in 60-minute deadline because GPU_HOST's normal resolver could not resolve the API hostname. The guest resolved it correctly. Recovery used `REDACTED_IP` only inside a temporary private mount namespace; TLS-verified `/readyz` returned `ok`, then bootstrap and install-complete both exited 0 in 2 seconds. The host `/etc/resolv.conf` was unchanged; the temporary resolver and runner were removed.
- The guest host key was obtained from GPU_HOST's private bridge only after proving the sole `example-nat` lease and running guest matched the expected MAC/IP. SSH used the pinned key with `StrictHostKeyChecking=yes`; QGA file/exec and console password access were unavailable. Do not use TOFU or weaken SSH verification.

### M2 evidence

- Final acceptance: `/path/to/lab-data` (0600).
- At the 00:49 UTC M2 handoff, `helm list --all-namespaces` was empty. The 17:05 UTC M3 preflight found no M3 CSVs, Subscriptions, or InstallPlans; only the built-in Package Server CSV was present.
- Detailed M2 commands, protected waiter logs/status, serial/SSH/guest, domain, GPU, pull, guard, and recovery evidence remain under `/path/to/lab-data`; all raw logs and secrets remain private.

## Milestone 1 record (historical; M2 state above supersedes it)

### Fixed run and retained assets

On GPU_HOST; reuse this build and ISO rather than generating a new run:

```bash
RUN_ROOT=/path/to/lab-data
BUILD_DIR=/path/to/lab-data
ISO=/var/lib/libvirt/images/ocp5-classic/agent.x86_64.iso
DSO_KUBECONFIG=/path/to/lab-data
```

| Item | Accepted value / location |
|---|---|
| Workstation | `/path/to/lab-data`; use ignored `.tmp/` for scratch, clean when finished |
| Access | `ssh -o BatchMode=yes GPU_HOST -- '<command>'`, outside sandbox; host `REDACTED_HOSTNAME` |
| Exact tools | `$RUN_ROOT/tools/openshift-install-v5rc3` and `oc-v5rc3`, both `5.0.0-rc.3`; versions/checksums matched the plan. Butane 0.27.0. Use the OCP 4 client/context for EXISTING_VM_A |
| Private input copies | `$RUN_ROOT/secrets/install-config.yaml` and `agent-config.yaml`, both `REDACTED_USER:REDACTED_USER 0600` |
| Matching generated auth | `$BUILD_DIR/auth/kubeconfig` and `kubeadmin-password`, both `REDACTED_USER:REDACTED_USER 0600`; existence/modes rechecked during consolidation, contents not printed |
| NAT XML | `$BUILD_DIR/config/example-nat.xml`, mode 0600 |
| Evidence | `$BUILD_DIR/evidence/`, mode 0700 directory / 0600 files; filenames below are relative to it |
| Planned VM | `SNO_VM`, 24 vCPU / 64 GiB, standalone 300 GiB `/var/lib/libvirt/images/ocp5-classic/SNO_VM.qcow2`, independent UUID/NVRAM, no autostart; use the plan's Q35/UEFI/MMIO/domain settings |
| Node | `REDACTED_HOSTNAME`, `REDACTED_IP`, MAC `REDACTED_MAC`, root `/dev/vda`; verify assumed NIC `enp1s0` at first boot |
| Topology / CIDRs | Classic platform-none SNO, 1 control plane / 0 workers, OVN; machine `REDACTED_IP`, **pods `REDACTED_IP`, hostPrefix 23**, services `REDACTED_IP` |

The original pod CIDR `REDACTED_IP` overlapped the workstation VPN's `REDACTED_IP`. Its replacement passed workstation checks and numeric GPU_HOST overlap checks across 21 prefixes, all libvirt networks and rootless/rootful Podman. Exclusions were defaults, loopback and only the new network's own entries.

### Accepted ISO and workload

- **Release:** `quay.io/openshift-release-dev/ocp-release@sha256:29da60c98a5347be6e412b4176b5c27e1ca6336158178d77afc9dc5dbd1ed922`, installer commit `b6f119779c7ce529a94c7b8956a057343675f212`; expected Kubernetes `1.36.3`. Record actual installed RHCOS/kernel separately; boot-media label `rhcos-4.22` is expected. Clear inherited release/RHCOS overrides before installer commands.
- **ISO:** `$ISO` matches `$BUILD_DIR/agent.x86_64.iso`; **823,836,672 bytes**, SHA-256 **`56f75fa08300013e227c0110befc020331d1030208c23d7605a2832f8c175689`**. Staged `0640 root:qemu`, directory `0750 root:qemu`; `virt_image_t`, `matchpathcon -V` and QEMU readability passed. Build ran 20:52:09–20:54:13 UTC, installer exit 0. Embedded checks passed for release/identity/MAC/IP, `/dev/vda`, classic topology, `92-serial-console`, `console=tty0 console=ttyS0,115200n8`, and `rootfs.img`; no e3/HPP/disconnected inputs. Preserve matching auth assets; installer consumed build input copies.
- **Workload:** `quay.io/example-namespace/validation-image@sha256:1cec2ced159c8a3081fb4c9e70aacc7e7f5b9828662f226e57eba6d57f2b095f`. Tag `ocp5-rc3-GPU_HOST-matmul-20261016-1cec2ced159c`; config `sha256:45a83030ade06dc14a3c6c2373c9b12a8717e162ef9099fab6a42169dbc0940b`. Anonymous independent inspection matched source/destination manifest bytes, all 19 layers and `linux/amd64`; publication completed 20:43:06 UTC. **Do not republish M1.**

### Preserved state at the M1 boundary

EXISTING_VM_A/EXISTING_VM_B remained **running, autostart disabled**. Final health/inventory capture: 20:59:43 UTC. EXISTING_VM_A: OCP 4.22.15 / Kubernetes v1.35.6, 1 Ready node, 34/34 healthy ClusterOperators, master MCP Updated. EXISTING_VM_B: `/srv/mirrors` mounted XFS, registry/guard services active, TLS-verified private registry `:5000/v2/` HTTP 200.

| Resource | Retained identity / state |
|---|---|
| EXISTING_VM_A | UUID `REDACTED_UUID`; only `isolated-example-net` NIC |
| EXISTING_VM_B | UUID `REDACTED_UUID`; `isolated-example-net` + `example-uplink` NICs |
| Selected PF | Host `REDACTED_BDF`, singleton group 24, `vfio-pci`, zero VFs, reset `flr bus`, 32 GiB BAR 2; **still owned by EXISTING_VM_A**, guest `REDACTED_BDF` / `renderD128` |
| Other PF | Host `REDACTED_BDF`, singleton group 16, `vfio-pci`, zero VFs; still owned by EXISTING_VM_A, guest `REDACTED_BDF` / `renderD129`; leave unassigned on host VFIO after shutdown |
| Guard / old networks | Guard active/enabled, all seven checks passed after NAT creation, five checker/rules/hook/service hashes unchanged. `default`, `isolated-example-net`, `example-uplink` preserved; 8090 closed; no host 5000/8090 listener expected |
| New NAT | `example-nat`, UUID `REDACTED_UUID`, `virbr-ocp5`, gateway `REDACTED_IP`; active/persistent/autostart disabled; A/PTR/apps/upstream/negative/AAAA DNS checks passed |

**No new VM, disk, NVRAM, domain XML, guest boot/shutdown or installed OCP 5/GPU validation occurred.** The new image directory contained only the ISO. the lab owner confirmed BMC console recovery access; no automated BMC login was tested. A bridge showing DOWN without a guest attached was observed despite working libvirt DNS.

Preserve the plan's complete old-lab artifact list, especially `/var/lib/libvirt/images/disconnected/`, `/path/to/lab-data`, `/path/to/lab-data`, EXISTING_VM_B `/srv/mirrors/`, guard/approved XML/C4 markers, EXISTING_VM_A's XPU monitoring claim, and the older pre-GPU checkpoint `/path/to/lab-data`. No live disk backup/integrity scan was performed.

### Evidence and resolved execution lessons

All remote filenames below are under `$BUILD_DIR/evidence/`; raw logs/configs remain private.

| Proof | Files |
|---|---|
| Preservation / final state | `EXISTING_VM_A-inactive.xml`, `EXISTING_VM_B-inactive.xml`, `network-{default,isolated-example-net,example-uplink}.xml`, `old-lab-storage.txt`, `gpu-ownership.txt`, `guard-preflight.txt`, `m1-post-network-health-guard.txt`, `m1-final-state.txt` |
| Publication / corrections | `m1-publication-verification.txt`, `destination-manifest-inspect.json`, `skopeo-copy*.log`, `skopeo-copy*-status.txt`, `ca-diagnosis/`; EXISTING_VM_B originals `/root/ocp5-publish/evidence/` |
| Network / collisions | `m1-network-dns-guard.txt`, `m1-cidr-overlap-proof.json`; REDACTED_HOSTNAME workspace `evidence/workstation-preflight-20261020T201626Z.txt` |
| ISO / release / exit | `m1-iso-preflight-success.txt`, `agent-create-image.status.txt`, `agent-iso-kargs-hash.txt`, `m1-iso-contents.txt`, `m1-iso-inspection.txt`, `m1-iso-stage.txt`; private `agent-create-image.raw.log` |

Resolved history: unavailable tool pin replaced with the lab owner-approved Skopeo 1.22.3 pin in the plan; source TLS fixed by explicit `--src-cert-dir` using accepted `/path/to/lab-data`, with TLS/SELinux retained and no global trust change. The SSH setup wrapper returned 1 while the durable copy succeeded: require real journal/exit and destination proof, not a missing unit's default success fields. ISO launcher exit 127 preceded any installer execution; the retry exited 0. A YAML 1.1 MAC-parsing quirk affected only verification; raw/embedded MAC matched and the plan example is quoted.

Task auth/CA/parser/job scratch and GPU_HOST `registry-login.sh` were removed; private inputs/auth remain. The Skopeo tool image was retained on EXISTING_VM_B; no shared pruning occurred. Push credentials are not in the SNO. The two task execution logs and their redundant REDACTED_HOSTNAME mirror are retired by this consolidation; preserve the separate disconnected-lab execution history used by guard fallback.
