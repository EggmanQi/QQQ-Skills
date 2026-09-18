# Root Cause Catalog

The 5 candidate root causes for intermittent failures in voice/social iOS features, in priority order. Each entry: mechanism → search patterns → signature → typical voice/social scenario.

Language-specific search syntax and code templates: `objc.md` / `swift.md`.

---

## 1. Stale In-Memory State / Flag Leak

**Mechanism**: A flag or cached value set during session A is **not cleared** when session A ends. The next session reads the stale value and short-circuits.

**Search pattern**:
- Grep for the flag's setters (`xxxSent = YES`, `hasShownXxx = true`, `isXxxEnabled = ...`).
- For each setter, find where it is cleared. Check the session-teardown hooks: `exitRoom`-equivalents, `dealloc`/`deinit`, `viewWillDisappear`/`viewDidDisappear`.
- A flag with **zero** clear sites is the prime suspect.

**Signature**: works on first run (clean state), fails on re-entry.

**Typical scenario**: "Car enter message sent" flag persists after exiting a room → re-entering the same room skips re-sending → animation never plays.

---

## 2. Same-Key Early-Exit in State Managers

**Mechanism**: Shared singletons (e.g. `TurboModeStateManager`) guard against redundant updates with a "current value unchanged" check. When re-entering the same room/session, the key (e.g. `roomId`) matches, the update is skipped, and downstream observers never fire.

**Search pattern**:
- Find shared manager `setCurrent*` / `update*` methods.
- Look for the early-exit guard comparing new value to the stored current value (`isEqualTo:` in ObjC, `==` / `guard` in Swift).
- Check which side effects are skipped when the guard returns early — those side effects are what downstream features depend on.

**Signature**: works when switching between different rooms/sessions, fails when re-entering the same one.

**Typical scenario**: Re-entering room A with `roomId == currentRoomId` → `giftEffects` update skipped → car-enter / gift animation blocked by a downstream guard.

**Why the guard exists**: it usually protects against redundant UI refresh or network calls. Do **not** remove it — force the side effect after the call instead (see fix patterns in the platform files).

---

## 3. Async Race Condition (A beats B)

**Mechanism**: Two async paths converge at a guard. If fast path B arrives before slow path A, B reads `nil` / uninitialized state and returns early — with no retry. The outcome depends purely on network timing.

**Search pattern**:
- Find the guard condition (usually `if (!userInfo) return;` / `guard let userInfo else { return }`).
- Trace **both** paths that populate it — e.g. HTTP `getUserInfo` response vs. NIM WebSocket connect callback.
- Check whether the slower path has a "retry if pending" mechanism. If not, every fast-B session silently drops the feature.

**Signature**: works when network is slow (path A wins), fails when network is fast (path B wins first). Testable by switching between Wi-Fi and weak 4G.

**Typical scenario**: NIM WebSocket connects before HTTP `getUserInfo` returns → car-enter message sender reads missing user info and returns early.

---

## 4. Guard Chain Misconfiguration

**Mechanism**: A feature passes through multiple sequential guards: feature flag → room setting → user setting → mode switch → playback queue. Any single guard returning early silently blocks the feature.

**Search pattern**:
- Read the full guard chain **in order**; do not jump to conclusions after the first early return.
- For each guard, trace what sets its value and verify the setter is actually called in your reproduction path.
- Grep for early returns: bare `return;` inside `if` (ObjC), `guard ... else { return }` (Swift).

**Signature**: works for other users (different guard state), fails for this user. Often correlates with a specific config change (VIP expiry, mode toggle).

**Typical scenario**: User's gift effects blocked because a mode-setting fetch failed once and cached `NO`; every subsequent entry hits the cached guard.

---

## 5. Duplicate Trigger Paths

**Mechanism**: The feature has **more than one trigger source**. Two paths fire in the same session; the second reads half-initialized state and returns early — so the *apparent* failure is intermittent even though the trigger is deterministic.

**Search pattern**:
- Map every call site of the feature's entry method:
  - ObjC: grep `handleXxx:` / `postNotificationName:` / `addObserver:`.
  - Swift: grep `handleXxx(` / `NotificationCenter.default.post` / `addObserver`.
- Draw the trigger graph: send-completion callback vs. SDK local echo (`onRecvMessages:`) is the classic IM pair.

**Signature**: the feature's entry log appears **≥2 times** for a single user action; or a failing session shows two trigger paths interleaving.

**Typical scenario**: Sending a car-enter message triggers playback twice — once via manual callback trigger, once via the SDK's local echo — producing double animation, or a crash-on-second-play that looks "random".

**Priority note**: this cause is diagnosed cheapest (just count entry logs), so check it early even though it is listed last.
