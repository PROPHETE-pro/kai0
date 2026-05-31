# Agilex Inference — Relative EEF Models (ROS 2 Humble)

This document describes how to deploy **pi0.5 / OpenPi policies trained on UMI Pika relative end-effector data** (e.g. `pika_lerobot`) on dual Piper arms under **ROS 2 Humble**.

It complements:

- [README.md](README.md) — policy server, IPC Python env, general Agilex setup  
- [readme_ros2.md](readme_ros2.md) — ROS 2 stack for **joint-space** Kai0 models  

Use the **EEF-specific scripts** in this guide:

| Purpose | Script |
|---------|--------|
| Inference client | [`agilex_inference_openpi_temporal_smoothing_ros2_eef.py`](inference/agilex_inference_openpi_temporal_smoothing_ros2_eef.py) |
| Dry-run mock feedback | [`mock_arm_feedback_ros2_eef.py`](inference/mock_arm_feedback_ros2_eef.py) |

---

## Training data vs joint-space Kai0

Your `pika_lerobot` dataset uses:

| Field | Format |
|-------|--------|
| `state` / `action` | 14-D: `[left xyzrpy, left_gripper, right xyzrpy, right_gripper]` |
| Pose (dims 0–5, 7–12) | **Relative to episode t=0** (xyzrpy ≈ 0 at frame 0) |
| Gripper (dims 6, 13) | **Absolute** encoder distance (~0.005–0.097 in training stats) |
| `top_head` | **All black** (constant zero video) |
| `hand_left` / `hand_right` | Real wrist cameras |
| Prompt | e.g. `"fold the rag"` |

Policy server training config must use **`use_delta_joint_actions=False`** and norm stats computed on this dataset.

---

## Architecture

```text
GPU host                          IPC (ROS 2)
────────                          ───────────
serve_policy.py  ◄── websocket ── agilex_inference_*_ros2_eef.py
                                  │
                                  ├─ state: relative EEF (ref = t=0)
                                  ├─ images: black top + wrist cams
                                  └─ cmd: PosCmd (/pos_cmd_left|right)
                                         Piper EndPoseCtrl (firmware IK)
```

**Episode reference (t=0):** When you press Enter at inference start, the client reads absolute EEF from `/end_pose_stamped_*`, stores it as `episode_ref`, and expresses all subsequent states/actions relative to that pose — matching training.

---

## One-time setup

Follow [readme_ros2.md — One-time setup](readme_ros2.md#one-time-setup-ros-2--piper-workspace):

1. ROS 2 Humble + Piper workspace (`piper_ros_humble`)  
2. RealSense wrist cameras (left + right; **top camera not required**)  
3. IPC conda env `kai0_inference` + `openpi-client` ([README.md](README.md))  

Every terminal:

```bash
source /opt/ros/humble/setup.bash
source "$PIPER_ROS2_ROOT/install/setup.bash"
```

---

## Step 1 — Policy server (GPU host)

From repo root on the GPU machine:

```bash
uv run scripts/serve_policy.py policy:checkpoint \
  --policy.config=<your_eef_train_config> \
  --policy.dir=<checkpoint_dir> \
  --port=8000
```

Requirements:

- Same `TrainConfig` name and checkpoint as SFT training  
- `LerobotAgilexDataConfig(..., use_delta_joint_actions=False)`  
- Norm stats bundled under checkpoint `assets/`  

### Check policy server is listening

On GPU host or IPC (replace IP):

```bash
# TCP port open
ss -ltnp | grep 8000
# or
nc -zv <gpu_host_ip> 8000

# Health endpoint (OpenPi websocket server)
curl -s http://<gpu_host_ip>:8000/healthz
# Expected: OK
```

Websocket inference itself is binary/msgpack; use the dry-run client (below) for end-to-end verification.

---

## Step 2 — Real robot inference

### 2.1 Start hardware stack (IPC)

**CAN:**

```bash
cd "$PIPER_ROS2_ROOT" && sudo bash ./can_config.sh
```

**Wrist cameras only** (no top camera needed):

```bash
ros2 launch realsense2_camera rs_launch.py \
  camera_name:=camera_l camera_namespace:=camera_l serial_no:=<SERIAL_L>

ros2 launch realsense2_camera rs_launch.py \
  camera_name:=camera_r camera_namespace:=camera_r serial_no:=<SERIAL_R>
```

**Dual Piper:**

```bash
ros2 launch piper start_two_piper.launch.py \
  can_left_port:=can_left can_right_port:=can_right \
  auto_enable:=true girpper_exist:=true gripper_val_mutiple:=2
```

### 2.2 Pre-flight topic checks

```bash
ros2 topic hz /camera_l/camera_l/color/image_raw
ros2 topic hz /camera_r/camera_r/color/image_raw
ros2 topic hz /end_pose_stamped_left
ros2 topic hz /end_pose_stamped_right
ros2 topic hz /joint_left
ros2 topic hz /joint_right
```

All should report stable rates (EEF feedback ~200 Hz from Piper node, cameras ~30 Hz).

### 2.3 Run inference client

```bash
conda activate kai0_inference
source /opt/ros/humble/setup.bash
source "$PIPER_ROS2_ROOT/install/setup.bash"

python train_deploy_alignment/inference/agilex/inference/agilex_inference_openpi_temporal_smoothing_ros2_eef.py \
  --host <gpu_host_ip> \
  --port 8000 \
  --use_temporal_smoothing \
  --chunk_size 50 \
  --prompt "fold the rag" \
  --skip_homing
```

**Workflow:**

1. Manually move arms to a safe start pose (or use your own homing procedure).  
2. Script prompts: *Press Enter to capture t=0 EEF reference*.  
3. Client logs `episode_ref` and verifies relative state at t=0 has xyzrpy ≈ 0.  
4. Warmup infer → closed-loop EEF commands on `/pos_cmd_left`, `/pos_cmd_right`.  

**Important:** This client publishes **`PosCmd` only** (not joint commands). Piper firmware runs internal IK via `EndPoseCtrl`.

### 2.4 Gripper mapping

Training gripper uses Pika encoder distance; Piper uses a different numeric range. Defaults map:

- Policy `[0.004881, 0.096707]` ↔ Piper `[0.0, 0.08]`  

Override if you use identical Pika gripper hardware:

```bash
  --gripper_policy_min 0.004881 --gripper_policy_max 0.096707 \
  --gripper_piper_min 0.0 --gripper_piper_max 0.08
```

---

## Step 3 — Dry-run (no robot motion)

Use dry-run to verify **cameras + policy server + EEF command generation** without enabling CAN or moving arms.

### 3.1 Terminal layout

| Terminal | Role |
|----------|------|
| A, B | RealSense wrist launches (same as above) |
| C | `mock_arm_feedback_ros2_eef.py` |
| D | `agilex_inference_*_ros2_eef.py --dry_run` |
| E (optional) | Monitor `*_dry` command topics |

**Do not** run `start_two_piper.launch.py` in pure dry-run mode.

### 3.2 Mock EEF feedback

```bash
source /opt/ros/humble/setup.bash
python train_deploy_alignment/inference/agilex/inference/mock_arm_feedback_ros2_eef.py
```

Publishes:

| Topic | Message |
|-------|---------|
| `/end_pose_stamped_left`, `/end_pose_stamped_right` | `geometry_msgs/PoseStamped` |
| `/joint_left`, `/joint_right` | `sensor_msgs/JointState` (gripper channel for mapping) |

### 3.3 Dry-run inference

```bash
conda activate kai0_inference
source /opt/ros/humble/setup.bash
source "$PIPER_ROS2_ROOT/install/setup.bash"

python train_deploy_alignment/inference/agilex/inference/agilex_inference_openpi_temporal_smoothing_ros2_eef.py \
  --host <gpu_host_ip> \
  --port 8000 \
  --use_temporal_smoothing \
  --chunk_size 50 \
  --prompt "fold the rag" \
  --dry_run \
  --max_publish_step 200 \
  --dry_run_log_file /tmp/eef_dry_run.log
```

Press Enter when prompted to capture mock t=0 reference.

### 3.4 What `--dry_run` does

- Sets `--skip_homing` automatically.  
- Publishes commands on **`/pos_cmd_left_dry`** and **`/pos_cmd_right_dry`** (Piper listens on `/pos_cmd_*` without `_dry`, so **nothing moves**).  
- Logs each step to console and optional log file with **both relative and absolute** EEF:  
  `rel_left`, `rel_right`, `abs_left`, `abs_right`.  

### 3.5 Monitor dry-run command output

Start inference first so publishers exist, then:

```bash
ros2 topic list | grep dry
ros2 topic hz /pos_cmd_left_dry
ros2 topic echo /pos_cmd_left_dry piper_msgs/msg/PosCmd
```

Example fields in `PosCmd`:

```text
x, y, z          # absolute EEF position (m)
roll, pitch, yaw # absolute EEF orientation (rad)
gripper          # Piper-scaled gripper command
```

Compare console/log **relative** values to training scale (typically small deltas from t=0).

### 3.6 Monitor policy server connectivity

If dry-run hangs at warmup or prints inference errors:

```bash
curl -s http://<gpu_host_ip>:8000/healthz
nc -zv <gpu_host_ip> 8000
```

Check GPU server logs for incoming websocket connections and inference timing.

---

## Topic reference (EEF client)

| Role | Default topic |
|------|----------------|
| EEF feedback (subscribe) | `/end_pose_stamped_left`, `/end_pose_stamped_right` |
| Gripper feedback (subscribe) | `/joint_left`, `/joint_right` (7th position) |
| EEF command (publish) | `/pos_cmd_left`, `/pos_cmd_right` |
| Dry-run commands | `/pos_cmd_left_dry`, `/pos_cmd_right_dry` |
| Left wrist camera | `/camera_l/camera_l/color/image_raw` |
| Right wrist camera | `/camera_r/camera_r/color/image_raw` |
| Top camera | **Not used** (synthetic black 480×640 image) |

Override via CLI: `--endpose_left_topic`, `--img_left_topic`, etc.

---

## CLI reference (EEF client)

| Flag | Default | Description |
|------|---------|-------------|
| `--host` | `localhost` | Policy server IP |
| `--port` | `8000` | Policy server port |
| `--prompt` | `fold the rag` | Language instruction |
| `--use_temporal_smoothing` | off | **Required** for this client |
| `--chunk_size` | `50` | Action chunk size |
| `--publish_rate` | `30` | Command publish Hz |
| `--inference_rate` | `3.0` | Background infer Hz |
| `--use_real_top_camera` | off | Use real top camera instead of black image |
| `--dry_run` | off | Publish to `*_dry` topics only |
| `--dry_run_log_file` | none | Append rel/abs commands to file |
| `--skip_homing` | off | Skip any homing prompt (recommended) |
| `--gripper_policy_min/max` | 0.004881 / 0.096707 | Training gripper range |
| `--gripper_piper_min/max` | 0.0 / 0.08 | Piper gripper range |

---

## Troubleshooting

| Symptom | Likely cause | Action |
|---------|--------------|--------|
| `syn fail when get_ros_observation` | Missing wrist cam or EEF feedback | Check `ros2 topic hz` for cameras, `/end_pose_stamped_*`, `/joint_*` |
| `Missing end_pose feedback` | Piper not running, wrong topic | Launch Piper or `mock_arm_feedback_ros2_eef.py` |
| Warmup infer fails | Policy server down / wrong config | `curl http://<host>:8000/healthz`; verify checkpoint + config name |
| rel@t0 not ≈ 0 after capture | Stale buffers / motion during Enter | Hold arms still; restart client |
| Arm does not move (live run) | Using `--dry_run` or wrong cmd topic | Confirm publishing `/pos_cmd_left` not `*_dry` |
| Arm does not move (live run) | Piper not in pose mode | Ensure `PosCmd` reaches Piper; check enable status |
| Gripper wrong | Mapping mismatch | Tune `--gripper_*` flags; use identical Pika gripper if possible |
| `_TYPE_SUPPORT` errors | Stale PYTHONPATH | `unset PYTHONPATH AMENT_PREFIX_PATH`; source Humble + piper workspace after conda |

---

## Related docs

- [readme_ros2.md](readme_ros2.md) — joint-space inference, CAN, RealSense, Piper launch  
- [README.md](README.md) — policy server, training, prompts  
- [`/home/zhenyang/data/convert_pika_umi_to_lerobot_relative.py`](/home/zhenyang/data/convert_pika_umi_to_lerobot_relative.py) — training relative EEF conversion  

---

## References

Same as [README.md — References](README.md#references).
