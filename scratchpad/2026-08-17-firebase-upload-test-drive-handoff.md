# Handoff: Firebase Storage upload test-drive — 2026-08-17

Git state: `issue-279-firebase-emulator-wiring` @ `92d59e899c5e966ce892c74f033b05239a5f793f`.
`docs` submodule on branch `docs-testing-policy-scope` @ `8c2eb54` (2 commits ahead
of `main` — see "docs submodule state" below).

## What this session was

Two threads, in order:

1. `/improve-codebase-architecture` on the Firebase storage/auth/upload
   cluster — produced an HTML report and 3 candidates. Only partially
   resolved (see "Architecture review" below).
2. User pivoted to: "prep for a physical device test drive... check if
   upload works." This became the bulk of the session — a long debugging
   chase, not yet resolved. **This is what "everything about this test
   drive" refers to and what the next session should pick up.**

## Test-drive investigation — current state: UNRESOLVED, narrowed to two candidate fixes

### What's confirmed true

- Live `storage.rules` on the default bucket (`project-e3d5659d-bc4f-438f-88c.firebasestorage.app`)
  matches local, byte-for-byte (verified via Firebase Rules API, not guessed).
- Upload fails every time with the **same 412** error as the
  2026-08-17-photo-upload-retest.md note documented before this session:
  `A required service account is missing necessary permissions` — but
  hitting `gs://feral-spotter-image-uploads` (the **original** bucket),
  not the default bucket.
- `src/lib/upload/firebaseUpload.ts:41`'s `BUCKET_URL` constant on disk
  **is** the default bucket (correct, per commit `b255d7e` from earlier
  today). `google-services.json`'s `storage_bucket` field also already
  says the default bucket. Both agree — yet the running app still uploads
  to the old bucket.
- `getStorage(getApp(), 'gs://...')` is confirmed via context7
  (`/invertase/react-native-firebase` docs) as the correct way to target a
  non-default bucket — not an API-usage bug.
- Tried twice to force a fresh JS bundle (dev-menu Reload after Metro
  reconnected) — same wrong-bucket behavior both times.
- Installed APK's `lastUpdateTime` (`adb shell dumpsys package`):
  **2026-08-16 18:33:02** — one day before the `BUCKET_URL` fix commit.
  Leading theory: stale native build and/or stale Metro transform cache,
  not a code defect.

### Left off asking the user

Two options, neither attempted yet:
- (a) restart Metro with `-c` (clears transform cache) and reload — faster,
  try first if it's a Metro cache issue.
- (b) full native rebuild (`npx expo run:android`) — the sure fix if the
  native build itself is stale.

Both require the user's OK since they touch their long-running dev-server
process / take real rebuild time.

### Also surfaced, not yet resolved

- User separately reported the analytics dashboard showing zero activity
  "not even auth." This turned out to be a mix of: (1) I initially
  misattributed it to PostHog rather than Firebase Analytics — corrected
  mid-session; (2) once Metro reconnected, PostHog *was* confirmed firing
  correctly (`photo_captured`, `camera_opened` events with a real
  `distinct_id` reached `flush` in the local log). Whether the user's
  Firebase Analytics dashboard concern is now moot or still real is
  **unconfirmed** — Firebase Analytics DebugView
  (`debug.firebase.analytics.app` prop, now enabled per below) was never
  actually checked this session.
- Two Firebase buckets now exist in play (`feral-spotter-image-uploads`,
  the default bucket) with inconsistent identifiers across
  `firebaseUpload.ts` (client), `functions/src/index.ts` (Cloud Functions
  triggers, still bound to `feral-spotter-image-uploads`), ADR-0005, and
  `backend.md`. This was flagged by the advisor earlier in the session as
  a real functional risk (Storage triggers / photoCount counter go dead if
  client and functions point at different buckets) — not yet reconciled.
  Whichever bucket wins the fix above, functions/ADR-0005/backend.md need
  updating to match.

### Standing change made to the test-drive process itself

`docs/agents/test-driving.md` now has a "Before every physical-device/
emulator run: enable Firebase debug logging" section (adb `setprop`
commands for `FirebaseStorage`/`FirebaseAuth`/`FirebaseApp` DEBUG,
`FA`/`FA-SVC` VERBOSE, `debug.firebase.analytics.app`). **This is
uncommitted** — sitting in the `docs` submodule working tree, not yet
staged, on top of a pile of *other* pre-existing uncommitted changes in
that submodule (see below) that are not this session's and should not be
touched.

### Device state right now

- Pixel 7 physical, `2A151FDH200HY4`, connected via USB.
- `adb reverse tcp:8081` set.
- Firebase debug props set via `adb shell setprop` (reset on device
  reboot, not persistent — re-set from `test-driving.md` if the device
  reboots).
- App is mid-draft on-screen (last capture attempt, photo local_id
  `a3a467a8-8b36-4c76-819d-05b55903a1c0`, failed upload). Safe to Reset
  from the Submission Details screen before the next attempt.
- Metro is running and connected (was dead for most of this session until
  reconnected via the Dev Launcher's "recently opened" entry + a
  dev-menu Reload — see the full conversation transcript if the "Metro
  dev-socket died" symptom recurs, the diagnosis steps are all there).

## Architecture review — paused, not abandoned

Report: `C:\Users\mattm\AppData\Local\Temp\architecture-review-20260817-164224.html`
(OS temp dir, not in repo). Progress log:
`docs/scratchpad/2026-08-17-architecture-review-progress.md`.

Three candidates were proposed for the storage/auth/upload cluster:

1. **Strong** — collapse the duplicated Firebase emulator-connect-and-verify
   procedure (`firebaseUpload.ts:51-68` vs `firebaseAuthProvider.ts:64-76`)
   into one shared seam. User confirmed: proceed with this one.
2. **Dropped** — deepen `src/lib/auth/index.ts`'s provider-selection logic.
   Advisor's call, user agreed: it's one boolean OR, not real complexity;
   deletion test already passes. **Offer to record an ADR for this so
   future architecture reviews don't re-suggest it** — not yet done, ask
   the user first (per this project's ADR-offer convention).
3. **Worth exploring** — unify bucket identity between `firebaseUpload.ts`
   and `functions/src/index.ts`. Advisor flagged the original framing as
   wrong (assumed both pointed at "the same bucket" — they don't, on
   purpose, mid-fix). **This candidate is now entangled with the
   test-drive bucket bug above** — resolve the test-drive bucket question
   first, then re-scope this candidate around whatever bucket wins.

Grilling (`/grilling` skill, per the architecture-review skill's step 3)
was never actually started for candidate 1 — it was launched and
immediately interrupted by the user's pivot to the test-drive. Needs a
restart, not a resume (no grilling questions were asked yet).

## docs submodule state — needs a decision, don't touch further without asking

`docs` submodule is on branch `docs-testing-policy-scope` (created this
session — it was in a **detached HEAD** state before, so it got branched
off to avoid an orphaned commit). Two commits on it:

- `bf236d2` — scopes the XState-only test policy to not block unrelated
  engineering decisions (user's explicit correction, see conversation).
- `8c2eb54` — adds `docs/agents/secrets.md` (public classification doc,
  safe-vs-sensitive split; the real inventory stays in the gitignored
  `.agents/secrets.md` at the main repo root).

Neither commit is pushed or PR'd. The submodule also has a large pile of
**pre-existing, unrelated uncommitted changes** (other scratchpad/handoff
files, `PROJECT_STRUCTURE.md`, `using-design-decisions.md`,
`project_instructions.md`) that predate this session — do not stage or
commit those, they're someone else's in-progress work.

The `test-driving.md` Firebase-debug-logging addition (see above) is
**also currently uncommitted** on this same branch, mixed in with those
pre-existing unrelated changes — it needs to be committed on its own
before this branch is pushed, without sweeping in the unrelated files.

## `.agents/secrets.md` update (private, gitignored, not this session's problem to solve further)

Added an entry for `firebase-adminsdk-fbsvc@project-e3d5659d-bc4f-438f-88c.iam.gserviceaccount.com`,
flagged sensitive by the user. Not referenced anywhere in the repo as of
this session (grepped, no hits) — just recorded so a future session
doesn't paste it somewhere public.

## Suggested skills for the next session

- `mattpocock-skills:diagnosing-bugs` — to resume the upload-bucket
  investigation methodically (Metro cache vs native rebuild) rather than
  ad hoc `adb`/logcat digging.
- `mattpocock-skills:grilling` — to actually run the grilling loop for
  architecture candidate 1 (and candidate 3, once its bucket question
  resolves), which never got past being proposed.
- `mattpocock-skills:domain-modeling` — if the bucket-identity decision
  (which bucket is canonical) ends up being recorded as an ADR amendment
  to ADR-0005.

## Not redacted, not needed

No API keys, tokens, or credentials appear in this handoff — the one
flagged-sensitive identifier (service account email) is recorded only in
`.agents/secrets.md` (gitignored), not here.
