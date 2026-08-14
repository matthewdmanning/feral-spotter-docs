---
status: implemented — issue #266 ("Connect to storage")
---

# Firebase Storage SDK for photo uploads, not the custom `upload+api.ts` route

Issue #10 built a validation-only Expo API route (`src/app/upload+api.ts`): the client
POSTs multipart form data, the route verifies a Google ID token against a tester
allowlist, validates MIME type/size (`src/lib/upload/fileValidation.ts`), enforces a
per-submission photo count cap, then writes to GCS via `@google-cloud/storage` with
Application Default Credentials. None of this is wired to a screen yet.

Decided 2026-08-13: switch to `@react-native-firebase/storage`'s `putFile`, uploading
directly from the client to the bucket, to get resumable uploads with built-in
pause/resume/retry — the custom route's one-shot multipart POST has no resilience
story for the spotty field network conditions this app runs in. This extends the
Firebase adoption started in [ADR-0001](0001-firebase-auth-over-google-signin.md)
(Auth) to Storage.

iOS is not blocked by this choice: `@react-native-firebase/storage` wraps the native
Firebase SDKs on both platforms (Android and the Firebase Apple/Swift SDK on iOS), so
it carries forward cleanly whenever ADR-0001's "Android-only for now" restriction
lifts.

## Resolved: target bucket

Corrected 2026-08-13 — `gs://feral-segmentor-alpha` (the MVP milestone description's
bucket name) is not the target; the real bucket is `gs://feral-spotter-image-uploads`.
Confirmed via `gcloud storage buckets list --project=project-e3d5659d-bc4f-438f-88c`:
it's already linked to this Firebase project, alongside the Firebase default bucket
(`project-e3d5659d-bc4f-438f-88c.firebasestorage.app`) and an unrelated
`feral-vision-smoke` bucket. Addressed as a non-default bucket:
`getStorage(getApp(), 'gs://feral-spotter-image-uploads')`. `upload+api.ts` and
`functions/src/index.ts` both default `GCS_BUCKET_NAME` to this value now.

## Resolved: photo-count enforcement moves to a Firestore counter

Storage Rules have no primitive for counting sibling objects under a prefix —
`resource` in a `write` rule is scoped to the single object at that same path, not a
folder listing, and `request.resource` describes only the incoming write. Confirmed
against current Firebase docs (`firebase.google.com/docs/storage/security/*`), not
assumed.

Design landed 2026-08-13, scoped **per submission** (matches `MAX_PHOTOS` in
`src/config/constants.ts`, not a per-user lifetime total):

- **Object path**: `submissions/{uidHash}/{submissionId}/{fileName}` — supersedes the
  original `{uid}` convention (distinct from `upload+api.ts`'s
  `uploads/{email}/{submissionId}/...`, which this replaces). `uidHash` is
  `sha256(USER_ID_HASH_SALT + auth uid)`, computed client-side
  (`hashUid()` in `src/lib/upload/firebaseUpload.ts`) and independently
  recomputed by `storage.rules`/`firestore.rules` via the rules language's
  `hashing.sha256()` — no Cloud Function round-trip, no new native module.
- **Counter**: Firestore doc `/submissions/{submissionId}`, field `photoCount`,
  maintained by `functions/src/index.ts`'s `onSubmissionPhotoUploaded` /
  `onSubmissionPhotoDeleted` Storage triggers (Admin SDK, bypasses rules — the
  recommended path over letting the client update the counter directly).
- **Gate**: `storage.rules` uses the cross-service `firestore.get()`/`firestore.exists()`
  functions (default Firestore database only, max 2 Firestore reads per rules
  evaluation) to check `photoCount < 10` before allowing a write. The
  `firestore.exists(...) == false` branch is required — without it the _first_ photo
  of a new submission is denied, since the counter doc doesn't exist until the
  Cloud Function creates it after that first write lands.
- **Known race**: the counter increments _after_ the object finalizes, not
  synchronously with the write. Two uploads issued close enough together can both
  read `photoCount` before either increment lands, both pass the rule, and leave the
  submission at 11 photos instead of 10. The hard synchronous gate the old
  `upload+api.ts` route had (`getFiles({ prefix })` counted at request time) doesn't
  carry over exactly — this trades a strict cap for resumable uploads.
- **Firestore access**: `firestore.rules` allows the owning user read-only access to
  their submission doc; all writes are `false` (Cloud-Function-only).

Tester allowlist enforcement is still open — not addressed by this change. Custom
claims or a Firestore-backed lookup would extend the same `firestore.get()` mechanism,
but that isn't designed here.

## Considered Options

- **Keep the custom `upload+api.ts` route** — rejected: no resumable/retry story,
  which is the reason for this change.
- **Firebase Storage SDK, direct client upload** (chosen) — resumable, works
  offline-to-online, reuses the Auth investment from ADR-0001. Trade-off: allowlist
  and count-cap enforcement need to be re-homed (see above) before this can fully
  replace the route.
