# ME5400A — Session Summary: Resolving the SP_hole/ALB_hook Axis Mismatch

**Date:** September 7, 2026

## The Problem

After earlier sessions validated `SP_hole` and `ALB_hook` at rest (hook seated,
no twist), physically twisting the hook revealed a serious issue: instead of
`SP_hole` staying fixed at `ALB_hook`'s origin and showing pure yaw, it swung
through a large arc — both position and orientation changing substantially,
with roll and pitch contaminating what should have been a clean single-axis
rotation.

## What Didn't Work (and why that mattered)

Several fixes were tried and ruled out, each one narrowing down where the
real problem could and couldn't be:

- **Correcting `SP_hole`'s rotation alone** — didn't help, and couldn't have:
  the true rotation axis, extracted directly from raw tag motion, is
  algebraically independent of anything `SP_hole` is calibrated to. No
  amount of retuning that one transform can move a physical axis.
- **Correcting `SP_hole`'s full pose (translation + rotation) to sit exactly
  on the measured axis** — mathematically valid, but broke the seated-rest
  alignment, since the true axis didn't pass through `ALB_hook`'s origin.
  Confirmed via two independent tags agreeing to within 5mm — real physical
  fact, not sensor noise.
- **Rebroadcasting tag poses under new names, or axis-locking them** — didn't
  address anything, since renaming or relabeling a frame doesn't change the
  underlying transform values.
- **Switching which frame was treated as "fixed" (`ALB_hook` vs `SP_hole`)**
  — no effect; rotation angle and translation magnitude were identical
  either way, since it's the same physical relationship viewed from two ends.

Each dead end was still useful: together they proved the issue had to live
in the **static geometry chain itself** (`camera_optical_frame → gripper_palm
→ ALB_hook`), not in `SP_hole`, not in sensor noise, and not in which
reference frame was used to view it.

## The Actual Root Cause

The camera is mounted **sideways** on the arm — a fact that had never been
modeled. Working through the physical geometry explicitly (rather than
guessing roll/pitch/yaw numbers) revealed multiple compounding errors in the
original hand-derived `camera_optical_frame → gripper_palm` transform:

- A translation offset placed along the wrong axis (camera's red instead of
  green).
- A missing ~90°+ rotation component entirely, from never accounting for the
  sideways mount.
- Confirming the **AprilTag's own axis convention** directly (red=right,
  green=up, blue=out of the tag face) rather than assuming a textbook
  default — an earlier assumption about this had been wrong.

Fixes were derived by describing the actual physical operation in each case
("spin this vector about the gripper's own blue axis," "sweep this frame
about the camera's Z axis, carrying children with it") and solving the
matrix algebra to match — rather than iterating on Euler angles by guesswork,
which had been the failure mode in earlier attempts.

## The Decisive Move: CAD-Verified Geometry

Rather than continuing with hand-measured angles and rulers, the robot arm
and camera mount were imported into Fusion 360 with named User Coordinate
Systems (`camera_optical_frame`, `gripper_palm`, `ALB_hook`, and separately
`Tag1`/`Tag2`/`Slot`), and a script was used to extract exact relative poses
directly from the CAD model — handling the added complexity of these frames
living in different components/occurrences within the assembly.

**Cross-check result:** the CAD-derived rotations matched the hand-derived
ones **exactly** — strong, independent confirmation that the physical
reasoning above was correct. Translations differed by a few centimeters —
expected, since hand measurement always carries some error, and CAD replaced
approximation with precision. The CAD-derived `SP_hole` transform also caught
a **missing 90° yaw term** and a **sign error in the z-offset** that hand
measurement had missed entirely.

## Final Result

With CAD-exact values loaded, one small residual remained at the seated
pose — attributed to a minor difference between where `camera_optical_frame`
was placed in CAD versus its true position on the physical arm. This was
corrected using the same algebraic technique as earlier sessions: solving
for the small adjustment to `camera_optical_frame → gripper_palm` needed to
make `ALB_hook` and `SP_hole` coincide exactly, while leaving
`gripper_palm → ALB_hook` untouched as required.

**Verified on hardware, 30° commanded twist:**
| | Result |
|---|---|
| Yaw | -30.5° (target: 30°) |
| Roll | 0.15° |
| Pitch | -0.43° |
| Translation | ~9mm |

Roll and pitch are now *smaller* than the noise floor measured in the
cleanest possible detection test earlier in the project. This is the first
result all session that looks like genuine convergence rather than a
coincidence at one measurement point.

## Outstanding

- Recommended: repeat the twist test at a different arm pose or twist angle
  to confirm this holds generally, not just at this one configuration.
- Whether the remaining ~9mm/±0.5° is acceptable depends on the physical
  tolerance of the hook-slot fit, which hasn't been separately characterized.
- Next major phase of the project: building a Gazebo simulation of the
  Kassow KR1410 (separate checklist), which will let this same validation be
  repeated against exact simulated ground truth rather than real-world
  sensor noise.
