# Which isolation backends meet the sandbox requirements?

Type: research
Status: open
Blocked by:

## Question

Compare Incus (VMs and containers), libkrun / microsandbox, Firecracker, Cloud Hypervisor, Kata Containers, and Docker `sbx` against the sandbox requirements: VM-level isolation; **live** resize of CPU/RAM (and disk grow) without recreating; GPU passthrough; per-sandbox egress filtering hooks; snapshot/clone; a programmable API a daemon can drive; Linux host support (macOS host as a bonus); licence and maturity. Which one or two are viable for v1?
