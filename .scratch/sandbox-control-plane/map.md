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
- [How can egress requests be held for live approval?](issues/03-live-egress-requests.md): Host-side proxy keyed on SNI/Host/CONNECT (DNS-map fallback), forced DNS + nftables default-drop; hold ~10–25 s then fail closed, approval applies on retry (OpenShell `/wait` pattern for slow phone taps). Accepted gaps: domain fronting, ECH, shared CDN IPs. Detail: [findings](../../research/live-egress-requests.md)
- [Which isolation backends meet the sandbox requirements?](issues/02-backend-candidates.md): Only Incus VMs meet all of them today (live CPU/RAM hotplug, live root-disk grow, shared AMD GPU via `native-context`, snapshot/copy, REST API, already on host); microsandbox runner-up (beta, no GPU). Egress on Incus is IP-only, so domain policy needs the host proxy. Detail: [findings](../../research/backend-candidates.md)
- [How to scope a sandbox to one repo with branch + PR rights only?](issues/04-repo-grant-credentials.md): Per-owner GitHub App, 1-hour installation token per sandbox narrowed to one repo (Metadata read, Contents + PRs write), injected by a proxy, re-minted ~55 min, revoked on delete. Token can't block default-branch push or self-merge, so back it with a ruleset + a control-plane filter on pushes/merges (the only guard on Free private repos). Detail: [findings](../../research/repo-grant-credentials.md)
- [How does the host proxy inject the repo-grant token on Incus?](issues/11-credential-injection-on-incus.md): The egress proxy decrypts only github.com / api.github.com via a per-sandbox CA; guest tools send a placeholder; proxy runs the push/merge filter (receive-pack refs, REST allowlist, GraphQL mutations) then swaps in the real token. vsock helper rejected (token enters VM); SSH git endpoint deferred. Daemon language not yet decided. Detail: [findings](../../research/credential-injection-on-incus.md)

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
