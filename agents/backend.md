# Backend / Firebase

Current state of this project's backend surface — Auth, Storage, uploads —
and which Firebase MCP resources and skills are approved to use. Ratified
design decisions live in `docs/adr/`; this file is the map to them plus the
bits not yet ratified.

Firestore is **not** a backend surface of this project — see "Firestore
skills are off-limits" below before touching anything Firestore-shaped.

## Current architecture

- **Auth**: Firebase Auth, Google + email/password live, Apple/Facebook
  version-gated off. See [ADR-0001](../adr/0001-firebase-auth-over-google-signin.md).
- **Photo uploads**: direct client upload via `@react-native-firebase/storage`
  to `gs://feral-spotter-image-uploads`, path
  `submissions/{uid}/{submissionId}/{fileName}` (raw Firebase Auth uid, not
  a hash — see ADR-0005's 2026-08-16 amendment). See
  [ADR-0005](../adr/0005-firebase-storage-for-uploads.md).
- **Firestore counter doc**: `/submissions/{submissionId}.photoCount`,
  written only by `functions/src/index.ts`'s Storage-trigger Cloud Functions
  (Admin SDK) to satisfy Storage Rules' need for a sibling-object count —
  not an application database. `firestore.rules` gives the owning uid
  read-only access; all client writes are `allow write: if false`. Don't
  extend this into a general Firestore data model — see below.
- **Final submission (cats + metadata)**: uploaded to Cloud Storage as
  `metadata.json` alongside the photos — see
  [ADR-0005](../adr/0005-firebase-storage-for-uploads.md). Not a Firestore
  write, not a separate Cloud Run/Functions endpoint.
- **Tester allowlist / abuse limits on uploads**: not enforced yet — open
  follow-up issues #267–#270 from the PR #271 security-rules audit.

## Firestore skills are off-limits

Never load `firebase:firebase-firestore` or any other skill scoped to
Firestore schema/query/index design in this project. The only Firestore
surface that exists is the single Admin-SDK-written counter doc above —
it is not, and must not become, a general-purpose data store. If a task
seems to call for a Firestore skill, that's a signal to stop and confirm
scope with the user rather than loading it.

## Approved resources

MCP resources:

- `firebase://docs/`
- `firebase://guides/init/backend`
- `firebase://guides/init/auth`
- `firebase://guides/app_id`

Skills — only these, filtered mechanically by one rule: the skill's
frontmatter `description` does not mention Firestore. Re-run that check
against the installed skill's `description` before adding any new one to
this list; don't add by name/topic judgment:

- `firebase:extension-to-functions-codebase`
- `firebase:firebase-ai-logic-basics`
- `firebase:firebase-app-hosting-basics`
- `firebase:firebase-auth-basics`
- `firebase:firebase-basics`
- `firebase:firebase-crashlytics`
- `firebase:firebase-data-connect`
- `firebase:firebase-hosting-basics`
- `firebase:firebase-remote-config-basics`
- `firebase:xcode-project-setup`

Never `firebase:firebase-firestore` (see above) or
`firebase:firebase-security-rules-auditor` (its description only mentions
Firestore, even though its checklist also covers Storage rules) — excluded
by the same mechanical rule, not a judgment call.
