# Punchlist 2026-08-07

Annotate Screen: Back button undoes boxing - want this to display box selection. Not in Photo button does not work for the
first or last photo, but works for other photos. Not in Photo button also work for first photo if user hit Confirm then Back
on the next screen. Bubble is at the bottom of the screen -> obscures buttons; should be at the top. Bubble is sometimes off
screen. Box gestures work but user can push photo so far it isn't visible. no "Not in photo" gesture -- maybe this feature
was dropped. _Cat Form screen_: Still initializes with Unknown/Unsure buttons selected. Bottom button still labelled Put the
at In a Box. Warning popup text is terribly formatted -- need change to @docs/references/ux_principles.md as well. Put the
cat in a box button wired back to Annote screen! It should go to Cat List screen. But Done button sends it Cat List screen as
expected. Header does not change size with Bubble shrink. Bubble disappears off screen here too. bubble begins maximized on
both screens -> should begin small. "Put the Cat in a Box" button goes to Cat List screen on second cat form.

# Punchlist 2026-08-19

_Edit Cat screen_: Clear button does nothing. _Annotate screen_: inset picture not showing for second cat.
_Observed Cat screen_: inset bubble minimized size too small, maximized size too large -- expands to screen width.
Add a Delete button on Edit Cat page that completely removes the entry from Cat List, including its annotations.
Handle hitbox (bounding-box resize handles) too small -- make configurable, increase current value by 50%. Rename
"Boxing Complete" button label to "Done With This Cat". Submission: "Submission Failed" popup should be removed
entirely -- submissions must be resilient to poor connectivity, retries happen in background, user should never see
a failure notice; document this decision in docs/agents/ux_decisions.md. Handles are on box ends -- move to halfway
points on crosshair arms per spec. Pinch no longer zooms the photo (original behavior) -- makes boxing distant cats
difficult. Pinch/pan should move the photo; handles should control box dimensions.
