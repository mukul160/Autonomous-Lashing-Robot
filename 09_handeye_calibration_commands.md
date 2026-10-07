# Hand-eye calibration session: commands in order

Goal: find the camera pose in the robot's end-effector frame (flange -> camera), validate the bench `gripper_palm` setup, and check the roll/pitch/yaw order of the robot's reported pose.

Assumes the robot startup sequence (see `kassow1410_real_robot_startup.md`) is done and the ROS2 CBun is active. Scripts live in `~/alb_ws/tools/handeye/`.

## 0. Every terminal

```bash
source /opt/ros/humble/setup.bash
source ~/alb_ws/install/setup.bash
```

## 1. Check the link and state (no motion)

```bash
ping -c 3 192.168.0.20
ros2 topic list --no-daemon --spin-time 5 | grep kr
ros2 topic echo --once /kr/system/state --field pos     # expect mm
ros2 topic echo --once /kr/system/state --field rot     # expect degrees
```

## 2. Frame check (optional, MOVES the arm: clear the workspace first)

```bash
ros2 run kr_example_python move_joint_stow_raised
ros2 topic echo --once /kr/system/state --field pos
ros2 topic echo --once /kr/system/state --field rot
```

Compare with the example's target of about `[26, -396, 1003]` mm. A match suggests WORLD (`ref: 0`). This assumes the script ends at that pose, which is not verified.

## 3. Start the camera and tag detection (own terminal)

```bash
ros2 launch apriltag_bringup apriltag_bringup.launch.py
```

If "file not found": `ls ~/alb_ws/src/apriltag_bringup/launch/`

Check the tag frame (own terminal). It should print a steady pose:

```bash
ros2 run tf2_ros tf2_echo camera_optical_frame tag36h11:1
```

If the frame name differs, pass it to the logger with `--tag-frame <name>`.

## 4. Inserted-state capture, then pose capture

Keep fixed for the whole session: tag, camera bracket and gripper finger state (grasp configuration).

Start the logger (own terminal):

```bash
cd ~/alb_ws/tools/handeye
python3 handeye_logger.py --out cap1.csv
```

1. Run your grab-and-insert script so the hook is seated in the slot. Press **Enter** to record this pose (the validation point).
2. Release, retreat, then jog from the pendant or hand-guide to about 12 more poses with the tag visible. Press **Enter** at each, holding the arm still.
3. Vary the orientation a lot: rotate 30 degrees or more about all three axes. Clustered poses give a wrong answer that still looks fine.
4. Keep clear of the jig and glass wall.
5. Type `q` + Enter to finish.

Each line should report the tag distance and a small detection jitter.

## 5. Solve

```bash
cd ~/alb_ws/tools/handeye
python3 handeye_solve.py cap1.csv --euler-sweep
```

If a clear winner is named, re-solve with it (use `xyz` if that is the winner):

```bash
python3 handeye_solve.py cap1.csv --euler <winner>
```

If the sweep says NOT CONCLUSIVE, capture more poses with larger, more varied rotations and repeat steps 4 and 5.

## 6. What to check in the output

- The diversity warning does not fire (3 of 3 rotation axes).
- PARK, HORAUD and PARK-NP agree. A split between them is a warning.
- No per-sample deviation is an outlier.
- Tag spread is near the detection jitter. A small spread alone does not prove accuracy.

## 7. Validate against the bench setup

Run your own `hook_convergence_check.py` at the inserted pose and compare its result with the new calibration. The twist lock has clearance, so expect a few mm of residual. Repeat the insertion a few times to see how much it varies.

## Cleanup

Stop the camera launch and the logger with Ctrl-C. Keep `cap1.csv` and the solver output: paste them back to check the result.

## Still unconfirmed until this session

- The logger against a live `kr_msgs/msg/SystemState` message.
- The roll/pitch/yaw order of the reported `rot`.
- Whether `pos`/`rot` is reported in WORLD or BASE.
- The tag frame name from `apriltag_ros`.
