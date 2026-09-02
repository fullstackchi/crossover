# Electron upgrade plan

## Decision

The runtime remains pinned to Electron 14 until the native-module gate below is
green. `config.electronUpgradeTarget` in `package.json` records the selected
destination: **Electron 44** (exactly `44.x`, not a
floating `latest` range). Before opening the runtime bump, verify that 44 is
still one of Electron's [three supported stable majors][releases] and select the
newest supported major if it is not. This check matters because Electron only
supports its latest three stable releases.

The eventual `package.json` change is deliberately deferred and should be made
as an exact, reviewable pair with its lockfile update:

```diff
-    "electron": "^14.0.0",
-    "electron-builder": "^24.6.4",
+    "electron": "44.x",
+    "electron-builder": "26.x",
```

Do not apply that diff until every preflight item is complete. In particular,
do not silence a native-module load failure by relying on the keyboard hook's
optional fallback.

## Why the upgrade is blocked today

| Component | Repository evidence | Electron 44 decision |
| --- | --- | --- |
| Node used to install/package | `.nvmrc` is 20.19.2 and `package.json` accepts Node 18 | Move CI, `.nvmrc`, and `engines.node` to a supported Node 22 LTS patch before upgrading Electron Builder. This is the packaging Node; Electron's embedded Node is independent. |
| `uiohook-napi` | Installed 1.5.5 is Node-API based and contains `win32-x64` and `win32-arm64` prebuilds | Promising, but not yet confirmed for the candidate. Require a clean Windows install and a packaged smoke test that loads, starts, receives one event, and stops the hook. Record package version and hashes of both Windows `.node` files. Node-API alone is not a substitute for this test. |
| `electron-winstore-auto-launch` | 2.0.6 depends on `@nodert-win10-au/windows.applicationmodel` 0.4.4, a NAN/native addon; Electron Builder currently has `npmRebuild: false` | **Blocker.** It has no Electron-44 prebuild in the installed package, and the current build explicitly prevents rebuilding it for Electron's ABI. Obtain a maintained Node-API/prebuilt release and verify it in an APPX/MSIX package, replace the module, or remove Store auto-launch. Do not bump Electron while this remains unresolved. |
| Electron Builder | 24.x predates the destination runtime | Upgrade to 26.x on the old runtime first. Produce unpacked, NSIS, portable, and APPX artifacts; inspect ASAR/native unpacking; install/uninstall; and exercise updater metadata. Only then bump Electron. |

For every dependency confirmation, use the published tarball from a clean lockfile
install—not a locally compiled or cached `node_modules` tree—and attach the
Windows build log to the upgrade PR.

## Incremental sequence

Each numbered item is a separate, revertible pull request. Unsupported Electron
majors may be used locally to narrow regressions, but must never be merged or
released.

1. **Land coverage on Electron 14.** Add tests for external-link denial,
   preload API shape and isolation, keyboard-hook load/start/stop, auto-launch
   fallback, capture-protection preference propagation, and the window-state
   assertions in the matrix below. Preserve all compatibility branches.
2. **Modernize the packaging host.** Move development and CI to Node 22 LTS,
   upgrade Electron Builder to 26.x, keep Electron 14, and build every target.
   `npmRebuild: false` must be removed once a compatible Store addon exists so
   Builder can rebuild native dependencies for Electron.
3. **Resolve native modules.** Verify `uiohook-napi`'s published Windows x64 and
   arm64 binaries in the packaged app. Replace or remove
   `electron-winstore-auto-launch` unless a compatible published build is
   available. Test the APPX startup-task lifecycle (enabled, disabled, and
   disabled-by-user).
4. **Create a candidate lane.** Test exact Electron majors in disposable CI
   branches to locate the first regression. Finish on 44.x and commit the exact
   version plus lockfile. Do not publish artifacts from this lane.
5. **Run automated and manual regression passes.** Run unit, Playwright,
   accessibility, dependency, packaging, install/uninstall, update, signing,
   native-hook, and Windows visibility checks.
6. **Remove old compatibility code last.** Once the external-link test proves
   `webContents.setWindowOpenHandler` is installed and denies navigation, remove
   the Electron 11 `new-window` fallback in `src/main/crossover.js`. Search for
   other version comments/branches and remove them only in the same change as
   their replacement coverage.
7. **Canary, then release.** Canary both installer and portable builds, retain
   the Electron 14 release for rollback, and compare crash/native-load telemetry
   before promotion.

## Breaking-change review checklist

### Transparent `BrowserWindow`

- Keep `transparent: true`, the alpha `backgroundColor`, `frame: false`, and
  `hasShadow: false` together; test both GPU enabled and the project's current
  `--disable-gpu-sandbox` launch path.
- On Windows, confirm pixels outside the crosshair remain transparent (no black
  rectangle), mouse hit-testing/lock behavior is unchanged, resizing preserves
  aspect ratio, and moving between unlike-DPI displays does not blur or offset
  the overlay.
- Treat compositor or DirectComposition changes as release blockers; a DOM
  screenshot cannot prove desktop transparency, so capture the installed app
  against a contrasting desktop with an external screenshot tool.

### Always-on-top and workspace visibility

- Verify `setAlwaysOnTop(true, 'screen-saver', 1)` for crosshair windows and the
  preferences window's `screen-saver` level. Check normal windows, borderless
  games, exclusive/full-screen applications where supported, Win+Tab, Show
  Desktop, lock/unlock, sleep/resume, and display hot-plug.
- `setVisibleOnAllWorkspaces(..., { visibleOnFullScreen: true })` is primarily a
  macOS spaces API. Do not treat it as the Windows always-on-top mechanism;
  Windows acceptance depends on the z-order tests above.

### Content protection

- Exercise `setContentProtection(true/false)` on the main and every shadow
  window. On Windows 10, Electron may fall back to `WDA_MONITOR`; on Windows 10
  version 2004 and later, `WDA_EXCLUDEFROMCAPTURE` can remove the window from
  capture. Validate both the enabled and disabled preference using Snipping Tool,
  Game Bar, Teams/Discord screen share, and OBS display/window capture.
- Never claim content protection is DRM or a security boundary. Its behavior is
  OS- and capture-tool-dependent.

### Preload and context isolation

- Retain `contextIsolation: true`, `nodeIntegration: false`, and the explicit
  preload. The preload must expose narrow functions through `contextBridge`;
  do not expose `ipcRenderer`, Electron objects, or raw event objects.
- Keep `sandbox: false` only while preload code requires Node. Inventory those
  calls and plan sandboxing separately rather than combining it with the runtime
  jump.
- Test that renderer globals exist, privileged Node globals do not, IPC arguments
  are cloned/validated, and every renderer works with the candidate runtime.

### Native modules

- Electron's Node ABI differs from the packaging host ABI. Run Electron Builder's
  dependency rebuild for the exact Electron/architecture combination and inspect
  the packaged ASAR for native files.
- Require x64 and arm64 Windows evidence. Verify whether Electron 44 still
  publishes and supports Windows ia32 before retaining that target; if not, drop
  it from `build:win`, `build:all`, and both Windows Builder YAML files in the
  same packaging PR.

### Electron Builder

- Review configuration validation warnings rather than copying the old config.
  Confirm NSIS/portable naming, APPX manifest merge, icons, signing, publishing,
  updater metadata, native dependency unpacking, and architecture selection.
- Build from a clean checkout with the same Node/npm versions as CI. A successful
  `--dir` build is necessary but does not validate installers, Store identity, or
  auto-update.

## Packaged-build Windows visibility matrix

This matrix requires interactive physical or GPU-backed Windows hosts; it cannot
be truthfully completed in Linux CI. Test the signed, installed artifact and the
portable artifact. Save the Electron version, Windows `winver`, GPU/driver,
monitor topology, scaling, artifact SHA-256, and evidence links for every row.

| OS | Displays | Scaling | Installer | Portable | Capture protection off/on | Result/evidence |
| --- | --- | --- | --- | --- | --- | --- |
| Windows 10 22H2 x64 | 1 × 1920×1080 | 100% | ☐ | ☐ | ☐ / ☐ | pending |
| Windows 10 22H2 x64 | 1 × 3840×2160 | 150% | ☐ | ☐ | ☐ / ☐ | pending |
| Windows 10 22H2 x64 | 1920×1080 + 2560×1440 | 100% + 125% | ☐ | ☐ | ☐ / ☐ | pending |
| Windows 11 23H2+ x64 | 1 × 1920×1080 | 100% | ☐ | ☐ | ☐ / ☐ | pending |
| Windows 11 23H2+ x64 | 1 × 3840×2160 | 150% | ☐ | ☐ | ☐ / ☐ | pending |
| Windows 11 23H2+ x64 | 1920×1080 + 3840×2160 | 100% + 150% | ☐ | ☐ | ☐ / ☐ | pending |
| Windows 11 23H2+ arm64 | native display + external | 150% + 100% | ☐ | ☐ | ☐ / ☐ | pending |

For each cell: launch/relaunch, drag and keyboard-move across every display edge,
center on each display, resize, lock/unlock click-through, create/remove shadow
windows, open/close preferences, run a borderless full-screen app, use Show
Desktop and Win+Tab, lock/unlock Windows, sleep/resume, disconnect/reconnect a
monitor, and repeat the capture tests. Pass only if transparency, physical
centering, size, click behavior, z-order, preferences visibility, and capture
behavior all match Electron 14.

## Minimum Windows release

The proposed runtime baseline is **Windows 10 version 22H2 (build 19045), x64,
or Windows 11 on x64/arm64**. Electron's broad platform floor is Windows 10, but
Windows 10 22H2 is the only Windows 10 release still in scope for this project;
older Windows 10 versions and Windows 7/8/8.1 are unsupported. Reconfirm the
destination major's [supported platforms][windows-support] immediately before
the bump and encode the floor in the website/download metadata and release notes.
Windows 10 itself reached Microsoft end of support on October 14, 2025, so this
is an application compatibility floor, not a promise of Microsoft security
support.

[releases]: https://www.electronjs.org/docs/latest/tutorial/electron-timelines
[windows-support]: https://www.electronjs.org/docs/latest/tutorial/support#supported-platforms
