# Spike microsandbox vs Incus VMs on the host

Type: task
Status: open
Blocked by:

## Question

On Saif's PC, confirm the facts the backend choice hangs on (HITL checklist). Background: [microsandbox deep dive](../../../research/microsandbox-deep-dive.md).

**microsandbox first**
- Install `msb`; build an agent image from a Dockerfile and `msb load` it.
- Create a sandbox with generous `max_cpus`/`max_memory`; `msb modify --cpus/--memory/--root-disk` while running. Does the guest see it?
- Set microsandbox policy to public-only and chain an upstream HTTP CONNECT proxy (mitmproxy). Does the proxy see the SNI and can it deny/allow live?
- Secret proxy: GitHub token bound to `github.com` + `api.github.com`; do `git push` (Basic auth) and `gh pr create` work with only the placeholder in the guest?
- Does the upstream proxy see decrypted GitHub traffic (for the push/merge filter), or only the tunnel?

**Incus VMs as fallback**
- Incus and QEMU versions.
- Change `limits.cpu` and `limits.memory` on a running VM. Do new vCPUs come online in the stock image?
- Grow the running VM's root disk on the current pool. Live, and does the guest filesystem grow?
- Change a network ACL on the running VM. Immediate?
- Does Incus anti-spoofing (`security.mac_filtering`, `security.ipv4_filtering`) apply to the VM NIC?

**Either backend**
- A push rejected mid-upload by the proxy: does git show a clean error? Do git, Node (`NODE_EXTRA_CA_CERTS`) and Python trust a CA scoped to GitHub domains?

Record versions and outcomes in the answer.
