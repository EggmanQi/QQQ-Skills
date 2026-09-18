# ObjC Patterns

Search syntax, logging templates, and fix code patterns for ObjC codebases. For root cause mechanisms and signatures, see `root-causes.md`.

---

## Searching

| Goal | Grep pattern |
|---|---|
| Flag setters | `xxxSent = YES`, `hasXxx = YES`, `setXxx:` |
| Early-exit guard in manager | `isEqualTo:` / `==` followed by `return;` inside `setCurrent*` / `update*` |
| Guard chain early returns | `^ *return;` inside `- (void)` / `- (BOOL)` methods |
| Notification triggers | `postNotificationName:` / `addObserver:` |
| Entry methods | `handleXxx:` |
| Teardown hooks | `dealloc`, `viewWillDisappear`, `exitRoom` |

## Logging

```objc
// Log BEFORE the return / skip, with the full decision state in one line.
NSLog(@"[FeatureName] guard check — key=%@ value=%@ expected=%@", key, actualValue, expectedValue);

// If you suspect multiple instances of the object:
NSLog(@"[FeatureName] instance=%p guard value=%@", self, actualValue);
```

## Fix Patterns

### Stale flag → add reset at teardown + entry

```objc
- (void)exitRoom {
    [self resetCarEnterMessageSentState];  // both teardown AND entry reset
}
```

### Same-key early-exit → force side effect after the set call

```objc
[self.turboModeManager setCurrentRoomId:roomId];
// Force the side effect the singleton's guard would skip — do NOT modify the guard.
[self.turboModeManager updateGiftEffectsForRoom:roomId enabled:YES];
```

### Async race → pending flag (associated object)

```objc
static const void *kEPCarEnterPendingKey;

// Fast path: mark pending before returning early.
objc_setAssociatedObject(self, kEPCarEnterPendingKey, @YES, OBJC_ASSOCIATION_RETAIN_NONATOMIC);

// Slow path (HTTP callback): consume the pending flag and re-run.
if ([objc_getAssociatedObject(self, kEPCarEnterPendingKey) boolValue]) {
    objc_setAssociatedObject(self, kEPCarEnterPendingKey, nil, OBJC_ASSOCIATION_RETAIN_NONATOMIC);
    [self sendCarEnterMessageIfPending];
}
```

## Cleanup

Remove diagnostic-only `@kWeakify` / `@kStrongify` pairs, imports (`objc/runtime.h` if only used for the diagnostic), and properties added solely for logging.
