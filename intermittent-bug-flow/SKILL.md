---
name: "intermittent-bug-flow"
description: "Systematic flow for debugging intermittent/probabilistic feature failures in voice/social iOS apps (ObjC & Swift) — flag leaks, same-key early-exits, async races, guard chains, duplicate triggers. Use when a feature works sometimes but not always, fails only on re-entry, or fires twice. Don't use for deterministic crashes, compile errors, or reproducible-every-time logic bugs."
metadata:
  version: 2.0.0
  category: debugging
---

# Intermittent Bug Investigation Flow

A structured debugging protocol for **non-deterministic** feature failures in voice/social iOS apps (语聊房 / IM 消息 / 礼物特效 / 座驾动效) — works on first run but not on retry, works for other users but not this one, works in one path but not an equivalent one.

## File Structure

| File | Load it when |
|---|---|
| `SKILL.md` (this file) | Always — the workflow skeleton |
| `references/root-causes.md` | Phase 1 — full details of the 5 candidate root causes |
| `references/objc.md` | Searching / instrumenting / fixing in ObjC code |
| `references/swift.md` | Searching / instrumenting / fixing in Swift code |

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

Read `references/root-causes.md` for the full catalog. Quick-match by signature:

| # | Root Cause | Signature | Details |
|---|---|---|---|
| 1 | Stale in-memory flag | Works first run, fails on re-entry | `references/root-causes.md` §1 |
| 2 | Same-key early-exit in singleton | Switching rooms works, re-entering the same one fails | `references/root-causes.md` §2 |
| 3 | Async race (no retry) | Works on slow network, fails on fast (or vice versa) | `references/root-causes.md` §3 |
| 4 | Guard chain misconfiguration | Works for other users, fails for this user | `references/root-causes.md` §4 |
| 5 | Duplicate trigger paths | Entry log fires ≥2 times per single user action | `references/root-causes.md` §5 |

---

## Phase 2 — Instrumentation Strategy

Add targeted logs **at each guard** in the feature's execution path. Code templates:
- ObjC: `references/objc.md` §Logging
- Swift: `references/swift.md` §Logging

Key principles:
- Log **before** the `return` / skip, not after.
- Include the object pointer if you suspect multiple instances.
- Include the full chain state in one log line to correlate across async boundaries.

Run the reproduction path. The first log that is **missing or has an unexpected value** is the failure point.

### Confirming the root cause before fixing

Do not touch code until the log evidence meets **all** of the following:
- The same guard shows the abnormal value **stably across ≥3 consecutive runs** of the minimal reproduction path.
- The root cause explains **every observed symptom** — including "why it works for other users / other rooms". A cause that explains only part of the symptoms is a false lead; keep instrumenting.

---

## Phase 3 — Fix Patterns

Match the root cause to the appropriate fix strategy:

| Root Cause | Fix Strategy |
|---|---|
| Stale flag not cleared | Add reset call in `exitRoom` / `dealloc`/`deinit` / `viewWillDisappear`. Also reset at entry to be safe. |
| Same-key early-exit in singleton | After the singleton's `setCurrent*` call, add an explicit force-refresh call. Do **not** modify the singleton's guard (it may exist for good reason). |
| Async race (no retry) | Add a pending flag: fast path sets it, slow path checks + consumes it. Reset on exit. |
| Guard chain misconfiguration | Fix the setter of the failing guard; do not bypass the guard itself. |
| Duplicate trigger paths | Ensure exactly one trigger path is authoritative; remove the redundant one. |

Concrete code patterns per language: `references/objc.md` §Fix Patterns / `references/swift.md` §Fix Patterns.

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
1. Remove all diagnostic logs added in Phase 2.
2. Review the final diff: remove imports, properties, and weak-self capture boilerplate that were only needed for the diagnostic path (per-language notes: `references/objc.md` §Cleanup / `references/swift.md` §Cleanup).
3. Confirm the only remaining changes are the minimal fix + explanatory comments.

---

## Reference: Real Case (EP Car Enter Animation)

**Symptoms**: User's own car-enter animation works on first app launch + room entry; fails on exit → re-enter same room; works when entering different rooms alternately.

**Root causes found**: root cause #2 (same-roomId early-exit in `TurboModeStateManager` → stale `giftEffects` blocked `handleCarEnter:`) **plus** root cause #3 (NIM WebSocket beat HTTP `getUserInfo` → `ep_sendCarEnterMessageIfNeeded` returned early, no retry).

**Anti-pattern avoided**: Do not manually trigger playback in the send-completion callback — the SDK's local-echo path (`sendMessage:didCompleteWithError:` → `handleNIMCustomMessage:` → `handleCarEnter:`) already handles it. Adding a second trigger causes double playback (root cause #5).
