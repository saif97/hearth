# Map: Sandbox Control Plane

Label: wayfinder:map

## Destination

A build / don't-build decision, and if "build", a v1 spec ready to hand off for an open-source, self-hosted **control plane**: a daemon on any **host** you own plus a web/phone UI to create, resize and govern agent **sandboxes**.

## Notes

- Domain: self-hosted agent infrastructure. Glossary: `CONTEXT.md` (Host, Sandbox, Repo Grant, Resource Profile, Egress Policy, Egress Request, Control Plane, Client, Agent App).
- Skills every session should consult: grilling, domain-modeling. Research tickets: research.
- Tracker: local markdown (`.scratch/sandbox-control-plane/`).
- Saif is user #1; open-source tool, not a venture.
- Settled constraints (round 1–2 grilling):
  - Product manages sandboxes only; agents are driven by existing agent apps over SSH/ACP.
  - Sandbox = long-lived, named, bound to one repo; repo grant limited to branches + PRs.
  - VM-level isolation; egress allowlist; secrets via proxy, never mounted.
  - Resource profile (CPU/RAM/disk/GPU) set from UI and **changeable without recreating** the sandbox.
  - Egress: static policy editable from phone **and** live egress requests (allow/deny).
  - Remote-first over Tailscale; tailnet identity is the auth for v1; PWA for phone.
  - One host in v1; data model must allow more hosts later.
- Docker Sandboxes pain points (the wedge): no UI for sizing/creating sandboxes; creating many is a hassle; egress allow/deny only from shell; resize requires recreating.
- Market context: `research/market-survey.md` (Oct 2026). Gap is real but narrow; don't build an agent UI.
- Standing preference: concise answers.

## Decisions so far

<!-- one line per closed ticket -->

- [Docker Sandboxes: are the deal-breakers fixed or on the roadmap?](issues/01-docker-sandboxes-limits-and-roadmap.md): No. Still no live resize, no public local API or bulk create, egress approval exists but CLI-only and deny-first, no published roadmap; `sbx` is at most an optional, version-pinned adapter. Detail: [findings](../../research/docker-sandboxes-roadmap.md)

## Not yet specified

- **How agent apps connect into a sandbox**: SSH host vs ACP endpoint vs both; how Claude Desktop / Codex discover sandboxes; how the agent's own API key reaches it (probably via the same credential proxy).
- **Snapshot / clone UX**: what snapshotting a sandbox means for its repo grant and egress policy.
- **Preview URLs**: exposing a dev server inside a sandbox to the tailnet.
- **GPU passthrough**: whether v1 sandboxes can use the host's GPU for local models.
- **Host lifecycle**: idle sleep / wake, what happens to sandboxes on host reboot.
- **Audit log**: what the owner can see of a sandbox's egress and repo activity after the fact.
- **Packaging**: single binary vs compose; Linux-only vs also macOS hosts.
- **v1 spec itself**: assembled once the decisions above land.

## Out of scope

- Agent session UI (transcripts, diffs, agent-action approvals): existing agent apps own this.
- Teams / multi-user / shared host fleets.
- Commercial product or hosted offering.
- Disposable per-task sandboxes (later effort).
