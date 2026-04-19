# Run Jekyll (fork of Jekyll Run) — Project Context

## What This Is

A maintained fork of the abandoned [Jekyll Run](https://github.com/Kanna727/jekyll-run) VS Code extension (Dedsec727.jekyll-run v1.7.0). The original author (Prasanth Kanna) hasn't committed since 2020. This fork fixes bugs, modernizes CI/CD, and will be republished as **Run Jekyll** on the VS Code Marketplace.

- **Original**: https://github.com/Kanna727/jekyll-run (upstream remote)
- **Fork**: https://github.com/mcgarrah/jekyll-run (origin remote)
- **License**: MIT (permits forking and republishing)

## Branch Strategy

| Branch | Purpose | Extension name | PR-able to upstream? |
|--------|---------|---------------|---------------------|
| `upstream-pr` | Bug fixes only, compatible with original repo | **Jekyll Run** | Yes |
| `main` | Full overhaul — rename, new tests, all 18 fixes, Marketplace publish | **Run Jekyll** | No |

**Workflow for bug fixes that should go upstream:**
1. Commit on `upstream-pr` first
2. Cherry-pick to `main`
3. PR from `upstream-pr` to `Kanna727/jekyll-run:master`

**Workflow for everything else (rename, tests, new features):**
- Commit directly on `main`

**Do NOT merge `main` into `upstream-pr`** — the rename and overhaul changes would make it un-PR-able.

## Current State (v1.7.1)

### Fixed (3 bugs, tagged v1.7.1)
1. **getConfiguration scoping** (`src/config/config.ts`) — `getConfiguration().get('jekyll-run')` returns null in multi-root workspaces; fixed to `getConfiguration('jekyll-run')`
2. **Null rejection** (`src/cmds/run.ts`) — stderr handler rejects with `null` when Ruby errors don't contain "Errno"; fixed with `|| error` fallback
3. **lsof parsing** (`src/utils/process-on-port.ts`) — macOS `lsof` uses variable-width columns; fixed `.split(' ')` to `.split(/\s+/).filter(Boolean)`

### CI/CD Modernized
- GitHub Actions: checkout v4, setup-node v4, CodeQL v4
- Node.js: 12 → 20
- Runner: ubuntu-18.04 → ubuntu-latest
- Test runner: vscode-test → @vscode/test-electron
- Packaging: vsce → @vscode/vsce v3
- Cross-platform testing: macOS, Ubuntu, Windows
- VSIX build workflow with GitHub Release artifacts
- Marketplace publish workflow (needs `PUBLISHER_TOKEN` secret)

### Dependencies Updated
- TypeScript 3.8 → 5.7, ESLint 6.8 → 8.57, Mocha 7.1 → 11.7
- Vulnerabilities: 42 → 3 (remaining are transitive, unfixable from our side)

## Remaining Work (15 issues from code review)

### High Priority — Crashes or Silent Failures
1. **Double rejection in stderr handler** (`src/cmds/run.ts:65-74`) — both `if(error.includes('Error'))` and `if(error.includes('ruby'))` can fire; use `else if`
2. **Raw Buffer passed to reject** (`src/cmds/run.ts:68`, `src/cmds/build.ts:37`) — `reject(data)` passes Buffer, caller does `.toString()` producing `[object Object]`; use `reject(error)`
3. **lsof crash on header-only output** (`src/utils/process-on-port.ts:22`) — `output.split('\n')[1]` is undefined if only header row; check line exists
4. **install.ts rejects with no error** (`src/cmds/install.ts:31`) — `reject()` with no argument; caller does `undefined.toString()`; pass `new Error(data.toString())`
5. **runBundleInstall not async-aware** (`src/extension.ts:175-188`) — cleanup runs before async install completes; move to `.finally()`
6. **deactivate() has no error handling** (`src/extension.ts:290`) — workspace may be torn down during shutdown; wrap in try-catch

### Medium Priority — Robustness
7. **exec-cmd.ts swallows all errors** (`src/utils/exec-cmd.ts:6`) — returns string `'error'` with no context
8. **getNumbersInString splits on single space** (`src/utils/get-numbers-in-string.ts:3`) — same pattern as lsof bug; fix to `.split(/\s+/)`
9. **kill-process-children doesn't await kill** (`src/utils/kill-process-children.ts:22`) — `executeCMD('kill -9 ...')` not awaited
10. **Dead code: VS Code < 1.31 check** (`src/utils/open-in-browser.ts:4`) — `compare-versions` dependency exists solely for this; remove both
11. **pid-on-port undefined array access** (`src/utils/pid-on-port.ts:8`) — `getNumbersInString(output)[0]` can be undefined

### Low Priority — Code Quality
12. **stopServerOnExit default is wrong type** (`package.json`) — `"default": "false"` is a truthy string, not boolean; server always stops regardless of setting
13. **Inline require** (`src/extension.ts:62`) — `var read = require('read-yaml')` should be top-level import
14. **Duplicate error handlers** (`src/extension.ts`) — identical `.then(rejection)` / `.catch()` blocks repeated 4 times; extract to shared function
15. **Commented-out code** (`src/utils/process-on-port.ts`, `src/utils/open-in-browser.ts`) — dead code from previous implementations

## New Features

See `FEATURE.md` in the repo root for planned features with full specs and implementation plans. Current planned features:

- **Jekyll Clean Command** — `jekyll-run.Clean` runs `bundle exec jekyll clean`. When stopped: cleans cache. When running: stops → cleans → restarts. Keybinding: `ctrl+f10`/`cmd+f10`. This is a `main`-branch-only feature.
- **Jekyll Doctor Command** — `jekyll-run.Doctor` runs `bundle exec jekyll doctor`. Read-only diagnostic, safe to run anytime. Keybinding: `ctrl+f11`/`cmd+f11`. `main`-branch-only.
- **Automated Test Suite** — Replace placeholder test with real unit and integration tests covering utility functions, error handling, and new commands. High priority prerequisite for safe refactoring.

## Rename Checklist (for `main` branch)

When renaming to "Run Jekyll", these files need changes:

| File | What to change |
|------|---------------|
| `package.json` | `name`, `displayName`, `description`, `publisher`, `repository`, `bugs`, command IDs (if desired) |
| `README.md` | Title, description, credit original author, link upstream |
| `CHANGELOG.md` | Add v1.8.0 section documenting the fork |
| `LICENSE` | Add fork copyright line (keep original) |
| `src/extension.ts` | Output channel name `"Jekyll Run"` → `"Run Jekyll"`, status bar text |
| `.github/workflows/build-vsix.yml` | VSIX filename |

**Command IDs** (`jekyll-run.Run`, etc.) can stay as-is for backward compatibility, or be renamed to `run-jekyll.Run` — but renaming breaks keybindings for anyone migrating.

## Testing

### Current State
- Only a placeholder test (`extension.test.ts`) that checks `[1,2,3].indexOf(5)`
- Test harness works: Mocha + @vscode/test-electron + Extension Development Host

### Tests to Write (documented in blog draft `2026-06-01-run-jekyll-testing-and-test-harness.md`)
- `process-on-port.test.ts` — lsof parsing with variable-width columns
- `error-handling.test.ts` — null rejection on non-Errno errors
- `config.test.ts` — getConfiguration scoping (integration test, needs Extension Development Host)
- `utils.test.ts` — getNumbersInString edge cases

### Running Tests
```bash
npm install
npm test          # compile + lint + launch Extension Development Host
npm run compile   # TypeScript only
npm run watch     # recompile on save
```

First run downloads a VS Code instance (~200MB, cached in `.vscode-test/`).

## Project Structure

```
jekyll-run/
├── src/
│   ├── extension.ts           # Main entry: activate/deactivate, command registration, status bar
│   ├── config/config.ts       # Settings resolution (getConfiguration wrapper)
│   ├── cmds/
│   │   ├── run.ts             # Jekyll serve lifecycle, stderr handling, progress notification
│   │   ├── build.ts           # Jekyll build (no server)
│   │   ├── stop.ts            # Kill server process tree
│   │   └── install.ts         # bundle install
│   ├── utils/
│   │   ├── exec-cmd.ts        # Shell command wrapper (returns 'error' string on failure)
│   │   ├── process-on-port.ts # lsof/netstat port detection (macOS bug was here)
│   │   ├── pid-on-port.ts     # Windows netstat PID extraction
│   │   ├── process-at-pid.ts  # Windows tasklist process name lookup
│   │   ├── get-numbers-in-string.ts  # Number extraction (has single-space split bug)
│   │   ├── kill-process-children.ts  # Recursive process tree kill
│   │   ├── look-path.ts       # PATH-based executable lookup (filters /mnt for WSL)
│   │   └── open-in-browser.ts # URL opener (has dead VS Code <1.31 check)
│   └── test/
│       ├── runTest.ts         # Test launcher (downloads VS Code)
│       └── suite/
│           ├── index.ts       # Mocha configuration
│           └── extension.test.ts  # Placeholder test (to be replaced)
├── .github/workflows/
│   ├── ci-release.yml         # Cross-platform test + semantic-release
│   ├── ci-publish.yml         # Marketplace publish on GitHub Release
│   ├── build-vsix.yml         # VSIX package + attach to Release
│   ├── codeql-analysis.yml    # Security scanning
│   └── stale.yml              # Stale issue/PR management
├── package.json               # Extension manifest, commands, keybindings, config schema
├── tsconfig.json              # TypeScript config (strict, skipLibCheck, target es6)
├── .eslintrc.json             # ESLint config (has duplicate 'semi' rule — minor)
├── .vscodeignore              # Files excluded from VSIX package
└── CHANGELOG.md               # Semantic-release generated (upstream history only)
```

## Key Technical Details

### How the Extension Works
1. Activates when `_config.yml` found in workspace (`workspaceContains:**/_config.yml`)
2. Reads port and baseurl from `_config.yml` (via `read-yaml` package) and CLI args
3. Spawns `bundle exec jekyll serve [args]` via `child_process.spawn` with `shell: true`
4. Monitors stdout for "Server running" to resolve the run promise
5. Monitors stderr for errors — this is where bugs #1 and #2 live
6. Detects existing server via `lsof` (Unix) or `netstat` (Windows) — bug #3

### Runtime Dependencies (only 2)
- `compare-versions` — Dead code, only used for VS Code <1.31 check. Safe to remove.
- `read-yaml` — Reads `_config.yml` for port/baseurl. Inline `require()` in extension.ts.

### The `/mnt` Filter
`look-path.ts` filters out PATH entries starting with `/mnt` — this prevents WSL from finding Windows executables. The same check appears in `extension.ts` where it tests if `jekyll` or `bundle` paths start with `/mnt`.

### The `stopServerOnExit` Bug
`package.json` declares `"default": "false"` (string) instead of `"default": false` (boolean). The string `"false"` is truthy in JavaScript, so `if (config.stopServerOnExit)` is always true. Every user who set this to `false` has been silently ignored since the feature was added in v1.7.0.

### Semantic Release
The project uses `semantic-release` with conventional commits. The `ci-release.yml` workflow runs it on push to main/master. Commit prefixes: `fix:` → patch, `feat:` → minor, `BREAKING CHANGE:` → major.

## Blog Article Series (in mcgarrah.github.io)

| Status | Date | File | Topic |
|--------|------|------|-------|
| Published | 2026-05-11 | `_posts/2026-05-11-jekyll-run-vscode-plugin-local-development.md` | Configuration guide |
| Draft | 2026-05-22 | `_drafts/2026-05-22-jekyll-run-plugin-multiroot-workspace-bug.md` | macOS debugging story |
| Draft | 2026-05-25 | `_drafts/2026-05-25-forking-jekyll-run-to-run-jekyll.md` | Fork rationale and CI/CD |
| Draft | 2026-05-28 | `_drafts/2026-05-28-run-jekyll-vscode-marketplace-publisher-setup.md` | VS Code Marketplace publisher account setup |
| Draft | 2026-05-29 | `_drafts/2026-05-29-run-jekyll-bug-fixes-and-code-review.md` | 18 issues documented |
| Draft | 2026-06-01 | `_drafts/2026-06-01-run-jekyll-testing-and-test-harness.md` | Test harness and regression tests |
| Draft | 2026-06-04 | `_drafts/2026-06-04-run-jekyll-new-features-clean-doctor-tests.md` | New features: Clean, Doctor, test suite |

All 4 drafts are marked `published: true` and read as finished — ready to promote to `_posts/`.

## VS Code Marketplace Publishing

### Prerequisites
1. Azure DevOps account → create a Personal Access Token (PAT) with "Marketplace: Manage" scope
2. Create a publisher at https://marketplace.visualstudio.com/manage
3. Add PAT as `PUBLISHER_TOKEN` secret in GitHub repo settings

### Publishing Flow
1. Create a GitHub Release (e.g., v1.8.0)
2. `build-vsix.yml` packages and attaches `.vsix` to the Release
3. `ci-publish.yml` publishes to Marketplace using `PUBLISHER_TOKEN`

### Manual Install (before Marketplace)
```bash
gh release download v1.7.1 --repo mcgarrah/jekyll-run --pattern '*.vsix'
code --install-extension jekyll-run.vsix
```

## File Operations

Always use `git mv` instead of `mv` when renaming or moving tracked files. This preserves Git history so `git log --follow` works. The only exception is untracked files.
