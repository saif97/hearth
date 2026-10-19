# Market survey: agent sandboxes on your own hardware (Oct 2026)

**Verdict:** no product combines a polished UI, real per-agent isolation (container/microVM), and running on a machine you own reachable from laptop/phone. The gap is real but narrow, and commercially weak.

## Three camps

1. **Orchestration UIs**: great UX and remote access, but isolation is only git worktrees. Conductor, Superset, Nimbalyst, Claude Desktop, Codex app, Happier.
2. **Sandbox runtimes**: strong isolation, egress, credential proxies, but CLI/SDK only. Docker `sbx`, microsandbox, NVIDIA OpenShell, code-on-incus, E2B Embed.
3. **Heavy platforms**: do both, but team-shaped, ops-heavy, enterprise-priced or winding down. Coder, Helix, OpenHands, Daytona/Nightona.

## Key products

| Product | Own box? | Isolation | UI | Remote | Egress/secrets | Licence | Note |
|---|---|---|---|---|---|---|---|
| Docker `sbx` | Yes (Linux KVM/mac/win) | microVM, private Docker daemon | CLI/TUI | SSH; v0.48 registers with Codex & Claude Desktop | Host proxy policies, proxied creds, MCP gateway | Closed; local free | Fast-moving, breaking CLI |
| Superset | Yes (daemon on any host) | Worktrees only | Polished desktop + iOS | Relay/SSH (Pro) | None | ELv2, Pro $20/seat | Closest on remote model |
| Sculptor | Yes (Mac/Linux) | Docker container per agent | Polished desktop | None | n/a | Closed beta | |
| Conductor | Mac only locally | Worktrees / Firecracker in their cloud | Excellent | Their cloud only | n/a | $50/mo Pro | |
| Claude Code Remote Control | Yes | Worktrees + bubblewrap | claude.ai, mobile, Desktop | First-party | Sandbox egress allowlist | Pro/Max+ | 32 sessions |
| Codex Remote | Yes | bubblewrap / Seatbelt | ChatGPT desktop/mobile | E2E relay, QR pairing | Codex sandbox modes | ChatGPT plans | GA Jun 2026 |
| Coder | Yes | Template (container/VM) | Web + IDE | Yes | Agent Firewall is Premium | AGPL core | Heavy; Tasks being removed |
| Helix | Yes | Streamed desktop per agent | Pair live, kanban | Browser | Per-agent net + creds | $199/yr Linux | Heavy |
| OpenHands | Yes | Docker per conversation | Web GUI | Fiddly | Limited | MIT | |
| microsandbox | Yes | libkrun microVM | CLI/SDK/MCP | No | Placeholder creds | Apache-2.0 | Pre-1.0 |
| NVIDIA OpenShell | Yes | Containers + kernel controls | CLI/SDK | Gateway | Strongest policy engine | Apache-2.0 | 0.1.x |
| code-on-incus | Yes (Incus) | Unprivileged Incus container | CLI + monitor | No | restricted/allowlist/open; nftables exfil detection | MIT | v0.10, matches Saif's stack |
| caic, sandboxed.sh, Happier | Yes | Docker / worktrees | Web/mobile | Tailscale/relay | Minimal | OSS | Hobby-grade |

Vibe Kanban: shut down Apr 2026 (no business model). Daytona: closed-source, archived self-host Oct 2026.

## What a new product should do differently

- Don't compete with agent UIs; expose each sandbox as an SSH host / ACP endpoint so existing agent apps drive it.
- Own what nobody does well: sandbox lifecycle across hosts (snapshots, branching, warm pools, sleep), per-sandbox egress policy + credential brokering visible in UI, Tailscale-native preview URLs, GPU passthrough, phone approvals.
- Single binary on any Linux box; backend-agnostic (Incus, Podman, `sbx`, libkrun).

## Sources

- https://docs.docker.com/ai/sandboxes · https://docs.docker.com/ai/sandboxes/release-notes.md
- https://docs.superset.sh/remote-access
- https://imbue.com/sculptor · https://www.conductor.build/pricing
- https://code.claude.com/docs/en/remote-control · https://learn.chatgpt.com/docs/remote-connections
- https://coder.com/docs/ai-coder/agent-firewall.md · https://www.helix.ml
- https://docs.openhands.dev/openhands/usage/sandboxes/docker.md
- https://github.com/NVIDIA/OpenShell · https://raw.githubusercontent.com/mensfeld/code-on-incus/master/README.md
- https://vibekanban.com/blog/shutdown · https://www.agent37.com/blog/daytona-alternative
- https://github.com/fishman/awesome-agent-sandbox
