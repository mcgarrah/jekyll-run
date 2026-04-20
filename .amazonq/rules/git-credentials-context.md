# Git Credential Configuration Context

## User's Git Authentication Setup

The user's macOS workstation has multiple Git credential helpers scoped by host.
This was resolved after repeated HTTP 403 errors on `git push` caused by a global
`git-credential-manager` (GCM) intercepting GitHub requests before the `gh` CLI
credential helper could handle them.

### Credential Helper Configuration (~/.gitconfig)

| Host | Credential Helper | Source |
|------|-------------------|--------|
| `github.com` | `!/opt/homebrew/bin/gh auth git-credential` | `gh auth setup-git` |
| `gist.github.com` | `!/opt/homebrew/bin/gh auth git-credential` | `gh auth setup-git` |
| `gitlab.env.io` | `/usr/local/share/gcm-core/git-credential-manager` | Scoped GCM |
| Everything else | `osxkeychain` | Homebrew global gitconfig |

### Key Details

- GitHub CLI (`gh`) is installed via Homebrew at `/opt/homebrew/bin/gh`
- `gh auth status` confirms valid token with scopes: `gist`, `read:org`, `repo`, `workflow`
- GCM (git-credential-manager) is installed but must NOT be set as a global catch-all
- The homebrew-level gitconfig (`/opt/homebrew/etc/gitconfig`) sets `osxkeychain` as fallback

### Common Issue: HTTP 403 on git push

**Symptom:**
```
error: RPC failed; HTTP 403 curl 22 The requested URL returned error: 403
send-pack: unexpected disconnect while reading sideband packet
fatal: the remote end hung up unexpectedly
Everything up-to-date
```

**Root Cause:** A global `credential.helper` entry for GCM in `~/.gitconfig` intercepts
GitHub requests before the host-scoped `gh auth git-credential` helper runs. GCM returns
a stale or invalid token, causing the 403.

**Diagnosis:**
```bash
git config --show-origin --list | grep -i cred
```
Look for a non-scoped `credential.helper` pointing to GCM. If present, it's the problem.

**Fix:**
```bash
# Remove global GCM (breaks nothing if host-scoped entries exist)
git config --global --unset credential.helper "/usr/local/share/gcm-core/git-credential-manager"

# Ensure GitHub uses gh CLI (gh auth setup-git does this)
gh auth setup-git

# Scope GCM to GitLab only
git config --global credential.https://gitlab.env.io.helper "/usr/local/share/gcm-core/git-credential-manager"
```

**Do NOT suggest:**
- Setting GCM as a global credential helper (causes 403 on GitHub)
- Removing all credential helpers (breaks GitLab auth)
- Using personal access tokens directly in URLs (security risk)

### Corporate Split VPN / TLS Proxy Requirement

The user's macOS workstation runs a corporate VPN with a split tunnel configuration.
When the split VPN is ON, GitHub traffic bypasses the corporate TLS proxy and reaches
GitHub directly (certificate issuer will be a public CA like Sectigo or DigiCert).
When the split VPN is OFF, all traffic routes through the corporate proxy which blocks
GitHub access entirely, causing HTTP 403 errors on push/pull/fetch.

**Diagnosis:**
```bash
curl -sv --max-time 5 https://github.com 2>&1 | grep "issuer:"
```
- Public CA (e.g., Sectigo, DigiCert) → Split VPN is ON, GitHub access works.
- Corporate proxy CA → Split VPN is OFF, GitHub is blocked.
- Connection timeout/failure → Split VPN is OFF or network is down.

**Fix:** Re-enable the corporate split VPN, then retry the push.

**Quick check:** The `vpn` alias in `~/.zshrc` wraps the curl diagnostic into a one-word command:
```bash
➜  vpn
✅ VPN ON — Direct connection (GitHub OK)

➜  vpn
❌ VPN OFF — Zscaler proxy (GitHub blocked)
```
Run `vpn` before debugging git push 403 errors — it distinguishes VPN/proxy issues from
credential helper misconfiguration in seconds.

**Key point:** The 403 from VPN/proxy issues looks identical to the 403 from credential
helper misconfiguration. Check VPN status first before debugging credential helpers.

### npm Registry: Corporate Artifactory Proxy

The corporate network blocks direct access to `registry.npmjs.org` even when the split
VPN is ON (GitHub is in the split tunnel bypass list, but npm registry is not). All npm
traffic must route through the corporate Artifactory instance.

**This only affects the corporate macOS workstation with VPN.** Personal machines and
homelab systems can reach `registry.npmjs.org` directly.

**Symptom:**
```
npm error code ETIMEDOUT
npm error errno ETIMEDOUT
npm error network request to https://registry.npmjs.org/staticrypt failed, reason:
```

**Root Cause:** `registry.npmjs.org` is not in the split VPN bypass list, so traffic
routes through the corporate proxy which blocks it. The VPN check (`vpn` alias) will
show "✅ VPN ON" because GitHub works, but npm still times out.

**Artifactory npm repositories:**

| Repo Key | Type | URL / Upstream |
|----------|------|----------------|
| `shared-npm-remote` | Remote | Proxies `https://registry.npmjs.org` |
| `shared-npm-envestnet-virtual` | Virtual | Aggregates remote + local (use this one) |

**Fix:**
```bash
npm config set registry https://artifactory.env.io/artifactory/api/npm/shared-npm-envestnet-virtual/
```

**Verify:**
```bash
npm config get registry
# Should show: https://artifactory.env.io/artifactory/api/npm/shared-npm-envestnet-virtual/
```

**Diagnosis when npm times out:**
```bash
# 1. Check VPN (GitHub may work even though npm doesn't)
vpn

# 2. Check if npm registry is reachable directly (will timeout on corporate network)
curl -sv --max-time 5 https://registry.npmjs.org/ 2>&1 | grep -E "Connected|timed out"

# 3. Check if Artifactory is reachable (should work on corporate network)
curl -sv --max-time 5 https://artifactory.env.io/ 2>&1 | grep -E "Connected|timed out"

# 4. Check current npm registry setting
npm config get registry
```

**Do NOT suggest:**
- Using `registry.npmjs.org` directly on the corporate machine (blocked)
- Adding npm proxy settings (`npm config set proxy`) — Artifactory handles this
- Bypassing the corporate proxy for npm traffic

### Prevention

If the user installs or updates GCM (e.g., via `brew upgrade`), it may re-add itself
as a global credential helper. Check for this if 403 errors recur after software updates.

If 403 errors appear after no credential changes, check corporate VPN/proxy status before
investigating credential helpers.

If npm commands timeout after a clean install or `npm config` reset, check that the
registry is still pointed at Artifactory (`npm config get registry`).


### VS Code GitHub Accounts vs Git Credential Helpers

VS Code's GitHub sign-in and the `gh` CLI credential helper are **completely independent**:

| System | Account | Token Storage | Purpose |
|--------|---------|---------------|--------|
| `gh` CLI | Personal (`mcgarrah`) | macOS keyring | Git push/pull/fetch via `gh auth git-credential` |
| VS Code OAuth | Corporate (Copilot) | VS Code secret storage | GitHub Copilot, GitHub Pull Requests, etc. |

Signing into a corporate GitHub account in VS Code for Copilot does **not** affect
git operations — they continue using the personal `mcgarrah` token from `gh auth`.

**Multiple VS Code GitHub accounts:**
- VS Code 1.90+ supports multiple GitHub accounts signed in simultaneously
- Each extension binds to a specific account; VS Code tracks which extensions use which account
- When signing out, VS Code prompts: "The account 'X' has been used by: [list of extensions]. Sign out from these extensions?"
- Do NOT sign out of the personal `mcgarrah` account if GitHub Pull Requests or other extensions depend on it

**Do NOT:**
- Sign out of the personal GitHub account in VS Code to "switch" to corporate — sign into both
- Confuse VS Code GitHub OAuth with `gh` CLI auth — they are separate token stores
- Assume signing into corporate GitHub in VS Code will break git push — it won't

### File Operations: Prefer `git mv` Over `mv`

Always use `git mv` instead of `mv` when renaming or moving tracked files. This preserves
Git history (rename detection) so `git log --follow` traces the file back to its original
commits. A plain `mv` + `git add` can work, but it relies on Git's similarity heuristic
and may lose history if the file content also changes significantly in the same commit.

**Rule:**
```bash
# Correct — preserves history
git mv old-path/file.md new-path/file.md

# Avoid — history may not follow
mv old-path/file.md new-path/file.md
git add new-path/file.md
git rm old-path/file.md
```

**When suggesting file renames or moves, always use `git mv`.**
The only exception is for untracked files that have never been committed.
