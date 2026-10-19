# How does the host proxy inject the repo-grant token on Incus?

Type: research
Status: open
Blocked by:

## Question

Incus has no built-in credential proxy (unlike `sbx` and microsandbox). How should the control plane's own host proxy inject the repo-grant token into git-over-HTTPS and GitHub API requests so the real token never enters the sandbox? Cover TLS interception for github.com only (per-sandbox CA trusted inside the guest) vs alternatives (git credential helper over a vsock/socket to the host, an SSH-based git endpoint the host fronts), how the push/merge filter from ticket 04 hooks in, and which existing proxies (mitmproxy, Envoy, OpenShell, Smokescreen-style) fit.
