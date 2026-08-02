# Hotkeys (in-house, zero dependencies)

`Core/HotKey/` holds:

- `KeyShortcut` — Sendable model, Carbon keycode + modifiers, layout-aware glyphs via `UCKeyTranslate`.
- `HotKeyCenter` — the Carbon `RegisterEventHotKey` layer, pausable.

`HotKeyManager` owns both: persistence, conflict lookup, and dispatch.

## Persistence

Shortcuts persist as JSON strings under `KeyboardShortcuts_<name>` UserDefaults keys — a **legacy
format** from the removed KeyboardShortcuts package, kept so old bindings survive. The set of bound
bundle IDs lives in `boundAppBundleIDs` and is re-registered on launch. System Settings panes use
`boundPaneBundleIDs`; custom commands use their stable UUIDs in `boundCustomCommandIDs`.

System actions and window commands are the fixed-catalog case: they persist under
`KeyboardShortcuts_systemActionHotkey.<raw-id>` and `KeyboardShortcuts_windowCommandHotkey.<raw-id>`
and need **no** bound-ID index, because `start()` and `conflictOwner` can just iterate `allCases` and
`register` no-ops on an unbound item. A registered window-command shortcut still runs nothing while the
feature switch is off — `AppCore.runWindowCommand` re-checks it (see
[window-management.md](window-management.md)); a system-action shortcut likewise goes through
`AppCore.runSystemAction(id:)`, so the confirmation gate holds for a hotkey exactly as it does for the
palette.

## Recorder

The settings recorder (`Features/Settings/ShortcutRecorder.swift`) is deliberately **not** a focusable
control: the active recorder is `HotKeyManager.recordingAction` state, and keys are captured by local
NSEvent monitors while all Carbon registrations are paused.
