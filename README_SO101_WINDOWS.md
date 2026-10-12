# SO-101 / LeRobot Robotic Arm: Complete Setup & Deployment Guide

This repository contains the complete codebase, Feetech servo driver patches, and pre-calibrated configurations for the **SO-ARM101 (SO-101 / SO-100)** 6-axis robotic arm system using Hugging Face **LeRobot**.

---

## Table of Contents
1. [Prerequisites & Software Installation (Windows)](#1-prerequisites--software-installation-windows)
2. [Cloning & Setting Up the Environment](#2-cloning--setting-up-the-environment)
3. [Pre-included Patches & Calibrations](#3-pre-included-patches--calibrations)
4. [Hardware Connection & COM Ports Setup](#4-hardware-connection--com-ports-setup)
5. [Servo ID Configuration (Optional / DIY Kits)](#5-servo-id-configuration-optional--diy-kits)
6. [Arm Calibration](#6-arm-calibration)
7. [Teleoperation](#7-teleoperation)
   - [7.1 Teleoperation Without Cameras](#71-teleoperation-without-cameras)
   - [7.2 Camera Setup & Teleoperation With Vision](#72-camera-setup--teleoperation-with-vision)
8. [Data Acquisition (Dataset Recording)](#8-data-acquisition-dataset-recording)
9. [Model Training](#9-model-training)
10. [Autonomous Model Inference](#10-autonomous-model-inference)
11. [Troubleshooting & FAQ](#11-troubleshooting--faq)

---

## 1. Prerequisites & Software Installation (Windows)

### 1.1 Install Git
- Download and install Git from [git-scm.com](https://git-scm.com/download/win).
- Select default options during installation.

### 1.2 Install Miniconda
- Download Miniconda for Windows (64-bit) from the [Official Miniconda Page](https://repo.anaconda.com/miniconda/).
- Run the installer.
- **Important**: Make sure to check **Add Miniconda3 to PATH** or use the **Anaconda Prompt / Miniconda Prompt** from the Start Menu.

---

## 2. Cloning & Setting Up the Environment

Open **Anaconda Prompt** or **Command Prompt (cmd.exe)** / **PowerShell**:

### 2.1 Clone This Repository
```cmd
git clone <YOUR_GITHUB_REPO_URL>
cd lerobot
```

### 2.2 Create Conda Virtual Environment
> [!IMPORTANT]
> LeRobot v0.6+ requires **Python >= 3.12**. FFmpeg 7.1.1 is required for video encoding (SVT-AV1).

```cmd
conda create -y -n lerobot python=3.12 ffmpeg=7.1.1 -c conda-forge
```

*(If prompted to accept Anaconda Terms of Service, run `conda tos accept --override-channels --channel https://repo.anaconda.com/pkgs/main` and repeat).*

### 2.3 Activate the Environment
```cmd
conda activate lerobot
```

### 2.4 Install LeRobot and Dependencies
Inside the `lerobot` directory, install LeRobot with all necessary driver and visualization extras:

```cmd
:: Install LeRobot with Feetech servo driver support
pip install -e ".[feetech]"

:: Install Rerun visualizer (required for real-time camera and teleoperation visualization)
pip install -e ".[viz]"

:: Install training dependencies (datasets, wandb, accelerate, etc.)
pip install -e ".[training]"
```

### 2.5 (Optional) Install PyTorch with CUDA for NVIDIA GPU
If your Windows PC has an NVIDIA discrete GPU (e.g., RTX 30xx/40xx):
```cmd
pip install torch torchvision torchaudio --index-url https://download.pytorch.org/whl/cu124
```
*(Verify by running: `python -c "import torch; print('CUDA Available:', torch.cuda.is_available())"`)*

---

## 3. Pre-included Patches & Calibrations

This repository already includes fixes for common issues encountered during setup:

1. **Negative Value Calibration Crash Patch**:
   - Fixed in `src/lerobot/teleoperators/so_leader/so_leader.py`, `src/lerobot/robots/so_follower/so_follower.py`, and `src/lerobot/motors/feetech/feetech.py`.
   - Prevents `ValueError: Negative values are not allowed: -156` by clamping recorded limits safely to `[0, 4095]`.
2. **Pre-calibrated Profiles**:
   - Working calibration JSON files are backed up in `./calibration/`:
     - Leader Arm: `calibration/teleoperators/so_leader/my_awesome_leader_arm.json`
     - Follower Arm: `calibration/robots/so_follower/my_awesome_follower_arm.json`
   - You can copy them to your user cache (`C:\Users\<YourUsername>\.cache\huggingface\lerobot\calibration\`) to reuse them immediately, or perform fresh calibration as described below.

---

## 4. Hardware Connection & COM Ports Setup

1. Connect the **Follower Arm** and **Leader Arm** to your PC via USB.
2. Plug in the **12V power supply** to both arms.
3. Open **Device Manager** on Windows (`Win + X` -> `Device Manager` -> expand `Ports (COM & LPT)`).
4. Identify which port belongs to the Leader Arm and Follower Arm:
   - Or run the interactive finder:
     ```cmd
     lerobot-find-port
     ```
     Unplug one arm and press `Enter` to see which port disappeared.
5. *(Recommended)* Set fixed COM port numbers:
   - In Device Manager, right-click the COM port -> **Properties** -> **Port Settings** -> **Advanced**.
   - Set Leader Arm to **COM22** and Follower Arm to **COM24**.
   - *(If using different COM port numbers, replace `COM22` and `COM24` in all commands below).*

---

## 5. Servo ID Configuration (Optional / DIY Kits)

> [!NOTE]
> Pre-assembled kits already have servo IDs set. Only run this if building from scratch or replacing servos.
> **Important**: When setting IDs, keep only ONE servo connected at a time.

```cmd
:: For Leader Arm:
lerobot-setup-motors --teleop.type=so101_leader --teleop.port=COM22

:: For Follower Arm:
lerobot-setup-motors --robot.type=so101_follower --robot.port=COM24
```
Follow the on-screen prompts to connect each motor (`gripper`, `wrist_roll`, `wrist_flex`, `elbow_flex`, `shoulder_lift`, `shoulder_pan`) sequentially.

---

## 6. Arm Calibration

Calibrate both arms so their joint limits and coordinate frames align.

### 6.1 Calibrate Leader Arm
```cmd
lerobot-calibrate --teleop.type=so101_leader --teleop.port=COM22 --teleop.id=my_awesome_leader_arm
```
1. When prompted:
   > `Press ENTER to use provided calibration file..., or type 'c' and press ENTER to run calibration:`
   - Type **`c`** and press **Enter** for a new calibration.
2. Align all joints to the middle reference diagram pose shown in the terminal, then press **Enter**.
3. **Move all joints smoothly through their full physical range of motion** (pan left/right, shoulder front/back, elbow up/down, wrist roll & flex, gripper fully open/closed).
4. Press **Enter** to save the calibration.

### 6.2 Calibrate Follower Arm
```cmd
lerobot-calibrate --robot.type=so101_follower --robot.port=COM24 --robot.id=my_awesome_follower_arm
```
1. Type **`c`** and press **Enter**.
2. Align the follower to the exact same middle reference diagram pose, then press **Enter**.
3. Move all joints through their full range of motion.
4. Press **Enter** to save.

---

## 7. Teleoperation

### 7.1 Teleoperation Without Cameras
Verify mechanical tracking between Leader and Follower arms:

```cmd
lerobot-teleoperate ^
  --robot.type=so101_follower ^
  --robot.port=COM24 ^
  --robot.id=my_awesome_follower_arm ^
  --teleop.type=so101_leader ^
  --teleop.port=COM22 ^
  --teleop.id=my_awesome_leader_arm
```
- Move the **Leader Arm** manually; the **Follower Arm** will mirror its motions synchronously.
- Press `Ctrl + C` in the terminal to stop teleoperation.

---

### 7.2 Camera Setup & Teleoperation With Vision

#### Step 1: Detect Connected Cameras
Connect your 2 USB cameras (e.g. front fixed environment camera + wrist handeye camera) and run:
```cmd
lerobot-find-cameras opencv
```
Sample images will be saved in `outputs\captured_images\`.
Check `opencv_0.png` and `opencv_1.png` to identify:
- `index_or_path: 0` -> Gripper/Wrist (handeye) camera
- `index_or_path: 1` -> Environment (fixed) camera

#### Step 2: Run Teleoperation with Cameras
```cmd
lerobot-teleoperate ^
  --robot.type=so101_follower ^
  --robot.port=COM24 ^
  --robot.id=my_awesome_follower_arm ^
  --robot.cameras="{ fixed: {type: opencv, index_or_path: 1, width: 640, height: 480, fps: 30}, handeye: {type: opencv, index_or_path: 0, width: 640, height: 480, fps: 30} }" ^
  --teleop.type=so101_leader ^
  --teleop.port=COM22 ^
  --teleop.id=my_awesome_leader_arm ^
  --display_data=true
```
A **Rerun** viewer window will open displaying live camera feeds and telemetry data.

---

## 8. Data Acquisition (Dataset Recording)

Collect imitation learning demonstrations:

```cmd
lerobot-record ^
  --robot.type=so101_follower ^
  --robot.port=COM24 ^
  --robot.id=my_awesome_follower_arm ^
  --robot.cameras="{ handeye: {type: opencv, index_or_path: 0, width: 640, height: 480, fps: 30}, front: {type: opencv, index_or_path: 1, width: 640, height: 480, fps: 30} }" ^
  --teleop.type=so101_leader ^
  --teleop.port=COM22 ^
  --teleop.id=my_awesome_leader_arm ^
  --display_data=true ^
  --dataset.repo_id=Admin/demo ^
  --dataset.num_episodes=20 ^
  --dataset.single_task="Grab the object" ^
  --dataset.push_to_hub=false
```

### Controls During Recording:
- Press **Enter** to start episode recording.
- Teleoperate the arm to complete the task.
- Press **Right Arrow (`->`)** to finish and save the episode.
- Reset the scene, then press **Right Arrow (`->`)** to proceed to the next episode.
- Press **Left Arrow (`<-`)** if you made a mistake and wish to discard and re-record the current episode.
- Press **Esc** to exit when complete.

Data is saved to: `C:\Users\<YourUsername>\.cache\huggingface\lerobot\Admin\demo`.

---

## 9. Model Training

Train an imitation learning policy (e.g. ACT - Action Chunking with Transformers):

```cmd
lerobot-train ^
  --dataset.repo_id=Admin/demo ^
  --policy.type=act ^
  --output_dir=outputs/train/act_so101 ^
  --job_name=act_so101 ^
  --policy.device=cuda ^
  --wandb.enable=false ^
  --policy.push_to_hub=false
```
*(On systems without an NVIDIA GPU, change `--policy.device=cuda` to `--policy.device=cpu` or `--policy.device=mps` for Apple Silicon).*

Trained checkpoints will be saved under: `outputs/train/act_so101/checkpoints/`.

---

## 10. Autonomous Model Inference

Deploy the trained model for autonomous execution:

```cmd
lerobot-record ^
  --robot.type=so101_follower ^
  --robot.port=COM24 ^
  --robot.id=my_awesome_follower_arm ^
  --robot.cameras="{ handeye: {type: opencv, index_or_path: 0, width: 640, height: 480, fps: 30}, front: {type: opencv, index_or_path: 1, width: 640, height: 480, fps: 30} }" ^
  --display_data=true ^
  --dataset.repo_id=Admin/eval_so101 ^
  --dataset.single_task="Grab the object" ^
  --policy.path=outputs/train/act_so101/checkpoints/last/pretrained_model ^
  --dataset.episode_time_s=60
```
*(The leader arm is not needed during inference; the follower arm acts autonomously based on visual observations).*

---

## 11. Troubleshooting & FAQ

### Q1: `RuntimeError: FeetechMotorsBus motor check failed: Missing motor IDs: - 6`
- **Cause**: Gripper servo signal wire is disconnected or loose.
- **Fix**: Check the 3-pin cable connecting motor 5 (`wrist_roll`) to motor 6 (`gripper`). Ensure 12V power supply is on.

### Q2: `ValueError: Negative values are not allowed` during calibration
- **Cause**: Joint angle exceeded boundary (-156).
- **Fix**: Already fixed in this repository code (bounds are clamped to `[0, 4095]`).

### Q3: Follower arm moves erratically or calibration feels inaccurate
- **Cause**: Calibration range was too narrow or `c` was not typed during recalibration.
- **Fix**: Run `lerobot-calibrate`, type `c`, press `Enter`, and slowly move every joint through its maximum physical reach.

### Q4: `ImportError: 'rerun-sdk' is required but not installed`
- **Fix**: Run `pip install -e ".[viz]"` inside the `lerobot` directory.

### Q5: Camera cannot be found or black screen
- **Fix**: Run `lerobot-find-cameras opencv`. Do not plug both cameras into an unpowered USB hub; connect them directly to PC USB ports.
