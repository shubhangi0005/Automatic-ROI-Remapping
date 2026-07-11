# ROI Remapping Under Camera Disturbance (Landmark-Based Template Matching)

## Problem

This project targets a **fixed, static surveillance camera** monitoring a road. A user manually
selects a rectangular/quadrilateral Region of Interest (ROI) on the road using an interactive
slider tool. Under normal conditions the camera does not move — but wind or rain can physically
nudge it, shifting the scene it captures.

When that happens, the goal is to:

- Automatically **detect** that the camera's view has shifted.
- Measure **how much** it shifted (position, and optionally scale/zoom).
- **Reposition the ROI** so it always captures the same real-world patch of road — never more,
  never less — regardless of how the camera was nudged.
- Leave the ROI untouched when the camera is genuinely stable.

## Approach

This notebook (`roi-tm.ipynb`) implements the final, validated approach after several iterations:

1. **Manual ROI selection** — interactive sliders let you draw a 4-point quadrilateral ROI on a
   reference image.
2. **Manual landmark selection** — a single distinct, static, high-contrast object near the ROI
   (e.g. a signboard) is also marked. This landmark is deliberately *not* part of the ROI itself,
   so it's unaffected by traffic or other transient content inside the ROI.
3. **Multi-scale template matching** — for each new image, `cv2.matchTemplate` searches a local
   window around the landmark's expected position across a range of scales, recovering both a
   translation `(dx, dy)` and a scale factor in a single search. This captures both a physical
   shift and any zoom/focal-length change.
4. **Confidence gating** — a normalized cross-correlation confidence score determines whether the
   measured shift is trusted. Low-confidence matches fall back to the original, unshifted ROI
   rather than risk an incorrect correction.
5. **Coordinate transform, not image warp** — the ROI's four corner points are scaled and
   translated relative to the landmark's own measured motion. This is a direct coordinate
   transform (not a re-fit or full-image warp), which guarantees the ROI's shape and size never
   change — it can only reposition.
6. **Visual verification** — an overlay-blend step (reference + corrected frame blended, ROI drawn
   on top) lets a human directly confirm alignment; any residual drift shows up as ghosting or a
   double outline.

## Why this approach (and not simpler ones)

Earlier iterations were tried and found to have real failure modes:

- **Applying a single fixed crop to every frame** — no disturbance detection at all; silently
  wrong the moment the camera moved.
- **Whole-frame ORB feature matching + full homography** — could badly skew/distort the image when
  matches were sparse, and could be fooled by moving vehicles/pedestrians or repetitive textures
  (e.g. tarpaulin folds, gravel) elsewhere in the frame.
- **Using the ROI's own content as a matching template** — works, but is sensitive to whatever is
  currently inside the ROI (a truck or workers standing in it contaminates the template), and only
  tolerates small disturbances due to a limited local search window.

The landmark-based method sidesteps all three: it tracks one specific, verifiable, static object
instead of trusting a statistical fit across the whole frame or the ROI's own (possibly
non-static) content.


## Usage

1. Open `roi-tm.ipynb` (designed for a Kaggle-style environment with images under
   `/kaggle/input/...`; adjust the hardcoded paths in the image-loading cell for your own setup).
2. Run the cells in order:
   - Load images, set the reference frame.
   - Use the sliders to draw the ROI on the reference image.
   - Use the sliders to mark a distinct, static landmark near the ROI (avoid low-contrast or
     repetitive objects — prefer something large and unambiguous like a signboard face).
   - Run the template-matching cell to measure `(dx, dy, scale, confidence)` for each image.
   - Run the crop/overlay cells to produce the corrected ROI crops and the visual verification
     overlay.
3. Tune `CONFIDENCE_THRESHOLD` and the search margin based on your own camera's real disturbance
   patterns — the values in this notebook are starting points validated on a small set of test
   images, not universal constants.

## Notes / Known limitations

- This method assumes **isolated snapshots**, not continuous video. If continuous or frequent
  video frames are available instead, a frame-to-frame optical flow tracker (validated separately
  in an earlier proof-of-concept) may be a more robust alternative.
- Landmark choice matters a lot: low-contrast or visually repetitive objects (e.g. a plain dark
  barrel) have been observed to produce false, low-confidence matches. Prefer large, flat,
  high-contrast, non-repetitive static objects.
- A landmark placed too close to the image edge can produce an empty/invalid template crop —
  leave at least the patch half-size of margin from any image boundary.

