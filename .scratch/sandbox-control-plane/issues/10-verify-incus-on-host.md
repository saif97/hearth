# Verify Incus capabilities on the host

Type: task
Status: open
Blocked by:

## Question

On Saif's PC, confirm the facts the backend choice hangs on (HITL checklist):
- Installed Incus, QEMU (need 11+), virglrenderer (need 1.1+) and kernel (need 6.13+) versions.
- Launch a test VM; change `limits.cpu` and `limits.memory` while running. Do new vCPUs come online by themselves in the stock image?
- Grow the root disk of the running VM on the current storage pool. Does it apply live, and does the guest filesystem grow?
- Change an Incus network ACL on the running VM. Does it apply immediately?
- Try a `native-context` GPU device on the RX 9060 XT. Does it attach, and does ROCm see it inside the VM?

Record versions and outcomes in the answer.
