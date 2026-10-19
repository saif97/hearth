# microsandbox deep dive (v0.7.8, main @ 59d14ba, 2026-10-19)

Checked against the repo source and its bundled docs (`docs/`), not just the website.

## Images
- Rootfs = any OCI image from any registry (Docker Hub, GHCR, local registry). Layers cached, copy-on-write, pinned on first pull. (`docs/images/overview.mdx`)
- **No build command.** Build with Docker/Podman/Buildah from a Dockerfile, then `docker save img | msb load` or push to a registry. (`docs/cli/image-commands.mdx`)
- Boot-time customisation: bootstrap scripts and patches. (`docs/sandboxes/bootstrap.mdx`)

## Live egress rule edits: **no**
- `msb modify` / `sandbox.modify()` patch covers only cpus, max_cpus, memory, max_memory, root disk, env, labels, ports, workdir, secrets. No network-policy field. (`packages/microsandbox-types/rust/lib/modify.rs`, `docs/sandboxes/tuning.mdx`)
- So the egress policy is fixed at creation.

## Approval hook: **no**, but an upstream proxy slot exists
- One HTTP CONNECT or SOCKS5 outbound proxy per sandbox; microsandbox's own policy is evaluated first, then traffic is tunnelled through it. TLS interception and secret injection still apply. (`docs/networking/outbound-proxy.mdx`)
- The CONNECT target is `IP:port` only, no hostname (`crates/network/lib/proxy/http_connect.rs`), so an upstream proxy must read the domain from the TLS SNI / HTTP Host in the tunnelled bytes.
- Workaround: microsandbox policy = public internet allowed (keeps its private-IP, metadata and DNS-rebinding protections); our proxy upstream does the live allowlist and egress requests.

## Daemon API: **none**
- SDKs (Rust, TS, Python, Go) embed the local runtime via native bindings; local is the default backend, cloud opt-in. (`docs/operations/backends.mdx`, `sdk/*/native`)
- Our control-plane daemon would be the SDK caller, and its API is ours.

## Secret proxy with git / gh
- Substitution in headers is on by default, **including Basic auth**, so git over HTTPS should work, not just `gh`/REST. Docs' example binds a GitHub token to `api.github.com`; `github.com` must be allowed too. (`docs/sandboxes/secrets.mdx`)
- Rotating a secret's value is live; adding a secret or changing its placeholder needs a restart. Fits re-minting a 1-hour token. (`docs/sandboxes/tuning.mdx`)
- No branch-level push filter: ticket 11's push/merge filter still has to live in our upstream proxy, which then needs to see decrypted GitHub traffic (TLS interception order to be tested).

## Live resize
- CPU and memory live within boot-time `max_cpus` / `max_memory`; raising the ceiling needs restart. Root disk grows live (ext4, grow-only). (`docs/sandboxes/tuning.mdx`)

## Not checked
- GitHub issues/roadmap for live network modify (issue search unavailable from this environment).
