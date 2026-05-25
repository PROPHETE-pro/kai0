# Agilex Inference (ROS 2 Humble)

This document is the **ROS 2** counterpart to [README.md](README.md). It mirrors the **Full setup (one-time and run-time)** flow using **ROS 2 Humble** and the in-repo package [`piper_ros_humble`](piper_ros_humble/).

Shared topics (policy server on GPU host, IPC Python env, prompts, AWBC, inference scripts, references) are unchanged — see [README.md](README.md).

---

## ROS 1 vs ROS 2 (quick reference)

| ROS 1 (README.md) | ROS 2 (this doc) |
|---|---|
| `roscore` | Not used. Source `/opt/ros/humble/setup.bash` and your workspace `install/setup.bash`. |
| `roslaunch piper start_ms_piper.launch` | `ros2 launch piper start_two_piper.launch.py` (dual-arm; no master/slave launch in Humble tree) |
| `roslaunch realsense2_camera multi_camera.launch` | `ros2 launch realsense2_camera ...` (see [RealSense (ROS 2)](#3-realsense-cameras-ros-2) below) |
| `Piper_ros_private-ros-noetic` | `piper_ros_humble` |
| `PKG_ROOT` | `PIPER_ROS2_ROOT` → path to `piper_ros_humble` |
| `agilex_inference_openpi_temporal_smoothing.py` (`rospy`) | [`agilex_inference_openpi_temporal_smoothing_ros2.py`](inference/agilex_inference_openpi_temporal_smoothing_ros2.py) (`rclpy`) |

**Recommended:** Use the native ROS 2 inference script [`agilex_inference_openpi_temporal_smoothing_ros2.py`](inference/agilex_inference_openpi_temporal_smoothing_ros2.py). It talks directly to `start_two_piper.launch.py` topics (no `ros1_bridge`).

**Legacy:** The original scripts under `inference/` still use **`rospy` (ROS 1)**. If you keep them, you need **ROS Noetic + `ros1_bridge`** — see [Legacy: ROS 1 inference + bridge](#legacy-ros-1-inference--bridge) below.

---

## One-time setup (ROS 2 + Piper workspace)

Run on the **IPC** (Ubuntu 22.04).

### 1. ROS 2 Humble and Piper dependencies

```bash
# ROS 2 (if not already installed)
sudo apt update
sudo apt install ros-humble-desktop

# Piper / ros2_control dependencies (from piper_ros_humble README)
sudo apt install -y \
  ros-humble-ros2-control \
  ros-humble-ros2-controllers \
  ros-humble-controller-manager \
  ros-humble-joint-state-publisher-gui \
  ros-humble-robot-state-publisher \
  ros-humble-xacro

# CAN tools
sudo apt install -y can-utils ethtool iproute2

# RealSense ROS 2 driver
sudo apt install -y ros-humble-realsense2-camera

# Inference client (rclpy + image conversion)
sudo apt install -y \
  ros-humble-cv-bridge \
  ros-humble-sensor-msgs \
  ros-humble-geometry-msgs \
  ros-humble-nav-msgs
```

Add to `~/.bashrc` or `~/.zshrc` (use **one** ROS distro per shell — do not mix Noetic and Humble in the same terminal):

```bash
source /opt/ros/humble/setup.bash   # or setup.zsh
```

### 2. Piper SDK (Python)

```bash
pip3 install python-can scipy piper_sdk
```

See also [Piper SDK](https://github.com/agilexrobotics/piper_sdk) and [piper_ros_humble/README.MD](piper_ros_humble/README.MD).

### 3. Build `piper_ros_humble`

```bash
export PIPER_ROS2_ROOT=/path/to/kai0/train_deploy_alignment/inference/agilex/piper_ros_humble
cd "$PIPER_ROS2_ROOT"
colcon build
```

Add to your shell (after Humble):

```bash
source "$PIPER_ROS2_ROOT/install/setup.bash"
```

### 4. IPC Python env (unchanged)

Follow [IPC Python environment (one-time setup)](README.md#ipc-python-environment-one-time-setup) in README.md (`kai0_inference` conda env, `requirements_inference_ipc.txt`, `openpi-client`).

### 5. Configure dual CAN (dual Piper arms)

Edit `piper_ros_humble/can_config.sh` (or use `can_muti_activate.sh` / `find_all_can_port.sh` as in [piper_ros_humble/README.MD §2](piper_ros_humble/README.MD)) so both arms get **1 Mbps** interfaces, typically:

- `can_left` — left arm  
- `can_right` — right arm  

Verify with `ifconfig -a | grep can` before run-time.

---

## Inference Setup

Unchanged from [README.md — Inference Setup](README.md#inference-setup):

1. **GPU host:** `uv run scripts/serve_policy.py ...`  
2. **IPC:** Start the ROS 2 robot stack below, then run the inference script with `--host <gpu_host_ip> --port 8000`.

Set **`lang_embeddings`** in the inference script (ROS 2: `agilex_inference_openpi_temporal_smoothing_ros2.py`) as in [Prompt and AWBC](README.md#prompt-and-awbc-important).

---

## Full setup (one-time and run-time)

Flow **on the IPC** after the policy server is running on the GPU host. Replace paths as needed.

### Prerequisites

- Piper SDK and hardware (CAN, arms) configured.  
- `kai0_inference` conda env and `openpi-client` installed ([README.md](README.md)).  
- **ROS 2 Humble**, `piper_ros_humble` built and sourced.  
- `PIPER_ROS2_ROOT` set; workspace sourced: `source "$PIPER_ROS2_ROOT/install/setup.bash"`.  
- Policy server IP and port (default `8000`).  
- Optional: **tmux** for multi-pane layout.  
- For inference: **only Humble** in the inference terminal (do not source Noetic in the same shell).

### Environment (every terminal)

```bash
source /opt/ros/humble/setup.bash
source "$PIPER_ROS2_ROOT/install/setup.bash"
export PIPER_ROS2_ROOT=/path/to/kai0/train_deploy_alignment/inference/agilex/piper_ros_humble
```

ROS 2 does **not** use `roscore`. Optionally check the daemon:

```bash
ros2 daemon status
# ros2 daemon start   # only if status reports not running
```

---

### Steps on the IPC (manual or via tmux)

#### 1. ROS 2 environment (replaces `roscore`)

In each pane that runs ROS 2 nodes:

```bash
source /opt/ros/humble/setup.bash
source "$PIPER_ROS2_ROOT/install/setup.bash"
```

No separate “master” process is required.

---

#### 2. CAN config (Piper; may need sudo)

Dual-arm default in-repo (`can_config.sh` expects 2 modules → `can_left` / `can_right`):

```bash
cd "$PIPER_ROS2_ROOT" && sudo bash ./can_config.sh
```

Single arm (example):

```bash
cd "$PIPER_ROS2_ROOT" && sudo bash ./can_activate.sh can0 1000000
```

Alternative: `bash can_muti_activate.sh` after editing `USB_PORTS` in that script — see [piper_ros_humble/README.MD §2.3](piper_ros_humble/README.MD).

---

#### 3. RealSense cameras (ROS 2)

ROS 1 used:

```bash
roslaunch realsense2_camera multi_camera.launch
```

On ROS 2, use **`ros-humble-realsense2-camera`**. When both `camera_name` and `camera_namespace` are set to the same value (as below), RealSense publishes under **`/<ns>/<ns>/color/image_raw`** (namespace appears twice). The ROS 2 inference script defaults match that layout:

| Camera | Color topic |
|---|---|
| Front | `/camera_f/camera_f/color/image_raw` |
| Left wrist | `/camera_l/camera_l/color/image_raw` |
| Right wrist | `/camera_r/camera_r/color/image_raw` |

**Option A — three launches (one serial per terminal)**  
Replace `<SERIAL>` with each device serial from `rs-enumerate-devices`:

```bash
# Terminal A
ros2 launch realsense2_camera rs_launch.py \
  camera_name:=camera_f camera_namespace:=camera_f serial_no:=<SERIAL_F>

# Terminal B
ros2 launch realsense2_camera rs_launch.py \
  camera_name:=camera_l camera_namespace:=camera_l serial_no:=<SERIAL_L>

# Terminal C
ros2 launch realsense2_camera rs_launch.py \
  camera_name:=camera_r camera_namespace:=camera_r serial_no:=<SERIAL_R>
```

**Option B — shorter topic names**  
To get `/camera_l/color/image_raw` (single namespace), use a different `camera_name`, e.g. `camera_name:=cam camera_namespace:=camera_l`. Then override inference CLI: `--img_left_topic /camera_l/cam/color/image_raw`.

**Option C — custom multi-camera launch**  
If you already have a ROS 2 `multi_camera` launch, ensure topic names match the script defaults or pass `--img_*_topic` flags.

Verify:

```bash
ros2 topic hz /camera_f/camera_f/color/image_raw
ros2 topic hz /camera_l/camera_l/color/image_raw
ros2 topic hz /camera_r/camera_r/color/image_raw
```

---

#### 4. Piper arms (dual-arm control)

ROS 1 equivalent:

```bash
roslaunch piper start_ms_piper.launch mode:=1 auto_enable:=true
```

ROS 2 Humble uses **two `piper_single_ctrl` nodes** (no `start_ms_piper` in this tree):

```bash
cd "$PIPER_ROS2_ROOT"
ros2 launch piper start_two_piper.launch.py \
  can_left_port:=can_left \
  can_right_port:=can_right \
  auto_enable:=true \
  girpper_exist:=true \
  gripper_val_mutiple:=2
```

Check topics:

```bash
ros2 topic list | grep -E 'joint_(left|right)|joint_ctrl_cmd'
```

**Note:** `mode:=1` master/slave teleop from Noetic is **not** available here; the inference stack commands joints directly on the control topics (see mapping below).

---

#### 5. Inference node (ROS 2 / rclpy)

In the **same terminal** as step 1, source Humble + `piper_ros_humble`, then run the ROS 2 client.

**Important:** Source ROS 2 *after* `conda activate`. If you only activate conda and run Python, you may get `_TYPE_SUPPORT` / ROS 1 message errors.

```bash
conda activate kai0_inference
source /opt/ros/humble/setup.bash
source "$PIPER_ROS2_ROOT/install/setup.bash"

python train_deploy_alignment/inference/agilex/inference/agilex_inference_openpi_temporal_smoothing_ros2.py \
  --host <gpu_host_ip> --port 8000 \
  --ctrl_type joint --use_temporal_smoothing --chunk_size 50
```

Or use the wrapper (sources Humble + workspace for you):

```bash
conda activate kai0_inference
bash train_deploy_alignment/inference/agilex/inference/run_inference_ros2.sh \
  --host <gpu_host_ip> --port 8000 \
  --ctrl_type joint --use_temporal_smoothing --chunk_size 50
```

The script will move the arms to a home pose, then wait for **Enter** before connecting to the policy server and starting closed-loop control.

##### Dry run: verify commands without moving the robot

Use this when the workspace is cluttered or you want to test the full pipeline (cameras + policy + command generation) **without** driving the real arms.

**Do not** run `start_two_piper.launch.py` in this mode (or Piper will still enable if launched). Instead:

| Terminal | Command |
|---|---|
| A | Three RealSense launches (as in step 3) |
| B | `python .../mock_arm_feedback_ros2.py` — fake `/joint_left`, `/joint_right` |
| C | `ros2 topic echo /joint_ctrl_cmd_left_dry` (optional monitor) |
| D | Inference with `--dry_run` |

```bash
# Terminal B — mock proprio (no CAN / no Piper)
source /opt/ros/humble/setup.bash
python train_deploy_alignment/inference/agilex/inference/mock_arm_feedback_ros2.py

# Terminal C — watch commands (optional; start AFTER dry-run inference is running)
ros2 topic list -t | grep dry
ros2 topic hz /joint_ctrl_cmd_left_dry
ros2 topic echo /joint_ctrl_cmd_left_dry
# If echo asks for a type before any publisher exists, use one of:
#   ros2 topic echo /joint_ctrl_cmd_left_dry sensor_msgs/JointState
#   ros2 topic echo /joint_ctrl_cmd_left_dry sensor_msgs/msg/JointState

# Terminal D — inference dry run
source /opt/ros/humble/setup.bash
conda activate kai0_inference
python train_deploy_alignment/inference/agilex/inference/agilex_inference_openpi_temporal_smoothing_ros2.py \
  --host <gpu_host_ip> --port 8000 \
  --ctrl_type joint --use_temporal_smoothing --chunk_size 50 \
  --dry_run --max_publish_step 200 \
  --dry_run_log_file /tmp/piper_dry_run.log
```

What `--dry_run` does:

- Skips homing (no move to `left0` / `right0`).
- Publishes commands on **`/joint_ctrl_cmd_left_dry`** and **`/joint_ctrl_cmd_right_dry`** only — Piper subscribes to `/joint_ctrl_cmd_*` without `_dry`, so **nothing moves**.
- Prints each command to the console; optional `--dry_run_log_file` saves them.

If Piper is already running and you only want to block motion, `--dry_run` is still safe: commands never reach the Piper nodes.

**Pre-flight checks** (with cameras + Piper already running):

```bash
ros2 topic hz /camera_f/camera_f/color/image_raw
ros2 topic hz /camera_l/camera_l/color/image_raw
ros2 topic hz /camera_r/camera_r/color/image_raw
ros2 topic hz /joint_left
ros2 topic hz /joint_right
```

Other OpenPi scripts (`*_rtc.py`, `*_sync.py`, etc.) are still ROS 1 only; port them the same way or use the bridge path below.

##### Inference topics (`start_two_piper` ↔ ROS 2 script)

The ROS 2 script defaults match `start_two_piper.launch.py`:

| Role | Topic |
|---|---|
| Arm state (subscribe) | `/joint_left`, `/joint_right` |
| Arm command (publish) | `/joint_ctrl_cmd_left`, `/joint_ctrl_cmd_right` |
| End-effector cmd | `/pos_cmd_left`, `/pos_cmd_right` |
| Front / left / right color | `/camera_f/camera_f/color/image_raw`, `/camera_l/camera_l/color/image_raw`, `/camera_r/camera_r/color/image_raw` |

Override with CLI flags if your namespaces differ, e.g. `--img_front_topic`, `--puppet_arm_left_topic`.

**Joint command names:** The ROS 2 Piper node expects `JointState.name` = `joint1`…`joint6`, `gripper` (not ROS 1 `joint0`…`joint6`). The `_ros2.py` script sets these automatically.

##### Legacy: ROS 1 inference + bridge

Only if you still run `agilex_inference_openpi_temporal_smoothing.py` (`rospy`):

| Role | ROS 1 default | ROS 2 (`start_two_piper`) |
|---|---|---|
| Arm state | `/puppet/joint_left`, `/puppet/joint_right` | `/joint_left`, `/joint_right` |
| Arm command | `/master/joint_left`, `/master/joint_right` | `/joint_ctrl_cmd_left`, `/joint_ctrl_cmd_right` |

Bridge example (separate terminal; requires Noetic + `ros1_bridge`):

```bash
source /opt/ros/noetic/setup.bash
source /opt/ros/humble/setup.bash
ros2 run ros1_bridge dynamic_bridge --bridge-all-topics
```

Run the ROS 1 script in a terminal with **Noetic** sourced (not Humble).

---

### Optional: multi-pane layout with tmux

- Session name, e.g. `piper_demo_ros2`.  
- Suggested panes (no `roscore` pane):

| Pane | Command |
|---|---|
| 1 | Source Humble + workspace (keep open or use for `ros2 topic echo`) |
| 2 | `sudo bash ./can_config.sh` in `$PIPER_ROS2_ROOT` |
| 3–5 | RealSense launches (`camera_f`, `camera_l`, `camera_r`) or one multi-camera launch |
| 6 | `ros2 launch piper start_two_piper.launch.py ...` |
| 7 | Inference: `agilex_inference_openpi_temporal_smoothing_ros2.py` (`conda activate kai0_inference`) |

Log each pane optionally:

```bash
2>&1 | tee "$LOG_DIR/<pane_name>.log"
```

Stop: `tmux kill-session -t piper_demo_ros2` or Ctrl+C in each pane.

Set `PIPER_ROS2_ROOT`, `DEMO_ROOT`, and `LOG_DIR` to your paths.

---

## Troubleshooting

| Symptom | Likely cause | Action |
|---|---|---|
| `command not found: roscore` | ROS 2 workflow | Do not use `roscore`; source Humble + workspace (step 1). |
| `command not found: ros2` | Humble not sourced | `source /opt/ros/humble/setup.bash` |
| `package 'piper' not found` | Workspace not built/sourced | `colcon build` && `source install/setup.bash` |
| CAN / arm not responding | Wrong bitrate or names | Use **1000000**; names `can_left` / `can_right` must match launch args |
| No camera topics | Namespace mismatch | Use `ros2 topic list`; with duplicate ns see `/camera_l/camera_l/color/image_raw` or pass `--img_*_topic` |
| Inference: no joint data | Topics not publishing | `ros2 topic hz /joint_left`; ensure Piper launch is running |
| `syn fail when get_ros_observation` | Camera or joint sync | Check all three color topics and `/joint_left` / `/joint_right` |
| Arm does not move | Wrong `JointState.name` | Use `_ros2.py` (sends `joint1`…`gripper`); ROS 1 names break ROS 2 Piper |
| `No module named 'piper_msgs'` | Workspace not sourced | `source "$PIPER_ROS2_ROOT/install/setup.bash"` before Python |
| `_TYPE_SUPPORT` / ROS 1 message type | Stale `PYTHONPATH` from parent shell (e.g. `~/pika_ros` in `.bashrc`) | Use `run_inference_ros2.sh` (clears paths before sourcing Humble); or `unset PYTHONPATH AMENT_PREFIX_PATH` then source Humble + piper workspace |
| Dry run: `syn fail` | No joint feedback | Run `mock_arm_feedback_ros2.py` or Piper for `/joint_left` / `/joint_right` |
| `The passed message type is invalid` (echo) | Wrong type string or no publisher yet | Start `--dry_run` inference first; then `ros2 topic echo /joint_ctrl_cmd_left_dry` with no type, or try `sensor_msgs/JointState` |
| Dry run: arm still moves | Piper + real cmd topics | Use `--dry_run`; confirm echo on `*_dry` topics, not `/joint_ctrl_cmd_left` |
| Mixed ROS errors | Noetic + Humble in one shell | Inference terminal: Humble only; bridge (if any) in another terminal |

---

## Related docs

- [README.md](README.md) — ROS 1 full guide, policy server, prompts, inference scripts  
- [piper_ros_humble/README.MD](piper_ros_humble/README.MD) — CAN activation, single/dual arm, MoveIt, simulation  
- [Piper_ros_private-ros-noetic](Piper_ros_private-ros-noetic/) — legacy ROS 1 stack used by README.md  

---

## References

Same as [README.md — References](README.md#references).
