# Automated ROI Re-Localization After Camera Displacement

Fixed site cameras (highway, yard, plant and checkpoint cameras) monitor a specific
ground area defined once as a 4-corner **Region of Interest (ROI)** — a stretch of
road used for vehicle detection, a checkpoint lane, a yard entrance, and so on.

Cameras on poles or masts get knocked, drift, or get re-mounted after maintenance.
Once that happens, the ROI drawn on the *old* view no longer points at the same
patch of ground, and any detection/counting logic built on it silently starts
looking at the wrong place.

**This notebook automatically re-locates a previously drawn ROI in a new frame from
the same camera after it has moved** — no manual re-drawing, and no need to know in
advance how much the camera moved.

> You draw the ROI once on **Image 1** (before the move). The notebook predicts
> where it belongs on **Image 2** (after the move).

---

## How it works

Given `image 1` (the reference frame the ROI was drawn on) and `image 2` (a later
frame from the same camera, possibly after it moved):

1. **Preprocess** — convert both frames to grayscale, apply CLAHE contrast
   equalization, and derive an edge map for each, so lighting changes and
   wet/dry surface differences don't confuse matching.
2. **Detect & match features** — SIFT finds distinctive keypoints in both frames;
   matches are filtered with a ratio test, a mutual (both-directions) check, and a
   per-region cap so no single busy area dominates the fit.
3. **Fit candidate motion models** — RANSAC/MAGSAC fits three transforms to the
   matched points: **similarity** (shift + rotate + scale), **affine**
   (+ shear/stretch), and **homography** (+ perspective — the right model for a
   camera that tilts or pans, which is the common real-world case).
4. **Refine with ECC** — the homography is refined further by directly aligning
   the two edge maps (coarse → fine), which is more precise than point matches alone.
5. **Score, ground-truth-free** — after sanity checks (the ROI must stay inside the
   frame, not collapse or flip), each candidate transform is scored by how well it
   aligns the edges around the ROI in image 2. The best-scoring model wins, with a
   bias toward the simpler model when scores are close.
6. **Predict the new ROI** — the winning transform is applied to the original 4 ROI
   corners to get their predicted location in image 2. Because the *whole
   transform* is predicted (not each corner independently), a corner can be placed
   correctly even if it ends up **outside the visible frame** in image 2.
7. **Estimate confidence** — the matched inlier points are bootstrapped (200
   resamples) to get a per-corner uncertainty and an overall **GOOD / CHECK**
   verdict, so a human knows when to double check the result.

### What changed from v1

| | v1 | v2 (this notebook) |
|---|---|---|
| Road segmentation (HSV + GrabCut) | required, fragile | **dropped** — asphalt has almost no texture anyway |
| Motion model | 4-DOF similarity only (no tilt/pan) | similarity, affine, **and homography** |
| Features | ORB + ratio test | **SIFT** on CLAHE-equalized images, ratio test + mutual check + spatial spreading |
| Refinement | none | **ECC** direct alignment on edge maps |
| Choosing between models | needed a hand-drawn ground truth | **ground-truth-free** edge-correlation score |
| Confidence | none | **bootstrap uncertainty** + inlier-coverage check |

---

## Measured accuracy

| Check | Result |
|---|---|
| **Synthetic ground truth** (15 random camera-bump trials on a real image, with added lighting change, blur, noise and fake moving objects) | median **0.11 px**, p95 **0.24 px** ROI-corner error |
| **Round-trip consistency** (image 1 → 2 → 1) | mean **0.02 px**, p95 **0.03 px** corner error, IoU **1.000** |
| **Real-image landmark check** (hand-picked static points on genuine camera movement) | not yet run — see [Limitations](#limitations) |

These numbers are a best case (single source image, no parallax) — real-world
accuracy on genuine camera movement should be confirmed with the landmark check
before relying on this for anything safety-critical.

Tested successfully across highway, plant-road, batching-plant, checkpoint,
fisheye (wide-angle) and **night-vision / IR** camera scenarios, including a case
where a predicted ROI corner correctly landed outside the visible frame in image 2.

---

## Getting started

### Requirements

```
opencv-python
numpy
matplotlib
ipywidgets
```

The ROI-picker sliders need a Jupyter/Kaggle notebook environment with widget
support — this won't run as a plain `.py` script as-is.

### Usage

1. Open the notebook and set your two image paths:
   ```python
   IMG1_PATH = "path/to/before.png"   # before the camera move
   IMG2_PATH = "path/to/after.png"    # after the camera move
   ```
2. Run all cells. When the ROI-picker sliders appear, drag them to draw the ROI on
   Image 1 — or skip the UI entirely by pasting known coordinates into
   `ROI1_MANUAL`.
3. Run the remaining cells. The predicted ROI on Image 2, the chosen model, the
   alignment score, the uncertainty estimate, and a GOOD/CHECK verdict are all
   printed and plotted.
4. (Optional) Draw a ground-truth ROI on Image 2 (`GT_ROI2_MANUAL`) purely to
   report IoU against a known answer.
5. The result is saved to `roi_result.json`:
   ```json
   {
     "roi_image1": [[x, y], ...],
     "roi_image2_predicted": [[x, y], ...],
     "model": "homography+ECC (ROI-focused)",
     "homography_img1_to_img2": [[...], [...], [...]],
     "alignment_score": 0.0,
     "uncertainty": {"mean_px": 0.0, "p95_px": 0.0, "iou_mean": 0.0},
     "verdict": "GOOD"
   }
   ```

### Running the accuracy check

A separate cell reruns the whole method on synthetic camera bumps and on a
round-trip test, and reports quantitative accuracy (see table above). It caches
Image 1's features once instead of recomputing them every trial, so 15 trials
finish in well under a minute with live progress printed per trial.

---

## Limitations

- The **GOOD/CHECK** verdict is a heuristic based on inlier coverage and
  bootstrap spread, not a ground-truth accuracy measurement.
- The **real-image landmark check** — matching hand-picked static points (sign
  corners, pole bases, etc.) between the two real images — has not been run yet.
  This is the only check that measures accuracy on genuine camera movement rather
  than a synthetic bump, so it's the most convincing number and the natural next
  step.
- Very low-texture or heavily obstructed scenes (dust storms, dense fog, or a
  scene with almost no matchable structure) can reduce the number of reliable
  keypoints and lower confidence.
- Extremely large camera displacements, where little or no common structure
  remains between image 1 and image 2, are outside the scope of what
  point-matching can recover.

---

## Author

Shubhangi — Computer Vision & Machine Learning, applied R&D
(construction-site monitoring and worker-safety systems).



