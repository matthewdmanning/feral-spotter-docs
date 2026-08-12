# Issue #249 — explicit consent gate on "Choose from Library"

## Context

`useLibraryPhotoPicker.pickFromLibrary()` currently calls `ImagePicker.launchImageLibraryAsync()` directly. The OS permission prompt happens invisibly *inside* that call — a user who declines gets back `{ canceled: true }`, indistinguishable from backing out of the picker having chosen nothing. The screen does nothing either way; a decline gets no explanation.

This violates the "explicit yes, never absence of no" consent principle already enforced for camera (#237) and location (#66), per #243's unification under `isXGated`-style explicit checks. Issue #249 (labeled `ready-for-agent`, fully specified) asks for the same explicit-check pattern applied to photo-library read access, but as a one-shot Alert on denial, not a full-screen gate like `ConsentScreen`'s "Permission Blocked" — a denial here just stops the current pick attempt.

## Change 1: `src/hooks/useLibraryPhotoPicker.ts`

Add an explicit check-then-request-then-gate step before `launchImageLibraryAsync()`, mirroring the check-only short-circuit pattern already used for the camera's write-only gallery-save permission in `useCameraCapture.tsx` (~line 160-165, ~255-257: `getPermissionsAsync(true)` → proceed if granted, else request).

```ts
import { Alert, Linking } from 'react-native'
// ...existing imports...

const isLibraryPermissionUsable = (
  r: ImagePicker.MediaLibraryPermissionResponse,
) => r.status === ImagePicker.PermissionStatus.GRANTED || r.accessPrivileges === 'limited'

const pickFromLibrary = useCallback(async () => {
  const current = await ImagePicker.getMediaLibraryPermissionsAsync()
  let usable = isLibraryPermissionUsable(current)

  if (!usable) {
    const requested = await ImagePicker.requestMediaLibraryPermissionsAsync()
    usable = isLibraryPermissionUsable(requested)
  }

  if (!usable) {
    Alert.alert(
      'Photo library access needed',
      'FeralSpotter needs access to your photo library to choose photos. Enable it in Settings to continue.',
      [
        { text: 'Cancel', style: 'cancel' },
        { text: 'Open Settings', onPress: () => Linking.openSettings() },
      ],
    )
    return
  }

  const result = await ImagePicker.launchImageLibraryAsync({
    mediaTypes: ImagePicker.MediaTypeOptions.Images,
    allowsMultipleSelection: true,
    quality: 1,
    exif: true,
  })
  // ...unchanged from here down...
}, [photos.length, addPhotos, setLocationType, setTimeType, setCapturedAt])
```

- Call both permission functions with no `writeOnly` arg (read access, unlike camera's `true`/write-only).
- Fast-path: once `getMediaLibraryPermissionsAsync()` itself reports usable, `requestMediaLibraryPermissionsAsync` is never called — check-only on repeat taps, no extra state needed (OS is the source of truth, same as `useCameraCapture`).
- The gate (Alert + `return`) sits directly after the permission check/request, before the picker launch — no separate step, no `AppState`/foreground-recheck listener like `ConsentScreen`'s. That listener exists there because it's a full-screen gate the user is trapped behind; here a denial just returns from the tap, and the next press re-invokes the function and re-checks fresh. No listener needed, ever.
- `limited` (iOS "Select Photos") counts as usable per issue story 3.
- Denial message: plain inline `Alert.alert`, matching this repo's established convention (`register/index.tsx`, `ConsentScreen`'s catch block, etc. all use inline `Alert.alert`, no shared copy file). `Linking.openSettings()` is the same recovery call `ConsentScreen` uses — reuse the call, not its full-screen "Permission Blocked" UI.
- No changes to `src/content/consentDisclosure.json` (deferred point-of-use permission per #91).
- Update the stale comment above the old `launchImageLibraryAsync` call ("Lazy, point-of-use permission prompt — no eager request() call") — it describes the old implicit-inside-launch behavior being replaced; note the new explicit flow and reference #249/#66/#237/#243.

## Change 2: new test — `src/hooks/__tests__/useLibraryPhotoPicker.permissionGate.model.test.ts`

Per `docs/agents/testing.md`: model-based test only, `getPathsFromEvents` (UX journeys), not `getShortestPaths` (coverage-only). Shape mirrors `src/hooks/__tests__/useCameraCapture.controls.model.test.ts` (hook test via `renderHook`, not `render`).

Mock `expo-image-picker` (`getMediaLibraryPermissionsAsync`, `requestMediaLibraryPermissionsAsync`, `launchImageLibraryAsync`, `MediaTypeOptions`, `PermissionStatus`), `expo-router`'s `router.navigate`, `react-native`'s `Alert`/`Linking`, and `@/src/hooks`'s `usePhotoStore`/`useSubmissionStore` selectors (same pattern as existing hook tests).

Machine:
```ts
const permissionGateMachine = createMachine({
  id: 'libraryPermissionGate',
  initial: 'idle',
  states: {
    idle: {
      on: {
        TAP_ALREADY_GRANTED: 'opened',
        TAP_NOT_DETERMINED_THEN_GRANT: 'opened',
        TAP_NOT_DETERMINED_THEN_DENY: 'blocked',
      },
    },
    opened: {},
    blocked: {},
  },
})
```

Journeys (each asserts observable behavior — mock call counts/args — not internal state), matching the issue's own Testing Decisions 1:1:
- already granted → picker opens directly, `requestMediaLibraryPermissionsAsync` never called (proves the short-circuit)
- not-determined → request → granted → picker opens, no Alert
- not-determined → request → denied → `Alert.alert` fires (assert its "Open Settings" button's `onPress` triggers `Linking.openSettings()`), `launchImageLibraryAsync` never called

`limited` (`accessPrivileges`) isn't a separate journey — it hits the same `isLibraryPermissionUsable` branch as `granted`, identical assertions, no new coverage. Story 3 is covered by the implementation's check itself, not a dedicated test.

`HomeScreen.photoSourceGate.model.test.tsx` and the other three tests mocking `useLibraryPhotoPicker` wholesale (`HomeScreen.entrypointActions.model.test.tsx`, `HomeScreen.gate.model.test.tsx`, `CreateScreen.addMorePhotosAction.model.test.tsx`) stay untouched — new coverage lives one layer down at the hook, per the issue's own testing decisions.

## Call sites — no changes needed

Confirmed via grep: only `src/screens/home/index.tsx` and `src/screens/submission/create/index.tsx` call `useLibraryPhotoPicker()` in production code; both just destructure and call `pickFromLibrary()` with no args. Signature (`() => Promise<void>`) is unchanged, so neither needs edits.

## Out of scope (per issue body)

- Camera's separate write-only gallery-save permission in `useCameraCapture.tsx` — untouched.
- The unreproduced "app exits on Don't allow" report from #64 — no repro, not investigated here.
- Android/iOS location-accuracy display gap — unrelated, separate spec if it resurfaces.

## Verification

- Run the test suite scoped to the new file: `npx jest useLibraryPhotoPicker.permissionGate` (check `package.json`'s `scripts.test` for the repo's exact invocation).
- Full suite + lint + typecheck before PR, per this repo's PR policy (tests/linter/formatter must pass).
- No manual device/emulator run strictly required (pure logic + mocked native module), but a quick real-device "Choose from Library" tap-through (first-time deny, then Settings-grant, then re-tap) is worth a test-drive note given this touches a live permission dialog.
