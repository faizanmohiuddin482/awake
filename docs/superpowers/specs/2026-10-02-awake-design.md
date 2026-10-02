# Awake — design

Date: 2026-10-02
Status: draft, awaiting review

## Purpose

A macOS menu bar app that keeps the Mac awake on demand, so a Claude Code
Remote Control session stays reachable from a phone while the laptop sits at
home. Built for Faizan and his team, shared through GitHub.

Success: one click keeps the Mac awake, one click lets it sleep, and the menu
bar icon always shows which state it is in.

## Non-goals

- No awareness of Claude sessions (no detection, no listing, no control).
- No closed-lid mode. macOS sleeps on lid close unless on power with an
  external display; the README says to leave the lid open.
- No App Store release, no auto-update, no settings window.
- No support below macOS 26.

## Design principles

Follow Apple's Human Interface Guidelines for menu bar extras:

- Template SF Symbol icon (monochrome, adapts to light/dark/tinted bars).
- Clicking the icon opens a standard menu, not a custom panel. On macOS 26 a
  standard menu renders in Liquid Glass with no custom code.
- System font and system controls only; no custom colours or materials.
- Accessibility (VoiceOver, keyboard, Reduce Transparency) comes from the
  standard menu; nothing custom to break it.

## User interface

Menu bar icon:

- Awake: `cup.and.saucer.fill`
- Not awake: `cup.and.saucer`
- Accessibility label: "Awake: on" / "Awake: off"

Menu:

```
✓ Keep Mac Awake            ⌘K
────────────────
  Duration           ▸  ✓ Until Turned Off
                          1 Hour
                          4 Hours
────────────────
  Open at Login
  Quit Awake                ⌘Q
```

- "Keep Mac Awake" is a checkmark toggle.
- While a timed duration is running, the toggle reads
  "Keep Mac Awake (until 14:30)" using the user's locale time format.
- Duration is a checkmark submenu; changing it while awake restarts the
  timer from now. Choosing a duration while off does not turn it on.
- Duration choice persists across launches (UserDefaults). The awake state
  itself does not persist: the app always launches with the Mac allowed to
  sleep.
- No Dock icon (`LSUIElement = YES`).

## Architecture

Swift Package (no Xcode required; Command Line Tools + Swift 6), SwiftUI
`MenuBarExtra` with `.menuBarExtraStyle(.menu)`. Three units:

1. `SleepBlocker` — wraps IOKit power assertions.
   - `start()` creates an `IOPMAssertionCreateWithName` assertion of type
     `kIOPMAssertionTypePreventUserIdleDisplaySleep` (same as
     `caffeinate -d`), reason "Awake: keeping Mac awake".
   - `stop()` releases it. Both are idempotent.
   - `isActive` reflects whether an assertion is held.
   - Releasing in `deinit` and on app termination; the kernel also drops the
     assertion if the process dies, so the Mac can never be stuck awake.
2. `AwakeModel` — `@Observable` state: `isAwake`, `duration`, `endsAt`.
   - Turning on calls `SleepBlocker.start()` and, for a timed duration,
     schedules a single `Task.sleep` until `endsAt`, then turns off.
   - Turning off cancels the timer task and calls `stop()`.
   - Clock and blocker are injected so tests run without real assertions or
     waiting.
3. `AwakeApp` — the `MenuBarExtra` scene and menu items; no logic beyond
   binding to `AwakeModel`.

Open at Login uses `SMAppService.mainApp.register()` / `unregister()`; the
menu item's checkmark reads `SMAppService.mainApp.status == .enabled`.

## Error handling

- If `IOPMAssertionCreateWithName` fails, the toggle stays off and the menu
  shows a disabled line "Couldn't keep the Mac awake" until the next attempt.
- If login-item registration fails (commonly when the app is not in
  /Applications), the checkmark stays off; the README tells users to move the
  app to /Applications first.

## Testing

- Unit tests (`swift test`) for `AwakeModel` with a fake blocker and a fake
  clock: on/off, timed expiry, duration change restarts the timer, failure
  path leaves state off, quit releases.
- `SleepBlocker` integration check: after `start()`, `pmset -g assertions`
  lists the "Awake" assertion; after `stop()`, it is gone.
- Manual check on macOS 26: icon swaps, menu renders in system glass, VoiceOver
  reads the toggle, Quit lets the display sleep.

## Build and distribution

- `scripts/build-app.sh` runs `swift build -c release`, assembles
  `Awake.app` (binary + `Info.plist` with `LSUIElement`,
  `LSMinimumSystemVersion = 26.0`, bundle id `com.qureos.awake`), and ad-hoc
  signs it with `codesign -s -`.
- GitHub release ships a zipped `Awake.app`.
- v1 is not notarized: first launch needs right-click → Open (README covers
  it). Developer ID signing + notarization ($99/yr) can be added later
  without code changes.

## Project layout

```
awake/
├── Package.swift
├── Sources/Awake/
│   ├── AwakeApp.swift
│   ├── AwakeModel.swift
│   └── SleepBlocker.swift
├── Tests/AwakeTests/AwakeModelTests.swift
├── Resources/Info.plist
├── scripts/build-app.sh
└── README.md
```
