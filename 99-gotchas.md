# Gotchas — things that wasted hours and the actual fix

A.k.a. "what I wish someone had told me before I spent a Sunday on it."

---

## 1. The "Record loop is running slower (17 Hz)" warning is misleading

**Symptom**: you run a policy and get spammed with `WARNING ... Record loop is running slower (15-17 Hz) than the target FPS (30 Hz)` every ~3 seconds.

**You think**: the loop is stuck at 17 Hz steady-state, the platform is too slow.

**Actually**: the warning is **per-tick instantaneous** (`1/dt_s` of a single tick that exceeded the budget), NOT a rolling average. With ACT (`n_action_steps=100`), one tick out of every 100 fires the full transformer forward pass (~60 ms, reported as "17 Hz") and triggers the warning; the OTHER 99 ticks pop a cached action and complete in ~5 ms. The loop genuinely runs at ~30 Hz steady-state.

**How to verify**: enable lerobot DEBUG logs and count `read state` lines per second (each line = one tick):
```bash
python3 << 'PYEOF'
import sys
from pathlib import Path
import lerobot.scripts.lerobot_record as r
_orig = r.init_logging
r.init_logging = lambda **k: _orig(**{**k, "console_level": "DEBUG", "log_file": Path("/tmp/lerobot_debug.log")})
sys.argv = ["lerobot-record", "...your args..."]
r.main()
PYEOF

grep "follower.py:185.*read state" /tmp/lerobot_debug.log | awk '{print $3}' | uniq -c
# ~30 per second = real loop rate, regardless of what the warnings say
```

**Don't fix what isn't broken**: I burned hours testing different `streaming_encoding`, `num_image_writer_processes`, `encoder_threads`, `display_data`, etc. None of it mattered because the loop was already at 30 Hz.

---

## 2. TTS deadlock on headless Jetson

**Symptom**: `lerobot-record` / `lerobot-teleoperate` hang forever right when the run should be exiting (no progress, no error). Or they look like they're stuck uploading.

**Cause**: lerobot calls `spd-say --wait "Stop recording"` (blocking text-to-speech). On a headless Jetson the `speech-dispatcher` daemon wedges and `spd-say` hangs forever.

**Fix**: always pass `--play_sounds=false` on record/teleop commands. **This is mandatory.**

To unstick a hung run without restarting: `pkill -u $USER spd-say`.

---

## 3. Orin Nano has NO hardware video encoder

NVIDIA's docs (verbatim): *"The NVIDIA Jetson Orin Nano does not have the NVENC engine."* — see https://docs.nvidia.com/jetson/archives/r36.4.3/DeveloperGuide/SD/Multimedia/SoftwareEncodeInOrinNano.html

Means:
- `--dataset.vcodec=h264_nvenc` fails (`libnvidia-encode.so.1` not present, and CAN'T be — no silicon).
- Jetson-native `nvv4l2h264enc` fails (`/dev/v4l2-nvenc` not present).
- `h264_v4l2m2m` also fails (same reason).

**There is nothing to install or patch.** All dataset video encoding must be CPU. Use `--dataset.vcodec=h264` (libx264). libsvtav1 (lerobot's default) is brutally slow on ARM cores — avoid.

This constraint is specific to Orin Nano. Orin NX and AGX Orin both have NVENC.

---

## 4. `nvpmodel -m 0` is the LOW-power mode on Orin Nano Super, NOT MAXN

```
ID=0 → 15 W
ID=1 → 25 W
ID=2 → MAXN_SUPER  ← what you actually want
```

On older Jetsons, mode 0 used to be MAXN. On Orin Nano Super (JetPack 6.2+) it's the 15 W profile. Set it via **jtop** (easiest — set power mode + jetson_clocks toggle in the UI) or on the CLI:
```bash
sudo nvpmodel -m 2 && sudo jetson_clocks
sudo nvpmodel -q                              # verify "NV Power Mode: MAXN_SUPER"
grep "POWER_MODEL ID" /etc/nvpmodel.conf      # confirm the mode→name mapping on YOUR board
```

`jetson_clocks` pins the CPU at max-for-current-mode but does NOT change the mode — so if you're in mode 0, your GPU and EMC clocks stay throttled even after `jetson_clocks`.

Subtle: `jetson_clocks` keeps the governor named `schedutil`. That's by design; it pins `scaling_min_freq = scaling_max_freq` instead of changing the governor. Verify with `cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_min_freq` — should equal `scaling_max_freq`.

Verify the GPU clock actually came up: `tegrastats --interval 500` should show `GR3D_FREQ ... ~1020 MHz` during inference bursts. Both settings revert on reboot (re-apply each boot — jtop makes this a two-click habit).

---

## 5. RealSense `TimeoutError: latest frame is too old: >500 ms` during recording

**Symptom**: recording crashes mid-run at an episode boundary with that error.

**Cause**: NOT USB bandwidth (verified — D435 alone on its own USB3 bus). The per-episode video-encoding burst (default: PNG-during-episode, encode-to-MP4-at-save_episode) saturates the Jetson's CPU for >500 ms, starving the RealSense background read thread, so its newest frame goes stale and the next `get_observation()` raises.

**Fix**: add to your record command:
```
--dataset.streaming_encoding=true --dataset.encoder_threads=1 --dataset.vcodec=h264
```
Streaming encoding spreads the work continuously instead of dumping it in a burst.

The 500 ms limit (`read_latest(max_age_ms=500)` in `camera_realsense.py`, called arg-less from `so_follower.get_observation`) is **not configurable via any CLI flag/env in lerobot 0.4.4**. Don't chase a flag for it.

---

## 6. ACT policy "freezes" on the arm — the fixed-point trap

**Symptom**: you run a trained ACT policy, the arm doesn't move at all (or barely twitches), looks completely stuck. Eventually you Ctrl+C thinking it hung.

**Cause**: behavior-cloning policies (ACT, BC) only work from **in-distribution starting states**. If the arm starts in a pose far from any training demo's start pose, the policy outputs near-zero "stay here"-class actions → arm doesn't move → observation doesn't change → next inference outputs the same → frozen in a fixed point forever.

**Fix**: before the run, drive the follower to the SAME starting pose your training demos started from. Use the leader for this (either run a quick teleop session first, or include `--teleop.type=so101_leader ...` in the policy-eval command so the leader controls during the reset window between episodes).

This is NOT a lerobot bug or a Jetson bug — it's a fundamental limitation of imitation learning. If it freezes from a known-good starting pose, your policy needs more / more diverse training data.

---

## 7. `lerobot-record` always records a dataset

There is no "just run the policy without saving" flag in 0.4.4. `lerobot-record --policy.path=` will always create a dataset of the rollouts. Use a throwaway repo_id (e.g. `bjahoor/eval_yellow_block`) and `rm -rf` it later.

---

## 8. `FileExistsError` on the dataset folder

```
FileExistsError: ~/.cache/huggingface/lerobot/<repo>/...
```

A previous failed attempt created the folder shell. Just `rm -rf ~/.cache/huggingface/lerobot/<repo>` and re-run.

**Exception**: if you want to resume an in-progress dataset, do NOT delete the folder; add `--resume=true` instead.

---

## 9. Default `pip install lerobot` gives you CPU-only PyTorch

The auto-resolved `torch` wheel on aarch64 is CPU-only. `torch.cuda.is_available()` returns `False`. You have to replace it with the Jetson AI Lab cu126 wheel — see [00-install.md](00-install.md).

---

## 10. `lerobot-train` on the PC at a newer lerobot version than the Jetson breaks model loading

**Symptom**: model trained on PC, copied to Jetson, lerobot-record errors out with `DecodingError: unknown field 'use_peft'` (or similar field name).

**Cause**: the PC's lerobot has fields the Jetson's lerobot doesn't know about.

**Fix**: pin both machines to the same patch version. I use `lerobot==0.4.4` on both.

---

## 11. `lerobot-train` crashes at startup with `'policy.repo_id' argument missing`

**Symptom**: you run `lerobot-train --policy.type=act --dataset.repo_id=... --job_name=... --output_dir=...` and it errors out immediately with `'policy.repo_id' argument missing. Please specify it to push the model to the hub.`

**Cause**: `push_to_hub` defaults to `True` (`lerobot/configs/policies.py:70`), and `lerobot/configs/train.py:138` has an explicit check that raises if push is on but `policy.repo_id` is unset.

**Fix**: either set the destination repo (optionally private):
```
--policy.repo_id=bjahoor/act_<TASK> --policy.private=true --policy.push_to_hub=true
```
or disable the push entirely (for speed tests / smoke tests):
```
--policy.push_to_hub=false
```
`--job_name` does NOT auto-populate `policy.repo_id` despite what you'd expect. The full working command in [11-pc-training.md](11-pc-training.md) includes it.

---

## 12. Cameras silently dropping fps

Some USB cameras negotiate down to YUYV (uncompressed, low fps over USB 2) if you don't explicitly request MJPG. For OpenCV cameras in lerobot, pass `fourcc: MJPG` in the camera config:
```
{type: opencv, index_or_path: 0, width: 640, height: 480, fps: 30, fourcc: MJPG}
```

RealSense doesn't have this problem in lerobot — it's configured via the librealsense pipeline directly.

---

## 13. PEP 668 / `error: externally-managed-environment` on the PC

**Symptom**: on the PC (Ubuntu 24.04), `pip install lerobot` fails with `error: externally-managed-environment`.

**Cause**: Ubuntu 24.04 marks the system Python as "externally managed" (PEP 668) so people don't break their distro by pip-installing on top of apt-managed packages. It's a feature, not a bug.

**Fix**: add `--break-system-packages`, don't bother with `--user`:
```bash
python3 -m pip install --break-system-packages "lerobot[smolvla]==0.4.4"
```

Without sudo, pip can't write to `/usr` anyway — it auto-defaults to `~/.local` (prints `Defaulting to user installation`). Same outcome, shorter command.

**Never `sudo pip`.** Writes into `/usr` on top of apt-managed packages and is exactly what PEP 668 was created to prevent.

---

## 14. `--wandb.enable=true` crashes at startup with `No API key configured`

**Symptom**: `lerobot-train` dies ~2 sec in (after printing the config) with:
```
wandb.errors.errors.UsageError: No API key configured. Use `wandb login` to log in.
```

**Cause**: with `--wandb.enable=true`, lerobot calls `wandb.init()` before training, which needs a stored wandb API key. If you've never `wandb login`'d on this box, no key is stored — crash.

**Fix**: either log in first:
```bash
wandb login    # paste a key from https://wandb.ai/authorize
```
or set `--wandb.enable=false` (the safe default for unattended overnight runs).

The crash happens before any output_dir is created, so no cleanup needed — just relaunch.

---

## 15. `Output directory ... already exists and resume is False`

**Symptom**: re-launching a (non-resume) training run, even just bumping `--steps`, errors at startup:
```
ValueError: Output directory <path> already exists and resume is False.
```

**Cause**: lerobot refuses to overwrite an existing output_dir for a fresh run. Even an empty dir from a previous Ctrl+C'd attempt counts (the dir is created early, before the first step).

**Fix**: either delete the dir or use a new `--output_dir` name:
```bash
rm -rf ~/outputs/train/<NAME>          # if no useful checkpoint inside
# or change --output_dir=outputs/train/<NAME>_v2
```

Checkpoints are written atomically (full file or none), so deleting an output_dir that has only the startup config dump (no real checkpoint) loses nothing.

---

## 16. `--steps` on a resume is the absolute total, NOT additional

**Symptom**: you resume a 30k-step run with `--steps=100000` thinking "100k more," and the run only does 70k.

**Cause**: lerobot's training loop is `for step in range(resumed_step, cfg.steps)` — `--steps` is the global target, not the count of new steps.

**Fix**: pass the new ABSOLUTE total. To extend a 30k run by 70k, use `--steps=100000`. To then add another 100k after that, use `--steps=200000`. The progress bar will read `…/200000` starting at `~100000` — that's the "did the resume actually take" check (if it starts at 0, the resume didn't load — abort, check the `--config_path`).

---

## 17. "Defaulting to user installation because normal site-packages is not writeable" is fine

**Symptom**: every `python3 -m pip install --break-system-packages …` prints
```
Defaulting to user installation because normal site-packages is not writeable
```
and you wonder if something's wrong.

**Cause**: nothing's wrong. Without sudo, pip can't write to `/usr/lib/python3.12/site-packages` (read-only for you), so it falls back to `~/.local/lib/python3.12/site-packages`. Same place `--user` would have put it, just automatic.

**Don't "fix" this** by adding sudo (dangerous — writes into `/usr`) or `--user` (just verbose for the same behavior).
