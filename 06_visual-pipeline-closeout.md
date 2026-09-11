# ME5400A — Session Summary: Visual Pipeline Closed Out

**Date:** September 2026 (final session in this thread)

## Final State — Validated and Convergent

All four static geometry transforms are now CAD-derived, cross-checked, and
confirmed to hold across a full ±90° twist sweep on real hardware:

| Transform | Source |
|---|---|
| `camera_optical_frame → gripper_palm` | CAD baseline + small verified residual correction |
| `gripper_palm → ALB_hook` | Exact CAD value, untouched throughout |
| `SP_hole_from_tag1` | Derived directly as `tag1 → ALB_hook` in corrected CAD |
| `SP_hole_from_tag2` | Derived directly as `tag2 → ALB_hook` in corrected CAD |

`ALB_hook` and `SP_hole` now coincide at rest and stay coincident (translation
near-zero, pure yaw) through the entire twist range — the target stated at
the start of this thread's debugging arc, finally met on real hardware, not
just in simulation-free math.

## How This Got Resolved

**The real breakthrough was fixing the CAD alignment itself**, not further
iteration on static-transform math. A geometry error in the CAD assembly had
been quietly present through many earlier calibration attempts — once
corrected, `tag → ALB_hook` could be computed directly and exactly, replacing
the entire earlier chain of hand-measurement, best-fit averaging, and
axis-location correction with a single clean derivation.

**Two infrastructure bugs masqueraded as calibration problems** late in this
process, and both were worth catching explicitly rather than continuing to
chase phantom math errors:

- **Stale `colcon install`:** without `--symlink-install`, edits to the
  launch file source had no effect until an explicit rebuild — several
  "verified correct, sanity-check-passes" corrections appeared to do nothing
  because the *previous* version was still what was actually running. Fixed
  by rebuilding with `--symlink-install`.
- **Hardcoded device path drift:** `set_camera_controls.sh` was still
  targeting `/dev/video2` after USB enumeration order shifted the camera to
  `/dev/video0` — silently succeeding against the wrong device (likely the
  laptop's built-in webcam) while auto-exposure stayed active on the real
  camera the whole time.

Both are worth remembering as a general lesson: **when a mathematically
verified fix appears to do nothing, suspect the deployment/infrastructure
layer before suspecting the math again.**

## Final Corrective Technique (for future reference)

Every real correction in this final phase used the same pattern: measure the
residual `ALB_hook → SP_hole` transform at a known configuration, then solve
for the adjustment to `camera_optical_frame → gripper_palm` (never
`gripper_palm → ALB_hook`, held fixed throughout per design constraint) via

```
T_cam_gripper_new = T_cam_gripper_old @ T_gripper_hook @ T_measured_residual @ inv(T_gripper_hook)
```

with an exact-zero sanity check (recompose and diff against the target)
applied before ever handing a correction back for use — a habit adopted after
an earlier unverified correction turned out to be wrong.

## Outstanding / Handed Off

- **Gazebo simulation of the Kassow 1410** is being tracked in a separate
  chat — the URDF/joint/controller work, and eventual porting of this same
  camera + AprilTag pipeline into the simulated environment.
- **Persistent device paths** (`/dev/v4l/by-id/...`) were recommended in
  place of `/dev/videoN` in both `usb_cam.yaml` and
  `set_camera_controls.sh`, to prevent the enumeration-drift bug from
  recurring — worth confirming this was actually applied.
- The **spreadsheet-based error log**
  (`SP_hole_ALB_hook_error_log.xlsx`) and the two Fusion pose-extraction
  scripts (`print_tag_slot_poses.py`, `print_all_frames_and_slot.py`) remain
  useful references for any future recalibration.
