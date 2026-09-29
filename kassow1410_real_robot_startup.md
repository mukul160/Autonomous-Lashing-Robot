# KR1410 Real Robot — Startup Guide

**🖥️** = needs its own terminal, left running  ·  **▶️** = one-off command

---

## Network Facts

| Device | IP | Notes |
|---|---|---|
| Robot controller | `192.168.0.20` | found with `arp-scan` |
| Laptop (wired) | `192.168.0.10` | USB-Ethernet adapter `enx6c1ff771d75e` |

- The pendant's **DEVICE IP** must be the **laptop's** IP (`192.168.0.10`)
- Pinging `.10` only reaches your own laptop (0.04 ms). Ping `.20` for the robot (~0.2 ms)
- The interface name may change if you use a different USB adapter

---

## Startup Sequence

### 1. Robot side (pendant)
1. Power on the controller, release the E-stop, unlock the robot
2. Check the Ethernet settings for **PROFINET warnings**. If present, ask your supervisor how to disable it
3. **Workcell → Custom interfaces → ROS2 Interface → ROS2 Control device**
   - DEVICE IP = `192.168.0.10`
   - Namespace = `kr`
   - Press **Activate**. It only turns green if the connection succeeds
4. Set the master speed low (3% was used)

### 2. Laptop side
```bash
# cable plugged in, then:
ping 192.168.0.20          # ▶️ expect ~0.2 ms, 0% loss

source ~/alb_ws/install/setup.bash    # in EVERY terminal

ros2 topic list --no-daemon --spin-time 5 | grep kr    # ▶️ expect /kr/... topics
ros2 run kr_example_python robot_state                 # ▶️ read-only check
```

---

## Verified So Far

| Test | Result |
|---|---|
| `robot_state` | ✅ live joint / pose data |
| Pendant jogging while watching `robot_state` | ✅ |
| `move_joint_stow` | ✅ |
| `move_joint_stow_raised` | ✅ |

**Reading `robot_state`:**
- `sensed_pos`: joint angles in **degrees**, J1–J7
- `pos`: flange position in **mm**; `rot`: orientation in **degrees**
- J5 and J7 read beyond ±180°, which confirms they are continuous joints
- `robot_mode: MANUAL` may affect whether external motion commands are accepted

---

## Scripts

Run with `ros2 run kr_example_python <name>`. **No `.py`**, since the name comes from `setup.py`.

| Script | Type | Target / notes |
|---|---|---|
| `robot_state` | read-only | prints robot state |
| `move_joint_stow` | move | Stow_Pose_2B `[-0.304, -55.928, 0.485, 148.297, -181.782, 1.029, 182.748]` |
| `move_joint_stow_raised` | move | `[-36.504, -28.048, -48.969, 98.634, -112.123, -85.624, 100.281]` |

**Not yet reviewed or tested:** `move_linear`, `jog_linear`, `follow_joint`, `self_motion`, the `farmstick` scripts, the `state_machine_*` scripts

- `jog_linear_farmstick_5_apriltag_and_serial` also opens an **Arduino on `/dev/ttyUSB0`**
- `state_machine_6_can` and `state_machine_7_yaml` use **CAN**, so they need the CAN setup below
- Read a script's source before running it. Targets and speeds are hardcoded

---

## Before Any Motion

- [ ] Hand near the pendant **E-stop**
- [ ] Area around the arm clear (jig, glass wall)
- [ ] Master speed low
- [ ] Script source read; target compared to current `sensed_pos`
- [ ] Speed (`tvalue`) lowered for first runs
- [ ] `move_joint` is joint-space, so the swept path is curved. A safe start and a safe end don't guarantee a safe path
- [ ] Ctrl+C does **not** cancel a move the controller has already accepted. Use the pendant stop

---

## Optional: CAN Setup

Only the CAN scripts need this. The robot link (Ethernet) doesn't.

```bash
sudo apt install can-utils     # one-time
sudo modprobe slcan            # after each reboot

ls /dev/ttyACM*                # ▶️ confirm the port matches can_start.sh
~/can_start.sh
~/can_sanity_check.sh          # expect: UP, ERROR-ACTIVE
candump can0                   # 🖥️ shows live traffic (Ctrl+C to stop)
```

- `can_start.sh` uses `-s6` (500 kbit/s) on `/dev/ttyACM0`. The bus speed must match the device
- `bota_start.sh` needs a Bota F/T sensor driver package that isn't in this workspace

---

## Shutdown

```bash
ros2 run kr_example_python move_joint_stow    # ▶️ return to parking pose (if safe)
~/can_shutdown.sh                             # ▶️ only if CAN was started
```
Then deactivate the ROS2 device on the pendant and power down.

---

## Troubleshooting

| Symptom | Cause / fix |
|---|---|
| `ros2 topic list` shows no `/kr` topics | Discovery takes a few seconds: use `--no-daemon --spin-time 5` |
| Stale results | `ros2 daemon stop && ros2 daemon start` |
| `No executable found` | You typed `.py`. Drop it |
| "service not available, waiting again" | Normal discovery delay |
| Pendant shows activated but nothing arrives | DEVICE IP is the robot's IP or `127.0.0.1` |
| Move commands rejected | Check `robot_mode` and pendant errors |
| `not found: .../kr_example_cpp/.../local_setup.bash` | Stale leftover, harmless |
| `sudo: slcand: command not found` | `sudo apt install can-utils` |

---

## Workspace Notes

- `~/.colcon/defaults.yaml` sets `symlink-install: true`
- **Rebuild needed** after: adding a script, or editing `setup.py`, `package.xml` or `CMakeLists.txt`
- **No rebuild** for edits to existing Python scripts, but restart the node
- Duplicate packages are hidden with `COLCON_IGNORE`:
  - `orange-ros2_f/kr_msgs` (identical to the original)
  - `orange-ros2_f/kr_example/cpp`
  - `orange-ros2/kr_example/python` (superseded by the supervisor's version)
- Active `kr_example_python` is in `~/alb_ws/src/orange-ros2_f/kr_example/python/`
