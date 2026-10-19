# Isolation backend candidates (Oct 2026)

Ticket: `.scratch/sandbox-control-plane/issues/02-backend-candidates.md`. Researched 2026-10-19 against official docs, source code and release notes. Host in mind: Arch-based desktop, Ryzen 9 9950X, 32 GB RAM, AMD RX 9060 XT 16 GB, already running Incus.

**Verdict:** **Incus VMs** are the only candidate that meets every hard requirement today: live CPU, RAM and disk grow on a *running VM*, a shared AMD GPU path, snapshot/copy, a stable REST API, and they are already installed. **microsandbox** is the one alternative worth keeping. It has the best agent-shaped features (live resize, domain egress, secret proxy, fork, macOS), but it is beta, has no GPU support and no documented live policy edits. Firecracker, Cloud Hypervisor and Kata are VMM building blocks, not products: you would rebuild most of Incus on top of them. Docker `sbx` fails the resize requirement and only supports NVIDIA GPUs.

## Comparison

✅ = yes, documented · ⚠️ = partial or with caveats · ❌ = no · ? = not documented

| Requirement | Incus (VM) | Incus (container) | microsandbox | libkrun (raw) | Firecracker | Cloud Hypervisor | Kata Containers | Docker `sbx` |
|---|---|---|---|---|---|---|---|---|
| VM-level isolation | ✅ QEMU/KVM | ❌ shared kernel | ✅ libkrun microVM | ✅ | ✅ | ✅ | ✅ (VM per pod) | ✅ microVM |
| Live CPU change | ✅ hotplug ≤64 vCPU, not with pinning [1][2] | ✅ cgroup | ✅ within boot-time `max_cpus` [11] | ❌ fixed at build [9] | ❌ no vCPU hotplug [5] | ✅ `vm.resize` (x86), needs `max` at boot [7][8] | ⚠️ via OCI `update`, `default_maxvcpus` cap [14][15] | ❌ create-time only [17] |
| Live RAM change | ✅ grow by hotplug, shrink by balloon (slow) [1][3] | ✅ | ✅ within `max_memory` [11] | ❌ (balloon is page reporting only) [9] | ⚠️ virtio-mem, pool sized pre-boot [6] | ✅ ACPI or virtio-mem, `hotplug_size` pre-boot [7] | ⚠️ via OCI `update` [15] | ❌ [17] |
| Live disk grow | ✅ QMP block resize on running VM; guest grows FS [4] | ✅ | ✅ ext4 root, grow-only, brief pause [11] | ? | ⚠️ resize backing file + rescan [5] | ✅ `vm.resize-disk` [8] | ? | ❌ no disk flag [17] |
| GPU passthrough | ✅ `physical` (VFIO, whole GPU, no hotplug) or `native-context` (shared, AMD OK, QEMU ≥ 11) [2a][2b] | ✅ `physical`, hotplug OK | ❌ not documented | ⚠️ virtio-gpu venus/native-context [9] | ❌ open issue, no ETA [5a] | ✅ VFIO, incl. hotplug via `vm.add-device` [7][8] | ⚠️ QEMU + NVIDIA only; AMD "placeholder" [14a] | ⚠️ experimental, NVIDIA + x86_64 Linux only [18] |
| Per-sandbox egress hooks | ⚠️ per-NIC ACLs, IP/CIDR only, logged rules; FQDN needs a proxy [10] | same | ✅ domain allowlist in host user-space stack, deny-by-default; ❌ live edit / approval hook not documented [12] | ⚠️ TSI uses host netns; filter it yourself [9a] | ❌ bring your own tap + nftables | ❌ bring your own | ⚠️ CNI netns | ✅ host proxy policies, `sbx policy log` [19] |
| Snapshot / clone | ✅ snapshots + `copy`; stateful needs `migration.stateful` (forces 9p not virtiofs) [13] | ✅ | ✅ snapshot, restore, live fork [11a] | ❌ | ✅ GA (diff snapshots preview) [5b] | ✅ paused snapshot, restore as clone [8a] | ❌ no checkpoint/restore [15] | ⚠️ `template save` only (see ticket 01) |
| Daemon-drivable API | ✅ REST over unix socket/HTTPS + Go client | ✅ | ⚠️ in-process SDKs (TS/Py/Rust/Go) + CLI; no local daemon API documented [12a] | ⚠️ C/Rust library; 2.0 API unstable [9] | ✅ REST per VM socket | ✅ REST per VM socket [8] | ⚠️ via containerd / Docker / nerdctl; no Podman [15] | ⚠️ CLI; SDK only for cloud sandboxes (see ticket 01) |
| Linux host | ✅ kernel ≥ 6.12, QEMU ≥ 8.2 [16] | ✅ | ✅ KVM | ✅ | ✅ | ✅ | ✅ | ⚠️ Ubuntu 24.04+ only; Arch unsupported [20] |
| macOS host (bonus) | ⚠️ client only; server via Colima VM (VMs need M3+) [21] | ⚠️ same | ✅ Apple Silicon | ✅ HVF | ❌ | ❌ | ❌ | ✅ |
| Licence | Apache-2.0 | Apache-2.0 | Apache-2.0 | Apache-2.0 | Apache-2.0 | Apache-2.0 + BSD-3 | Apache-2.0 | Closed, free locally |
| Maturity (latest) | 7.5 (25 Sep 2026); 7.0 LTS (May 2026) | same | **beta**, v0.7.8 (9 Oct 2026) | 2.0 on `main` unstable | v1.17.0 | v53.0 (12 Jul 2026) | 4.2.0 (15 Sep 2026) | v0.48 (13 Oct 2026), breaking CLI |

## Live-resize details (VMs specifically)

- **Incus VM CPU:** `limits.cpu` is live-updatable and hotplugs vCPUs into a running VM. Limits: max 64, no live change with CPU pinning or explicit topology (`sockets=…`), and "the guest OS may need a restart or manual steps before the new CPUs come online" [1][2c]. *Implication:* the control plane must keep `limits.cpu` as a plain count and online CPUs in the guest (udev rule or agent).
- **Incus VM RAM:** `limits.memory.hotplug` defaults to `true`, with a ceiling equal to host RAM on standalone hosts. Raising `limits.memory` adds DIMMs over QMP; lowering it inflates the balloon, which is slow and may need reapplying. Hugepages block live change [1][3].
- **Incus VM disk:** changing the root `size` on a running VM resizes the storage volume, then calls QMP block resize (`internal/server/device/disk.go` around L1963-2004; `drivers/driver_qemu.go` around L11276-11292). If the driver reports the volume in use, the change is deferred to next boot (`volatile.<dev>.apply_quota`). Shrinking block volumes isn't supported [4]. The guest still has to grow its partition and filesystem.
- **microsandbox:** `msb modify --cpus/--memory/--root-disk` and `sandbox.modify()` work live within `max_cpus`/`max_memory`, which are fixed at boot (raising them means a restart). Root disk growth is ext4 and grow-only, with a brief VM pause. Local only, not cloud [11]. The mechanism isn't documented.
- **Cloud Hypervisor:** `vm.resize` (CPUs on x86, memory) and `vm.resize-disk` are first-class API calls. Hotplug headroom must be declared at boot [7][8].
- **Firecracker:** memory via virtio-mem only, from a hotplug pool configured pre-boot (guest ≥ 5.16, `CONFIG_VIRTIO_MEM`). No vCPU hotplug [5][6].

## GPU on this host (AMD RX 9060 XT)

- **Incus `native-context`** (added in 7.3): virtio-gpu with DRM native context. It is shared by several VMs, and the host keeps using the GPU [2b][2d]. QEMU documents AMDGPU support with host kernel ≥ 6.13, virglrenderer ≥ 1.1, guest kernel ≥ 6.14 and guest Mesa ≥ 25.0 [22]. Incus needs **QEMU ≥ 11.0** for this type. It gives the guest Mesa graphics and Vulkan, which is the likely route for local models (for example llama.cpp's Vulkan backend). ROCm/HIP compute through it is **not documented**.
- **Incus `physical`** (VFIO): the whole GPU goes to one VM, with no hotplug for VMs [2a]. The 9950X has an iGPU, so the host desktop could run on it while the dGPU is passed through. Forum reports describe reset problems with RX 9060 XT VFIO passthrough [23] (secondary sources, not verified).
- **Everything else:** Docker `sbx` is NVIDIA-only [18]. Kata documents NVIDIA only [14a]. Firecracker has none [5a]. Cloud Hypervisor does VFIO with hotplug but has no shared-GPU path [7]. microsandbox documents none.

## Recommended shortlist

1. **Incus VMs: v1 backend.** It meets every requirement: live CPU, RAM and disk grow on running VMs, shared AMD GPU via native-context, snapshots and copies, per-NIC ACLs with logging, and a mature REST API with a Go client. It is Apache-2.0 with an LTS line, and it is already running on the host. The gaps are ones the control plane owns anyway: FQDN egress (route the VM through a host proxy and allow only the proxy by ACL; see ticket 03), guest-side CPU onlining and filesystem growth, and macOS hosts.
2. **microsandbox: the alternative to watch, not the v1 base.** It is the closest match to the sandbox concept (domain egress, a secret proxy that "never enters the VM", live fork, live resize, macOS and Windows hosts). But it is beta with frequent breaking changes, has no GPU support, documents no live egress-policy edits or approval hooks, and exposes no local daemon API. A pluggable backend layer could add it later as a second backend for macOS hosts.

The other options aren't viable for v1. Firecracker lacks GPU and vCPU hotplug. Cloud Hypervisor and raw libkrun would mean rebuilding image, storage, network and snapshot management. Kata is shaped for Kubernetes and containerd, with no Podman or checkpoint support. Docker `sbx` can't resize, is NVIDIA-only and doesn't support Arch.

## Unknowns (verify on the host before ticket 05)

- **QEMU ≥ 11.0 and a native-context-enabled virglrenderer on Arch:** current package versions weren't reachable from the research environment. Check with `qemu-system-x86_64 --version` and `pacman -Qi virglrenderer`.
- **RDNA4 (gfx1200) under native-context:** whether the RX 9060 XT works, and whether guest compute (Vulkan for llama.cpp; ROCm almost certainly not) performs usefully. This needs a spike.
- **CPU onlining in Incus guests:** whether the stock `images:` cloud images auto-online hotplugged vCPUs, or need a udev rule.
- **Live root-disk grow per Incus storage driver:** the code path exists, but whether the host's pool driver (ZFS, btrfs, dir or LVM) returns `ErrInUse` and defers is untested.
- **Incus ACL changes on a running NIC:** whether they apply instantly (likely, since they are nftables rules) isn't stated in the docs.
- **microsandbox:** whether network rules can be changed live, whether a local HTTP API is planned, and what mechanism its live memory resize uses.

## Sources

1. Incus instance options (`limits.cpu`, `limits.memory`, `limits.memory.hotplug`): https://linuxcontainers.org/incus/docs/main/reference/instance_options/
2. Incus API extensions `cpu_hotplug`, `memory_hotplug`, `limits_memory_hotplug`, `gpu_native_context`: https://github.com/lxc/incus/blob/main/doc/api-extensions.md
   - 2a. Incus GPU devices: https://linuxcontainers.org/incus/docs/main/reference/devices_gpu/
   - 2b. `gpu_native_context` extension text ("One host GPU can be shared by several VMs … Requires QEMU 11.0.0 or newer"), same file as [2]
   - 2c. `cpu_hotplug` extension ("Hotplugging is disabled when using CPU pinning"), same file
   - 2d. Incus 7.3 announcement: https://discuss.linuxcontainers.org/t/incus-7-3-has-been-released/27033
3. Incus memory hotplug and balloon code (`updateMemoryLimit`, `hotplugMemory`): https://github.com/lxc/incus/blob/main/internal/server/instance/drivers/driver_qemu.go
4. Incus live disk resize: https://github.com/lxc/incus/blob/main/internal/server/device/disk.go and `driver_qemu.go` (QMP `UpdateBlockSize`); resizing how-to: https://linuxcontainers.org/incus/docs/main/howto/storage_volumes/
5. Firecracker README and features: https://github.com/firecracker-microvm/firecracker/blob/main/README.md; device hotplug (developer preview): https://github.com/firecracker-microvm/firecracker/blob/main/docs/device-hotplug.md
   - 5a. GPU issue #1179 (open, "no ETA"): https://github.com/firecracker-microvm/firecracker/issues/1179
   - 5b. Snapshots: https://github.com/firecracker-microvm/firecracker/blob/main/docs/snapshotting/snapshot-support.md; changelog: https://github.com/firecracker-microvm/firecracker/blob/main/CHANGELOG.md
6. Firecracker memory hotplug: https://github.com/firecracker-microvm/firecracker/blob/main/docs/memory-hotplug.md
7. Cloud Hypervisor hotplug and VFIO: https://github.com/cloud-hypervisor/cloud-hypervisor/blob/main/docs/hotplug.md · https://github.com/cloud-hypervisor/cloud-hypervisor/blob/main/docs/vfio.md
8. Cloud Hypervisor REST API: https://github.com/cloud-hypervisor/cloud-hypervisor/blob/main/docs/api.md · releases: https://github.com/cloud-hypervisor/cloud-hypervisor/releases
   - 8a. Snapshot and restore: https://github.com/cloud-hypervisor/cloud-hypervisor/blob/main/docs/snapshot_restore.md
9. libkrun README and VM lifecycle: https://github.com/containers/libkrun/blob/main/README.md · https://github.com/containers/libkrun/blob/main/docs/vm-lifecycle.md
   - 9a. libkrun networking: https://github.com/containers/libkrun/blob/main/docs/networking.md
10. Incus network ACLs: https://linuxcontainers.org/incus/docs/main/howto/network_acls/; NIC `security.acls.default.egress.*`: https://github.com/lxc/incus/blob/main/doc/config_options.txt
11. microsandbox tuning and live modify: https://docs.microsandbox.dev/sandboxes/tuning.md · CLI: https://docs.microsandbox.dev/cli/sandbox-commands.md
    - 11a. microsandbox README (fork, snapshot, beta warning, licence): https://github.com/superradcompany/microsandbox · releases: https://github.com/superradcompany/microsandbox/releases
12. microsandbox network security and networking: https://docs.microsandbox.dev/security/network.md · https://docs.microsandbox.dev/networking/overview.md
    - 12a. microsandbox backends: https://docs.microsandbox.dev/operations/backends.md
13. Incus `migration.stateful` (and its virtiofs/9p trade-off): https://github.com/lxc/incus/blob/main/doc/config_options.txt
14. Kata hypervisors table: https://github.com/kata-containers/kata-containers/blob/main/docs/hypervisors.md · vCPU sizing: https://github.com/kata-containers/kata-containers/blob/main/docs/design/vcpu-handling-runtime-rs.md
    - 14a. Kata GPU: https://github.com/kata-containers/kata-containers/blob/main/docs/use-cases/GPU-passthrough-and-Kata.md
15. Kata limitations (update, checkpoint, Podman): https://github.com/kata-containers/kata-containers/blob/main/docs/Limitations.md · releases: https://github.com/kata-containers/kata-containers/releases
16. Incus requirements: https://linuxcontainers.org/incus/docs/main/requirements/ · releases: https://github.com/lxc/incus/releases · news: https://linuxcontainers.org/incus/news/
17. `sbx create` flags: https://docs.docker.com/reference/cli/sbx/create/
18. `sbx` GPU passthrough: https://docs.docker.com/ai/sandboxes/configuration/gpu-passthrough/
19. Docker Sandboxes overview and release notes: https://docs.docker.com/ai/sandboxes/ · https://docs.docker.com/ai/sandboxes/release-notes/
20. Docker Sandboxes install requirements: https://docs.docker.com/ai/sandboxes/install/
21. Colima README (Incus runtime on macOS): https://github.com/abiosoft/colima/blob/main/README.md
22. QEMU virtio-gpu DRM native context: https://www.qemu.org/docs/master/system/devices/virtio/virtio-gpu.html
23. RX 9060 XT passthrough reports (secondary): https://forum.proxmox.com/threads/problem-with-gpu-passthrough-amd-rx-9060-xt.177974/ · https://forum.proxmox.com/threads/amd-radeon-gpu-passthrough-infamous-reset-bug.186112/
