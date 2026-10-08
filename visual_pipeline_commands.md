# Visual pipeline: activation and checks

Camera -> rectify -> AprilTag detection -> TF frames. Package: `~/alb_ws/src/apriltag_bringup`.
Status as of 8 Oct 2026: verified working on the real robot (camera ~29 Hz, tag pose steady).

## 0. Every terminal

```bash
source /opt/ros/humble/setup.bash
source ~/alb_ws/install/setup.bash
```

## 1. Before launching

Check the robot camera is detected and which node it has:

```bash
v4l2-ctl --list-devices
```

Robot camera = "USB 2.0 Camera: USB Camera" (Sonix, USB ID 0c45:636d). Use its **lower** node (was `/dev/video2`; `/dev/video3` is metadata). The laptop's ASUS webcam is `/dev/video0`/`video1`; do not use it.

If the number changed, update `~/alb_ws/src/apriltag_bringup/config/usb_cam.yaml` (`video_device`). Use the plain `/dev/videoN` path there: `usb_cam` fails on `/dev/v4l/by-id/...` symlinks (it builds `/dev/../../video2`).

Make sure no old launch is still holding the camera:

```bash
lsof /dev/video2
ps aux | grep -E "usb_cam|apriltag_bringup" | grep -v grep
```

## 2. Launch (own terminal, leave running)

```bash
ros2 launch apriltag_bringup apriltag_bringup.launch.py
```

What it starts, in order:
1. `~/alb_ws/scripts/set_camera_controls.sh` (v4l2-ctl: manual exposure, `exposure_time_absolute=350`, dynamic framerate off). Runs on the by-id path of the robot camera, then exits.
2. After it exits: `usb_cam` (1920x1080, 30 fps, `frame_id: camera_optical_frame`), `rectify_node`, `apriltag_node` (tags `tag36h11:1`, `tag36h11:2`).
3. Immediately: static TFs `camera_optical_frame -> gripper_palm`, `gripper_palm -> ALB_hook` (0.12 m along Z), `tag36h11:1 -> SP_hole_from_tag1`, `tag36h11:2 -> SP_hole_from_tag2`.

Not launched: `tag_in_hook_broadcaster` (removed after an "Exec format error"; the launch file says to run it manually with `python3 scripts/tag_in_hook_broadcaster.py` from the workspace until fixed).

Stop with Ctrl-C.

## 3. Verify (separate terminals)

Image stream, expect ~29 Hz:

```bash
ros2 topic hz /image_raw
ros2 run rqt_image_view rqt_image_view        # pick /image_raw; /image_rect is the rectified one
```

Exposure landed on the robot camera (expect `auto_exposure` value=1, `exposure_time_absolute` value=350):

```bash
v4l2-ctl -d /dev/v4l/by-id/usb-Sonix_Technology_Co.__Ltd._USB_2.0_Camera_SN0001-video-index0 --list-ctrls | grep -i exposure
```

Tag detected, steady pose (the first "frame does not exist" line is just a startup wait):

```bash
ros2 run tf2_ros tf2_echo camera_optical_frame tag36h11:1
```

Hook-to-hole error (both frames hang off `camera_optical_frame`; near zero when the hook is seated, apart from slot clearance):

```bash
ros2 run tf2_ros tf2_echo ALB_hook SP_hole_from_tag1
```

To see what else is publishing: `ros2 topic list | grep -E "image|detections|tf|camera_info"`

## 4. Files that control it

| File | What it sets |
|---|---|
| `apriltag_bringup/config/usb_cam.yaml` | device, resolution, fps, pixel format, `frame_id`, calibration URL |
| `apriltag_bringup/config/apriltag.yaml` | tag family, IDs, tag size (must equal the physical black square) |
| `apriltag_bringup/launch/apriltag_bringup.launch.py` | order of startup and the static TFs |
| `~/alb_ws/scripts/set_camera_controls.sh` | manual exposure via v4l2-ctl |
| `~/.ros/camera_info/gxivision_s2m03.yaml` | intrinsics (1920x1080, matches the stream) |

After editing a config, relaunch. If the change doesn't take effect, rebuild: `colcon build --packages-select apriltag_bringup`, source, relaunch.

## 5. Troubleshooting seen so far

| Symptom | Cause / fix |
|---|---|
| Image shows the laptop webcam | `usb_cam.yaml` or the exposure script pointed at `/dev/video0` |
| rqt_image_view shows a grey gradient, `/image_raw` has 0 publishers | `usb_cam` failed to open the device; read the launch terminal. Seen: by-id symlink rejected ("not a valid V4L2 device: `/dev/../../video2`") |
| Camera "busy" | an old launch or another program holds the device (`lsof`) |
| Exposure not manual | script targeted the wrong device |
| Logger says tag seen 0/10 | stale-age check; use the current `handeye_logger.py` (`--max-age`) |

## 6. Known open points

- `camera_optical_frame -> gripper_palm` in the launch (x -0.10533, z 0.12354, rpy 0.010 -0.32083 3.12895) differs from the value earlier recorded as confirmed (x -0.0933, z 0.1197). Decide which is current.
- `SP_hole_from_tag1/2` come from the bench CAD. The real test-bed (slot inclined 30 deg) needs its own tag-to-slot transforms.
- The `hook_convergence_check.py` and `tag_in_hook_broadcaster.py` scripts are not documented here: their arguments and locations were not checked.
- Permanent fix for the "wrong camera" problem: see section 7 (applied and checked working on 8 Oct).

## 7. Permanent fix: never fall back to the laptop webcam

**Why it keeps happening.** `/dev/videoN` numbers depend on what is plugged in and enumerated first, so they change. Two things used a hard-coded number: `usb_cam.yaml` (and `usb_cam`'s own default is `/dev/video0`, the webcam) and the exposure script. When the number is wrong, nothing fails: the pipeline just starts the wrong camera.

**The fix.** Identify the camera by its stable by-id name (tied to the USB device, not to the number), resolve it to the real node at launch, and refuse to start if it is missing. One constant, used by both the camera node and the exposure script.

1. `launch/apriltag_bringup.launch.py`: add near the top (after the imports):

   ```python
   CAM_BY_ID = ('/dev/v4l/by-id/'
                'usb-Sonix_Technology_Co.__Ltd._USB_2.0_Camera_SN0001-video-index0')
   ```

   inside `generate_launch_description()`, before the `set_camera_controls` definition:

   ```python
   if not os.path.exists(CAM_BY_ID):
       raise RuntimeError(
           f'Robot camera not found at {CAM_BY_ID}. Is it plugged in? '
           'Refusing to fall back to another camera.')
   cam_dev = os.path.realpath(CAM_BY_ID)   # the real node, e.g. /dev/video2
   ```

   in `set_camera_controls`, pass the device to the script:

   ```python
   cmd=['bash', os.path.expanduser('~/alb_ws/scripts/set_camera_controls.sh'), cam_dev],
   ```

   in `usb_cam_node`, override the YAML device:

   ```python
   parameters=[usb_cam_params, {'video_device': cam_dev}],
   ```

   (`usb_cam` rejects the by-id symlink itself, which is why the path is resolved with `realpath` first.)

2. `~/alb_ws/scripts/set_camera_controls.sh`: take the device as an argument instead of hard-coding it:

   ```bash
   #!/bin/bash
   DEV="${1:?usage: set_camera_controls.sh <video device>}"
   v4l2-ctl -d "$DEV" -c auto_exposure=1
   v4l2-ctl -d "$DEV" -c exposure_time_absolute=350
   v4l2-ctl -d "$DEV" -c exposure_dynamic_framerate=0
   ```

3. `config/usb_cam.yaml`: leave `video_device` as is; the launch file overrides it. Add a comment: `# overridden by apriltag_bringup.launch.py (resolved from the by-id name)`.

4. Rebuild if the launch change doesn't take effect: `colcon build --packages-select apriltag_bringup`, source, relaunch.

**Test it.** (a) Camera plugged in: launch works, `rqt_image_view` shows the robot's view. (b) Unplug the camera: launch stops immediately with the "Robot camera not found" message instead of showing the webcam.

**Limits.** The by-id name contains the serial `SN0001`, which is generic. Two identical cameras plugged in at once could collide, so use only one. If the camera is ever swapped for a different model, run `ls -l /dev/v4l/by-id/` and update `CAM_BY_ID`.
