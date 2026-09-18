# Swift Patterns

Search syntax, logging templates, and fix code patterns for Swift codebases. For root cause mechanisms and signatures, see `root-causes.md`.

---

## Searching

| Goal | Grep pattern |
|---|---|
| Flag setters | `xxxSent = true`, `hasXxx = true`, `var xxx:` |
| Early-exit guard in manager | `==` / `guard newValue != currentValue else { return }` inside `setCurrent*` / `update*` |
| Guard chain early returns | `guard .* else { return }` |
| Notification triggers | `NotificationCenter.default.post` / `addObserver` |
| Entry methods | `handleXxx(` |
| Teardown hooks | `deinit`, `viewWillDisappear`, `viewDidDisappear`, `exitRoom` |

## Logging

```swift
// Log BEFORE the return / skip, with the full decision state in one line.
print("[FeatureName] guard check — key=\(key) value=\(String(describing: actualValue)) expected=\(expectedValue)")

// If you suspect multiple instances of the object:
print("[FeatureName] instance=\(ObjectIdentifier(self)) guard value=\(String(describing: actualValue))")
```

## Fix Patterns

### Stale flag → add reset at teardown + entry

```swift
func exitRoom() {
    resetCarEnterMessageSentState()  // both teardown AND entry reset
}
```

### Same-key early-exit → force side effect after the set call

```swift
turboModeManager.setCurrentRoomId(roomId)
// Force the side effect the singleton's guard would skip — do NOT modify the guard.
turboModeManager.updateGiftEffectsForRoom(roomId, enabled: true)
```

### Async race → pending flag (stored property, or associated object via `objc_setAssociatedObject` in mixed targets)

```swift
private var isCarEnterPending = false

// Fast path: mark pending before returning early.
isCarEnterPending = true

// Slow path (HTTP callback): consume the pending flag and re-run.
if isCarEnterPending {
    isCarEnterPending = false
    sendCarEnterMessageIfPending()
}
```

## Cleanup

Remove diagnostic-only `[weak self]` / `guard let self` capture boilerplate, imports added solely for logging, and stored properties used only for instrumentation.
