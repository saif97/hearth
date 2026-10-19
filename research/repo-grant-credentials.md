# Repo grant credentials: scoping a sandbox to one repo (Oct 2026)

Ticket: `.scratch/sandbox-control-plane/issues/04-repo-grant-credentials.md`

**Verdict:** use a **GitHub App** (one per owner, registered via the manifest flow) and have the control plane mint a fresh **installation token per sandbox**, narrowed to that sandbox's one repo and to `contents:write` + `pull_requests:write` (+ `metadata:read`), every ~55 minutes. Inject it through the sandbox runtime's credential proxy so the sandbox only sees a placeholder. **No token permission can say "branches but not the default branch" or "open PRs but don't merge them"**: `contents:write` covers pushing to any branch *and* merging PRs. Those limits have to come from a GitHub ruleset on the default branch (where the plan allows it) and/or from the control plane's own proxy filtering git ref updates and API routes.

## 1. Options compared

| | GitHub App installation token | Fine-grained PAT |
|---|---|---|
| Repo scoping | Per token: `repositories` / `repository_ids` in the mint request; can't exceed what the installation was given [1] | Per token: one resource owner, then "only select repositories" [2] |
| Permission scoping | Per token: `permissions` body param, a subset of the app's permissions [1] | Fixed when the token is created [2] |
| Lifetime | **1 hour**, fixed [1] | 1–366 days or `none` (no expiry) unless an org policy caps it [2] |
| Can the control plane create it programmatically? | Yes: `POST /app/installations/{id}/access_tokens` signed with the app JWT [1] | **No documented API.** Only the web UI, with URL pre-fill for name/description/target/expiry/permissions but **not repositories** [2] |
| Revocation | `DELETE /installation/token` (self-revoke) [3] | Manual delete in the UI [2] |
| Acts as | The app (`app-name[bot]`); activity attributed to the app [4] | The user, with all the user's roles, including admin bypass on their own repos [2][7] |
| Gaps | None relevant | Can't call the **Checks API**; can't contribute to repos where the user is an outside collaborator or non-member public repos; max 50 per user [2] |
| Setup cost | One-time: register the app (manifest flow, works for personal accounts, done within 1 h [5]) and install it on repos | Per sandbox: owner clicks through the PAT UI and pastes the token |

**Why the App wins:** the control plane can mint a token per sandbox with no human in the loop, the token is narrowed per repo and per permission at mint time, a leaked token is useless after an hour, and the token *isn't* the owner. That last point matters for rulesets. A PAT acts as the repo owner, who is an admin. Classic branch protection lets admins push unless "Do not allow bypassing" is set [7], and rulesets let you put repo admins on the bypass list [8]. The App is a separate actor that you simply leave off the bypass list. A fine-grained PAT can't read CI check runs either [2], and agents want those.

**Cost of the App:** the app's private key "grants access to every account that the app is installed on" [4]. It lives on the host, on the trusted side next to the proxy. GitHub recommends a key vault over env vars and never hard-coding it [4].

## 2. Minimal permission set

Sources: GitHub App permissions reference [6] and REST endpoint pages [9][10].

| Permission | Level | Why | Notes |
|---|---|---|---|
| Metadata | read | `GET /repos/{o}/{r}` needs it [11] | Granted to every app (not re-checked in this session) |
| Contents | **write** | git push over HTTPS needs Contents [12]; `POST/PATCH/DELETE git/refs` = Contents write [9] | **Also allows** `PUT /pulls/{n}/merge` and `POST /merges` (both Contents write) [6] |
| Pull requests | write | `POST /pulls` and `PATCH /pulls/{n}` = Pull requests write [6][10] | |
| Workflows | **withhold** | Creating or updating refs that touch `.github/workflows` needs Contents write **plus** Workflows write [9] | Withholding it stops the agent from adding a workflow that would run with repo secrets |
| Administration | **withhold** | `PATCH /repos`, `DELETE /repos`, branch protection and rulesets endpoints all need Administration write [6] | Withholding it covers "can't change settings" |
| Optional: Checks, Commit statuses, Actions | read | Let the agent see CI results | Checks API isn't available to fine-grained PATs at all [2] |
| Optional: Issues | read/write | Issue context or PR conversation comments | Exact endpoint mapping not verified here |

**What the token alone still allows** (and has to be blocked elsewhere):
- Pushing to the default branch, or force-pushing or deleting any branch (Contents write).
- Merging its own PR via the API (Contents write) [6].
- Reading other repos is **not** possible: the token can't reach a repo outside `repository_ids` [1]. The exception is public repos, which are readable anyway.

### Backing it with rulesets / branch protection

Target the default branch with a ruleset using **Restrict updates**, **Restrict deletions**, **Block force pushes** and **Require a pull request before merging** with ≥1 approval. Keep the App off the bypass list. Bypass can be granted to roles, teams or GitHub Apps [8][13]. Required approvals also block self-merge via the API, because the App can't approve its own PR. "Restrict creations"/"Restrict updates" on other patterns can confine the agent to e.g. `agent/**` [8].

**Plan gap:** rulesets and protected branches on **private** repos need GitHub Pro/Team/Enterprise. On GitHub Free they only work on public repos [8][7]. So for a Free user's private repo, GitHub can't enforce "no default-branch push" at all, and the control plane's proxy has to do it (§3.3).

## 3. Proxy injection

### 3.1 What GitHub expects on the wire

- **git over HTTPS:** HTTP Basic auth with username `x-access-token` and the installation token as password: `https://x-access-token:TOKEN@github.com/owner/repo.git` [12]. git smart-HTTP uses standard Basic auth [14].
- **REST/GraphQL (`gh`, API):** `Authorization: Bearer <token>` [12].
- Installation tokens are moving to a stateless `ghs_APPID_JWT` format of about 520 characters, variable length. Don't assume 40 characters [15][1]. Rollout ran Apr–Jun 2026. The header `X-GitHub-Stateless-S2S-Token: enabled|disabled` forces either format per mint request [16].

So a proxy must substitute both inside a **Basic auth header** (decode, swap, re-encode) and inside a **Bearer header**, and it must terminate TLS to do either.

### 3.2 How the candidate runtimes do it

**Docker Sandboxes (`sbx`, v0.48.0 as of 2026-10-13 [17])**
- A host-side proxy intercepts outbound HTTP/HTTPS, looks up the credential and rewrites the auth header. The sandbox sees the placeholder `proxy-managed` [18]. "The raw credential values never enter the VM" [19]. Substitution happens "after the request has left the microVM" [20].
- The built-in **`github` service** injects on `github.com`, `api.github.com`, `raw.githubusercontent.com`, `gist.github.com` and Copilot hosts, with in-VM env `GH_TOKEN`/`GITHUB_TOKEN` [18].
- **Dynamic source:** `sbx secret set github --command '<cmd>'` runs a host command and caches its output for 55 minutes by default, or per use with `--refresh on-demand` [21]. `--sandbox <name>` scopes the secret to one sandbox, and sandbox-scoped secrets win over global ones [18]. Updates take effect in running sandboxes without a restart [18].
  - This maps directly onto per-sandbox App tokens: `sbx secret set github --sandbox foo --command 'scp mint-token --repo owner/foo'`.
- **git over HTTPS:** the core docs don't spell it out. The first-party `docker/sbx-kit-github-clone` kit writes a `--system` git credential helper pointing at `gh auth git-credential`, so `git` and `gh` both send the `proxy-managed` sentinel and the proxy "substitutes the real token onto every outbound request to `api.github.com` and `github.com`" [22]. That implies the proxy handles the sentinel inside git's Basic auth header.
- **Custom secrets** (experimental): a placeholder `sbx-cs-<rand>` is replaced "anywhere in the request" to configured hosts, with wildcard support [18].
- Non-HTTP TCP is forwarded transparently with no injection [20], so git over SSH gets no token. Use HTTPS remotes.

**microsandbox (beta)**
- A secret is bound to allowed hosts. The guest gets a placeholder env such as `$MSB_GITHUB_TOKEN` and the real value "never enters the guest VM" [23][24].
- Substitution covers headers by default, "including Basic authentication" (decoded and re-encoded). Bodies and query strings are opt-in [23][24]. Restrict substitution to `Authorization` so a reflecting endpoint can't echo the secret back in another header [25].
- It requires TLS interception by default. A CA from `~/.microsandbox/tls/` is added to the guest trust store. Apps with their own CA stores may need `msb-trust.sh`. Certificate-pinning clients must be bypassed, and bypassed hosts get no injection [26][25].
- A placeholder sent to a non-allowed host is blocked and logged by default, or optionally the sandbox is terminated [23].
- You can rotate a secret or change its allowed hosts live "when supported". Adding a secret or changing its placeholder needs a restart [23].

Both runtimes therefore cover **git over HTTPS and `gh`/REST** with the token kept on the host, as long as remotes are HTTPS and the in-VM CA trust is set up.

### 3.3 What neither runtime does: ref- and route-level policy

Neither runtime documents filtering on *what* the request does. The control plane needs that wherever GitHub can't enforce it (Free private repos), and as defense in depth everywhere else. Because the proxy already terminates TLS:
- **git push:** `POST /{owner}/{repo}.git/git-receive-pack` carries pkt-lines `old-id new-id refname` before the packfile [14]. Reject any update whose refname is the default branch (or outside `refs/heads/agent/<sandbox>/**`), and deletes (`new-id` = zeros).
- **REST:** deny `PUT /repos/*/pulls/*/merge`, `POST /repos/*/merges`, the GraphQL `mergePullRequest`/`enablePullRequestAutoMerge` mutations, and anything outside the granted repo path.
- `sbx` can chain to an upstream HTTP/HTTPS proxy [20]. Whether it chains *after* injection (so a control-plane filter would see real tokens) or before is undocumented (see Unknowns). A filter in front of the runtime proxy, which sees only placeholders, is the safer layout.

## 4. Rotation

- **Installation tokens:** no refresh endpoint. Re-mint with a fresh app JWT, since tokens expire after 1 h [1][12]. Mint about every 55 min, which matches `sbx`'s default cache [21], or on demand. On sandbox delete or suspend, call `DELETE /installation/token` [3] and drop the runtime secret.
- **App private key:** up to 25 keys at once. Rotate by generating a new key, switching to it, then deleting the old one, with no downtime [27].
- **Fine-grained PAT:** manual. The owner regenerates it in the UI before expiry (≤366 days, or never) and updates the runtime secret. microsandbox notes that updating a secret in the sandbox "does not rotate or revoke it at the issuing service" [23].
- **Repo add/remove:** after the Oct 2026 changes, repo admins can add their repos to an already-installed app themselves. A new `installation_repositories:write` permission lets an app or PAT manage installation repos [28]. `PUT /user/installations/{id}/repositories/{repo_id}` requires admin on the repo [3].

## 5. Recommendation

1. **Credential type:** one private GitHub App per owner, created via the manifest flow from the control plane UI [5] and installed on the repos they want sandboxes for. Permissions: Metadata r, Contents rw, Pull requests rw, plus optional Checks/Statuses/Actions r and Issues rw. Never Workflows or Administration.
2. **Per sandbox:** the control plane mints `POST /app/installations/{id}/access_tokens` with `repository_ids: [repo]` and the narrowed `permissions` [1]. It hands the token to the backend's credential proxy (`sbx secret set github --sandbox <name> --command …`, or a microsandbox secret bound to `github.com` and `api.github.com`, header substitution limited to `Authorization`). The sandbox sees only a placeholder. It revokes the token on delete.
3. **Branch and merge limits:** two layers.
   - (a) Offer to create a default-branch ruleset (restrict updates and deletions, block force push, require PR + 1 approval, App not on bypass). The Administration permission for this belongs to the owner's own setup step, not to the sandbox token.
   - (b) The control-plane proxy filters `git-receive-pack` ref updates and merge routes. Layer (b) is the only enforcement for private repos on GitHub Free.
4. **Fallback:** support a pasted fine-grained PAT for users who won't create an App. The UI should warn that it acts as them (admin bypass), that it can't read Checks, and that rotation is manual.
5. **Remotes:** force HTTPS remotes inside the sandbox. SSH gets no injection [20].

This fits the settled constraints in the map: secrets via proxy, never mounted; repo grant limited to branches and PRs; the repo grant visible and revocable from the UI.

## 6. Unknowns

- **Docker `sbx` Basic auth substitution:** shown only by the `github-clone` kit README [22], not stated in the core credentials docs. Also not documented: the exact header format the `github` service writes (`Bearer` vs `token`), and whether its `--command` output can be a ~520-character stateless token without issue.
- **`sbx` upstream-proxy ordering:** whether the upstream proxy sees requests before or after credential injection [20].
- **Revoking stateless tokens:** whether `DELETE /installation/token` revokes a stateless (JWT-format) installation token immediately. The notice says nothing about revocation [15].
- **Full `.github/workflows` push behavior:** endpoint docs list "Contents write + Workflows write" as an alternative permission set [9], and community reports show pushes refused without Workflows. The authoritative docs page for git-push behavior wasn't found.
- **PR comment endpoints:** the exact permission for PR conversation comments (Issues vs Pull requests) was not verified.
- **Metadata read is automatic:** taken as known GitHub behavior, not re-fetched in this session.
- **microsandbox trust setup for git:** whether `msb-trust.sh` covers git, or git needs `http.sslCAInfo` [26].
- **Rulesets that confine creations to `agent/**`:** whether "Restrict creations" with an App-only allow pattern works without a bypass entry. Rulesets restrict *for everyone except bypass*, so allowing only certain prefixes may need inverted patterns. Not tested.

## Sources

1. https://docs.github.com/en/apps/creating-github-apps/authenticating-with-a-github-app/generating-an-installation-access-token-for-a-github-app
2. https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/managing-your-personal-access-tokens
3. https://docs.github.com/en/rest/apps/installations
4. https://docs.github.com/en/apps/creating-github-apps/about-creating-github-apps/best-practices-for-creating-a-github-app
5. https://docs.github.com/en/apps/sharing-github-apps/registering-a-github-app-from-a-manifest
6. https://docs.github.com/en/rest/authentication/permissions-required-for-github-apps
7. https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/managing-protected-branches/about-protected-branches
8. https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/managing-rulesets/available-rules-for-rulesets
9. https://docs.github.com/en/rest/git/refs
10. https://docs.github.com/en/rest/pulls/pulls
11. https://docs.github.com/en/rest/repos/repos
12. https://docs.github.com/en/apps/creating-github-apps/authenticating-with-a-github-app/authenticating-as-a-github-app-installation
13. https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/managing-rulesets/creating-rulesets-for-a-repository
14. https://git-scm.com/docs/http-protocol
15. https://github.blog/changelog/2026-04-24-notice-about-upcoming-new-format-for-github-app-installation-tokens/
16. https://github.blog/changelog/2026-05-15-github-app-installation-tokens-per-request-override-header
17. https://docs.docker.com/ai/sandboxes/release-notes/
18. https://docs.docker.com/ai/sandboxes/configuration/credentials/
19. https://docs.docker.com/ai/sandboxes/security
20. https://docs.docker.com/ai/sandboxes/architecture
21. https://docs.docker.com/ai/sandboxes/workflows/authentication/
22. https://hub.docker.com/r/docker/sbx-kit-github-clone
23. https://docs.microsandbox.dev/sandboxes/secrets
24. https://docs.microsandbox.dev/sdk/typescript/secrets
25. https://docs.microsandbox.dev/security/secrets.md
26. https://docs.microsandbox.dev/networking/tls
27. https://docs.github.com/en/apps/creating-github-apps/authenticating-with-a-github-app/managing-private-keys-for-github-apps
28. https://github.blog/changelog/2026-10-13-github-app-improvements-for-least-privilege-and-management
