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

**Important:** Inference scripts under `inference/` still use **`rospy` (ROS 1)**. For end-to-end OpenPi inference you must either:

1. Run a **ROS 1 ↔ ROS 2 bridge** (`ros1_bridge` or `ros1_bridge` dynamic bridge) with the topic mapping in [Inference topics (ROS 2 ↔ scripts)](#inference-topics-ros-2--scripts), **or**
2. Port the inference nodes to `rclpy` (not covered here).

The steps below bring up the **robot stack on ROS 2**. Step 5 describes running the existing inference client with bridging or adjusted launch order.

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

Set **`lang_embeddings`** in the inference script as in [Prompt and AWBC](README.md#prompt-and-awbc-important).

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
- For inference with existing scripts: **ROS Noetic** available for `rospy` + **`ros1_bridge`** (see step 5).

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

On ROS 2, use **`ros-humble-realsense2-camera`**. Inference defaults expect these image topics (see script defaults):

| Camera | Color topic |
|---|---|
| Front | `/camera_f/color/image_raw` |
| Left wrist | `/camera_l/color/image_raw` |
| Right wrist | `/camera_r/color/image_raw` |

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

**Option B — custom multi-camera launch**  
If you already have a ROS 2 `multi_camera` launch (ported from Noetic), run it with `ros2 launch` and ensure namespaces match the table above.

Verify:

```bash
ros2 topic list | grep -E 'camera_[flr]/color'
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

#### 5. Inference node

From repo root (or directory containing `inference/`), with `kai0_inference` active:

```bash
conda activate kai0_inference
python train_deploy_alignment/inference/agilex/inference/agilex_inference_openpi_temporal_smoothing.py \
  --host <gpu_host_ip> --port 8000 \
  --ctrl_type joint --use_temporal_smoothing --chunk_size 50
```

Other scripts: see [Inference scripts](README.md#inference-scripts) in README.md.

##### Inference topics (ROS 2 ↔ scripts)

Scripts default to **ROS 1** names (`rospy`). With **`start_two_piper.launch.py`**, the natural ROS 2 names are:

| Role | ROS 1 default (script) | ROS 2 (`start_two_piper`) |
|---|---|---|
| Arm state (subscribe) | `/puppet/joint_left`, `/puppet/joint_right` | `/joint_left`, `/joint_right` |
| Arm command (publish) | `/master/joint_left`, `/master/joint_right` | `/joint_ctrl_cmd_left`, `/joint_ctrl_cmd_right` |
| End-effector cmd | `/pos_cmd_left`, `/pos_cmd_right` | `/pos_cmd_left`, `/pos_cmd_right` |

**A. `ros1_bridge` (recommended with current scripts)**  

Terminal with **Noetic** + bridge (example static bridge config — adjust if your bridge version differs):

```bash
source /opt/ros/noetic/setup.bash
source /opt/ros/humble/setup.bash

# Example: bridge puppet/master names to ROS 2 arm topics
ros2 run ros1_bridge dynamic_bridge \
  --bridge-all-topics
```

Or use a YAML bridge mapping `/puppet/joint_left` ↔ `/joint_left`, `/master/joint_left` ↔ `/joint_ctrl_cmd_left`, etc.

Run inference in a terminal with **Noetic** sourced (for `rospy`):

```bash
source /opt/ros/noetic/setup.bash
conda activate kai0_inference
python train_deploy_alignment/inference/agilex/inference/agilex_inference_openpi_temporal_smoothing.py \
  --host <gpu_host_ip> --port 8000 --ctrl_type joint --use_temporal_smoothing --chunk_size 50
```

**B. CLI overrides (only if something subscribes/publishes on ROS 1 with those names)**  

If you remap ROS 2 → ROS 1 via bridge under the **default** names, you do not need extra flags. If you run native ROS 2 inference in the future (`rclpy`), use:

```bash
python .../agilex_inference_openpi_temporal_smoothing.py \
  --puppet_arm_left_topic /joint_left \
  --puppet_arm_right_topic /joint_right \
  --puppet_arm_left_cmd_topic /joint_ctrl_cmd_left \
  --puppet_arm_right_cmd_topic /joint_ctrl_cmd_right \
  ...
```

(Camera topics can be overridden with `--img_front_topic`, `--img_left_topic`, `--img_right_topic` if your RealSense namespaces differ.)

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
| 7 | `ros1_bridge` (if using ROS 1 inference scripts) |
| 8 | Inference Python script (`conda activate kai0_inference`) |

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
| No camera topics | Namespace mismatch | Align RealSense `camera_namespace` with `/camera_f`, `/camera_l`, `/camera_r` |
| Inference: no joint data | Topic / bridge mismatch | See [Inference topics](#inference-topics-ros-2--scripts); check `ros2 topic hz /joint_left` |
| Mixed ROS errors | Noetic + Humble in one shell | Use separate terminals or bridge; avoid sourcing both distros in one process |

---

## Related docs

- [README.md](README.md) — ROS 1 full guide, policy server, prompts, inference scripts  
- [piper_ros_humble/README.MD](piper_ros_humble/README.MD) — CAN activation, single/dual arm, MoveIt, simulation  
- [Piper_ros_private-ros-noetic](Piper_ros_private-ros-noetic/) — legacy ROS 1 stack used by README.md  

---

## References

Same as [README.md — References](README.md#references).
