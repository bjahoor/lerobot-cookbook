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

| File | What |
|---|---|
| [00-install.md](00-install.md) | Native pip install of `lerobot[all]==0.4.4` on aarch64 + CUDA PyTorch fix |
| [01-power-and-clocks.md](01-power-and-clocks.md) | `nvpmodel` and `jetson_clocks` setup (mode 2 = MAXN_SUPER, NOT mode 0!) |
| [02-calibration.md](02-calibration.md) | Motor ID assign + leader/follower calibration + elbow re-clocking trick |
| [03-teleop.md](03-teleop.md) | Leader → follower teleop one-liner |
| [04-recording.md](04-recording.md) | Dataset recording — 2 cameras at 640×480, h264 streaming encoding, resume |
| [05-training.md](05-training.md) | `lerobot-train` ACT on the PC |
| [06-policy-eval.md](06-policy-eval.md) | Running a trained policy on the real arm (with leader teleop for reset) |
| [10-pc-install.md](10-pc-install.md) | PC install — lerobot 0.4.4 on Ubuntu 24.04 + Python 3.12, native (no venv), PEP 668 / `--break-system-packages` |
| [11-pc-training.md](11-pc-training.md) | Full PC training playbook — speed test, fresh train, resume to higher step count, tmux, status, GPU monitor, Hub verify, log tricks, cleanup |
| [12-pc-tuning.md](12-pc-tuning.md) | Measured findings on the 3060 Ti — step rates, batch/workers sweet spot, batch-16 VRAM, power cap, convergence/overfitting math |
| [99-gotchas.md](99-gotchas.md) | Everything that wasted hours and the actual fix |

## Style

These files are working command recipes for THIS hardware. They are NOT a polished tutorial. If something here saves you a frustrating evening of debugging, great.

Last updated: 2026-05-28
