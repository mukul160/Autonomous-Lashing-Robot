# ME5400A — Session Summary: TF Restructuring & Axis-Convention Rebuild

**Date:** August 27, 2026
**Continues from:** `03_gripper-hook-calibration_Aug25.md`

## What Was Done

### 1. TF Tree Restructured — `gripper_palm` as Root
Reparented the tree so `gripper_palm` is now the fixed frame, with
`camera_optical_frame` and `ALB_hook` both as its direct children (previously
`camera_optical_frame` was the root). Since TF only allows one parent per
frame, this meant replacing the `camera → gripper_palm` edge with its
algebraic inverse (`gripper_palm → camera_optical_frame`), not adding a
second edge. `gripper_palm → ALB_hook` needed no change — it was already
oriented correctly for the new structure. Nothing downstream (tag detection,
`SP_hole`) was affected, since `apriltag_node` only cares that
`camera_optical_frame` exists, not which direction the tree runs above it.

### 2. Diagnosed "Frames Moving the Wrong Way" — Not a Bug
Rotating the camera CCW appeared to move tag frames CW on screen. Root cause:
RViz's Fixed Frame was set to a frame rigidly attached to the moving camera,
so stationary real-world tags appeared to move relative to it — ordinary
parallax, not a calibration or sign error. Confirmed numerically by switching
Fixed Frame to a genuinely stationary tag and checking `tf2_echo`: CCW camera
rotation correctly increased yaw (right-hand rule about the up-pointing Z
axis), closing out the question with a verified, non-visual answer.

### 3. Full Frame Rebuild With a Consistent Axis Convention
Prior ad-hoc axis choices (`SP_hole`'s offset along X for `ALB_hook`, tuned
rotations found by trial and error) were discovered to be internally
inconsistent — specifically, the hook's physical offset direction and the
"into slot" direction had ended up exactly perpendicular rather than
parallel, a structural consequence of the original parametrization rather
than a measurement error. Rather than patch around it, all frames were
cleared and remeasured in SolidWorks with a single explicit convention set
up front: **Z = axis of insertion, X = downward**, for every frame.

New CAD-based values under this convention:
- `camera_optical_frame → gripper_palm`: dx=142.4mm, dz=89.3mm, camera tilted
  18.5° (positive pitch, since tipping the forward/Z axis toward
  downward/X is a positive rotation about Y under this convention).
- `gripper_palm → ALB_hook`: pure dz=120mm, no lateral offset, no orientation
  difference from `gripper_palm` (parallel surfaces).

### 4. `SP_hole` Corrected via Full-Pose Back-Calculation
Rather than remeasuring the tag-to-slot CAD offsets a second time, the
relative pose between the (known-correct) `ALB_hook` and the old `SP_hole`
was measured directly (`tf2_echo`, hook seated) and used to solve for the
**full** corrected `tag → SP_hole` transform — translation and rotation
together, not orientation alone. An orientation-only fix was tried first and
found insufficient, since the underlying issue was inconsistent measurement
axes affecting position too, not just rotation. The full-pose solve
(`T_new = T_old ⊗ T_measured`) was verified via an exact-identity sanity
check before being applied. The same correction was mirrored (X sign
flipped) onto `SP_hole_from_tag2`, since only `tag1` was directly measured.

## Result

Both sides of the pipeline — gripper/hook geometry and tag/slot geometry —
are now confirmed sensible and mutually consistent under one shared axis
convention, per direct confirmation.

## Outstanding Follow-ups

- ~~`SP_hole_from_tag2` is unverified.~~ **Confirmed.** Independently checked
  via `tf2_echo SP_hole_from_tag2 ALB_hook` (hook seated) — result is good,
  not just inherited from the tag1 mirror assumption.
- **`apriltag_bringup.launch.py`** should be updated to reflect today's final
  static transform values for all four nodes (`gripper_palm`, `ALB_hook`,
  `SP_hole_from_tag1`, `SP_hole_from_tag2`).
- **Twist-lock control logic** — approach, seat, and 30° twist — is the next
  major piece of work, now that `ALB_hook` reliably converges on `SP_hole`.
