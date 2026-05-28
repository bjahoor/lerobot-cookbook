# Recording a dataset

Recipe for the 50-episode `yellow_block` dataset (yellow block → white plate), 2 cameras at 640×480, h264 video, streaming encoding to avoid RealSense timeouts.

## Pre-flight

```bash
sudo nvpmodel -m 2 && sudo jetson_clocks    # see 01-power-and-clocks.md
```

## Recording command (1 line, dataset gets created)

```bash
lerobot-record --robot.type=so101_follower --robot.port=/dev/ttyACM0 --robot.cameras="{ wrist: {type: opencv, index_or_path: 0, width: 640, height: 480, fps: 30, fourcc: MJPG}, overhead: {type: intelrealsense, serial_number_or_name: '135622077272', width: 640, height: 480, fps: 30}}" --teleop.type=so101_leader --teleop.port=/dev/ttyACM1 --dataset.repo_id=bjahoor/so101_yellow_block --dataset.single_task="Pick up the yellow block and place it on the white plate" --dataset.num_episodes=50 --dataset.fps=30 --dataset.episode_time_s=15 --dataset.reset_time_s=5 --dataset.vcodec=h264 --dataset.streaming_encoding=true --dataset.encoder_threads=1 --play_sounds=false
```

## Key flags (and why each one matters)

| Flag | Why |
|---|---|
| `--dataset.fps=30` | Standard SO-101 control rate. |
| `--dataset.episode_time_s=15` | 15 s per demo. Tune to your task. |
| `--dataset.reset_time_s=5` | Time to reposition arm + object between episodes. Bump to 15+ if you need more. |
| `width: 640, height: 480` | NOT 720p. 640×480 is the lerobot community standard; 720p will pin the Jetson's CPU and cause RealSense timeouts. |
| `fourcc: MJPG` | Wrist USB cam: MJPG over USB2 ≫ YUYV (lower bandwidth, higher fps). |
| `type: intelrealsense, serial_number_or_name: '...'` | Address the D435 by **serial number**, not `/dev/videoN`. Find with `realsense-viewer` or `rs-enumerate-devices`. |
| `--dataset.vcodec=h264` | libsvtav1 (the default) is brutally slow on Jetson CPU. **h264_nvenc does NOT work on Orin Nano** — there is no NVENC silicon on this chip. |
| `--dataset.streaming_encoding=true` | Spreads h264 encoding across capture instead of dumping it in a burst at each episode boundary. Without this, the burst starves the RealSense read thread → `TimeoutError: latest frame is too old: >500ms`. |
| `--dataset.encoder_threads=1` | 1 encoder thread leaves cores free for the capture loop. |
| `--play_sounds=false` | **Mandatory.** TTS deadlocks on headless Jetson. See [99-gotchas.md](99-gotchas.md). |

## If the run crashes mid-recording

Add `--resume=true` and set `--dataset.num_episodes=N` where **N = episodes to ADD this session, not the new total**. Example after 35/50 got saved before a crash:

```bash
lerobot-record ... --dataset.num_episodes=15 --resume=true   # 35 + 15 = 50
```

**Don't `rm -rf` the dataset folder when resuming.** Do `rm -rf` only on a brand-new attempt where you want to start over.

## Common errors

- **`FileExistsError`**: dataset folder from a prior failed attempt. Run `rm -rf ~/.cache/huggingface/lerobot/<repo_id>` then re-run from scratch (NOT with `--resume`).
- **`TimeoutError: latest frame is too old: 5XX ms (max allowed: 500 ms)`**: the streaming-encoding flag wasn't on, or the Jetson is otherwise CPU-saturated. Verify flags. Confirm jetson_clocks is on.
- **`ConnectionError: ... There is no status packet!` / `Incorrect status packet!`**: a motor isn't responding. Reseat cables at that motor and check power. See [02-calibration.md](02-calibration.md).

## Deleting bad episodes (without losing the good ones)

```bash
lerobot-edit-dataset --repo_id=bjahoor/so101_yellow_block \
  --operation.type=delete_episodes \
  --operation.episode_indices="[35,36,37,38,39,40,41,42]" \
  --push_to_hub=true
```

This is non-destructive: lerobot first renames your folder to `..._old` as a backup, then writes the pruned dataset to the original path. Delete the `..._old` backup once you're happy:
```bash
rm -rf ~/.cache/huggingface/lerobot/bjahoor/<repo>_old
```
