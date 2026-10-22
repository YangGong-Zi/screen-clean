# Screen Clean

Raycast extension for cleaning your screen and keyboard. It covers every display with a pure black
window and, optionally, locks the keyboard globally. It works on **macOS** and **Windows**. No data
is collected or sent anywhere.

## Commands

| Command                            | What it does                                               |
| ---------------------------------- | ---------------------------------------------------------- |
| **Start Cleaning Mode**            | Blacks out all displays. Press `Esc` 5 times to exit.      |
| **Stop Cleaning Mode**             | Terminates the overlay immediately.                        |
| **Grant Keyboard Lock Permission** | macOS only: requests Accessibility permission.             |

The session always ends automatically (default 60 seconds, configurable in the extension
preferences). The "until manually stopped" option is hard-capped at 30 minutes as a fail-safe.

## How it works

Raycast commands cannot draw their own full-screen windows, so the extension ships a native helper
per platform. The Raycast command spawns it detached and exits; the helper keeps running.

### macOS

Helper: `assets/ScreenCleaner` — universal binary (arm64 + x86_64), built from `native/main.swift`.

```
Raycast no-view command
  └─ spawn detached → ScreenCleaner
       ├─ one borderless black NSWindow per display, level = CGShieldingWindowLevel()
       ├─ optional CGEventTap that swallows keyboard events globally
       ├─ 5 Esc presses (counter on screen) / auto-exit timer / SIGTERM → fade out and quit
       └─ prevents display sleep while cleaning
```

- Strict keyboard lock requires Accessibility permission. Without it, the helper falls back to
  passive mode (the overlay still receives and swallows keys while it is focused) and shows a hint.
- The helper is copied to the extension's support directory before every run, so its path and code
  signature stay stable (important for the Accessibility permission).

### Windows

Helper: `assets/ScreenCleaner.ps1`, run with Windows PowerShell 5.1. It compiles a small C# helper
in memory with `Add-Type`:

- one borderless, topmost black WinForms window per monitor (`Screen.AllScreens`)
- optional `WH_KEYBOARD_LL` low-level keyboard hook that swallows every key system-wide
- the same 5 Esc press gesture with an on-screen counter, plus the auto-exit timer
- prevents display sleep with `SetThreadExecutionState`
- `Stop Cleaning Mode` terminates the PowerShell process with `taskkill`

The first start takes an extra second or two because the embedded C# code is compiled on the fly.
If compilation or startup fails, the error is written to `<extension support dir>/screen-cleaner.log`.

Common behavior on both platforms:

- Mouse input is not blocked. Pressing `Esc` 5 times works in both modes. The counter (`2/5`,
  `3/5`, …) is shown in the center of the screen at 18 pt; it resets if you pause for more than
  2 seconds.
- The hint text and counter are centered on every display.

## Development

```bash
npm install
npm run dev          # refreshes helpers, then starts Raycast dev mode
```

Other scripts:

```bash
npm run build:helper # copies the Windows helper; on macOS also compiles assets/ScreenCleaner
npm run build        # helpers + Raycast production build
npm run lint
```

Manual tests (each blacks the screen for 2 seconds):

```bash
# macOS
./assets/ScreenCleaner --seconds 2 --no-hint

# Windows PowerShell
powershell.exe -NoProfile -ExecutionPolicy Bypass -STA -File .\assets\ScreenCleaner.ps1 -Seconds 2
```

## Strict keyboard lock setup

**macOS**

1. Run **Grant Keyboard Lock Permission** from Raycast.
2. Enable **ScreenCleaner** in _System Settings → Privacy & Security → Accessibility_. The helper
   path is copied to the clipboard; if it is not in the list, click `+` and paste it.
3. Enable **Strict Keyboard Lock** in the extension preferences.

TCC permissions are bound to the binary's path and signature. Rebuilding the helper with different
contents may require granting the permission again.

**Windows**

No permission is required. Enable **Strict Keyboard Lock** in the extension preferences; the helper
installs a low-level keyboard hook. If the hook cannot be installed, it falls back to passive mode
and shows a hint.

## Safety notes

- This is a cleaning cover, **not a security lock**. The power button, Touch ID / Windows Hello,
  `Ctrl+Alt+Del`, UAC secure desktop and other OS-level mechanisms still work.
- The auto-exit timer is always active. The "until manually stopped" option exits after 30 minutes.
- If the overlay ever gets stuck, the timer will release it. On macOS, as a last resort, force
  restart — strict mode blocks keyboard input globally, so typing `pkill` is not an option while it
  is active. On Windows, `Ctrl+Alt+Del` can still open the secure screen.
- The cursor stays visible.
- On Windows, some antivirus products may flag a process that installs a global keyboard hook. The
  hook is required for strict keyboard lock; passive mode works without it.

## Project layout

```
src/
  start-cleaning.ts           # Raycast commands
  stop-cleaning.ts
  grant-accessibility.ts
  lib/screen-cleaner.ts       # per-platform helper copy/spawn/PID management
native/
  main.swift                  # macOS: overlay, event tap, timers
  build.sh                    # macOS universal build + ad-hoc signing
  windows/ScreenCleaner.ps1   # Windows: WinForms overlay + keyboard hook
scripts/
  build-helper.cjs            # cross-platform helper build/copy step
assets/
  ScreenCleaner               # generated macOS binary
  ScreenCleaner.ps1           # generated copy of the Windows helper
```

## Store publishing

Keep `native/` and `scripts/build-helper.cjs` in the repository so reviewers can verify the sources.
The macOS helper is a prebuilt universal binary; confirm the Raycast Store policy on bundled
binaries before publishing. Rebuild with `npm run build:helper` whenever a helper source changes.
