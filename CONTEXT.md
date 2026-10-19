# Sandbox Control Plane

A self-hosted control plane for running coding agents inside isolated sandboxes on hardware you own, governed remotely from a laptop or phone.

## Language

**Host**:
A machine the owner provides (an idle PC or laptop) on which sandboxes run.
_Avoid_: server, node, VPS, box

**Sandbox**:
A long-lived, named, VM-isolated environment bound to exactly one repo, in which coding agents work.
_Avoid_: container, workspace, environment, machine

**Repo Grant**:
The single repository a sandbox may access, and the limited permissions it holds on it (create branches, open PRs).
_Avoid_: repo access, token, credential

**Resource Profile**:
The CPU, RAM, disk and GPU allocation of a sandbox.
_Avoid_: size, spec, plan

**Egress Policy**:
The set of domains a sandbox is allowed or denied to reach.
_Avoid_: firewall, network rules, allowlist

**Egress Request**:
An outbound connection to a domain not yet covered by the egress policy, held until the owner allows or denies it.
_Avoid_: network prompt, approval

**Control Plane**:
The daemon on the host plus the web/phone UI through which the owner creates, resizes and governs sandboxes.
_Avoid_: dashboard, orchestrator, manager

**Client**:
A device the owner uses to reach the control plane (MacBook, phone).
_Avoid_: frontend, remote

**Agent App**:
An existing agent interface (Claude Desktop, Codex, a terminal) that connects into a sandbox to drive an agent; not part of this product.
_Avoid_: agent UI, harness
