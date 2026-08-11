# Handoff — Issue #253: Camera crashes on resume after long background/lock

Work from `C:\GitHub\feral-spotter` (main checkout) — not a worktree. This handoff was written from a worktree session that will not persist.

## Task

Fix #253: https://github.com/matthewdmanning/feral-spotter/issues/253

Full repro, stack trace, and root-cause analysis are in the issue body — don't duplicate here, read it there. In short: `src/screens/camera/index.tsx`'s `<Camera isActive .../>` hardcodes `isActive` to `true`, never bound to screen-focus or app-foreground state. Android reclaims camera hardware from backgrounded apps regardless; without `isActive` toggling off, vision-camera never releases its side of the session, so on resume it tries to reconfigure Camera2 streams against a device the OS already reclaimed, and it throws uncaught.

## Status as of this handoff

Not started, no branch created. Root cause confirmed by reading `src/screens/camera/index.tsx:111` live — not yet fixed.

## Key file

`src/screens/camera/index.tsx` — line ~111, the `<Camera isActive .../>` JSX prop.

## Fix direction — needs verification before implementing

Bind `isActive` to actual focus + foreground state, e.g. `isActive={isFocused && appState === 'active'}`. Two things to check before writing this, not yet verified this session:

1. **Focus hook availability**: this is an Expo Router app (`expo-router: ~56.2.11`), not a direct `@react-navigation/native` consumer (`@react-navigation/native` isn't in `package.json`'s direct dependencies, only transitive via expo-router). `useIsFocused()` may still resolve since expo-router is built on React Navigation under the hood, but this wasn't confirmed — check whether it's safely importable, or whether Expo Router has its own recommended focus-detection hook, before assuming the exact hook name.
2. **vision-camera's own recommended pattern**: check `react-native-vision-camera`'s docs (Context7 MCP is configured for this repo — use it) for whether it ships a purpose-built hook/util for this exact `isActive` lifecycle binding, rather than hand-rolling `AppState` + focus tracking from scratch.

## Repro details not yet nailed down

Found incidentally, 2026-08-11, Pixel 7 — the device sat locked mid-session for an unmeasured period (long enough for Android to reclaim the camera) while the Camera screen was still the foregrounded route underneath. Exact minimum lock duration to trigger this is unknown — not measured, only observed once. If a tight repro loop matters for verifying the fix, that timing would need establishing (or the fix verified via `isActive` state actually changing in Metro logs / a temporary trace, same technique used to verify the #224 fix — see `docs/test-drives/run-notes/2026-08-11-issue224-library-pick-drive.md` on `main` if it's been merged from PR #252, otherwise on branch `issue-249-photo-library-consent`).

## Device-testing environment gotchas (all confirmed live this session, Pixel 7 physical device)

- A fresh worktree will be missing `.env.local` and `google-services.json` — both gitignored, not copied automatically. Copy both from the main checkout (`C:\GitHub\feral-spotter\.env.local`, `C:\GitHub\feral-spotter\google-services.json`) before running Metro. **Do not** paste their contents into any committed file or issue/PR text — they contain live keys.
- `.env.local` sets `EXPO_PUBLIC_AUTH_MOCK=true` — sign-in during manual testing is mocked, not real Google auth.
- Metro must run with `CI=1` on this Windows machine, or the file watcher can hang on startup. `CI=1` also disables Fast Refresh — **a source edit needs a full Metro process restart to be picked up, not just an app reload**. Confirmed the hard way in the prior session.
- `adb reverse tcp:8081 tcp:8081` needed after any Metro restart or long device-lock period.
- After a Metro restart, Expo's dev-launcher drops back to its own project-picker screen — reconnect manually (tap "Recently Opened" if present, or type `127.0.0.1:8081` into the connect field).
- `adb shell` commands with a `/sdcard/...` path need `MSYS_NO_PATHCONV=1` prefix in git-bash.
- UI verification: `adb shell uiautomator dump` + `MSYS_NO_PATHCONV=1 adb pull`, parse `content-desc`/`text`/`bounds` attributes, tap bounds center. Avoid `adb shell screencap` for state verification (repo convention); prefer uiautomator dumps + Metro log / logcat.
- To reproduce the actual bug: open Camera, lock the device (`adb shell input keyevent KEYCODE_SLEEP` or leave idle), wait, then wake + resume the app foreground and watch for the crash. Consider adding a temporary `console.log` around the `isActive` prop / `AppState` transitions to confirm the fix actually toggles the value at the right moments, same technique that resolved a confusing device-behavior mystery on #224's fix (see run-note referenced above) — cheaper than guessing from blind adb cycles.

## Suggested skills

- **advisor** — consult before implementing the `isActive` binding (the exact focus-hook choice matters, see open questions above) and again before declaring done.
- **context7-mcp** — fetch current `react-native-vision-camera` docs specifically for `isActive` / camera lifecycle guidance before writing the fix; don't rely on training-data assumptions about the API.
- **mattpocock-skills:diagnosing-bugs** — if the fix doesn't fully resolve the crash on first attempt (e.g., focus hook doesn't fire in the expected order under Expo Router), this skill's diagnosis loop fits better than ad hoc device-cycle guessing.
- **run** — to launch and drive the app for the live repro/verification pass.
