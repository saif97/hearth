# How can egress requests be held for live approval?

Type: research
Status: open
Blocked by:

## Question

What mechanisms can intercept a sandbox's outbound connections per domain and hold an unknown one pending owner approval (allow/deny, once/always)? Cover DNS-based filtering, an HTTP(S)/SNI forward proxy, nftables/eBPF on the host bridge, and how prior art does it (Docker `sbx` proxy, NVIDIA OpenShell, code-on-incus nftables, Little Snitch-style). What breaks: non-HTTP traffic, hard-coded IPs, DNS-over-HTTPS, long-held connections and timeouts while waiting on a phone tap?
