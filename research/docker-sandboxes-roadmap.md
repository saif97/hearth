# Docker Sandboxes (`sbx`): deal-breakers, fixes and roadmap (Oct 2026)

Ticket: `.scratch/sandbox-control-plane/issues/01-docker-sandboxes-limits-and-roadmap.md`. Researched 2026-10-19 against `sbx` v0.48.0 (released 2026-10-13).

**Verdict:** Of the four pain points, only live egress requests have moved: v0.48.0 surfaces them, but only through the CLI. Resizing without recreating is not supported. There is no public local API and no bulk create. Docker publishes no roadmap that names any of these. A third-party control plane can drive `sbx` today only through its CLI's JSON output, which broke a week ago, or through an undocumented local daemon socket.

## 1. Can CPU/RAM/disk change without recreating the sandbox?

**No.**

- CPU and memory are set only when the sandbox is created: `sbx create` / `sbx run` take `--cpus` (default `0` = all host CPUs, capped at 16 on Linux arm64) and `-m, --memory` (default 50% of host memory, clamped to 512 MiB–32 GiB). The create page has no disk flag and no note about changing resources later. https://docs.docker.com/reference/cli/sbx/create/
- The CLI has no `update`, `resize` or `set` verb for a sandbox. Its top-level commands are attach, cp, create, daemon, diagnose, env, exec, kit, login, logout, ls, mcp, move, policy, ports, prune, reset, rm, run, secret, settings, setup, skills, stop, template, ttl, tui, version and volume. https://docs.docker.com/reference/cli/sbx
- Disk has no flag either. A maintainer pointed users to the env vars `DOCKER_SANDBOXES_ROOT_SIZE`, `DOCKER_SANDBOXES_CLONED_WORKSPACE_SIZE` and `DOCKER_SANDBOXES_DOCKER_SIZE` (issue #426). An open bug (#676, 2026-10-07) reports "no space left on device" with an EROFS `default_size: 20G` and has no maintainer answer on how to grow it. https://github.com/docker/sbx-releases/issues/426 · https://github.com/docker/sbx-releases/issues/676
- The cloud API is no better. In the Cloud Sandboxes API, `PATCH /v1/sandboxes/{sandbox}` "changes only the display label". Cloud sizes are fixed shapes, micro through xl (1 CPU/2 GiB to 16 CPUs/32 GiB). https://docs.docker.com/reference/api/sandboxes/latest/ · https://docs.docker.com/ai/sandboxes-api/limits/
- Other settings already change live. An issue opened 2026-10-07 lists mounts (`sbx mount`), ports, network rules and secrets as changeable "without recreating it", and asks for skills mode to join them. Resources are not on that list. https://github.com/docker/sbx-releases/issues/680
- History: memory was hard-capped at about 4 GB in Feb 2026 (#36, #56, both closed). `--cpus` arrived in v0.25.0 (#16), and v0.43.0 lowered the minimum memory to 512 MiB. https://github.com/docker/sbx-releases/issues/36 · https://github.com/docker/sbx-releases/issues/56 · https://github.com/docker/sbx-releases/issues/16 · https://github.com/docker/sbx-releases/releases/tag/v0.43.0
- The only path today is snapshot and recreate: `sbx template save` a snapshot, then create from it with new `--cpus`/`--memory`. v0.48.0 made creating from a local snapshot skip registry requests "including when using the daemon API". https://docs.docker.com/ai/sandboxes/release-notes/

## 2. Is there an API or control plane beyond the CLI (bulk create, remote egress edits)?

**Locally, not a public one.**

- **Local daemon API: it exists, but it is private.** Release notes mention a "daemon API" (0.48.0) and a "runtime API" (0.46.0), but Docker documents neither. The `sbx daemon` docs give no socket, port or listen address. https://docs.docker.com/ai/sandboxes/release-notes/ · https://docs.docker.com/reference/cli/sbx/daemon/start/
- An unofficial SDK reverse-engineered the API from v0.34.0. It reports REST over a Unix socket at `~/.local/state/sandboxes/sandboxes/sandboxd/sandboxd.sock`, with routes `/sandbox`, `/policy/rules`, `/policy/profiles`, `/network/log`, `/runtime` and others. It calls the API "private, undocumented … can change without notice between `sbx` releases" and says auth was not enforced on the socket. https://github.com/itbm/sbx-sdk
- `sbx setup ssh` says the socket or named pipe "enforces the OS user boundary". https://docs.docker.com/reference/cli/sbx/setup/ssh/
- **Public API is cloud-only.** The Sandboxes API/SDK (connect.docker.com) covers 58 operations for cloud sandboxes only. Its network-policy endpoints are read-only (`GET` policies and policy-logs), and it has no approval endpoints. https://docs.docker.com/ai/sandboxes-api/concepts/ · https://docs.docker.com/reference/api/sandboxes/latest/
- **Org governance is paid.** Docker AI Governance (Docker Home plus the AI Governance API) lets admins push network, filesystem and MCP policy centrally to local sandboxes. When it is active, local allow rules stop being evaluated. It needs "a separate paid subscription" through Docker Sales. https://docs.docker.com/ai/sandboxes/governance/ · https://docs.docker.com/ai/sandboxes/faq/
- **No bulk create.** `sbx env` is declarative, but it is experimental and describes one sandbox per `sbxenv.yaml`. Its docs mention CPU/memory sizing only for cloud. Batch operations exist only for `rm` and `stop`. https://docs.docker.com/reference/cli/sbx/env/ · https://docs.docker.com/ai/sandboxes/release-notes/
- **Egress edits are CLI-only, and they apply immediately.** `sbx policy allow|deny|rm network`, global or `--sandbox <name>`, take effect without a restart. https://docs.docker.com/ai/sandboxes/security/policy/ · https://docs.docker.com/ai/sandboxes/governance/access-controls/local/

## 3. Can egress requests be surfaced live for approval?

**Partly, as of v0.48.0, and CLI-only.**

- **How it works:** Under the Balanced and Locked Down presets, and with no org governance, "a request with no matching allow or deny rule also asks for approval rather than being denied outright." https://docs.docker.com/ai/sandboxes/governance/access-controls/network/
- **The request is not held:** "The request that triggers the prompt doesn't wait for an answer. It's denied, and approving the destination affects later requests." Agents that retry pick up the new access. https://docs.docker.com/ai/sandboxes/governance/access-controls/network/
- **What the sandbox sees:** an HTTP 403 with the body `Approval required for example.com:443. Review and respond with: sbx policy approval ls`. A third-party tool observed this on v0.48.0 (2026-10-16). On v0.45.1 the same request got a plain "Blocked by network policy". https://github.com/swat9013/sbxr/issues/98
- **How the owner responds:** `sbx policy approval ls | inspect <id> | respond <id> --option allow|dismiss`. There is no allow-once vs allow-always choice, and no documented TUI, `sbx events`, notification or API surface. https://docs.docker.com/ai/sandboxes/governance/access-controls/network/
- **Docs lag:** the `sbx policy` CLI reference does not list an `approval` subcommand yet, and the v0.48.0 release notes don't mention the feature. https://docs.docker.com/reference/cli/sbx/policy/ · https://github.com/docker/sbx-releases/releases/tag/v0.48.0
- **Scripts can't approve:** confirmations need a TTY, and piped answers no longer confirm as of v0.48.0. A related request for programmatic approval of kit credential bindings is open (#519). https://docs.docker.com/ai/sandboxes/workflows/automation/ · https://github.com/docker/sbx-releases/issues/519

## 4. Roadmap, changelog, issue tracker: is any of this being fixed, and when?

- **No public roadmap.** The docs send feedback to GitHub issues on `docker/sbx-releases`. That repo holds only a README, a proprietary LICENSE ("All rights reserved") and the release binaries; the source is closed. https://docs.docker.com/ai/sandboxes · https://github.com/docker/sbx-releases
- **Cadence:** roughly weekly (0.46.0 on 09-28, 0.47.0 on 10-05, 0.48.0 on 10-13), with RC and nightly channels. None of the 0.46–0.48 notes mention resize, a public local API or bulk create. https://docs.docker.com/ai/sandboxes/release-notes/
- **The only forward-looking statement** is in the Cloud Sandboxes launch post (2026-09-24): "Centralized governance is coming soon for enterprises through Docker AI Governance." That is aimed at enterprises, not at a self-hosted owner. https://www.docker.com/blog/introducing-cloud-sandboxes-start-on-your-laptop-finish-in-the-cloud/
- **No timeline found** for resize, a public local API or bulk create. No open issue requesting live resize turned up (see Unknowns).

## 5. Can a third-party control plane drive `sbx` through a stable interface?

**Not a stable one.**

- **The supported surface is the CLI's structured output.** Docker's automation guidance: use `--format json|yaml` with snake_case fields; results go to stdout, errors to stderr; `sbx events --json` is an NDJSON stream; use `--force` to skip confirmations; "human-readable tables … can change between releases." https://docs.docker.com/ai/sandboxes/workflows/automation/
- **That surface just broke.** v0.48.0 "changes flags, structured output, and prompts used by existing scripts": `sbx ls --json` now returns a bare array, several flags were renamed, and confirmations fail without a TTY. https://docs.docker.com/ai/sandboxes/release-notes/
- **Integrators pin versions.** Kubernetes agent-sandbox drives `sbx` by `kubectl exec … sbx create`, pins v0.46.0, and warns "Commands and flags change between releases." https://agent-sandbox.sigs.k8s.io/docs/use-cases/examples/docker-sbx/
- **Key surfaces are still experimental:** `sbx env`, `sbx kit`, `sbx setup` (SSH and agent-app registration) and `sbx skills`. https://docs.docker.com/reference/cli/sbx

## Implication for the control plane

- **Live resize** (a settled constraint) cannot be delivered on `sbx`. At best the control plane can fake it with snapshot → recreate → restore, which loses running processes and costs a restart. That argues for a backend with native live resize (Incus/LXD VMs or containers, libvirt/QEMU) as the primary target, and treating `sbx` as an optional adapter if at all.
- **Live egress requests** are feasible on `sbx`, but only by shelling out to `sbx policy approval ls/respond` and polling. Requests are denied, not held. The approval CLI is undocumented in the reference and may change.
- **Driving `sbx`** means a CLI adapter pinned to an exact version, with contract tests on JSON shapes. The private `sandboxd.sock` is richer but carries even more breakage risk. The tool is closed-source, so we can't fix what breaks.
- **Wedge check:** Docker is converging on cloud and enterprise features (Cloud Sandboxes, paid AI Governance), not a local owner-facing control plane. The "no UI, no resize, CLI-only egress" gap is likely to stay open for self-hosters in the near term, but the backend choice should not depend on `sbx`.

## Unknowns

- **Whether an open feature request for live resize exists, or what Docker's position on it is.** GitHub issue search was not reachable (robots and API restrictions), so issues were found via web search only, which is not exhaustive.
- **Which release first shipped `sbx policy approval`.** Behaviour was confirmed on v0.48.0 by a third party; the v0.48.0 notes and CLI reference don't mention it. It is also unclear whether approvals appear in `sbx events --json` or the TUI.
- **Whether the disk env vars (`DOCKER_SANDBOXES_*_SIZE`) are read only at creation, and whether they are supported or internal.** They are undocumented beyond a maintainer comment.
- **The current shape and auth of the private `sandboxd.sock` API.** Last mapped by a third party at v0.34.0, versus v0.48.0 today.
- **Whether Docker plans a public local API.** No statement found.
