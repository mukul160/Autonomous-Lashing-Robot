# Seated-hook experiment: startup sequence

Goal: with the key (hook) seated in the lock (slot), measure camera -> gripper_palm and the twist motion.
Two captures per insertion, in either order. Your state machine goes TWISTED (locked, `SP_final_locked`) first, then INSERTED (untwisted, aligned with the slot, `SP_final_unlocked`), then removes the key. About 5 insertions.

Needs: robot powered with your usual startup sequence done, camera plugged in, tags 1 and 2 visible from the seated pose.

## 0. Every terminal

```bash
source /opt/ros/humble/setup.bash
source ~/alb_ws/install/setup.bash
```

## 1. Robot startup

Your documented sequence (power, brakes, ROS2 CBun active). Skip nothing.
Check `/kr/system/state` is publishing:

```bash
ros2 topic echo /kr/system/state --once
```

(The seated script reads TF only, but this confirms the robot link is alive.)

## 2. Camera + tags (own terminal, leave running)

```bash
ros2 launch apriltag_bringup apriltag_bringup.launch.py
```

Launch stops with "Robot camera not found" if the camera is unplugged. That is intended.

## 3. Checks (separate terminal)

```bash
ros2 topic hz /image_raw                                   # ~29 Hz
ros2 run tf2_ros tf2_echo camera_optical_frame tag36h11:1  # steady pose
ros2 run tf2_ros tf2_echo camera_optical_frame tag36h11:2
ros2 run tf2_ros tf2_echo ALB_hook SP_hole_from_tag1       # frames exist
```

Both tags must be detected from the seated pose. Check with the key inserted, before the first capture.

## 4. Run the script (own terminal)

```bash
cd ~/alb_ws/tools/handeye
python3 seated_calib.py --tags 1 2 --expected-twist 30 --first twisted --hole-state twisted --out seated1.csv
```

Add `--palm-hook-xyz 0 0 0.12` only if the palm-to-hook offset is not the launch value.

## 5. Per insertion (with the state machine, option 11 drill)

1. Palm docked at `SP_final_locked`, hook locked in the slot (the existing step-3 pause). Hold still.
2. Press **Enter** in `seated_calib`: TWISTED capture.
3. Press Enter in the state machine: it closes the gripper and untwists to `SP_final_unlocked`.
4. At the new pause after `SP_final_unlocked` (add `input("INSERTED: capture, then Enter: ")` after the step-5 print), press **Enter** in `seated_calib`: INSERTED capture.
5. Press Enter in the state machine to continue (it removes the key). Repeat for the next insertion.

Keys (in `seated_calib`):
- **Enter**: captures the next state of this insertion (twisted first with `--first twisted`). When both are in, the next capture starts a new insertion.
- **t** / **i**: force TWISTED / INSERTED (replaces it if already taken)
- **u**: undo the last capture
- **n**: abandon this insertion and start a new one
- **q**: finish and print the summary

Do not move the camera, tags or slot box during the session.

## 6. Read the summary

- TWISTED (hook upright, locked): hook -> hole near zero if the chain is right (SP_hole is the upright bench slot). INSERTED (aligned with the tilted slot): about 30 deg off, and the suggested `camera_optical_frame -> gripper_palm` static TF.
- Twisted: angle (expect about 30), axis, pitch, and hook-origin-to-axis offset.
- Tag 1 vs tag 2 agreement: a small difference means the result is trustworthy.

Paste the summary back for review before changing the launch file.

## 7. If you apply the suggested transform

Edit `gripper_palm_static_tf` in `apriltag_bringup.launch.py` with the printed `--x --y --z --roll --pitch --yaw` values, then:

```bash
cd ~/alb_ws && colcon build --packages-select apriltag_bringup
source install/setup.bash
```

Relaunch, re-seat the key, and check `ros2 run tf2_ros tf2_echo ALB_hook SP_hole_from_tag1` is near zero.

## Troubleshooting

| Symptom | Fix |
|---|---|
| Tag not found or stale | check the launch terminal; `--max-age` can be raised from 1.0 |
| Only one tag visible | run with `--tags 1`; agreement check is lost |
| Webcam image | camera by-id fix in `visual_pipeline_commands.md`, section 7 |
| Twist angle far from 30 | undo with `u` and redo that insertion; check the key fully seated first |
| Camera dies mid-run | see the USB notes: camera on its own port, not behind the CAN hub; `respawn=True` on `usb_cam` |
| Palm suggestion looks wrong | check `--hole-state`: it must be the state where the hook frame equals SP_hole (twisted = hook upright) |
