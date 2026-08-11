# Handoff — Issue #249: Photo library access must be an explicit yes

Work from `C:\GitHub\feral-spotter` (main checkout) — not a worktree. This handoff was written from a worktree session that will not persist.

## Task

Implement #249: https://github.com/matthewdmanning/feral-spotter/issues/249

Full spec, acceptance criteria, and implementation decisions are in the issue body — don't duplicate here, read it there. In short: `useLibraryPhotoPicker.pickFromLibrary()` currently calls `ImagePicker.launchImageLibraryAsync()` directly with no explicit permission check (confirmed by reading the file live, 2026-08-11 — see below). Any outcome short of an actual selection (user cancels, OS denies) collapses to the same silent no-op. #249 wants an explicit `getMediaLibraryPermissionsAsync`/`requestMediaLibraryPermissionsAsync` check before invoking the picker, with a distinct denial message reusing the existing Permission Blocked / Settings-recovery pattern.

## Status as of this handoff

Not started. Confirmed live by reading current source, 2026-08-11:

```
src/hooks/useLibraryPhotoPicker.ts — pickFromLibrary():
  const result = await ImagePicker.launchImageLibraryAsync({...})
  if (result.canceled || result.assets.length === 0) return
```

No permission-check code exists anywhere in this file or elsewhere in `src/`.

## Key files

- `src/hooks/useLibraryPhotoPicker.ts` — the file to change (add the explicit check before `launchImageLibraryAsync`).
- `src/screens/consent/index.tsx` — has the existing Permission Blocked gate / Settings-recovery pattern (`Linking.openSettings()`) the issue says to reuse rather than invent new copy/UI.
- `src/hooks/useCameraAccess.ts` — another existing `Linking.openSettings()` call site, useful as a second reference for the established pattern.
- `docs/agents/domain.md` — "Library pick" glossary entry, useful context on how this concept fits the domain model. Read this file's Language section before starting per repo convention (`docs/agents/domain.md`'s own instructions).
- Related prior work referenced by the issue: #66, #237 (camera/location consent pattern this extends), #145/#146 (the "check() only, never request() on every call" pattern to mirror).

## Branch strategy — needs your judgment

`issue-249-photo-library-consent` already exists (local + pushed to origin) but its current tip contains **two unrelated bug fixes**, not #249's work — see PR #252 (draft, open): https://github.com/matthewdmanning/feral-spotter/pull/252. That PR's commits are about a stale submission-cache bug found while device-testing #224, folded onto this branch on explicit direction from the user in a prior session — not #249 itself.

Before starting: either continue on `issue-249-photo-library-consent` (PR #252 will presumably merge first, or #249's work stacks on top), or ask whether to branch fresh off `main` instead. Don't assume either way.

## Device-testing environment gotchas (all confirmed live this session, Pixel 7 physical device)

- This worktree (not present in your session) was missing `.env.local` and `google-services.json` — both gitignored, not copied when a worktree is created. If you're in a fresh worktree, copy both from the main checkout (`C:\GitHub\feral-spotter\.env.local`, `C:\GitHub\feral-spotter\google-services.json`) before running Metro. **Do not** paste their contents into any committed file or issue/PR text — they contain live keys.
- `.env.local` sets `EXPO_PUBLIC_AUTH_MOCK=true` — sign-in during manual testing is mocked (any provider tap grants a stub user), not real Google auth.
- Metro must run with `CI=1` on this Windows machine, or the file watcher can hang on startup. `CI=1` also disables Fast Refresh — **a source edit needs a full Metro process restart to be picked up, not just an app reload or even a fresh `am start`**. This cost real time in the prior session (tested stale code by accident more than once).
- `adb reverse tcp:8081 tcp:8081` is needed after any Metro restart or long device-lock period — the tunnel can silently drop.
- After a Metro restart, Expo's dev-launcher drops back to its own project-picker screen instead of auto-reconnecting — has to be manually reconnected (tap "Recently Opened" if present, or type `127.0.0.1:8081` into the connect field and tap Connect).
- `adb shell` commands with a `/sdcard/...` path need `MSYS_NO_PATHCONV=1` prefix in git-bash, or the path gets mangled into a Windows path — applies to `screencap`, `pull`, and any other shell command taking a Unix path, not just `pull`.
- UI verification approach: `adb shell uiautomator dump` + `MSYS_NO_PATHCONV=1 adb pull`, parse `content-desc="..."`/`text="..."`/`bounds="[x1,y1][x2,y2]"`, tap the bounds center. Do not use `adb shell screencap` for verification (repo convention — screenshots only when explicitly asked); uiautomator dumps + logcat/Metro-log are the standard technique here.
- To inspect app state (submission cache, photo pool), pull the AsyncStorage-backed SQLite db: `adb exec-out run-as com.mmanning.feralspotter cat /data/data/com.mmanning.feralspotter/databases/RKStorage > local.db`, then query with Python's `sqlite3` module — `catalystLocalStorage` table, `key`/`value` columns. A raw `cat` of a live SQLite file can occasionally miss very recent writes; if a result looks stale, prefer adding a temporary `console.log` in the relevant code path and reading Metro's log directly over re-pulling and re-guessing.

## Suggested skills

- **advisor** — consult before writing the permission-check code (the exact `getMediaLibraryPermissionsAsync`/`requestMediaLibraryPermissionsAsync` branching logic is easy to get subtly wrong — cross-check against the issue's own Implementation Decisions section and the #145/#146 pattern it says to mirror) and again before declaring done.
- **mattpocock-skills:tdd** — the issue's own Testing Decisions section explicitly calls for an XState model test on the permission-branching logic, mirroring `ConsentScreen.cameraGate.model.test.tsx` / `ConsentScreen.locationGate.model.test.tsx`'s proven pattern. Build the model at design/implementation start, not after.
- **run** — to launch and drive the app for the live device-testing pass; check for a project-specific launch skill first before falling back to the manual Metro/adb steps above.
