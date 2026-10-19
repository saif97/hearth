# Which isolation backends meet the sandbox requirements?

Type: research
Status: resolved
Blocked by:

## Question

Compare Incus (VMs and containers), libkrun / microsandbox, Firecracker, Cloud Hypervisor, Kata Containers, and Docker `sbx` against the sandbox requirements: VM-level isolation; **live** resize of CPU/RAM (and disk grow) without recreating; GPU passthrough; per-sandbox egress filtering hooks; snapshot/clone; a programmable API a daemon can drive; Linux host support (macOS host as a bonus); licence and maturity. Which one or two are viable for v1?

## Answer

Incus VMs are the only candidate that meets every requirement today: live CPU hotplug, RAM hotplug and root-disk grow on running VMs, a shared AMD GPU via `native-context` (QEMU ≥ 11), snapshot/copy, and a mature REST API. They are also already on the host.
microsandbox is the runner-up: live resize, domain egress and a secret proxy, plus macOS hosts. It is beta, though, with no GPU support and no documented live policy edits.
Firecracker, Cloud Hypervisor and Kata are VMM building blocks, and `sbx` can't resize and only supports NVIDIA GPUs. The main unknowns are RDNA4 under native-context and the QEMU/virglrenderer versions on Arch.

Findings: [research/backend-candidates.md](../../../research/backend-candidates.md)
