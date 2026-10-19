# How to scope a sandbox to one repo with branch + PR rights only?

Type: research
Status: open
Blocked by:

## Question

What is the best GitHub mechanism for a repo grant: fine-grained PAT vs GitHub App installation token scoped to one repo? Which minimal permissions allow creating branches and PRs but not pushing to the default branch, changing settings or reading other repos (and do rulesets/branch protection need to back it)? How do credential proxies (Docker `sbx`, microsandbox placeholder creds) inject the token so it never exists inside the sandbox, and does that work for both `git` over HTTPS and `gh`/API calls? How are tokens rotated?
