# Spike Incus VMs vs microsandbox on the host

Type: task
Status: open
Blocked by:

## Question

On Saif's PC, confirm the facts the backend choice hangs on (HITL checklist):
- Installed Incus, QEMU (need 11+), virglrenderer (need 1.1+) and kernel (need 6.13+) versions.
- Launch a test VM; change `limits.cpu` and `limits.memory` while running. Do new vCPUs come online by themselves in the stock image?
- Grow the root disk of the running VM on the current storage pool. Does it apply live, and does the guest filesystem grow?
- Change an Incus network ACL on the running VM. Does it apply immediately?
- Check that Incus anti-spoofing (`security.mac_filtering`, `security.ipv4_filtering`) applies to the VM NIC.
- Push through a test proxy that rejects mid-upload: does git show a clean error? Do git, Node (`NODE_EXTRA_CA_CERTS`) and Python trust a CA scoped to GitHub domains?
- Repeat the resize test with microsandbox (`msb modify --cpus/--memory/--root-disk` within `max_cpus`/`max_memory`). Can its domain egress rules be changed on a running sandbox? Does its secret proxy work for git over HTTPS and `gh`?

Record versions and outcomes in the answer.
