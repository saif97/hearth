# Credential injection on Incus: how the host proxy injects the repo-grant token (Oct 2026)

Ticket: `.scratch/sandbox-control-plane/issues/11-credential-injection-on-incus.md`. Researched 2026-10-19/20 against official docs and source code. Builds on ticket 03 (egress proxy, `research/live-egress-requests.md`), ticket 02 (Incus VMs, `research/backend-candidates.md`) and ticket 04 (GitHub App tokens, `research/repo-grant-credentials.md`).

**Verdict:** extend the ticket 03 egress proxy, which already sits on the host and sees every sandbox TCP connection, so that it **terminates TLS for GitHub hosts only** (`github.com`, `api.github.com`). It uses a **per-sandbox CA** that is installed in the guest at create time. Every other host stays SNI-spliced. Inside the VM, `git` and `gh` are configured to send a **per-sandbox placeholder**. On the host, the proxy runs the **repo-grant filter** on the decrypted request (git ref updates, REST routes, GraphQL mutations), then swaps the placeholder for the real installation token in the Basic or Bearer header and forwards the request. Build it in-process in the Go daemon, using `crypto/tls` plus `net/http` or goproxy, so there is one binary and one policy point. Use mitmproxy only to spike it. A credential helper over vsock fails the "never enters the sandbox" constraint. An SSH git front-end only covers git, so neither alternative replaces MITM; at most the SSH front-end is a later option.

## 1. Options compared

| | A. Scoped TLS interception (MITM GitHub only) | B. Plain-HTTP reverse proxy (tokenizer pattern) | C. Credential helper over vsock/socket | D. Host-fronted SSH git endpoint |
|---|---|---|---|---|
| Real token enters the VM? | **No.** Guest holds only a placeholder | **No** | **Yes.** The helper hands the token to `git`, and `git credential fill` prints it [gitcredentials, git-credential] | **No** |
| Covers `git` over HTTPS | ✅ unchanged `https://github.com/...` remotes | ⚠️ only after rewriting remotes to `http://proxy/...` (`url.<base>.insteadOf`) | ✅ | ⚠️ only after rewriting remotes to SSH |
| Covers `gh` / REST / GraphQL | ✅ | ⚠️ needs `gh` routed to the proxy. `api_host` is experimental and "not a security boundary"; `http_unix_socket` dials a Unix socket that Incus can't forward into a VM [gh config, Incus proxy] | ✅ (`GH_TOKEN` = real token) | ❌ needs A or B anyway |
| Covers arbitrary tools (curl, SDKs, MCP servers) | ✅ if they trust the CA | ❌ per-tool config | ⚠️ only tools that call the helper | ❌ |
| Needs a CA in the guest | **Yes** | No | No | No |
| Where the push/merge filter runs | On the decrypted request in the proxy | Same | **Nowhere.** Traffic goes straight to GitHub, so the filter needs MITM anyway | git: a `pre-receive`-style check on the host. API: needs A/B |
| Incus support needed | None beyond ticket 03's fence and bridge | Same | A vsock or socket channel. Incus documents no user vsock ports, and its proxy device is NAT-only (TCP/UDP) for VMs [Incus proxy, Incus ext] | A host port reachable from the bridge |
| Verdict | **Recommended** | Fallback for CA-hostile tools only | **Reject**: breaks a settled constraint | Possible later add-on, not v1 |

### A. Scoped TLS interception (recommended)

- **Prior art does exactly this.** Docker `sbx` (a host proxy rewrites the auth header, and the VM sees `proxy-managed`), microsandbox (a host-bound secret, with Basic auth decoded and re-encoded, and a CA added to the guest trust store) and NVIDIA OpenShell all inject this way (see ticket 04 §3.2). OpenShell: "At startup, OpenShell replaces each credential with an opaque placeholder token in the agent's environment… the proxy resolves it immediately before forwarding the request". It resolves placeholders in header values, in Basic auth ("The proxy decodes, resolves, and re-encodes"), in query and path, and in opt-in bodies. Its GitHub profile resolves `GITHUB_TOKEN`/`GH_TOKEN` only for `api.github.com:443` and `github.com:443`, and returns `credential_endpoint_mismatch` for any other host [OpenShell providers].
- **Scope of interception.** Only hosts that carry the grant: `github.com` (git smart-HTTP) and `api.github.com` (REST and GraphQL). All other egress keeps ticket 03's SNI/CONNECT splice with no decryption. mitmproxy expresses the same split as `allow_hosts`/`ignore_hosts` [mitmproxy options]. OpenShell endpoints terminate TLS "unless the endpoint sets `tls: skip`" [OpenShell network rules].
- **Wire formats the proxy rewrites** (ticket 04 §3.1): git sends HTTP Basic with username `x-access-token` and the token as the password. `gh` and the REST API send `Authorization: Bearer <token>`. Installation tokens can be about 520 characters (stateless format), so nothing should assume a 40-character length.
- **CA in the guest** (an Incus VM boots a cloud image, so cloud-init does this at create):
  - cloud-init `ca_certs.trusted` "adds CA certificates to the system's CA store" on Debian, Ubuntu, Fedora, Alpine, RHEL and others [cloud-init modules]. Go reads the system bundle (`/etc/ssl/certs/ca-certificates.crt` and others) [Go root_linux.go]. git uses libcurl and the system store, and `http.sslCAInfo`/`GIT_SSL_CAINFO` can override it [git http config].
  - **Node does not use the system store by default.** Set `NODE_EXTRA_CA_CERTS` (read only at process start) or `--use-system-ca` (Linux support since v23.9.0) [Node CLI]. Python `requests` uses its own bundle unless `REQUESTS_CA_BUNDLE` (fallback `CURL_CA_BUNDLE`) is set [requests advanced]. Set both in `/etc/environment` so every agent and MCP server inherits them.
  - **Per-sandbox CA:** generate it at sandbox create, keep the private key only in control-plane state, and delete it with the sandbox. OpenShell uses a per-sandbox ephemeral CA for the same reason (ticket 03 findings). Optional hardening is an X.509 name-constraints extension limiting the CA to the GitHub hosts. Go enforces RFC 5280 name constraints "in the intermediates" [Go x509 verify.go], so it would have to be a constrained intermediate under a host root. Whether every client also honours constraints on a root is unverified (see Unknowns).
- **Guest-side placeholder wiring**, written by cloud-init:
  - `GH_TOKEN=<placeholder>`. `gh` uses `GH_TOKEN` for `github.com` and overrides stored credentials [gh environment].
  - A system git credential helper scoped to GitHub, for example `credential.https://github.com.helper = "!f(){ test \"$1\" = get && printf 'username=x-access-token\npassword=%s\n' \"$GH_TOKEN\"; }; f"`. A `!` helper is a shell snippet [gitcredentials]. The other route is `gh auth setup-git`, which is what Docker's `sbx-kit-github-clone` does (ticket 04 [22]).
  - The placeholder should be random and per sandbox, so the proxy can check that a request's placeholder matches the sandbox it came from.
- **No proxy env vars needed:** ticket 03's fence already REDIRECTs or TPROXYs all guest TCP to the egress proxy. mitmproxy's transparent mode is the same gateway setup: "Set the test device up to use the host on which mitmproxy is running as the default gateway" plus the `REDIRECT` rules [mitmproxy transparent].
- **Sandbox identity:** the proxy maps the source IP to a sandbox. Each sandbox has its own Incus NIC, and `security.ipv4_filtering`/`security.mac_filtering` "Prevent the instance from spoofing another instance's IPv4 address" [Incus NIC]. The docs don't say explicitly that these apply to VMs (see Unknowns).
- **Costs and risks:** certificate-pinning clients fail. GitHub CLIs aren't known to pin, but any that do need `tls: skip`-style bypass and therefore get no token. Every intercepted byte of a push flows through the daemon, so it must stream (§2.1). HTTP/2 to the client can be avoided by advertising only `http/1.1` in ALPN on the intercepted leg (an implementation choice, untested).

### B. Plain-HTTP reverse proxy (tokenizer pattern)

- Fly.io's **tokenizer** (Apache-2.0) is "an HTTP proxy that injects third party authentication credentials into requests". Clients "speak plain HTTP to the proxy server, not HTTPS", and "the proxy transparently switches to HTTPS for connections to upstream services". It assumes the client-to-proxy leg is a secure transport such as a VPN. Secrets are sealed to the proxy's key, carried in a `Proxy-Tokenizer` header and limited by `allowed_hosts`. It even has a `github_app_processor` that returns a sealed installation token [tokenizer].
- **Fit:** it avoids a guest CA, and on a single host the guest-to-host leg is a local bridge. But every tool has to be pointed at `http://` URLs. git can do this with `url.<base>.insteadOf` and `http.extraHeader` [git http config]. `gh`, curl scripts and SDKs that hard-code `https://api.github.com` cannot, and the fence would then drop them (fail closed, but broken). Tokenizer also does no ref or route filtering. **Keep it as a fallback** for one tool that refuses a custom CA, not as the main path.

### C. Credential helper over vsock or a socket to the host

- git's helper protocol returns `username`/`password`, or, with the `authtype` capability, a pre-encoded `credential` that git turns into `Authorization: <authtype> <credential>`. Either way the helper's output is the **real** token inside the guest's `git` process [git-credential]. Anything in the VM can run `git credential fill` and read it. That breaks the map's settled "secrets via proxy, never mounted" constraint. The 1-hour token lifetime limits the damage but doesn't meet the constraint.
- Plumbing on Incus is awkward too. The VM proxy device is "NAT mode only" with `tcp <-> tcp` / `udp <-> udp`, so it can't put a host Unix socket in a VM [Incus proxy]. vsock exists (`VMADDR_CID_HOST` = 2 [vsock(7)]), and Incus uses it for its own agent (`vsock_api`, `volatile.vsock_id` [Incus ext, Incus config]), but no user port forwarding is documented. `/dev/incus/sock` is Incus's own fixed guest API [Incus dev-incus].
- It also gives the push/merge filter nothing to hook into, because traffic goes straight to GitHub. **Reject.**

### D. SSH git endpoint the host fronts

- The host runs an SSH server per sandbox key. The guest remote becomes `ssh://host/...`. The host side runs `git-receive-pack`/`git-upload-pack` against a local mirror and relays to GitHub over HTTPS with the token, or bridges the protocol directly. Ref checks become natural: a `pre-receive` hook gets `<old-oid> SP <new-oid> SP <ref-name>` per ref, and a non-zero exit rejects the whole push [githooks].
- It covers git only. `gh`, the REST API and GraphQL (PRs, merges) still need A or B, so it adds a second mechanism rather than replacing MITM. It also changes the remote URL the agent sees, needs a mirror or protocol bridge, and SSH has no SNI for the egress proxy. **Not v1.** Revisit only if MITM of `github.com` proves fragile for git (for example, if push streaming can't be made reliable).

## 2. How the push/merge filter hooks in

Order inside the proxy, for a request on an intercepted GitHub host:

1. **Identify the sandbox** from the source IP and check that the placeholder matches. Any client-supplied credential that *isn't* this sandbox's placeholder is stripped and the request is denied. Never forward a credential the guest supplied.
2. **Scope the path to the grant:** `github.com/{owner}/{repo}.git/...` and `api.github.com/repos/{owner}/{repo}/...` only, plus `/graphql`. The token is already repo-scoped (ticket 04), so this is defence in depth and gives better error messages.
3. **Run the grant filter** (below) on the decrypted request.
4. **Inject:** replace the placeholder in `Authorization` (Basic: decode, swap, re-encode; Bearer: swap) with the current installation token from the control plane's 55-minute minting cache (ticket 04 §4).
5. **Forward and log** to the audit log (method, path, refs, verdict).

Because the control plane owns this proxy, the filter always sees the request before injection and with placeholders only. That removes the ordering question ticket 04 raised about chaining to `sbx`'s upstream proxy.

### 2.1 git over smart HTTP

- Push = `GET /{o}/{r}.git/info/refs?service=git-receive-pack`, then `POST /{o}/{r}.git/git-receive-pack` with body `command_list` (`old_id SP new_id SP name`, capabilities after a NUL on the first line, then a flush), optional push-options, then `PACK…` [http-protocol, gitprotocol-pack]. Fetch = `info/refs?service=git-upload-pack` + `POST git-upload-pack`, which OpenShell's push tutorial also allow-lists by path [OpenShell push].
- **Check each command:** allow create or update only for refs matching the grant (for example, not the default branch, or only `refs/heads/agent/<sandbox>/**`). Deny delete (`new_id` all zeros) outside that prefix, and deny `refs/tags/*` unless granted. If the command list carries a `push-cert`, the commands live inside the cert [gitprotocol-pack]. Simplest is to reject push-cert pushes, since agents won't sign.
- **Body handling:** git never gzips push bodies. `push_git()` leaves `gzip_request` at 0, and only fetch and v2 `stateless_connect` set it. Large requests switch to chunked transfer, sized by `http.postBuffer` [git remote-curl.c]. So the proxy reads pkt-lines up to the first flush (a few hundred bytes), decides, then **streams** the packfile through untouched. Don't buffer whole pushes. In mitmproxy this is `flow.request.stream = <callable>` set in `requestheaders` ("a transformation function… called for each chunk") [mitmproxy http.py]. In Go it's a wrapping `io.Reader`.
- **Rejecting a push:** the simplest correct answer is HTTP 403, which git reports as an error. A friendlier answer is a synthesised `application/x-git-receive-pack-result` with `unpack ok` and `ng <ref> denied by repo grant` [gitprotocol-pack]. If `side-band-64k` was negotiated, `receive-pack` sends the report on band 1 (`send_sideband(1, 1, …)` in `builtin/receive-pack.c`) [git receive-pack.c]. Whether git accepts a reply sent before it finishes uploading the pack is untested (see Unknowns).
- Protocol v2 has no push command (it defines `ls-refs`, `fetch`, `object-info` and `bundle-uri`) [gitprotocol-v2], so receive-pack parsing stays v0/v1.

### 2.2 REST (`api.github.com`)

Use an **allowlist**, not a denylist, because `contents:write` reaches many write paths (ticket 04 §2):
- **Allow:** `GET`/`HEAD` under `/repos/{o}/{r}/**`. `POST /repos/{o}/{r}/pulls`, `PATCH /repos/{o}/{r}/pulls/{n}`, PR, issue and review comment endpoints. Checks, statuses and Actions reads.
- **Deny** (each is reachable with Contents write):
  - `PUT /repos/{o}/{r}/pulls/{n}/merge`, `POST /repos/{o}/{r}/merges`, `POST /repos/{o}/{r}/merge-upstream`, `POST /repos/{o}/{r}/branches/{b}/rename` [GitHub branches].
  - `PUT`/`DELETE /repos/{o}/{r}/contents/{path}`, whose `branch` parameter defaults to the default branch [GitHub contents]. Allow it only when the JSON body names a granted branch.
  - `POST`/`PATCH`/`DELETE /repos/{o}/{r}/git/refs*`. Allow it only for granted ref names.
- OpenShell's `rest` rules are a good model for the matcher. `*` matches one segment and `**` matches across segments. "A matching deny rule takes precedence over any allow". `access: read-only` maps to `GET`, `HEAD` and `OPTIONS` [OpenShell network rules].

### 2.3 GraphQL (`POST api.github.com/graphql`)

- "A REST rule sees every GraphQL request as `POST /graphql`" [OpenShell network rules], so parse the operation. Allow `query`. Allow only listed mutation fields: `createPullRequest`, `updatePullRequest`, review and comment mutations.
- Deny `mergePullRequest`, `enablePullRequestAutoMerge` and `enqueuePullRequest` [GitHub GraphQL pulls]. Deny `createRef`, `updateRef`, `updateRefs` and `deleteRef` [GitHub GraphQL git], and `createCommitOnBranch` ("moves the branch to point at it") [GitHub GraphQL commits], unless their ref or branch argument is granted. As in OpenShell, "every top-level field in an operation must match", and a disallowed field denies the whole batch.

Any denial returns a readable JSON 403 (`{"message":"blocked by sandbox repo grant: merge"}`) and an audit entry. It's the same UX contract as ticket 03's egress denials.

## 3. Which existing proxies fit

| Proxy | TLS MITM | Credential injection | Body/ref filtering | Fit |
|---|---|---|---|---|
| **In-process Go** (`crypto/tls` + `net/http`, or **goproxy** BSD-3 with `MitmConnect` and a custom CA via `TLSConfigFromCA` [goproxy, goproxy https.go]) | ✅ | write it (about 50 lines for Basic and Bearer) | write it (pkt-line parser, route table, GraphQL parser) | **v1 choice.** One binary with the egress proxy (ticket 03), token minting (ticket 04) and the audit log. No IPC on the hot path. goproxy has no transparent mode [goproxy], so the transparent listener plus SNI peek is our own code either way |
| **mitmproxy** (MIT, Python, v12.2.3 [mitmproxy releases]) | ✅ regular, transparent and WireGuard modes [mitmproxy modes] | addon | addon. Chunk-callable streaming for push bodies [mitmproxy http.py] | **Spike tool.** Fastest way to prove the git and `gh` flows end to end. A Python sidecar is the wrong shape for a single-binary daemon |
| **Envoy** | ✅ | `credential_injector` (generic or OAuth2), but "functional but has not had substantial production burn time" and "unknown security posture" [Envoy credential_injector] | needs an `ext_proc`/`ext_authz` gRPC service [Envoy ext_proc, ext_authz] | **No.** Too heavy for a single-host tool, and the interesting logic would sit in our gRPC service anyway |
| **NVIDIA OpenShell** (Apache-2.0, v0.0.116, 28 Aug 2026 [OpenShell releases]) | ✅ per endpoint | ✅ placeholders incl. Basic auth [OpenShell providers] | REST and GraphQL rules. gRPC middleware for bodies, but "The middleware API is still evolving" [OpenShell middleware]. Its push tutorial filters by **path only**, no ref check [OpenShell push] | **Design reference, not a dependency.** Its supervisor runs under its own drivers ("A process on the host" for VM) [OpenShell architecture], and fronting an externally managed Incus VM isn't documented |
| **Smokescreen** (Stripe) | ❌ "a HTTP CONNECT proxy" with hostname ACLs | ❌ | ❌ | Prior art for ticket 03's CONNECT ACLs only [Smokescreen] |
| **tokenizer** (Fly.io, Apache-2.0) | ❌ (plain HTTP in, HTTPS out) | ✅ sealed secrets, `github_app_processor` | ❌ | Pattern for option B only [tokenizer] |

## 4. Recommended approach

1. **At sandbox create:** the control plane generates a per-sandbox CA (key kept on the host), a random placeholder and a source-IP binding. cloud-init then installs the CA via `ca_certs`. It writes `/etc/environment` with `GH_TOKEN=<placeholder>`, `NODE_EXTRA_CA_CERTS=<ca>`, `REQUESTS_CA_BUNDLE=/etc/ssl/certs/ca-certificates.crt` and `SSL_CERT_FILE`, and adds the GitHub-scoped git credential helper that echoes the placeholder. Remotes stay `https://github.com/...`.
2. **Proxy (in the Go daemon, same listener as ticket 03):** SNI peek. On `github.com` or `api.github.com`, terminate TLS with a leaf certificate minted from that sandbox's CA. Everywhere else, splice as before. On intercepted requests: check identity and placeholder → path scope → grant filter (§2) → inject the token → forward → log.
3. **Token lifecycle** (ticket 04): mint per sandbox, cache for about 55 minutes, revoke on delete, and drop the CA and placeholder with the sandbox.
4. **Spike first, with mitmproxy** on Saif's host against an Incus VM, alongside ticket 10. Cover clone, fetch, push to an allowed branch, push to `main` (expect a denial), `gh pr create`, `gh pr merge` (expect a denial) and a 100 MB push. That measures the streaming and rejection behaviour before writing the Go version.
5. **Later, optional:** option B for one specific CA-hostile tool. Option D only if git-over-MITM proves unreliable.

## 5. Unknowns

- **Early push rejection:** whether git cleanly shows a synthesised `ng` report-status (or a 403) when the proxy answers before the packfile upload finishes, or whether it reports a broken pipe. The proxy may need to drain the body first. Needs a test.
- **Name-constrained CA:** whether OpenSSL (git, curl, Python), Node and Go all enforce name constraints when they sit on a *trust anchor* rather than an intermediate. Go's docs mention intermediates [Go x509 verify.go]. Root-in-store enforcement wasn't verified for any library.
- **Incus NIC anti-spoofing on VMs:** `security.ipv4_filtering`/`mac_filtering` are documented for bridged NICs without saying explicitly that they cover VMs [Incus NIC]. Check on the host (ticket 10).
- **Incus user vsock:** no documented way to expose a host vsock port to a VM beyond Incus's own agent. Moot for the recommendation.
- **git LFS and other GitHub hosts:** whether `codeload.github.com`, `uploads.github.com` (release assets) or LFS object storage need the token or interception. LFS batch calls go to `github.com/{o}/{r}.git/info/lfs` (assumed, not verified this session).
- **Agent CLIs' own TLS stacks:** whether Claude Code, Codex and common MCP servers honour `NODE_EXTRA_CA_CERTS` and the system store when they call GitHub directly. Only matters for GitHub calls, since other hosts aren't intercepted.
- **HTTP/2 on the intercepted leg:** whether forcing `http/1.1` via ALPN causes any issue with `gh` (Go) or git (libcurl). It shouldn't, but it's untested.
- **OpenShell standalone:** whether its supervisor or proxy can run against a VM it didn't create. Not documented.

## Sources

- [OpenShell providers] https://docs.nvidia.com/openshell/latest/how-it-works/providers/overview
- [OpenShell network rules] https://docs.nvidia.com/openshell/latest/how-it-works/policies/network-rules
- [OpenShell push] https://docs.nvidia.com/openshell/latest/tutorials/github-push-access
- [OpenShell architecture] https://docs.nvidia.com/openshell/latest/about/architecture
- [OpenShell middleware] https://docs.nvidia.com/openshell/latest/extensibility/supervisor-middleware
- [OpenShell releases] https://github.com/NVIDIA/OpenShell/releases · README/licence: https://github.com/NVIDIA/OpenShell
- [tokenizer] https://github.com/superfly/tokenizer
- [Smokescreen] https://github.com/stripe/smokescreen
- [Envoy credential_injector] https://www.envoyproxy.io/docs/envoy/latest/configuration/http/http_filters/credential_injector_filter · https://www.envoyproxy.io/docs/envoy/latest/api-v3/extensions/filters/http/credential_injector/v3/credential_injector.proto
- [Envoy ext_proc, ext_authz] https://www.envoyproxy.io/docs/envoy/latest/configuration/http/http_filters/ext_proc_filter · https://www.envoyproxy.io/docs/envoy/latest/configuration/http/http_filters/ext_authz_filter
- [mitmproxy options] https://docs.mitmproxy.org/stable/concepts/options/
- [mitmproxy modes] https://docs.mitmproxy.org/stable/concepts/modes/
- [mitmproxy transparent] https://docs.mitmproxy.org/stable/howto/transparent/
- [mitmproxy certificates] https://docs.mitmproxy.org/stable/concepts/certificates/
- [mitmproxy http.py] https://github.com/mitmproxy/mitmproxy/blob/main/mitmproxy/http.py (`Message.stream`)
- [mitmproxy releases] https://github.com/mitmproxy/mitmproxy/releases
- [goproxy] https://github.com/elazarl/goproxy · [goproxy https.go] https://github.com/elazarl/goproxy/blob/master/https.go
- [gitcredentials] https://git-scm.com/docs/gitcredentials
- [git-credential] https://git-scm.com/docs/git-credential (`authtype`, `credential`, `ephemeral`)
- [git http config] https://github.com/git/git/blob/master/Documentation/config/http.adoc (`http.sslCAInfo`, `http.extraHeader`, `http.proxy`)
- [http-protocol] https://git-scm.com/docs/http-protocol
- [gitprotocol-pack] https://git-scm.com/docs/gitprotocol-pack
- [gitprotocol-v2] https://git-scm.com/docs/gitprotocol-v2
- [gitprotocol-capabilities] https://git-scm.com/docs/gitprotocol-capabilities
- [githooks] https://github.com/git/git/blob/master/Documentation/githooks.adoc (`pre-receive`)
- [git remote-curl.c] https://github.com/git/git/blob/master/remote-curl.c (`push_git`, `post_rpc`)
- [git receive-pack.c] https://github.com/git/git/blob/master/builtin/receive-pack.c (`send_sideband(1, 1, …)` for report-status)
- [gh environment] https://cli.github.com/manual/gh_help_environment
- [gh config] https://cli.github.com/manual/gh_config · Unix-socket transport: https://github.com/cli/go-gh/blob/trunk/pkg/api/http_client.go
- [GitHub contents] https://docs.github.com/en/rest/repos/contents
- [GitHub branches] https://docs.github.com/en/rest/branches/branches
- [GitHub GraphQL pulls] https://docs.github.com/en/graphql/reference/pulls
- [GitHub GraphQL git] https://docs.github.com/en/graphql/reference/git
- [GitHub GraphQL commits] https://docs.github.com/en/graphql/reference/commits
- [Incus proxy] https://linuxcontainers.org/incus/docs/main/reference/devices_proxy/
- [Incus NIC] https://linuxcontainers.org/incus/docs/main/reference/devices_nic/
- [Incus config] https://github.com/lxc/incus/blob/main/doc/config_options.txt (`volatile.vsock_id`)
- [Incus ext] https://github.com/lxc/incus/blob/main/doc/api-extensions.md (`vsock_api`)
- [Incus dev-incus] https://github.com/lxc/incus/blob/main/doc/dev-incus.md
- [vsock(7)] https://man7.org/linux/man-pages/man7/vsock.7.html
- [cloud-init modules] https://docs.cloud-init.io/en/latest/reference/modules.html (CA Certificates)
- [Node CLI] https://github.com/nodejs/node/blob/main/doc/api/cli.md (`NODE_EXTRA_CA_CERTS`, `--use-system-ca`)
- [requests advanced] https://github.com/psf/requests/blob/main/docs/user/advanced.rst (`REQUESTS_CA_BUNDLE`)
- [Go root_linux.go] https://github.com/golang/go/blob/master/src/crypto/x509/root_linux.go
- [Go x509 verify.go] https://github.com/golang/go/blob/master/src/crypto/x509/verify.go · https://pkg.go.dev/crypto/x509
