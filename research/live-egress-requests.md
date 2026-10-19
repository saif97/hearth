# Live egress requests: holding unknown connections for owner approval (Oct 2026)

Ticket: `.scratch/sandbox-control-plane/issues/03-live-egress-requests.md`

**Gist:** Put the control point on the host, outside the VM. All sandbox TCP goes through a host-side proxy that reads the domain from TLS SNI or the HTTP Host header, and the sandbox can only use the control plane's own DNS resolver. UDP is dropped. An unknown domain creates an **egress request**. The proxy holds the connection only for a short window (about 10–25 s, enough for a fast phone tap). After that it fails closed with a readable error, and the request stays pending. Approval applies to the *retry*, which is the same model Docker `sbx` and NVIDIA OpenShell use. Don't promise to hold a connection until the owner taps: client timeouts make that unreliable.

## 1. Mechanisms compared

| Mechanism | Sees domain? | Can hold a connection? | Bypassed by | Verdict |
|---|---|---|---|---|
| **DNS-only filter** (control-plane resolver returns NXDOMAIN or delays the answer) | Yes (query name) | Only by delaying the answer. glibc gives up after `timeout` 5 s × `attempts` 2 by default, so about 10 s [resolv.conf(5)] | Hard-coded IPs, DoH, cached answers, `/etc/hosts` | Necessary but not sufficient |
| **nftables / eBPF on host bridge** (IP-level) | No. It sees IPs only, so it needs DNS correlation (e.g. dnsmasq `--nftset` adds resolved IPs to a set [dnsmasq(8)]; Cilium `toFQDNs` learns IPs from DNS responses through its DNS proxy [Cilium L3]) | Yes, via NFQUEUE: packets wait for a userspace verdict. With no listener, packets are dropped unless `bypass` is set [nftables wiki]. A held SYN is retransmitted, and the kernel gives up after about 131 s by default [ip-sysctl] | Shared CDN IPs (one IP serves many names), IP churn after resolution | Good **outer fence**, poor policy point |
| **HTTP(S)/SNI forward or transparent proxy on host** | Yes: CONNECT target, Host header, or TLS ClientHello SNI, with no decryption needed (Squid peek/splice reads the SNI and then splices) [Squid] | Yes. It can delay the CONNECT reply or the upstream dial while the request is pending | Domain fronting (SNI ≠ inner Host) [Claude Code sandboxing]; ECH (the outer SNI is the provider's `public_name`) [RFC 9849]; non-TLS, non-HTTP TCP has no name | **Primary policy point**. Everyone in prior art uses it |
| **TLS-terminating (MITM) proxy** | Yes, the real Host and path | Yes | Certificate pinning; needs a CA installed in the VM | Only for domains that need credential injection or L7 rules |
| **eBPF in the guest** (cgroup `connect4/6` hooks) | No | Unknown whether it can block on userspace (unverified) | Root in the guest can detach it. The agent owns the VM | Reject: wrong side of the trust boundary |
| **macOS NetworkExtension** (Little Snitch style) | Yes | `NEFilterNewFlowVerdict.pause()` can pause TCP flows indefinitely, UDP flows ≤10 s [Apple] | n/a | Only relevant for macOS hosts. Linux v1 doesn't need it |

## 2. How prior art does it

**Docker `sbx`:** "The only way traffic can leave a sandbox is through an HTTP/HTTPS proxy on your host" [sbx local policy]. The VM uses a forward proxy for HTTP(S) and a transparent proxy for other TCP. DNS goes through "the sandbox's internal resolver, which enforces network policy". UDP is off by default (experimental opt-in) and ICMP is blocked [sbx isolation, sbx architecture]. Non-HTTP TCP (e.g. SSH) needs an IP+port rule [sbx local policy]. The proxy does intercept HTTPS: v0.48.0 fixed "a malformed request sent through an intercepted HTTPS connection", and v0.47.0 rejects handshakes that overflow the ClientHello inspection buffer and closes incomplete handshakes after 2 min [sbx release notes].
  - **Live approval exists, but it is new and only partly documented.** Under the Balanced/Locked Down presets, an unmatched request "is blocked and asks for your approval instead of being denied outright". You review it with `sbx policy approval ls` / `respond`. "Approving applies to later requests, not the one that was blocked, so run the operation again afterward" [sbx local policy]. The docs PR for it is still open and says the feature is behind a feature flag. A reviewer notes that the tray companion app can list and approve requests [docker/docs#26059]. The CLI reference does not list an `approval` subcommand yet [sbx CLI ref]. Blocked hosts show up in `sbx policy log` with HOST, PROXY (`transparent`), RULE (`domain-blocked`), REASON (`default-deny`) and COUNT [sbx monitoring].
  - So **`sbx` follows the deny-then-retry model**, and approval happens from the CLI or the desktop tray, not remotely.

**NVIDIA OpenShell:** The workload has no network of its own: the container's networking is off, and a VM gets no NIC. A trusted supervisor intercepts TCP opens and DNS through seccomp user-notify, resolves DNS itself, identifies the calling binary, and "opens the real connection and relays traffic" only if policy allows it [OpenShell architecture]. A connection-level denial looks like `403 from proxy after CONNECT`. An L7 (`rest`) rule terminates TLS with a per-sandbox ephemeral CA and returns a JSON `policy_denied` 403 [OpenShell first policy, OpenShell best practices]. Non-HTTP traffic "is relayed without inspection" [best practices].
  - **Approval:** each denial is turned into a draft rule ("one binary to reach one host and port"). The operator approves or rejects it with `openshell rule approve|reject` or in the `openshell term` TUI. "The blocked request stays denied". Approved rules hot-reload, and connections opened under the old rules are closed. The agent can call `/v1/proposals/{id}/wait` (example timeout 300 s) to block until the decision arrives [OpenShell advisor, run-your-first-agent]. This is **deny-then-retry with an agent-side wait**, which is the best pattern for a slow phone approval.

**code-on-incus:** Uses static allowlisting only. It resolves the allowlisted hostnames *on the host*, puts the IPs into nftables and into the container's `/etc/hosts`, and blocks DNS egress. Wildcards are rejected. Re-resolution isn't documented, so CDN IP churn is a risk. nftables monitoring detects DNS tunnelling and allowlist violations, then auto-pauses or kills the container. There is **no interactive approval** [code-on-incus README]. Incus's own network ACLs match IP/CIDR and address sets only, not domains, and on a bridge they apply only at the bridge-to-host boundary [Incus ACLs].

**Claude Code sandbox / `sandbox-runtime` (srt):** On Linux it removes the network namespace. A host HTTP proxy and a SOCKS5 proxy are reached over bind-mounted Unix sockets [srt README]. An `ask` callback decides destinations that no rule covers. **The connection is held while the user decides**: "the command stays in the sandbox and waits for a decision". The choices are "Yes" (rest of session) or "Yes, and don't ask again" (persisted rule). By default the proxy decides on the client-supplied hostname without terminating TLS. Its docs warn this allows domain fronting, and that tools ignoring proxy env vars (ssh, DB drivers), plus UDP and QUIC, can't get out. After the allow check it re-resolves the name and refuses loopback, link-local and host addresses [Claude Code sandboxing].

**Little Snitch / OpenSnitch (desktop firewalls):** Both **hold** the connection. Little Snitch says "The connection is stalled until the alert is answered", and "Once affects only one data packet". Its durations are forever, until logout or restart, until the process quits, or for a set time [Little Snitch alert]. OpenSnitch applies the default action after 30 s (configurable) [OpenSnitch getting started]. Its durations are `once`, `30s`, `1h`, `5h`, `12h`, `always` [OpenSnitch rules]. With the GUI disconnected, the daemon's DefaultAction applies [OpenSnitch config]. Holding works there because the user is at the same screen and answers in seconds.

**Pattern:** tools whose owner sits at the screen hold the connection (Little Snitch, OpenSnitch, Claude Code). Tools built for remote or async approval deny and let the agent retry (`sbx`, OpenShell). A phone-first control plane is the second kind, with a short hold added for fast taps.

## 3. What breaks

**Client timeouts while waiting on a phone tap.** These numbers are the real limit on how long a connection can be held:
- Go `net/http` DefaultTransport: dial 30 s, TLS handshake 10 s, CONNECT to proxy capped at 1 min [Go transport.go]
- Node `fetch` (undici): connect 10 s, headers/body 300 s [undici Client]
- pip: socket timeout 15 s [pip CLI]; npm: `fetch-timeout` 5 min, `fetch-retries` 2 [npm config]
- glibc DNS: 5 s × 2 attempts [resolv.conf(5)]; kernel TCP SYN: about 131 s [ip-sysctl]

A transparent proxy accepts the TCP handshake immediately, so a held TLS connection runs into the *TLS-handshake* timeout (10 s for Go). A held CONNECT runs into the proxy-CONNECT timeout. A phone notification plus a tap usually takes longer than 10 s. **Design for deny-then-retry. Holding is an optimisation, not the contract.**

**Non-HTTP TCP** (SSH to a non-GitHub host, Postgres, Redis): there's no SNI or Host header, so the only name available comes from DNS correlation. Map the destination IP back to names this sandbox's resolver recently returned (the dnsmasq nftset and Cilium approach). If no name matches, raise the request as `IP:port`.

**Hard-coded IPs:** these have no DNS query and maybe no SNI. Raise them as `IP:port` requests. Always refuse loopback, link-local (`169.254.169.254`) and the host's own addresses, as srt and OpenShell do [Claude Code sandboxing, OpenShell best practices].

**UDP / QUIC / HTTP/3:** drop all UDP except DNS to the control plane's resolver. HTTP/3 clients fall back to TCP, and `sbx` and srt both do this.

**DNS-over-HTTPS / DoT:** DoT on port 853 is blocked by default-deny. DoH is just HTTPS to a domain like `dns.google`, so it shows up as an ordinary egress request and is denied by default. The UI should label known DoH endpoints. DNS bypass doesn't matter much anyway, because the proxy enforces on SNI and Host, not on resolution. Firefox's canary domain only switches off *default-enabled* DoH, not DoH the user turned on [Mozilla canary].

**ECH:** with ECH the visible SNI is the provider's `public_name`, not the real host [RFC 9849]. Clients get ECH configs from DNS HTTPS/SVCB records [RFC 9849 §3.2 → RFC 9848; RFC 9460]. Mitigation (inference, untested): the control-plane resolver strips the `ech` parameter, or refuses HTTPS-type queries, so clients send a plain SNI. Clients with a pre-configured ECH config would still evade this.

**Domain fronting:** SNI-only matching trusts a hostname the client supplies [Claude Code sandboxing]. Accept this for v1. Optionally MITM only the domains that already need credential injection.

**Shared and rotating IPs:** an IP-level allow for one domain also allows every other site on that CDN IP. This is why the policy decision belongs in the proxy, with nftables only as the outer fence.

**Long-lived connections and revocation:** once a connection is allowed it stays open. When a rule is revoked, the control plane must kill the connections that rule allowed (OpenShell closes connections opened under the previous rules [OpenShell advisor]). A held, unanswered request must also not tie up proxy resources: cap the number held per sandbox.

**Control plane down:** a fail-open NFQUEUE (`bypass`) leaks traffic. Fail closed: with no listener, packets are dropped [nftables wiki].

## 4. Recommended approach

1. **Fence (host, nftables on the sandbox bridge or tap):** default drop. Allow UDP/TCP 53 only to the control-plane resolver. Redirect all other TCP to the egress proxy (TPROXY/REDIRECT [kernel tproxy]). Drop other UDP and ICMP. Add an Incus ACL as defence in depth if the backend is Incus.
2. **Resolver (control plane):** answers every name (real answers) so clients proceed to connect. It records a per-sandbox name↔IP map for non-SNI traffic, logs queries for the audit log, and strips ECH hints.
3. **Egress proxy (control plane):** one policy point. Name source in priority order: HTTP Host or CONNECT target → TLS SNI → DNS correlation → bare `IP:port`. Policy outcomes: allow, deny, or **pending**. Use MITM only for domains with a brokered secret.
4. **Pending flow:** create an egress request (sandbox, name, port, first-seen, count, process if known) and push it to the phone. Hold the connection for a short configurable window (start with 10–25 s; the right value is still unknown). If the owner answers within the window, splice or refuse the live connection. Otherwise fail closed with a *readable* error: an HTTP 403 body saying "egress request pending for X; retry after approval" for HTTP and CONNECT, and a TCP reset for transparent TLS. Later identical attempts merge into the same pending request instead of creating new notifications. Consider an in-sandbox `wait` endpoint like OpenShell's, so an agent can block on the decision instead of polling.
5. **Choices** (for grilling ticket 07): Deny · Allow for a duration (e.g. 1 h or until the sandbox restarts; a literal "once" is meaningless when tools open many connections) · Allow always (writes to the sandbox's egress policy). Scope: exact host vs `*.domain`.

**Why not DNS-only or nftables-only:** neither can tell apart names that share an IP, both are bypassed by hard-coded IPs, and neither can hold a connection longer than a 10 s resolver timeout. **Why not in-guest eBPF:** the agent controls the guest. **Why not reuse `sbx` approvals:** they are feature-flagged, local-only, approved from the CLI or tray, and undocumented as a stable API (see ticket 01).

## 5. Unknowns

- Docker `sbx` approval details: `respond` flags, once/always semantics, expiry, and whether a third party can drive it. Docs PR #26059 is unmerged and the CLI reference lacks the subcommand.
- Whether OpenShell's TUI offers once versus persist. Its docs say each approval permanently adds a rule.
- The right hold window. It needs measuring against the real agent toolchains (Claude Code, Codex, npm, pip, git, Go) inside a sandbox.
- Whether Linux cgroup `sock_addr` eBPF programs can defer to userspace. Moot for the recommendation, but unverified.
- How common ECH is among CLI tools (curl, Go, Node) in 2026, and whether stripping HTTPS RRs breaks anything.
- Whether code-on-incus ever re-resolves allowlisted IPs.

## Sources

- [sbx local policy] https://docs.docker.com/ai/sandboxes/governance/access-controls/local.md
- [sbx network rules] https://docs.docker.com/ai/sandboxes/governance/access-controls/network/ · https://docs.docker.com/ai/sandboxes/governance/concepts/
- [sbx isolation] https://docs.docker.com/ai/sandboxes/security/isolation/
- [sbx architecture] https://docs.docker.com/ai/sandboxes/architecture/
- [sbx monitoring] https://docs.docker.com/ai/sandboxes/governance/monitor-and-enforce/monitoring.md
- [sbx release notes] https://docs.docker.com/ai/sandboxes/release-notes.md
- [sbx CLI ref] https://docs.docker.com/reference/cli/sbx/policy/
- [docker/docs#26059] https://github.com/docker/docs/pull/26059
- [OpenShell architecture] https://docs.nvidia.com/openshell/latest/about/architecture
- [OpenShell advisor] https://docs.nvidia.com/openshell/latest/how-it-works/policies/advisor
- [OpenShell policies] https://docs.nvidia.com/openshell/latest/how-it-works/policies/overview
- [OpenShell first policy] https://docs.nvidia.com/openshell/latest/tutorials/first-network-policy
- [OpenShell run-your-first-agent] https://docs.nvidia.com/openshell/latest/about/run-your-first-agent
- [OpenShell best practices] https://docs.nvidia.com/openshell/latest/security/best-practices
- [code-on-incus README] https://raw.githubusercontent.com/mensfeld/code-on-incus/master/README.md
- [Incus ACLs] https://linuxcontainers.org/incus/docs/main/howto/network_acls/
- [Claude Code sandboxing] https://code.claude.com/docs/en/sandboxing
- [srt README] https://github.com/anthropic-experimental/sandbox-runtime
- [Little Snitch alert] https://help.obdev.at/littlesnitch5/alert-overview
- [OpenSnitch] https://github.com/evilsocket/opensnitch/wiki/Getting-started · https://github.com/evilsocket/opensnitch/wiki/Rules · https://github.com/evilsocket/opensnitch/wiki/Configurations
- [Apple] https://developer.apple.com/documentation/networkextension/nefilternewflowverdict/pause()
- [nftables wiki] https://wiki.nftables.org/wiki-nftables/index.php/Queueing_to_userspace
- [kernel tproxy] https://docs.kernel.org/networking/tproxy.html
- [ip-sysctl] https://docs.kernel.org/networking/ip-sysctl.html (`tcp_syn_retries`)
- [dnsmasq(8)] https://thekelleys.org.uk/dnsmasq/docs/dnsmasq-man.html (`--nftset`)
- [Cilium L3] https://docs.cilium.io/en/stable/security/policy/layer3/ · https://docs.cilium.io/en/stable/security/dns/
- [Squid] https://wiki.squid-cache.org/Features/SslPeekAndSplice
- [Smokescreen] https://github.com/stripe/smokescreen (prior art: CONNECT proxy with hostname ACLs and private-IP blocking)
- [RFC 9849] https://www.rfc-editor.org/rfc/rfc9849.html (TLS ECH, Mar 2026) · [RFC 9460] https://www.rfc-editor.org/rfc/rfc9460.html
- [Mozilla canary] https://support.mozilla.org/en-US/kb/canary-domain-use-application-dnsnet
- [resolv.conf(5)] https://man7.org/linux/man-pages/man5/resolv.conf.5.html
- [Go transport.go] https://go.dev/src/net/http/transport.go
- [undici Client] https://github.com/nodejs/undici/blob/main/docs/docs/api/Client.md
- [pip CLI] https://pip.pypa.io/en/stable/cli/pip/ · [npm config] https://docs.npmjs.com/cli/v11/using-npm/config
