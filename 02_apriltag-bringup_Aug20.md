# AprilTag Detection Bringup — GXIVISION-S2M03 + ROS2 Humble

## Overview

This document covers the bringup of a full AprilTag detection and pose-visualization
pipeline using a GXIVISION-S2M03 USB camera on Ubuntu 22.04 / ROS2 Humble. The end
goal: detect a `tag36h11` AprilTag marker and visualize its live 6-DOF pose in
rviz2, as a foundation for camera-guided manipulation work on the ALB
(Automatic Lashing Bar) project.

Testing was done using a **screen-displayed AprilTag** (no printer available at the
time) rather than a printed marker — this is a valid substitute for bringup and
calibration purposes, with a couple of caveats noted below.

---

## Pipeline Structure

```
GXIVISION-S2M03 (USB, MJPG native)
        │
        ▼
┌───────────────────┐
│   usb_cam_node     │  Captures frames, decodes MJPG → RGB8,
│   (usb_cam pkg)     │  publishes /image_raw + /camera_info
└───────┬────────────┘
        │ /image_raw, /camera_info
        ▼
┌───────────────────┐
│   rectify_node      │  Undistorts the image using calibrated
│   (image_proc pkg)  │  intrinsics (K, D) → /image_rect
└───────┬────────────┘
        │ /image_rect, /camera_info
        ▼
┌───────────────────┐
│   apriltag_node     │  Detects tag36h11 markers, estimates
│   (apriltag_ros pkg) │  pose, publishes /detections + TF
└───────┬────────────┘
        │ TF (camera_optical_frame → tag36h11:N), /detections
        ▼
┌───────────────────┐
│      rviz2          │  Visualizes live image + TF tree;
│                      │  tag pose renders as a moving frame
└───────────────────┘
```

### What each stage does

| Stage | Package | Input | Output | Purpose |
|---|---|---|---|---|
| Camera driver | `usb_cam` | Physical USB camera (`/dev/video2`) | `/image_raw`, `/camera_info` | Talks to the V4L2 driver, grabs MJPG frames, decodes to RGB8, publishes as ROS image + calibration metadata |
| Rectification | `image_proc` (`rectify_node`) | `/image_raw`, `/camera_info` | `/image_rect` | Removes lens distortion using the calibrated intrinsics, required for accurate pose math |
| AprilTag detection | `apriltag_ros` | `/image_rect`, `/camera_info` | `/detections`, TF | Finds tag corners in the image, solves for 6-DOF pose relative to the camera, broadcasts as a TF frame |
| Visualization | `rviz2` | TF, `/image_rect` | — | Renders the live feed and the tag's pose as a 3D axis marker, moving in real time |

### How the camera interacts with ROS2

The GXIVISION-S2M03 is a UVC-compliant camera, so it needs no custom driver — the
Linux kernel's V4L2 subsystem exposes it directly as a `/dev/videoN` device. The
`usb_cam` node is a thin ROS2 wrapper around V4L2: it opens the device, requests a
specific format/resolution/framerate combination, and republishes each captured
frame as a `sensor_msgs/Image` message on `/image_raw`, plus a
`sensor_msgs/CameraInfo` message on `/camera_info` (populated from a calibration
file once calibration is done). Everything downstream — rectification, detection,
visualization — consumes these two topics; nothing talks to the camera hardware
directly except `usb_cam`.

---

## Setup Steps (condensed)

1. **Identify the camera** — `v4l2-ctl --list-devices` and `v4l2-ctl --list-formats-ext`
   to confirm the correct `/dev/videoN` node and supported resolutions/framerates
   (MJPG @ 1920x1080 @ 30/60fps confirmed).
2. **Scaffold a ROS2 package** (`apriltag_bringup`) with `package.xml`,
   `CMakeLists.txt`, and a `config/` directory for parameter files.
3. **Configure and verify `usb_cam`** standalone — confirm the correct device,
   resolution, and framerate are actually being used (not silently falling back
   to defaults).
4. **Calibrate the camera** using `camera_calibration`'s `cameracalibrator` tool
   and a checkerboard pattern (screen-displayed, since no printer was available),
   sized using a PPI-based pixel-to-physical-size calculation. Installed the
   resulting `K`/`D` intrinsics to `~/.ros/camera_info/gxivision_s2m03.yaml`.
5. **Rectify** the image using `image_proc`'s standalone `rectify_node`.
6. **Generate a test AprilTag** (`tag36h11`, ID 0) from the official
   [AprilRobotics/apriltag-imgs](https://github.com/AprilRobotics/apriltag-imgs)
   repo, scaled to a known physical size on-screen using the same PPI method.
7. **Run `apriltag_node`** configured for the `36h11` family and the tag's
   physical size, subscribed to the rectified image.
8. **Visualize in rviz2** — Fixed Frame set to `camera_optical_frame`, with TF
   and Image displays confirming a live, continuously-tracking tag pose.

---

## Challenges Faced and Fixes

### 1. Wrong camera device selected
**Symptom:** rviz2/rqt showed the laptop's built-in webcam instead of the GXIVISION.
**Cause:** `/dev/video0` was assumed to be the USB camera; it was actually the
laptop's internal webcam, which enumerates first.
**Fix:** Used `v4l2-ctl --list-devices` to positively identify the GXIVISION as
`/dev/video2` (first of its two device nodes — the second is a metadata node).

### 2. Parameter file silently ignored (node name mismatch)
**Symptom:** `usb_cam_node_exe` ran with all hardcoded defaults (`default_cam`,
`/dev/video0`, 640x480, YUYV, 30fps) despite passing `--params-file`.
**Cause:** ROS2 params files are keyed by node name. The yaml used
`/usb_cam_node:` as the top-level key, but the executable actually registers
itself as `/usb_cam`. Non-matching keys are silently ignored, not errored on.
**Fix:** Renamed the yaml's top-level key to `/usb_cam:`, matching the node name
visible in the log's bracketed logger tag.

### 3. Camera calibration board size mismatch
**Symptom:** Nearly ran calibration with the wrong `--size` argument.
**Cause:** Assumed a generic 8x6-corner checkerboard; the actual downloaded
pattern (OpenCV's `pattern.png`) is a 9x6-corner (10x7 square) board.
**Fix:** Verified square dimensions via `identify` (ImageMagick) before running
calibration, confirming the correct `--size 9x6` argument.

### 4. Calibration file camera-name mismatch
**Symptom:** Warning: `[gxivision_s2m03] does not match narrow_stereo in file`.
**Cause:** `cameracalibrator` always writes `camera_name: narrow_stereo` into the
saved yaml regardless of the actual camera. Cosmetic only — the K/D values still
loaded correctly.
**Fix:** Edited the saved calibration file's `camera_name` field to match, purely
for log cleanliness.

### 5. Bandwidth bottleneck misdiagnosed as the cause of frame-rate decay
**Symptom:** `/image_raw` frame rate degraded over time (down to ~15-19Hz) and
`image_proc`'s synchronizer reported 0 synchronized pairs, once a second
subscriber (`image_proc`) attached.
**Initial (incorrect) theory:** Raw RGB8 bandwidth (~186MB/s at 1080p30) was
overwhelming ROS2's DDS transport with multiple subscribers.
**Actual cause (discovered later):** A frozen/orphaned `cameracalibrator` process
from an earlier step was still running in the background, silently competing for
resources and/or subscribing. A full reboot cleared it, and `/image_raw` +
`/image_rect` held a clean, stable ~30Hz immediately afterward with no code
changes.
**Lesson:** Always verify background processes are actually dead
(`ps aux | grep ...`) rather than assuming a `pkill`/window-close succeeded,
especially after a GUI tool (like `cameracalibrator`) appears to hang.

### 6. Detour: `raw_mjpeg` pixel format mislabeling
**Symptom (during the bandwidth investigation above):** Switching `usb_cam`'s
`pixel_format` to `raw_mjpeg` (to avoid a decode/re-encode round-trip) caused
rviz2 to show random noise instead of a valid image.
**Cause:** In this driver, `raw_mjpeg` mode decodes to a raw YUV422 buffer but
mislabels the ROS image `encoding` field as `yuv422` in a way that doesn't match
the actual byte ordering expected by consumers (`image_proc`, `cv_bridge`,
rviz2), causing channels to be read misaligned.
**Fix:** Reverted to `pixel_format: "mjpeg2rgb"` (full decode to RGB8 inside
`usb_cam`), which reports a correct, unambiguous encoding downstream. This
config was already proven stable once issue #5's real cause (stale process) was
fixed, so no compressed-transport workaround was actually needed.

### 7. `image_proc`'s composed container publishes duplicate/colliding nodes
**Symptom:** `[rcl.logging_rosout] Publisher already registered for provided
node name` warning, plus intermittent sync failures, when running the plain
`ros2 run image_proc image_proc` executable.
**Cause:** That executable is a composed container that internally loads
multiple components — a debayer node and (in this setup) two rectify-related
nodes — some of which collided on the same default node name (`/RectifyNode`
appeared twice in `ros2 node list`), fighting over the same topics.
**Fix:** Ran the single needed component directly instead of the full container:
`ros2 run image_proc rectify_node ...`. Debayering was unnecessary anyway, since
the GXIVISION output is already full RGB (via `mjpeg2rgb`), not a raw Bayer
pattern.

### 8. Wrong parameter name for the camera's TF frame
**Symptom:** TF transforms for the detected tag were published under
`frame_id: default_cam` instead of the intended `camera_optical_frame`, breaking
rviz2's TF display since it had no path from the configured Fixed Frame to the
tag frame — this looked like "no pose tracking" even though detection was
working correctly underneath.
**Cause:** The yaml used the parameter name `camera_frame_id`, which does not
exist on this node — the correct parameter name is `frame_id`. ROS2 silently
drops unknown parameter file entries rather than erroring, so this went
unnoticed until specifically checked with `ros2 param get`.
**Fix:** Corrected the yaml key to `frame_id: "camera_optical_frame"`, confirmed
with `ros2 param get /usb_cam frame_id`, and re-verified the TF output showed
the correct parent frame.

---

## Final Working Configuration

**`config/usb_cam.yaml`**
```yaml
/usb_cam:
  ros__parameters:
    video_device: "/dev/video2"
    image_width: 1920
    image_height: 1080
    framerate: 30.0
    pixel_format: "mjpeg2rgb"
    frame_id: "camera_optical_frame"
    camera_name: "gxivision_s2m03"
    camera_info_url: "file:///home/mukul/.ros/camera_info/gxivision_s2m03.yaml"
```

**`config/apriltag.yaml`**
```yaml
/apriltag:
  ros__parameters:
    family: "36h11"
    size: 0.05
    max_hamming: 0
    detector:
      threads: 2
      decimate: 1.0
      blur: 0.0
      refine: true
```

**Launch sequence** (each in its own terminal, sourced with
`/opt/ros/humble/setup.bash` then `~/alb_ws/install/setup.bash`):

```bash
# 1. Camera
ros2 run usb_cam usb_cam_node_exe --ros-args \
  --params-file ~/alb_ws/src/apriltag_bringup/config/usb_cam.yaml

# 2. Rectification (standalone component, not the composed container)
ros2 run image_proc rectify_node --ros-args \
  -r image:=/image_raw \
  -r camera_info:=/camera_info \
  -r image_rect:=/image_rect \
  -p approximate_sync:=true

# 3. AprilTag detection
ros2 run apriltag_ros apriltag_node --ros-args \
  --params-file ~/alb_ws/src/apriltag_bringup/config/apriltag.yaml \
  -r image_rect:=/image_rect \
  -r camera_info:=/camera_info

# 4. Visualization
rviz2
# Fixed Frame -> camera_optical_frame
# Add -> TF
# Add -> Image (topic: /image_rect)
```

**Result:** Live rectified video in rviz2, with a `tag36h11:0` TF frame that
tracks the marker's motion continuously and smoothly.

<img width="1920" height="1200" alt="image" src="https://github.com/user-attachments/assets/b212a717-2a16-422e-96c2-6484166f1c58" />

---

## Known Caveats / Follow-ups

- **Screen-displayed tag jitter:** minor pose jitter observed is attributed to
  the tag and rviz2 sharing the same physical screen (visual interference) and
  possible rolling-shutter-vs-LCD-refresh interaction — expected to improve with
  a printed tag.
- **Calibration principal point offset:** the calibrated `cx` was offset from
  the image's geometric center by more than initially expected (~116px on a
  1920px-wide image). Not currently causing visible pose problems, but flagged
  as the first thing to revisit if lateral pose offsets look implausible later.
- **Not yet done:** tying `camera_optical_frame` into the ALB's actual TF tree
  (`base_link` → arm → camera) via a static or dynamic transform, so tag poses
  resolve into a frame usable for manipulator control rather than staying
  camera-relative.
- **Printed tag switch-over:** once printing is available, re-measure the tag's
  physical black-square size directly (replacing the PPI-estimated value) and
  update `size:` in `apriltag.yaml` accordingly — no other config changes
  needed.

---

## Addendum: Disabling Auto-Exposure ("Auto-Brightness")

### Background

The GXIVISION-S2M03 has no literal "auto brightness" control — `brightness` is
manual-only. The control actually responsible for the image self-adjusting over
time is `auto_exposure`, which ships in **Aperture Priority Mode** (auto) by
default. This needed to be disabled and pinned to a manual value for stable,
repeatable AprilTag detection.

### Camera's actual V4L2 controls

```
v4l2-ctl -d /dev/video2 --list-ctrls
```

```
User Controls
                     brightness 0x00980900 (int)    : min=-64 max=64 step=1 default=0 value=50
                       contrast 0x00980901 (int)    : min=0 max=64 step=1 default=32 value=32
                     saturation 0x00980902 (int)    : min=0 max=128 step=1 default=64 value=64
                            hue 0x00980903 (int)    : min=-40 max=40 step=1 default=0 value=0
        white_balance_automatic 0x0098090c (bool)   : default=1 value=1
                          gamma 0x00980910 (int)    : min=72 max=500 step=1 default=100 value=100
                           gain 0x00980913 (int)    : min=0 max=100 step=1 default=0 value=0
           power_line_frequency 0x00980918 (menu)   : min=0 max=2 default=1 value=1 (50 Hz)
      white_balance_temperature 0x0098091a (int)    : min=2800 max=6500 step=1 default=4600 value=4600 flags=inactive
                      sharpness 0x0098091b (int)    : min=0 max=6 step=1 default=3 value=3
         backlight_compensation 0x0098091c (int)    : min=0 max=3 step=1 default=1 value=1
Camera Controls
                  auto_exposure 0x009a0901 (menu)   : min=0 max=3 default=3 value=3 (Aperture Priority Mode)
         exposure_time_absolute 0x009a0902 (int)    : min=1 max=5000 step=1 default=157 value=157 flags=inactive
     exposure_dynamic_framerate 0x009a0903 (bool)   : default=0 value=1
```

Two things stand out: `exposure_dynamic_framerate` was already `value=1` despite
a `default=0` (something had enabled it), and `exposure_time_absolute` was
`flags=inactive` — locked out because auto exposure owned it.

### 9. `usb_cam`'s legacy control names don't match this camera's driver

**Symptom:** On every launch, the log showed:
```
unknown control 'white_balance_temperature_auto'
[INFO] ... Setting 'white_balance_temperature_auto' to 1
unknown control 'exposure_auto'
[INFO] ... Setting 'exposure_auto' to 3
unknown control 'focus_auto'
[INFO] ... Setting 'focus_auto' to 0
```
Setting `autoexposure: false` in `usb_cam.yaml` had no effect on the actual
hardware control.
**Cause:** The `usb_cam` node's internal code calls V4L2 controls by older/
legacy names (`exposure_auto`, `white_balance_temperature_auto`, `focus_auto`),
but this camera's driver reports them under different current names
(`auto_exposure`, `white_balance_automatic`) — notice the words are swapped
between the two naming schemes. `focus_auto` fails for a separate, harmless
reason: this module is fixed-focus and has no such control at all. Because the
calls error out rather than falling back to a default, this is silent — the
node starts fine and the warnings are easy to miss in the scroll of startup
logs.
**Fix:** Bypass `usb_cam.yaml` for these settings entirely. Set the real
controls directly via `v4l2-ctl` *before* launching `usb_cam`, using the exact
names the camera itself reports (`auto_exposure`, `exposure_time_absolute`,
`exposure_dynamic_framerate`). Removed `autoexposure`/`auto_white_balance` from
the yaml since they do nothing on this camera/driver combination — left in,
they're just confusing dead weight.

### Fix: manual control via v4l2-ctl

```bash
# Kill any running usb_cam node/executable first — v4l2-ctl and usb_cam
# can't both control the device cleanly at once
ps aux | grep usb_cam
kill <pid1> <pid2>
ps aux | grep usb_cam   # confirm clean before continuing

# Disable auto exposure, pin a manual value, and stop dynamic framerate
# from fighting the fixed 30fps usb_cam config
v4l2-ctl -d /dev/video2 -c auto_exposure=1
v4l2-ctl -d /dev/video2 -c exposure_time_absolute=157
v4l2-ctl -d /dev/video2 -c exposure_dynamic_framerate=0

# Verify
v4l2-ctl -d /dev/video2 --list-ctrls | grep -E "auto_exposure|exposure_time_absolute"
```

Verified result:
```
auto_exposure 0x009a0901 (menu)   : min=0 max=3 default=3 value=1 (Manual Mode)
exposure_time_absolute 0x009a0902 (int) : min=1 max=5000 step=1 default=157 value=157
```
No `flags=inactive` on `exposure_time_absolute` this time — confirms it's live
and controllable now that auto mode is off.

Tuning: with `usb_cam` running and `rqt_image_view` watching `/image_raw`, the
exposure value was adjusted live via repeated
`v4l2-ctl -d /dev/video2 -c exposure_time_absolute=<value>` calls until the
image looked correct for the room's lighting, then the chosen value was locked
in as the new default (see script below).

### Persistence: a session-start script

Since these controls must be set *before* `usb_cam` starts and can't be driven
through `usb_cam.yaml` (per issue #9 above), they're set with a small script
run once per session ahead of the camera launch, rather than typed by hand:

**`~/alb_ws/scripts/set_camera_controls.sh`**
```bash
#!/bin/bash
v4l2-ctl -d /dev/video2 -c auto_exposure=1
v4l2-ctl -d /dev/video2 -c exposure_time_absolute=157
v4l2-ctl -d /dev/video2 -c exposure_dynamic_framerate=0
```

```bash
chmod +x ~/alb_ws/scripts/set_camera_controls.sh
```

Usage, each session, before launching `usb_cam`:
```bash
~/alb_ws/scripts/set_camera_controls.sh
ros2 run usb_cam usb_cam_node_exe --ros-args \
  --params-file ~/alb_ws/src/apriltag_bringup/config/usb_cam.yaml
```

Re-checked after relaunch to make sure the node's failed
`exposure_auto`/`focus_auto` calls didn't silently reset anything:
```bash
v4l2-ctl -d /dev/video2 --list-ctrls | grep -E "auto_exposure|exposure_time_absolute"
```
Controls held as set — no reset observed.

### Follow-up

- **Not yet done:** a udev rule to run `set_camera_controls.sh` automatically
  on device plug-in, removing the manual per-session step entirely. Deferred
  until the exposure value is fully settled, since the script still needs
  hand-tuning occasionally as lighting conditions change.
