# lerobot-cookbook

Personal notes — what actually works on **my** hardware for [Hugging Face LeRobot](https://github.com/huggingface/lerobot) imitation learning.

## My setup

- **Compute**: NVIDIA Jetson Orin Nano "Super" (8 GB), JetPack 6.2.2, L4T R36.5.0, CUDA 12.6
- **Python**: system 3.10.12, `pip --user` (no venv/conda)
- **LeRobot**: 0.4.4 (highest version supporting Python 3.10 — 0.5.0+ needs 3.12)
- **Arms**: SO-101 leader + follower (Feetech STS3215 motors)
- **Cameras**: USB wrist cam (`/dev/video0`, MJPG) + Intel RealSense D435 overhead (color only, addressed by serial number)
- **Training rig**: separate PC with NVIDIA RTX 3060 Ti
- **Access**: Jetson is headless, accessed via Cursor remote over Tailscale (`100.82.20.54`)

## Files

Numbered in the order you actually do them, start to finish.

**Jetson — setup & data collection**

| File | What |
|---|---|
| [00-install.md](00-install.md) | Native pip install of `lerobot[all]==0.4.4` on aarch64 + CUDA PyTorch fix |
| [01-calibration.md](01-calibration.md) | Motor ID assign + leader/follower calibration + elbow re-clocking trick |
| [02-teleop.md](02-teleop.md) | Leader → follower teleop one-liner |
| [03-camera-view.md](03-camera-view.md) | View a camera headless over Tailscale (ustreamer) to position/focus it |
| [04-recording.md](04-recording.md) | Dataset recording — 2 cameras at 640×480, h264 streaming encoding, resume |

**PC — training**

| File | What |
|---|---|
| [05-pc-install.md](05-pc-install.md) | PC install — lerobot 0.4.4 on Ubuntu 24.04 + Python 3.12, native (no venv), PEP 668 / `--break-system-packages` |
| [06-pc-training.md](06-pc-training.md) | Full PC training playbook — speed test, fresh train, resume to higher step count, status, GPU monitor, Hub verify, log tricks, cleanup |
| [07-pc-tuning.md](07-pc-tuning.md) | Measured findings on the 3060 Ti — step rates, batch/workers sweet spot, batch-16 VRAM, power cap, convergence/overfitting math |

**Jetson — deployment**

| File | What |
|---|---|
| [08-policy-eval.md](08-policy-eval.md) | Running a trained policy on the real arm (with leader teleop for reset) |

**Reference**

| File | What |
|---|---|
| [09-tmux.md](09-tmux.md) | tmux essentials — detach/reattach so long runs survive disconnect |
| [99-gotchas.md](99-gotchas.md) | Everything that wasted hours and the actual fix |

## Style

These files are working command recipes for THIS hardware. They are NOT a polished tutorial. If something here saves you a frustrating evening of debugging, great.

Last updated: 2026-07-29
