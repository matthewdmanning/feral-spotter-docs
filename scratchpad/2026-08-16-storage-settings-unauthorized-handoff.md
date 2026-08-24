> **Stale — historical reference only.** This is a past session's handoff/note; it reflects state at the time it was written, not current fact. Do not treat its findings, plans, or open items as up to date.


# Handoff: storage/unauthorized still failing post-merge/deploy

Branch: `main` (post-merge of PR #276 main repo + PR #10 docs submodule).
Git state: `ab2ef84d22f1b6c2e7d7e3faf4ab315f343118a8` on `main`. Device:
Pixel 7 physical, `arm64-v8a`. Full plan this session came from:
`C:\Users\mattm\.claude\plans\typed-shimmying-leaf.md` (has the PostHog
expected-event table, step-by-step verification checklist — not
duplicated here).

Prior mission doc this session started from (currently **stashed**, not
on disk — see "Uncommitted work" below):
`docs/scratchpad/2026-08-16-storage-settings-firebase-connect-next-steps.md`.

## Current to-do, in order

1. **Wait for user's Firebase Console rules check** (in progress as of
   this handoff) — confirms whether the live ruleset actually matches
   what `firebase deploy --only firestore:rules,storage` pushed this
   session, before doing anything else.
2. If console rules match deployed content: add **temporary** diagnostic
   logging in `src/lib/upload/firebaseUpload.ts` around the `putFile`
   call — log `error.code` and any `error.userInfo`/native error detail,
   not just the message already visible. Reproduce one upload, capture
   the fuller error, then **fully revert** the instrumentation (repo
   convention — see `docs/test-drives/run-notes/2026-08-15-rebuild-
   verification-drive.md`'s "Google Sign-In: confirmed fixed" section for
   the pattern: added, verified, reverted, confirmed clean `git diff`).
3. **Recover the docs submodule stash** (`stash@{0}` in the `docs` repo —
   see "Uncommitted work" below) before it's forgotten. Don't discard.
4. Once root cause is confirmed: ask the user whether to file a GH issue
   for the storage/unauthorized regression (per
   `docs/agents/issue-tracker.md`'s convention of recommending an issue
   before implementation work starts).
5. Ask the user whether to file issues for two other findings from this
   drive (neither filed yet):
   - GPS/location never fired the entire session (zero
     `FusedLocationProvider`/`GnssLocationProvider`/`LocationManagerService`
     logcat lines).
   - Submit gives no visible error feedback when the pre-submit upload
     guard should have caught a permanently-failed upload — silent to
     the user.
6. Once uploads work: finish the PostHog event/property cross-check from
   the plan file's step 3 table (not reached this drive — the upload
   failure took priority).
7. Create `docs/agents/cloudops.md` (plan step 4, still pending) — Firebase
   deploy/ops conventions, see plan file for exact scope.
8. **Do not touch Firestore directly** (console or CLI) this session or
   next — explicit user directive, see `feedback_no_firestore_touching`
   in Claude's memory. Any check that would normally involve reading the
   Firestore counter doc must be reasoned about from `storage.rules`'
   text only, or asked of the user.

## What's been ruled out for storage/unauthorized, and exactly how

| Ruled out | How, precisely |
|---|---|
| Old `allowedTester` gate | `git show d0a34d4 -- storage.rules` diff shows it removed with no replacement; confirmed again by `grep -n "allowedTester" storage.rules` on `main` post-merge — no match. `storage.rules` currently only checks `request.auth != null && isOwner(uid) && (metadata-shape or photo-shape)`. |
| Stale/undeployed ruleset | Ran `npx firebase deploy --only firestore:rules,storage` live, immediately before this drive. CLI output confirmed `storage.rules`/`firestore.rules` both "compiled successfully" and "released" to `project-e3d5659d-bc4f-438f-88c`. |
| Bucket/project mismatch | Cross-checked three independent sources: `firebase.json`'s `storage.bucket` = `"feral-spotter-image-uploads"`; `src/lib/upload/firebaseUpload.ts`'s `BUCKET_URL` constant = same value; `android/app/google-services.json`'s `project_id` = `project-e3d5659d-bc4f-438f-88c` (matches `.firebaserc`/`firebase use`). All agree. (Note: `google-services.json`'s own `storage_bucket` field names the project's *default* bucket, not this one — expected, since the client explicitly addresses the non-default bucket by URL via `getStorage(getApp(), BUCKET_URL)`, not the no-arg default.) |
| Broken/stale auth token | Completed a real Google sign-in on-device via the actual account picker (not `EXPO_PUBLIC_AUTH_MOCK`, which is `false` in `.env.local`), reached Home successfully. Read `src/lib/auth/firebaseAuthProvider.ts` and `GoogleSignIn.ts` source directly: `signInWithCredential` path is unchanged since the already-verified-fixed `d28dad3`; the native Firebase Auth SDK attaches a real ID token to every subsequent SDK call automatically — no manual token plumbing in the upload path to get wrong. |
| Rules logic itself | PR #276's own `npm run test:rules` run (in its PR body's testing checklist) reports 22/22 passing against the real local Firebase rules emulator, for this exact object-path/write shape, before merge. |

**Not yet checked**: whether the live ruleset in Firebase Console's UI
matches what was deployed (user is checking this now); the cross-service
`firestore.get()`/`firestore.exists()` calls inside `isValidPhotoWrite()`/
`isValidMetadataWrite()` behaving correctly against the live default
Firestore database (blocked from checking directly by the Firestore
directive above — can only be inferred from rules text or asked of the
user, not queried).

Full failing error, verbatim, from Metro's log
(`.expo/dev/logs/start.log`):
```
[uploadNewPhoto] <local_id> [Error: [storage/unauthorized] User is not
authorized to perform the desired action.]
  at uploadSubmissionPhoto (src/lib/upload/firebaseUpload.ts:73:12)
```

## Uncommitted work — docs submodule stash

`docs` submodule has a stash, not yet recovered:
`stash@{0}: On storage-settings: wip before storage-settings pointer sync
(2026-08-16)`. Created because `git submodule update --init` (needed to
snap `docs` to `main`'s pinned commit after the PR merges) would have
overwritten uncommitted work. Contents (from the stash's diff, captured
before stashing):

- Modified: `docs/adr/0003-time-capture-model.md`
- Modified: `docs/test-drives/run-notes/2026-08-15-rebuild-verification-
  drive.md` — a continuation log that root-caused the *previous*
  `storage/unauthorized` incident (undeployed Cloud Functions leaving the
  old `allowedTester` claim never set) — valuable prior-art for the
  *current* investigation even though the root cause has since changed
  shape (that gate no longer exists in rules at all).
- Untracked: `docs/scratchpad/2026-08-15-code-simplification-review-
  rebuild-handoff.md`, `docs/scratchpad/2026-08-15-code-simplification-
  review-test-drive-continuation-handoff.md`, `docs/scratchpad/2026-08-
  16-storage-settings-firebase-connect-next-steps.md` (this session's
  own starting mission doc), `docs/test-drives/screen-captures/2026-08-
  15-map-picker-center-pin-check.png`.

`docs` is currently in **detached HEAD** at `3474013...` (main's pinned
commit, post-merge) — the stash was made on `storage-settings` but stash
entries are repo-level, not branch-scoped, so `git -C docs stash pop`
should still work; expect possible conflicts since the base commit moved.
Reconcile carefully, don't force-discard either side.

## Local docs read this session, and why

- [`docs/agents/backend.md`](../agents/backend.md) — current Firebase
  architecture map (Auth, Storage, the one Firestore counter doc, what's
  off-limits). Read first per the mission's "Read first" list.
- [`docs/agents/secrets.md`](../agents/secrets.md) (public classification)
  and main repo's `.agents/secrets.md` (private, gitignored, real
  resource inventory) — checked before treating any found value as a
  leak; not relevant findings this session, just the required check.
- [`docs/adr/0005-firebase-storage-for-uploads.md`](../adr/0005-firebase-storage-for-uploads.md)
  — the 2026-08-16 amendment (raw-uid path scheme) this whole branch
  implements.
- [`docs/agents/test-driving.md`](../agents/test-driving.md) — device
  test-drive conventions: where reports/logs/screen-captures go, the
  PostHog-in-Metro-not-logcat / location-in-logcat split, the
  `adb reverse`/Metro-restart/ABI-mismatch gotchas actually hit this
  session.
- [`docs/agents/issue-tracker.md`](../agents/issue-tracker.md) and
  [`docs/agents/triage-labels.md`](../agents/triage-labels.md) — `gh`
  CLI conventions and the `needs-verification` label, read before this
  session's first `gh issue`/`gh pr` write.
- [`docs/agents/project_instructions.md`](../agents/project_instructions.md)
  — the routing index itself; pointed at `emulator/` for device-driving
  conventions.
- `docs/test-drives/run-notes/2026-08-15-rebuild-verification-drive.md`
  (in the stash, see above) — prior session's continuation log; its
  root-cause section for the *previous* incarnation of this same symptom
  was the main reason today's investigation didn't stop at "rules must
  be wrong again" and instead verified the gate was actually gone.

## Tools, commands, and scripts used, and what they do

- **`gh` CLI** (`gh pr view/list/checks`, `gh issue view/list`) — read PR
  merge/CI state and issue state directly from GitHub; no `gh` writes
  happened this session except this handoff's own recommendation step 4/5
  are still pending, not yet executed.
- **`git`** — `status`/`fetch`/`log`/`checkout`/`pull` for normal repo
  sync; `submodule status`/`update --init` to move the `docs` submodule
  to whatever commit `main`'s tree pins; `stash push -u` to checkpoint
  the docs submodule's uncommitted work before that update (see above);
  `merge-base --is-ancestor` to confirm the submodule pin is reachable
  from `docs`' own `origin/main`, not a dangling commit; `show <sha> --
  <path>` to read one commit's diff to a single file (used to confirm
  `d0a34d4` actually removed the `allowedTester` rule).
- **Firebase CLI** (`npx firebase use`, `projects:list`,
  `deploy --only firestore:rules,storage`) — confirmed the active
  project, then deployed both rulesets live. (`firestore:databases:list`
  was run once early in planning, before the user's no-Firestore-touching
  directive — not repeated after.)
- **`adb`** — `devices -l`/`shell getprop ro.product.cpu.abi` to confirm
  the connected device before building; `reverse tcp:8081 tcp:8081` to
  fix the dev-client's `DevLauncherErrorActivity` (missing USB tunnel
  after a Metro/daemon restart — known issue, see Claude's
  `project_live_run_adb_reverse` memory); `shell am start -a
  android.intent.action.VIEW -d "exp+feral-spotter://…url=localhost:8081"`
  to force the dev client to reconnect over the USB tunnel instead of a
  cached LAN IP that the device couldn't reach; `shell input tap/swipe/
  keyevent` to drive the UI; `shell uiautomator dump` + `pull` to read
  screen state; `logcat -d -t N` to check for native crashes and
  location-provider activity.
- **`npm run android`** (`CI=1 npm run android`, i.e. `expo run:android`)
  — fresh Gradle build + install, run in background since it's a
  multi-minute build; `CI=1` per `test-driving.md`'s Windows note
  (without it, Metro's file watcher can hang on startup).
- **`emulator/tap_text.py` / `tap_desc.py`** — small scripts (not a
  package, just repo-local Python) that grep a pulled `window_dump.xml`
  for a `text=`/`content-desc=` match, walk backward for the nearest
  preceding `clickable="true"` ancestor's `bounds`, and `adb shell input
  tap` its center. Used for most of the onboarding/sign-in flow.
  **Caveat hit this session**: when a label string appears more than
  once in the dump (e.g. a wrapping container's description repeating a
  child's text), the script can grab the wrong/inactive ancestor's
  bounds and silently no-op the tap — fell back to manually parsing
  `bounds="[...]"` from the XML and calling `adb shell input tap`
  directly with the computed center for the camera/location permission
  dialogs and the Cat Form's radio buttons.
- **`advisor`** — consulted once, before finalizing the original plan
  (caught five real gaps: bucket-config check, submodule merge-method
  risk, stale-APK risk, the `email_verified`-isn't-free assumption, and
  the #267-closure question — all folded into the plan before execution).
- **Plan mode / `ExitPlanMode`** — used for the initial plan approval
  round (three iterations: Firestore-scope correction, PostHog-detail
  addition, then direct "1, ping me if you need an uplock" go-ahead that
  effectively exited plan mode by giving a direct execution instruction).
- No Skill-tool invocations this session (no `firebase-basics` etc.) —
  the CLI was already authenticated against the right project, so the
  setup-focused skills in `backend.md`'s approved list weren't needed.
