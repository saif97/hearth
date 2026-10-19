# How to scope a sandbox to one repo with branch + PR rights only?

Type: research
Status: resolved
Blocked by:

## Question

What is the best GitHub mechanism for a repo grant: fine-grained PAT vs GitHub App installation token scoped to one repo? Which minimal permissions allow creating branches and PRs but not pushing to the default branch, changing settings or reading other repos (and do rulesets/branch protection need to back it)? How do credential proxies (Docker `sbx`, microsandbox placeholder creds) inject the token so it never exists inside the sandbox, and does that work for both `git` over HTTPS and `gh`/API calls? How are tokens rotated?

## Answer

Use a per-owner GitHub App and mint a 1-hour installation token per sandbox, narrowed to that one repo with Metadata read, Contents write and Pull requests write (never Workflows or Administration). Inject it via the backend's credential proxy (`sbx secret set github --sandbox … --command`; microsandbox host-bound secret). Both runtimes swap the placeholder in Basic auth (git over HTTPS) and Bearer headers (`gh`/REST).
No token permission blocks default-branch pushes or self-merge: Contents write covers both. Back the token with a default-branch ruleset (App not on bypass), plus a control-plane filter on `git-receive-pack` refs and merge routes. The filter is the only guard for private repos on GitHub Free. Rotate by re-minting about every 55 min. Revoke the token on sandbox delete.
Findings: [research/repo-grant-credentials.md](../../../research/repo-grant-credentials.md)
