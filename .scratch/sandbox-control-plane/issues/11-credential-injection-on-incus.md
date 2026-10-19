# How does the host proxy inject the repo-grant token on Incus?

Type: research
Status: resolved
Blocked by:

## Question

Incus has no built-in credential proxy (unlike `sbx` and microsandbox). How should the control plane's own host proxy inject the repo-grant token into git-over-HTTPS and GitHub API requests so the real token never enters the sandbox? Cover TLS interception for github.com only (per-sandbox CA trusted inside the guest) vs alternatives (git credential helper over a vsock/socket to the host, an SSH-based git endpoint the host fronts), how the push/merge filter from ticket 04 hooks in, and which existing proxies (mitmproxy, Envoy, OpenShell, Smokescreen-style) fit.

## Answer

Extend the ticket 03 host egress proxy to terminate TLS only for `github.com` and `api.github.com`. Use a per-sandbox CA installed by cloud-init. The guest's `git` (via a credential helper) and `gh` (via `GH_TOKEN`) send a per-sandbox placeholder. The proxy runs the repo-grant filter on the decrypted request: `git-receive-pack` ref commands are read before the streamed pack, plus a REST route allowlist and GraphQL mutation checks. It then swaps the placeholder for the installation token in the Basic or Bearer header.
A credential helper over vsock puts the real token in the VM, so reject it. An SSH git front-end covers git only, so it's not for v1. Build the proxy in-process in Go (goproxy or `crypto/tls`). Spike it with mitmproxy; Envoy and OpenShell are references, not dependencies.
Findings: [research/credential-injection-on-incus.md](../../../research/credential-injection-on-incus.md)
