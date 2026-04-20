# Planned Features

## Jekyll Clean Command

**Status:** Planned
**Branch:** `main` only (new feature, not a bug fix for upstream)
**Priority:** Medium — quality-of-life improvement for `--incremental` users

### Problem

When using `--incremental` (or even without it), Jekyll caches build artifacts in `_site/`, `.jekyll-cache/`, and `.jekyll-metadata`. New files — especially new drafts added while the server is running — don't appear after a restart because the stale cache tells Jekyll to skip them.

The current workaround is to stop the server, run `bundle exec jekyll clean` from a terminal, and restart. This should be a single button click.

### Proposed Solution

Add a `jekyll-run.Clean` command that runs `bundle exec jekyll clean`. It should work in two contexts:

1. **Server is stopped** — run `jekyll clean` standalone (like Build works today)
2. **Server is running** — stop the server, run `jekyll clean`, restart the server (like a "clean restart")

### User-Facing Behavior

| Context | Action | Result |
|---------|--------|--------|
| Server stopped | Command Palette → "Jekyll Clean" | Runs `bundle exec jekyll clean`, shows progress notification |
| Server stopped | `Ctrl+F10` / `Cmd+F10` | Same as above |
| Server running | Command Palette → "Jekyll Clean" | Stops server → cleans → restarts server |
| Server running | `Ctrl+F10` / `Cmd+F10` | Same as above |

The command should appear in:
- Command Palette (always, when `jekyll-run` context is active)
- Status bar (alongside Stop/Restart when running, alongside Run when stopped)
- Editor title menu (optional)

### Implementation Plan

#### 1. New file: `src/cmds/clean.ts`

Modeled after `build.ts`. Spawns `bundle exec jekyll clean` with a progress notification.

```typescript
import { spawn } from 'child_process';
import { window, ProgressLocation, OutputChannel } from 'vscode';

export class Clean {
    constructor() { }

    async clean(workspaceRootPath: string, outputChannel: OutputChannel) {
        return await window.withProgress(
            {
                location: ProgressLocation.Notification,
                title: 'Jekyll Cleaning',
                cancellable: false,
            },
            async () => {
                return new Promise(function (resolve, reject) {
                    var child = spawn('bundle exec jekyll clean', {
                        cwd: workspaceRootPath,
                        shell: true,
                    });

                    child.stdout.on('data', function (data) {
                        outputChannel.append(data.toString());
                        outputChannel.show(true);
                    });

                    child.stderr.on('data', function (data) {
                        var error = data.toString();
                        if (error.includes('Error')) {
                            reject(new Error(error));
                        }
                    });

                    child.on('close', function (code) {
                        if (code === 0) {
                            outputChannel.appendLine('Clean complete.');
                            outputChannel.show(true);
                            resolve(undefined);
                        } else {
                            reject(new Error('Jekyll clean exited with code ' + code));
                        }
                    });
                });
            },
        );
    }
}
```

#### 2. Register in `extension.ts`

```typescript
import { Clean } from './cmds/clean';

// In activate():
const clean = commands.registerCommand('jekyll-run.Clean', async () => {
    if (isStaticWebsiteWorkspace() && currWorkspace) {
        if (isRunning) {
            // Stop → Clean → Restart
            const stop = new Stop();
            await stop.Stop(pid, outputChannel);
            isRunning = false;
            commands.executeCommand('setContext', 'isRunning', false);

            const cleanCmd = new Clean();
            await cleanCmd.clean(currWorkspace.uri.fsPath, outputChannel);

            // Restart
            commands.executeCommand('jekyll-run.Run');
        } else {
            // Clean only
            const cleanCmd = new Clean();
            cleanCmd.clean(currWorkspace.uri.fsPath, outputChannel)
                .catch((error) => {
                    if (error) {
                        window.showErrorMessage(error.toString());
                    }
                });
        }
    }
});

context.subscriptions.push(clean);
```

#### 3. Add to `package.json`

```jsonc
// In contributes.commands:
{
    "command": "jekyll-run.Clean",
    "title": "Jekyll Clean",
    "category": "Jekyll"
}

// In contributes.keybindings:
{
    "mac": "cmd+f10",
    "win": "ctrl+f10",
    "linux": "ctrl+f10",
    "key": "ctrl+f10",
    "command": "jekyll-run.Clean",
    "when": "jekyll-run&&!isBuilding"
}

// In contributes.menus.commandPalette:
{
    "command": "jekyll-run.Clean",
    "when": "jekyll-run&&!isBuilding"
}

// In contributes.activationEvents:
"onCommand:jekyll-run.Clean"
```

Note: The `when` clause is `!isBuilding` only (no `!isRunning` check) because Clean handles both running and stopped states internally.

### Testing

- **Stopped + clean**: Verify `_site/`, `.jekyll-cache/`, `.jekyll-metadata` are removed
- **Running + clean**: Verify server stops, cache clears, server restarts, new drafts appear
- **Cancel during clean**: Verify no partial state (clean is fast, cancellation may not be needed)
- **No workspace**: Verify error message shown
- **No Jekyll installed**: Verify error message shown

### Notes

- **`bundle exec jekyll clean` does NOT remove `.jekyll-cache/`** — it only removes `_site/` and `.jekyll-metadata`. `.jekyll-cache` is a **directory** managed separately by Jekyll's caching layer.
- To fully fix incremental staleness, the Clean command must also explicitly remove the `.jekyll-cache` directory after running `jekyll clean`. Use a directory existence check before removal:
  ```typescript
  // After jekyll clean completes, also remove .jekyll-cache if present
  // (jekyll clean does not remove it)
  import * as fs from 'fs';
  import * as path from 'path';
  const cacheDir = path.join(workspaceRootPath, '.jekyll-cache');
  if (fs.existsSync(cacheDir)) {
      fs.rmSync(cacheDir, { recursive: true, force: true });
  }
  ```
- The command is fast (< 1 second) — no need for cancellation support
- When running, the stop → clean → restart sequence should feel like a single operation to the user
- The progress notification title changes: "Jekyll Stopping..." → "Jekyll Cleaning" → "Jekyll Building"

---

## Jekyll Doctor Command

**Status:** Planned
**Branch:** `main` only (new feature, not a bug fix for upstream)
**Priority:** Low — diagnostic tool, useful for troubleshooting

### Problem

`bundle exec jekyll doctor` checks for deprecation warnings, config issues, and URL conflicts. Currently users must open a terminal to run it. Integrating it into the extension makes diagnostics a single click.

### Proposed Solution

Add a `jekyll-run.Doctor` command that runs `bundle exec jekyll doctor` and displays the output in the Jekyll Run output channel.

### User-Facing Behavior

| Context | Action | Result |
|---------|--------|--------|
| Any (running or stopped) | Command Palette → "Jekyll Doctor" | Runs `bundle exec jekyll doctor`, shows output |
| Any (running or stopped) | `Ctrl+F11` / `Cmd+F11` | Same as above |

Doctor is read-only — it doesn't modify the site or interfere with a running server, so it's available in all states.

### Implementation Plan

#### 1. New file: `src/cmds/doctor.ts`

Modeled after `build.ts`. Spawns `bundle exec jekyll doctor` with a progress notification.

```typescript
import { spawn } from 'child_process';
import { window, ProgressLocation, OutputChannel } from 'vscode';

export class Doctor {
    constructor() { }

    async doctor(workspaceRootPath: string, outputChannel: OutputChannel) {
        return await window.withProgress(
            {
                location: ProgressLocation.Notification,
                title: 'Jekyll Doctor',
                cancellable: false,
            },
            async () => {
                return new Promise(function (resolve, reject) {
                    var child = spawn('bundle exec jekyll doctor', {
                        cwd: workspaceRootPath,
                        shell: true,
                    });

                    child.stdout.on('data', function (data) {
                        outputChannel.append(data.toString());
                        outputChannel.show(true);
                    });

                    child.stderr.on('data', function (data) {
                        // Doctor outputs warnings to stderr — show them, don't reject
                        outputChannel.append(data.toString());
                        outputChannel.show(true);
                    });

                    child.on('close', function (code) {
                        outputChannel.appendLine('Doctor complete (exit code: ' + code + ')');
                        outputChannel.show(true);
                        resolve(undefined);
                    });
                });
            },
        );
    }
}
```

Note: Doctor outputs warnings to stderr even on success. Unlike `build.ts` and `clean.ts`, the stderr handler should **not** reject — it should display the warnings in the output channel.

#### 2. Register in `extension.ts`

```typescript
import { Doctor } from './cmds/doctor';

// In activate():
const doctor = commands.registerCommand('jekyll-run.Doctor', async () => {
    if (isStaticWebsiteWorkspace() && currWorkspace) {
        const doc = new Doctor();
        doc.doctor(currWorkspace.uri.fsPath, outputChannel)
            .catch((error) => {
                if (error) {
                    window.showErrorMessage(error.toString());
                }
            });
    }
});

context.subscriptions.push(doctor);
```

#### 3. Add to `package.json`

```jsonc
// In contributes.commands:
{
    "command": "jekyll-run.Doctor",
    "title": "Jekyll Doctor",
    "category": "Jekyll"
}

// In contributes.keybindings:
{
    "mac": "cmd+f11",
    "win": "ctrl+f11",
    "linux": "ctrl+f11",
    "key": "ctrl+f11",
    "command": "jekyll-run.Doctor",
    "when": "jekyll-run"
}

// In contributes.menus.commandPalette:
{
    "command": "jekyll-run.Doctor",
    "when": "jekyll-run"
}

// In contributes.activationEvents:
"onCommand:jekyll-run.Doctor"
```

Note: The `when` clause is just `jekyll-run` — no `!isRunning` or `!isBuilding` checks because Doctor is read-only and safe to run anytime.

### Testing

- **Clean site**: Verify Doctor runs and reports no issues
- **Site with deprecation warnings**: Verify warnings appear in output channel
- **While server is running**: Verify Doctor runs without interfering with serve
- **No workspace**: Verify error message shown

---

## Automated Test Suite

**Status:** Planned
**Branch:** `main` only
**Priority:** High — prerequisite for safe refactoring and new features

### Problem

The test suite has a single placeholder test that checks `[1,2,3].indexOf(5)`. The test harness works (Mocha + @vscode/test-electron + Extension Development Host), but there are no real tests. Every bug fix and new feature is manually verified. This doesn't scale — especially with 15 remaining issues from the code review and new features (Clean, Doctor) planned.

### Current State

- **Test harness**: Working. `npm test` compiles, lints, and launches the Extension Development Host.
- **Test runner**: Mocha with `tdd` UI, auto-discovers `**/*.test.js` files in `out/test/`.
- **Actual tests**: One file (`extension.test.ts`) with one placeholder assertion.
- **CI**: Cross-platform testing (macOS, Ubuntu, Windows) runs in GitHub Actions but only exercises the placeholder test.

### Proposed Test Files

| File | What it tests | Type |
|------|--------------|------|
| `process-on-port.test.ts` | `lsof` parsing with variable-width columns, header-only output, missing lines | Unit |
| `get-numbers-in-string.test.ts` | Number extraction with single/multiple spaces, empty strings, no numbers | Unit |
| `pid-on-port.test.ts` | Windows `netstat` PID extraction, undefined array access | Unit |
| `config.test.ts` | `getConfiguration` scoping in single-folder and multi-root workspaces | Integration (needs Extension Development Host) |
| `error-handling.test.ts` | Null/undefined rejection in stderr handlers, Buffer vs string errors | Unit |
| `clean.test.ts` | Clean command spawns correct process, handles errors | Unit (mock `child_process`) |
| `doctor.test.ts` | Doctor command handles stderr warnings without rejecting | Unit (mock `child_process`) |

### Implementation Approach

1. **Start with pure unit tests** — `process-on-port`, `get-numbers-in-string`, `pid-on-port` are pure functions that take input and return output. No VS Code API mocking needed. These cover the bugs already fixed (lsof parsing) and prevent regressions.

2. **Add error-handling tests** — Mock `child_process.spawn` to simulate stderr output with and without "Error" and "ruby" strings. Verify the correct rejection/resolution behavior.

3. **Add integration tests last** — `config.test.ts` needs the Extension Development Host to test `vscode.workspace.getConfiguration()` scoping. These are slower and more fragile.

4. **Test new features** — Each new command (Clean, Doctor) gets a test file verifying the spawn command and error handling.

### Testing Patterns

For unit tests that don't need VS Code APIs:

```typescript
import * as assert from 'assert';
import { getNumbersInString } from '../../utils/get-numbers-in-string';

suite('getNumbersInString', () => {
    test('extracts numbers from space-separated string', () => {
        assert.deepStrictEqual(getNumbersInString('pid 1234 name'), [1234]);
    });

    test('handles multiple spaces between tokens', () => {
        assert.deepStrictEqual(getNumbersInString('pid   1234   name'), [1234]);
    });

    test('returns empty array for no numbers', () => {
        assert.deepStrictEqual(getNumbersInString('no numbers here'), []);
    });

    test('handles empty string', () => {
        assert.deepStrictEqual(getNumbersInString(''), []);
    });
});
```

For tests that need mocked `child_process`:

```typescript
// Use sinon or manual stubs to mock spawn()
// Verify the correct command string is passed
// Simulate stdout/stderr events and verify resolve/reject behavior
```

### Success Criteria

- All 3 fixed bugs (v1.7.1) have regression tests
- All utility functions have edge-case coverage
- New features (Clean, Doctor) have tests before merge
- CI runs tests on all 3 platforms (macOS, Ubuntu, Windows)
- No test depends on a real Jekyll installation or running server

### Related

- Blog draft: `_drafts/2026-06-01-run-jekyll-testing-and-test-harness.md` documents the test harness setup and planned tests
- The 15 remaining issues from code review (see project-context.md) should each get a test when fixed
