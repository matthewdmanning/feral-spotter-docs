# Handoff: Firebase upload test-drive, continued — 2026-08-17 (session 2)

Supersedes `2026-08-17-firebase-upload-test-drive-handoff.md` (same topic,
same day — read that one first for the session-1 bucket/412 investigation;
this doc picks up where it left off and does not repeat its content).

Git state: `issue-279-firebase-emulator-wiring`. One uncommitted change:
`src/config/location.ts` (`LOCATION_STALE_THRESHOLD_MS`: 5min → 15s, see
below). Nothing else dirty.

PR #281 (session-1's emulator-wiring work) **merged** during this session,
closing issue #279. Two new issues filed this session:

- **#282** — annotate screen confirm button does nothing on the last photo.
  Reported by Matthew mid-session, not investigated (out of scope for this
  test-drive thread). Report only, `needs-triage`.
- **#283** — `watchPositionAsync`'s location request never reaches Android's
  OS-level location manager at all. Root-caused via `adb shell dumpsys
  location` (transport-independent — see below): across the whole session,
  `com.mmanning.feralspotter` never once appears as a listener on any
  provider (`fused`/`gps`/`network`/`passive`), while other installed apps
  correctly do. Not a GPS-signal/accuracy issue — the request never leaves
  the JS→native bridge. Full evidence and suggested next steps are in the
  issue body, not repeated here.

## What this session actually resolved

**Root cause of the blank/white Google Maps screen, found and partially
fixed:**

1. First hypothesis (wrong, but instructive): thought a stale Metro JS
   bundle was the cause. Tried `expo start -c` (clear cache) — didn't fix
   it, and also didn't fix the original wrong-bucket upload symptom from
   session 1 either. Ruled out.
2. Second hypothesis (also wrong): thought a stale *native* build
   (`npx expo run:android`) was the cause, since `MAPS_API` in
   `.agents/secrets.md` had been rotated 2026-08-16 but the installed APK's
   `AndroidManifest.xml` still had the old, **deleted** key baked in
   (confirmed via `adb logcat`'s `Authorization failure` error, which prints
   the exact key and cert fingerprint it's checking). Ran `npx expo
   run:android` — rebuilt and reinstalled, but the manifest **still** had
   the old key. Ruled out plain rebuild as sufficient.
3. **Actual root cause**: `android/` is gitignored (confirmed via `git
   check-ignore`) — it's an Expo Continuous Native Generation (CNG)
   artifact, not committed source. `npx expo run:android` reuses whatever
   `android/` already exists on disk without regenerating it from
   `app.config.js`; only `npx expo prebuild` does that. This local
   machine's `android/` folder predates the 2026-08-16 `MAPS_API` rotation,
   so it kept baking in the dead key no matter how many times it was
   rebuilt.
4. **Fix applied**: `npx expo prebuild --clean --platform android`,
   confirmed the regenerated `AndroidManifest.xml` now has the current
   correct key (`***REMOVED-ROTATED-MAPS-KEY***`, matches
   `.env.local`'s `MAPS_API`). This part is done and verified.

**Not yet verified end-to-end**, because the follow-up `npx expo
run:android` (to actually build+install with the regenerated `android/`)
**failed**:

```
BUILD FAILED in 3m 38s
```

Two native modules hit a `clang` crash mid-link/compile:
`react-native-skia`'s `buildCMakeDebug[arm64-v8a]` (`linker command failed
due to signal`) and `react-native-reanimated`'s `buildCMakeDebug[arm64-v8a]`
(`clang frontend command failed due to signal`, with a garbled/truncated
argument list in the crash dump — looks like a corrupted command line, not
a normal compile error). Full log: `docs/test-drives/logs/2026-08-17-*.log`
(several files from this session — `expo-run-android.log`, `prebuild.log`,
and the second `run:android` attempt's output, which streamed to a
background task file that may no longer be on disk — check
`bfolxdlnw.output` under the Claude Code task temp dir if still present, or
just rerun).

This looks like a Windows/NDK toolchain resource issue (signal crash on two
different native modules in the same build, both C++/CMake), not related to
the Maps-key fix itself — but it means **the device is still running the
old broken-Maps APK**, and the fix above is unverified live. Likely just
needs a clean retry (`npx expo run:android` again — Gradle build caches
mean it won't repeat the full 3m38s if the crash was transient), possibly
with `--no-build-cache` or a Gradle daemon restart if it recurs.

## Location bug (#283) — investigation notes not in the issue body

The investigation kept getting derailed by a second, independent problem:
Metro's dev-socket (the one that ships device `console.log` lines back to
`expo start`'s terminal, written to `docs/test-drives/logs/2026-08-17-
metro-restart.log`) died silently and repeatedly — at least 3 times, each
time right around when the camera screen / location watch was opened. Not
proven causally related to location code; could easily be this Windows
Metro setup being generally flaky (matches session 1's handoff, which
documented the same "Metro dev-socket died" symptom). A `console.log` added
temporarily to `watchPositionAsync`'s callback (since reverted, no trace
left in the diff) never printed even once — consistent with, but not
independently conclusive of, the `dumpsys location` finding, because the
transport was unreliable throughout.

**If continuing this**: don't trust the Metro log-forwarding socket for this
investigation. Either (a) get a reliable `console.log`-independent signal —
persist findings to `AsyncStorage` and read them back via the same
`adb exec-out run-as com.mmanning.feralspotter cat databases/RKStorage`
trick used successfully this session to inspect `submission-store`, or
(b) use a proper debugger (Flipper/Chrome DevTools) attached directly,
bypassing Metro's own forwarding.

## Also fixed this session (small, unrelated to the above)

`LOCATION_STALE_THRESHOLD_MS` (`src/config/location.ts`) changed from 5
minutes to 15 seconds, per Matthew's explicit complaint ("10 minutes is
stupidly long"). This constant drives both the single-watch-attempt timeout
*and* the automatic-retry delay, so an unresolved fix now rechecks roughly
every 30s instead of every 10min. Test suite
(`src/lib/__tests__/location.model.test.ts`) re-run and still passes — it's
written symbolically against the constant, not a hardcoded duration.
**Uncommitted.**

## Device/environment state right now

- Pixel 7 physical, `2A151FDH200HY4`, USB-connected, `adb reverse
  tcp:8081` set.
- Metro running in background (`expo start --android --clear`, PID chain
  rooted around a `bash.exe`/`node.exe` pair — check `adb reverse --list`
  and `tasklist` if it needs finding again). Confirmed alive and serving
  bundles as of end of session.
- App on-device: still the **old** APK (blank/broken Maps, deleted key).
  The submission draft in `AsyncStorage` (`submission-store` key, pulled via
  the `run-as`/`sqlite3` trick above) has a manually-pinned location
  (`36.2006492, -81.660061`, Western NC — plausible real coordinate from
  `getLastKnownPositionAsync()`, not the Clemson dev-stub) that was set
  blind, without ever seeing the map render, because of the bug this
  session was chasing.
- `docs` submodule state: **untouched this session**, carries forward
  unchanged from session 1's handoff (uncommitted
  `docs/agents/test-driving.md` addition on branch
  `docs-testing-policy-scope`, mixed with unrelated pre-existing uncommitted
  changes — still needs the same careful commit-only-the-relevant-bits
  treatment session 1 flagged, still not done).
- Original session-1 bucket/412 upload bug: **status unclear**. Never
  actually got a clean retest this session — every attempt either hit the
  Maps bug instead (different screen, but blocking the same submission
  flow) or the log transport died before a result could be confirmed. Next
  session should retest upload specifically once the Maps rebuild lands,
  since PR #281's emulator-wiring work is merged and the `BUCKET_URL` fix
  from session 1 should already be live in this build.

## Suggested skills for the next session

- `mattpocock-skills:diagnosing-bugs` — for the native-toolchain build
  crash (`react-native-skia`/`reanimated` clang signal failure) and for
  resuming #283's silent-bridge-call investigation with a non-Metro
  transport.
- No `/grilling` or architecture-review follow-up needed this session —
  session 1's paused architecture review (candidates 2 and 3) is untouched
  and still applies as previously noted; not revisited here.
