# Camera Calibration — GXIVISION-S2M03 + ROS2 Humble

## Overview

This document covers the intrinsic calibration of the GXIVISION-S2M03 USB camera,
performed as part of the AprilTag detection bringup for the ALB (Automatic
Lashing Bar) project. The goal of calibration is to recover the camera's
intrinsic matrix (`K`) and distortion coefficients (`D`), which `image_proc`'s
`rectify_node` then uses to undistort raw frames before AprilTag pose estimation
— without accurate intrinsics, the 6-DOF pose solved by `apriltag_node` will be
systematically wrong, especially at the image edges.

Calibration was done with **ROS2's `camera_calibration` package** using its
`cameracalibrator` tool, against a **screen-displayed checkerboard** (no printer
available at the time — the same constraint noted for the AprilTag marker
itself). A screen-displayed pattern is a valid substitute here, provided its
physical on-screen size is measured accurately rather than assumed.

> **Note:** This document reconstructs the calibration procedure and the two
> issues actually hit during bringup (board-size mismatch, camera-name
> mismatch). The exact PPI/square-size numbers used are not captured in the
> original bringup notes — placeholders are marked below where you should
> fill in the specific values you used, so this file is a fully accurate
> record rather than an approximation.

---

## Why Calibration Is Needed

Every real lens introduces distortion (radial and tangential) and has a
principal point that isn't perfectly centered in the sensor. `apriltag_node`
solves for a tag's pose assuming a clean pinhole projection — feeding it a
raw, uncorrected image means it's solving pose against a geometrically
inaccurate model of the camera, producing pose error that grows with distance
from the image center. `image_proc`'s `rectify_node` removes this distortion
using the `K`/`D` values recovered here, so this step has to happen before
AprilTag detection is trustworthy, not just before it "works."

---

## Prerequisites

- `usb_cam` already confirmed to publish `/image_raw` at the correct
  resolution/format (see the main bringup doc, setup step 3) — calibration
  should not be the first time you're looking at the camera's live feed.
- ROS2's calibration tooling installed:
  ```bash
  sudo apt install ros-humble-camera-calibration
  ```
- A checkerboard pattern, displayed on a screen large enough to be moved
  through a range of distances/angles in front of the camera during
  calibration.

---

## Step-by-Step Process

### 1. Obtain and display the checkerboard pattern

OpenCV's standard `pattern.png` was used rather than a hand-drawn board, to
avoid introducing measurement error from an irregular grid.

- Pattern: OpenCV `pattern.png`
- **Actual grid: 9x6 *internal corners* (i.e. a 10x7 *square* board)** — this
  matters because `cameracalibrator`'s `--size` argument counts internal
  corners, not squares, and this distinction caused issue #1 below.

Before trusting any square-size number, the pattern's actual square dimensions
in the downloaded image were confirmed directly rather than assumed:

```bash
identify pattern.png
```

### 2. Determine the physical square size (PPI method)

Since the board was screen-displayed rather than printed, its physical size in
the real world depends on the display's actual pixel density, not just the
image's pixel dimensions. The process:

1. Find the display's native resolution and physical screen size (from specs
   or `xrandr --verbose`), and compute PPI:
   `PPI = diagonal_resolution_px / diagonal_size_in`
2. Convert to pixel pitch: `mm_per_px = 25.4 / PPI`
3. Multiply by the checkerboard square's size in pixels (from the `pattern.png`
   file, confirmed via `identify` in step 1) to get the physical square size in
   mm, then convert to meters for the `--square` argument.

> **Fill in:** your actual display PPI, computed pixel pitch, and the
> resulting `--square` value in meters — these were computed during bringup
> but not recorded in this file.

This is the same PPI-based approach later reused for scaling the physical
AprilTag marker itself, for consistency between calibration and detection.

### 3. Launch the calibration tool

```bash
ros2 run camera_calibration cameracalibrator \
  --size 9x6 \
  --square <SQUARE_SIZE_METERS> \
  --ros-args -r image:=/image_raw -p camera:=/usb_cam
```

- `--size 9x6` — internal corner count, confirmed in step 1 (not the naive
  8x6 assumption — see issue #1).
- `--square` — physical square edge length in meters, from step 2.
- Topic remap should match whatever `usb_cam` is actually publishing on
  (`/image_raw`, per the main bringup doc's parameter setup).

### 4. Collect calibration samples

With the tool's GUI window open and the camera feed live, the checkerboard
was moved through the camera's field of view to fill out the tool's **X / Y /
Size / Skew** progress bars:

- **X, Y** — move the board across the frame horizontally and vertically,
  including near the edges, not just the center.
- **Size** — move the board closer and farther from the camera.
- **Skew** — tilt the board at angles relative to the camera, not just
  face-on.

The **CALIBRATE** button activates once all four bars are sufficiently full.
Skewed/angled samples matter disproportionately for solving distortion
coefficients accurately — a pile of only face-on, centered samples will
converge to a poor `D` even with many images.

### 5. Calibrate and save

- Click **CALIBRATE** — this can take a noticeable pause depending on sample
  count.
- Reprojection error is reported in the terminal/GUI; check this is small
  (sub-pixel, typically well under 1.0) before accepting the result.
- Click **SAVE**, then **COMMIT**. This writes a `calibrationdata.tar.gz`
  (raw images + `ost.yaml`/`ost.txt`) to `/tmp/`, and prints the resulting
  `camera_info` YAML to the terminal.

### 6. Install the calibration file

The tool's output YAML was copied into ROS2's standard camera_info location so
`usb_cam` picks it up automatically via `camera_info_url`:

```bash
mkdir -p ~/.ros/camera_info
cp /tmp/calibrationdata.tar.gz ~/.
# extract ost.yaml (or ost.txt, converted to yaml) from the archive
cp ost.yaml ~/.ros/camera_info/gxivision_s2m03.yaml
```

Referenced in `config/usb_cam.yaml` (see main bringup doc's final working
config):
```yaml
camera_info_url: "file:///home/mukul/.ros/camera_info/gxivision_s2m03.yaml"
```

### 7. Verify

With `usb_cam` running against the updated config:

```bash
ros2 topic echo /camera_info --once
```

Confirm `K` and `D` are populated (non-zero, non-identity) and match what the
calibration tool reported, and that `width`/`height` match the camera's actual
capture resolution (1920x1080).

---

## Challenges Faced and Fixes

### 1. Checkerboard size mismatch (internal corners vs. squares)

**Symptom:** Nearly launched `cameracalibrator` with `--size 8x6`.
**Cause:** Assumed a generic 8x6-corner board without checking the actual
downloaded pattern. OpenCV's `pattern.png` is a 9x6-corner (10x7-square)
board — a very common source of calibration tool confusion since "8x6
checkerboard" is often used loosely to describe either the square count or
the corner count depending on the source.
**Fix:** Verified the pattern's actual square dimensions with `identify`
(ImageMagick) before running calibration, confirming `--size 9x6` was
correct.

### 2. Calibration file camera-name mismatch

**Symptom:** On loading the calibration file, `usb_cam` logged a warning:
```
[gxivision_s2m03] does not match narrow_stereo in file
```
**Cause:** `cameracalibrator` always writes `camera_name: narrow_stereo` into
the saved YAML, regardless of what camera was actually used — this is a
hardcoded default in the tool, not something tied to the actual device.
**Impact:** Cosmetic only. `usb_cam` loads the `K`/`D` values by URL, not by
matching camera name, so detection/rectification worked correctly even before
this was fixed — the warning is purely a log-cleanliness issue.
**Fix:** Edited the saved calibration file's `camera_name` field from
`narrow_stereo` to `gxivision_s2m03` to match, then re-verified via
`ros2 topic echo /camera_info --once` that the name field updated with no
change to the `K`/`D` values.

---

## Known Caveats / Follow-ups

- **Principal point offset:** the calibrated `cx` was offset from the image's
  geometric center by more than initially expected (~116px on a 1920px-wide
  image). Not currently causing visible pose problems in AprilTag detection,
  but flagged as the first thing to revisit if lateral pose offsets look
  implausible later. Worth re-running calibration with more edge/corner
  samples if this needs investigating.
- **Screen-displayed board limitations:** the same caveat as the
  screen-displayed AprilTag applies here — potential for rolling-shutter vs.
  LCD-refresh interaction and a display that isn't perfectly flat/reflective
  compared to a printed board. If pose accuracy needs tightening later,
  re-calibrating against a printed board (with directly measured square size,
  removing the PPI-estimation step entirely) is the natural next step.
- **Re-run trigger:** if the camera's focus, zoom, or mounting position ever
  changes, this calibration is invalid and needs to be redone — intrinsics
  are specific to a fixed optical configuration.
