# Test Drive Summary — code-simplification-review (2026-08-15)

Device: Pixel 7, real hardware, dev-client build. Two purposes: (1) verify finding #4 from the ponytail audit (`formatDateTime.ts`'s `Intl.DateTimeFormat` rewrite) on the real Hermes runtime, since Jest runs on V8 and ICU AM/PM spacing differs between engines; (2) walk GH issues #160, #76, #129 and check Home screen button shapes per request. Files in this folder: `nav-log.txt` (action-by-action adb trail) and `screenshot-*.png` (numbered roughly in visit order; some numbers repeat where a tap missed and was retried).

## Confirmed

- **`formatDateTime.ts` (finding #4) is safe to commit.** On-device render: `"Aug 11 2026     12:48 PM"` (`screenshot-02-reports.png`). Multi-space date/time separator intact, no ICU narrow-no-break-space (`U+202F`) substitution around AM/PM — the risk this test drive existed to rule out.
- **Home screen entrypoint buttons match the design decision** in `docs/agents/ux_decisions.md`: circular, side-by-side, icon above label inside the circle, `Take Photos` primary / `Upload Photos` secondary (`screenshot-01-home.png`).
- **Mock auth path works end-to-end**: `EXPO_PUBLIC_AUTH_MOCK=true` grants a stub user on any sign-in tap (Google/Apple/Facebook/email all resolve identically per `src/lib/auth/index.ts`'s `createDevAuthProvider`), then routes through the analytics-consent screen into Home.
- **Cat Form (`Edit Cat`) fully exercised**: every attribute (Age, Sex, Ear Tipped, Hair Length, Pattern, Color, Owned/Domesticated, Health) taps and updates correctly; saving via "Put the Cat in a Box" returns to Submission Details with the summary line updated (`screenshot-05-catform-FILLED.png`).
- **Submission confirmation dialog fires correctly**: "Submit Submission — Submit 1 cat and N photos?" with Cancel/Submit (`screenshot-07-submit-warning-popup.png`) — matches the `Alert.alert` confirm pattern in `docs/agents/ux_decisions.md`.

## Root cause found — blocks further Storage-dependent testing

`@react-native-firebase/storage`'s native module is not linked into this dev-client build. Every upload throws immediately inside `firebaseUpload.ts:66` (`getStorage()`):

```
[uploadNewPhoto] error: You attempted to use a Firebase module that's not installed natively on your project by calling firebase.storage().
Ensure you have installed the npm package '@react-native-firebase/storage', have imported it in your project, and have rebuilt your native application.
```

The error is caught silently, so the UI just shows "Photos Still Uploading" forever (`screenshot-08-photos-uploading-popup.png`) regardless of wait time (retried after ~15s and ~40s, same result — `screenshot-13-retry-finish.png` through `screenshot-16-3rd-attempt.png`). Auth is unaffected (mock bypass, no native Storage dependency). **A native rebuild is required before any Storage-dependent flow — including full submission and #76 — can be tested again.**

## Issue-specific findings

| Issue | Status | Notes |
|---|---|---|
| #160 (library reselect persistence) | Not directly exercised | Picker selection state looked unreliable during my own photo-picker interactions (taps landed on unintended thumbnails while the grid was still loading); worth a dedicated re-check once rebuilt. |
| #76 (timestamp verification) | Blocked | Never reached the upload leg — see root cause above. Cache/local-capture leg was not separately isolated in this run. |
| #129 (no-GPS/no-map fallback) | Partially covered | Map Picker (`screenshot-06-location-tap.png` → `screenshot-06b-map-wait.png`) loaded tiles fine after ~4s (real network delay, not a rendering failure). But the "fixed center pin" described in `docs/agents/domain.md`'s Map Picker definition never rendered, checked twice after dismissing an obscuring toast (`screenshot-07-map-nopin-check.png`). This looks like a separate bug from #129's actual scope (#129 is about the fallback when *no* map/GPS is available at all, not a rendering bug in a map that *does* load) — worth its own issue. |

## Other bugs noticed (out of scope for this task, not fixed)

- Feral Reports screen: status filter chip labels are text-truncated ("In", "Sendin", "Submitte", "Faile" instead of "In Progress"/"Sending"/"Submitted"/"Failed"); header also overlaps the status bar (`screenshot-02-reports.png`).
- A raw debug toast `[uploadNewPhoto] <uuid>...` leaks into the UI on every photo add, and at one point overlapped the Cancel/Set Location buttons on the Map Picker screen.
- "Take Photos" home entrypoint loaded `enabled=false` (stale persisted single-source photo-pool lock from a prior session, per the single-source-by-construction design in `domain.md`) with no visible disabled styling difference from the enabled state — a user could tap it and see nothing happen with no explanation.

## Process notes

- Session started with the app stuck on `DevLauncherActivity`; root cause was the device being asleep/dozing, not a Metro/bundle issue — waking the device and re-firing the dev-client deep link resolved it.
- One navigation slip during this run opened the notification shade briefly; per user instruction the resulting dumps were deleted immediately and not acted on. No personal content is included in this folder's screenshots.
