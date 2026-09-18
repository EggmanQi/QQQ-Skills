---
name: "intermittent-bug-flow"
description: "Systematic flow for debugging intermittent/probabilistic feature failures in voice/social iOS apps (ObjC & Swift) — flag leaks, same-key early-exits, async races, guard chains, duplicate triggers. Invoke when a feature works sometimes but not always, fails only on re-entry, or fires twice."
---

# Intermittent Bug Investigation Flow

A structured debugging protocol for **non-deterministic** feature failures in voice/social iOS apps (语聊房 / IM 消息 / 礼物特效 / 座驾动效) — works on first run but not on retry, works for other users but not this one, works in one path but not an equivalent one. Applies to both ObjC and Swift codebases.

---

## Phase 0 — Scene Convergence (before touching code)

Narrow down the reproduction matrix first. Ask:

| Dimension | Questions |
|---|---|
| **Who** | Only this user? Only new users? Only after specific user actions (e.g. mall config)? |
| **When** | First launch only? First entry only? After re-entry? After background/foreground? |
| **Where** | Same room re-entry vs. different room? Same device vs. others? |
| **What changed** | Any recent user-facing config (vehicle, avatar frame, VIP status)? |

Goal: produce a **minimal reproduction path** (e.g. "open app → enter room A → exit → re-enter room A → feature missing").

---

## Phase 1 — Identify Candidate Root Causes

For voice/social iOS features that are intermittent, the most common causes in priority order:

### 1. Stale In-Memory State / Flag Leak
- A flag or cached value set during session A is **not cleared** when session A ends.
- Next session reads the stale value and short-circuits.
- **Search pattern**: grep for flag setters and check where they are cleared. Look for `dealloc`/`deinit`, `viewWillDisappear`, `exitRoom`-equivalent hooks that should reset them.
- **Signature**: works on first run (clean state), fails on re-entry.

### 2. Same-Key Early-Exit in State Managers
- Shared singletons (e.g. `TurboModeStateManager`) often guard against redundant updates using a "current value unchanged" check.
- When re-entering the same room/session, the key matches, the update is skipped, and downstream observers never fire.
- **Search pattern**:
  - ObjC: `if ([newValue isEqualTo:_currentValue]) return;`
  - Swift: `if newValue == currentValue { return }` / `guard newValue != currentValue else { return }`
- **Signature**: works when switching between different rooms/sessions, fails when re-entering the same one.

### 3. Async Race Condition (A beats B)
- Two async paths (HTTP + WebSocket, or auth + data) converge at a guard. If fast path B arrives before slow path A, it reads `nil`/uninitialized state and returns early — with no retry.
- **Search pattern**: find the guard condition and trace both paths that populate it (e.g. `URLSession` callback vs. NIM WebSocket delegate). Check if the slower path has a "retry if pending" mechanism.
- **Signature**: works when network is slow (HTTP wins), fails when network is fast (WebSocket wins first).

### 4. Guard Chain Misconfiguration
- A feature passes through multiple sequential guards (feature flag → room setting → user setting → mode switch → playback queue). Any single guard returning early silently blocks the feature.
- **Search pattern**: read the full guard chain in order. For each guard, trace what sets it and verify the setter is actually called in your path.
  - ObjC: grep for early `return;` inside `if` checks.
  - Swift: grep for `guard ... else { return }`.
- **Signature**: works for other users (different guard state), fails for this user.

### 5. Duplicate Trigger Paths
- The feature has **more than one trigger source** — e.g. IM SDK local echo (`onRecvMessages:`) plus send-completion callback, or `NSNotification`/`NotificationCenter` plus a direct delegate call, or `enterRoom` invoked from multiple entry points.
- Two paths fire in the same session; the second one reads half-initialized state and returns early — so the *apparent* failure is intermittent even though the trigger is not.
- **Search pattern**: map every call site of the feature's entry method.
  - ObjC: grep `handleXxx:` / `postNotificationName:` / `addObserver:`.
  - Swift: grep `handleXxx(` / `NotificationCenter.default.post` / `addObserver`.
- **Signature**: the feature's entry log appears **≥2 times** for a single user action; or a failing session shows two trigger paths interleaving.

---

## Phase 2 — Instrumentation Strategy

Add targeted logs **at each guard** in the feature's execution path:

```objc
// ObjC
NSLog(@"[FeatureName] guard check — key=%@ value=%@ expected=%@", key, actualValue, expectedValue);
```

```swift
// Swift
print("[FeatureName] guard check — key=\(key) value=\(String(describing: actualValue)) expected=\(expectedValue)")
```

Key principles:
- Log **before** the `return` / skip, not after.
- Include the object pointer (`%p` / `ObjectIdentifier(self)`) if you suspect multiple instances.
- Include the full chain state in one log line to correlate across async boundaries.

Run the reproduction path. The first log that is **missing or has an unexpected value** is the failure point.

### Confirming the root cause before fixing

Do not touch code until the log evidence meets **all** of the following:
- The same guard shows the abnormal value **stably across ≥3 consecutive runs** of the minimal reproduction path.
- The root cause explains **every observed symptom** — including "why it works for other users / other rooms". A cause that explains only part of the symptoms is a false lead; keep instrumenting.

---

## Phase 3 — Fix Patterns

Match the root cause to the appropriate fix:

| Root Cause | Fix Pattern |
|---|---|
| Stale flag not cleared | Add reset call in `exitRoom`/`dealloc`/`deinit`/`viewWillDisappear`. Also reset at entry to be safe. |
| Same-key early-exit in singleton | After the singleton's `setCurrent*` call, add an explicit `updateX:value:` force-refresh call. Do **not** modify the singleton's guard (it may exist for good reason). |
| Async race (no retry) | Add a `pendingFlag`. Fast path sets it; slow path checks + consumes it. Reset `pendingFlag` on exit. |
| Guard chain misconfiguration | Fix the setter of the failing guard; do not bypass the guard itself. |
| Duplicate trigger paths | Trace all paths that trigger the feature. Ensure exactly one path is authoritative; comment out or remove the redundant one. |

---

## Phase 4 — Verification

Test all 5 scenarios before closing:

1. **Clean launch → first entry**: should work.
2. **Exit → re-enter same room**: should work.
3. **Switch to different room → come back**: should work.
4. **Other users' equivalent actions**: should still work (no regression).
5. **Network timing flip**: repeat the path under weak network / airplane-mode-off recovery. A race-condition fix must hold under **both** fast and slow timing — local-only verification is exactly how race bugs "fixed" in dev reappear in production.

For each scenario, confirm the feature fires **exactly once**.

---

## Phase 5 — Cleanup

After verification:
1. Remove all diagnostic `NSLog`/`print` added in Phase 2.
2. Review the final diff: remove any imports, properties, `@kWeakify`/`@kStrongify` (ObjC) or `[weak self]` captures (Swift) that were only needed for the diagnostic path.
3. Confirm the only remaining changes are the minimal fix + explanatory comments.

---

## Reference: Real Case (EP Car Enter Animation)

**Symptoms**: User's own car-enter animation works on first app launch + room entry; fails on exit → re-enter same room; works when entering different rooms alternately.

**Root causes found**:
1. `TurboModeStateManager.setCurrentRoomId:` had a same-roomId early-exit → `giftEffects` retained stale `NO` from previous session → `handleCarEnter:` guard blocked playback.
2. NIM WebSocket connected before HTTP `getUserInfo` response → `ep_sendCarEnterMessageIfNeeded` returned early with no retry path.

**Fixes applied**:
- Force `updateGiftEffectsForRoom:enabled:` after `setCurrentRoomId:` call, bypassing the singleton's own guard.
- Added `kEPCarEnterPendingKey` associated object; HTTP callback path consumes it via `ep_sendCarEnterMessageIfPending`.
- Reset both flags in `ep_resetCarEnterMessageSentState`, called on room exit.

**Anti-pattern avoided**: Do not manually trigger playback in the send-completion callback — the SDK's local-echo path (`sendMessage:didCompleteWithError:` → `handleNIMCustomMessage:` → `handleCarEnter:`) already handles it. Adding a second trigger causes double playback.
