# Android MVP QA checklist

## Launch

- Launch app, sign in (Google or email — Firebase Auth)
- Consent screen (camera + location permission gate) shown before first use

## Home

- "Take a Photo" opens Camera
- "Choose from Library" opens the system photo picker
- "Resume Submission" appears only when an in-progress draft exists

## Camera

- Grant camera permission -> camera opens
- Deny camera permission -> blocked/error state shown, not a crash
- Capture photo(s); cycle flash (auto/on/off); flip front/back camera
- Cancel/close camera without capturing -> no photo added
- Repeated shutter presses don't re-trigger the gallery-save permission prompt

## Library picker

- Grant media permission -> picker opens
- Deny permission -> blocked/error state shown
- Select one photo, and multiple photos in one pick
- An EXIF-less photo (e.g. a screenshot) falls back to manual time entry later, not a silent "now" default

## Annotate (per cat)

- First box confirmed on a photo declares a new cat; boxing on further photos with no active cat starts the next one (via "Add a Cat")
- Framing gesture: pinch/pan/double-tap the photo under the fixed center crosshair, long-press (or Confirm) to lock the box in
- "Not in this photo" — swipe-up flick or the explicit pill button — skips the photo without boxing, advances to the next
- "Boxing Complete" with zero boxes drawn abandons the pass and returns to Home; does not save an empty cat
- Going back to an already-boxed photo restores the same crop-frame position
- Removing a photo mid-pass sweeps that photo's box(es) for the active cat, no orphaned data left behind
- Hardware back mid-pass abandons the current cat (already-drawn boxes are not cleaned up — by design)

## Cat List

- Zero cats recorded -> skips straight into annotate automatically, no tap required
- One or more cats recorded -> renders normally; "Add a Cat" explicitly re-enters annotate
- Tapping an existing cat row opens Cat Form directly

## Cat Form

- Static crop of the cat's first box shown at top (no manual photo re-selection)
- All 8 fields (age, sex, ear-tipped, owned/domesticated, pattern, hair length, color, health) are optional selectors
- Save always succeeds; unset fields are listed as a warning, never block Save
- Editing an existing cat's fields persists
- Clear wipes all fields, after a confirm prompt
- Save routes back to Cat List

## Submission Details

- Location was captured automatically back at Camera/Library open — verify it (or a staleness flag) is present here
- Manual-time entry button appears only when a photo lacked EXIF timestamp
- Submit shows a confirm dialog with cat/photo counts
- Reset clears the entire draft (cats, photos, submission fields), after a confirm prompt

## Save / recovery

- Full flow: Camera or Library -> annotate -> Cat Form -> Submission Details -> Submit
- Background/kill the app mid-flow and resume -> in-progress draft persists locally
- After a successful Submit, starting a new sighting begins a fresh draft, not the just-submitted one
- Repeated Submit taps don't create duplicate local drafts or duplicate server submissions

## Network

- Submit with normal connectivity succeeds
- Submit with no connectivity fails gracefully: error alert shown, draft kept locally as "Failed", not lost
- Toggle airplane mode mid-upload or mid-Submit -> no partial/corrupt local draft state
- Re-tapping Submit after a "Failed" status doesn't double-submit or duplicate the draft
- No automatic retry/backoff exists yet — confirm that's expected, not a regression

## Device / system

- Low-memory conditions during a full capture -> annotate -> submit pass
- Low-storage device: camera capture / gallery-save failure paths
- Incoming call or notification mid-flow doesn't lose in-progress state
- App restart after device reboot: in-progress local draft still present
