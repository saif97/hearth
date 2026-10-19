# How can egress requests be held for live approval?

Type: research
Status: resolved
Blocked by:

## Question

What mechanisms can intercept a sandbox's outbound connections per domain and hold an unknown one pending owner approval (allow/deny, once/always)? Cover DNS-based filtering, an HTTP(S)/SNI forward proxy, nftables/eBPF on the host bridge, and how prior art does it (Docker `sbx` proxy, NVIDIA OpenShell, code-on-incus nftables, Little Snitch-style). What breaks: non-HTTP traffic, hard-coded IPs, DNS-over-HTTPS, long-held connections and timeouts while waiting on a phone tap?

## Answer

A host-side proxy is the policy point. It reads the name from SNI, the Host header or the CONNECT target, and falls back to DNS correlation, then `IP:port`. The sandbox's DNS is forced to the control plane's resolver, and an nftables default-drop fence (no UDP except DNS) sits behind both. An unknown destination becomes an egress request. The proxy holds the connection only briefly (about 10–25 s, because client TLS/connect timeouts are about 10–30 s), then fails closed with a readable error. Approval applies to the retry, as in Docker `sbx` and NVIDIA OpenShell. Accepted gaps: domain fronting and ECH (SNI-only matching), and shared CDN IPs at the IP layer.

Findings: [research/live-egress-requests.md](../../../research/live-egress-requests.md)
