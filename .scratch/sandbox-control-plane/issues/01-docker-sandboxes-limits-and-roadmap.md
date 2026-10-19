# Docker Sandboxes: are the deal-breakers fixed or on the roadmap?

Type: research
Status: resolved
Blocked by:

## Question

For Docker Sandboxes (`sbx`) as of today: can a sandbox's CPU/RAM/disk be changed without recreating it? Is there an API or control plane (beyond the CLI) to create sandboxes in bulk and edit egress allow/deny remotely? Can egress requests be surfaced live for approval? What does Docker's roadmap, changelog and issue tracker say about fixing these, and on what timeline? Can a third-party control plane drive `sbx` through a stable interface?

## Answer

As of v0.48.0, resizing still means recreating the sandbox: CPU and memory are set only at creation, there is no disk flag, and even the cloud API can't resize. There is no public local API and no bulk create; the documented API is cloud-only, its egress endpoints are read-only, and the local `sandboxd.sock` is undocumented. Live egress approval now exists (`sbx policy approval ls/respond`), but it is CLI-only and the first request is denied, not held. Docker has no public roadmap or timeline for any of these, and its CLI JSON contract broke in 0.48.0, so a third-party control plane could only use `sbx` through a version-pinned CLI adapter.

Findings: [research/docker-sandboxes-roadmap.md](../../../research/docker-sandboxes-roadmap.md)
