# Running a trained policy on the real arm

There is no `lerobot-eval` for real robots — you use `lerobot-record` with `--policy.path=`. It always creates a dataset of the eval rollouts; pass a throwaway repo_id.

## The working command (5-episode eval with leader teleop for reset)

```bash
rm -rf ~/.cache/huggingface/lerobot/bjahoor/eval_yellow_block && lerobot-record --robot.type=so101_follower --robot.port=/dev/ttyACM0 --robot.cameras="{ wrist: {type: opencv, index_or_path: 0, width: 640, height: 480, fps: 30, fourcc: MJPG}, overhead: {type: intelrealsense, serial_number_or_name: '135622077272', width: 640, height: 480, fps: 30}}" --teleop.type=so101_leader --teleop.port=/dev/ttyACM1 --dataset.repo_id=bjahoor/eval_yellow_block --dataset.single_task="Pick up the yellow block and place it on the white plate" --dataset.num_episodes=5 --dataset.fps=30 --dataset.episode_time_s=30 --dataset.reset_time_s=15 --dataset.vcodec=h264 --dataset.streaming_encoding=true --dataset.encoder_threads=1 --policy.path=bjahoor/act_so101_yellow_block --policy.device=cuda --display_data=false --play_sounds=false
```

## What makes this work

- **`--teleop.type=so101_leader --teleop.port=/dev/ttyACM1`**: even though the policy drives during episodes, lerobot uses **teleop during the reset window**. Without teleop, the follower locks in place during reset (torque stays on; you can't move it freely by hand). With the leader connected, you use it to drive the follower back into the start pose and reposition the object.
- **`--dataset.reset_time_s=15`**: gives you time to do that reposition.
- **`--dataset.episode_time_s=30`**: enough time for the policy to attempt the task. Tune to your task length.
- **`--policy.device=cuda`**: actually uses the Jetson GPU for inference.
- **`--play_sounds=false`**: TTS deadlock — see [99-gotchas.md](99-gotchas.md).

## Pre-flight checklist

1. `sudo nvpmodel -m 2 && sudo jetson_clocks` is on.
2. Cameras in the SAME physical positions they were in during recording.
3. Before running, **manually pose the leader at the training start position** — when the run starts, the follower mirrors and that becomes episode 0's starting state.

## Running without teleop (one clean attempt only)

If you want to run the policy without the leader, drop the teleop flags and set `--dataset.num_episodes=1`. The follower will lock at the end of episode 0 (no teleop to drive it during reset), so multiple episodes without teleop are useless:

```bash
... [same as above but without --teleop.* flags and with] --dataset.num_episodes=1 ...
```

## If the arm doesn't move ("frozen")

This is the most common failure of behavior-cloning policies (ACT, BC). The policy got stuck in a **fixed point** because the starting state was out-of-distribution — it had never seen a starting pose like the one it's in, so the best action it can predict is "stay roughly here," which produces no movement, which produces the same observation, which produces the same action, forever.

**Fix**: drive the follower to the SAME starting pose your training demos started from (using the leader during reset, or by hand before the run starts) and retry. If it still freezes from a known good starting pose, it's a policy quality issue — retrain with more demos.

## On the "loop running slower (17 Hz)" warnings

**Ignore them — the loop is actually at 30 Hz.** The warning at `lerobot_record.py:423` reports the instantaneous Hz of any single tick that exceeded the budget. With ACT, one tick per 100 fires the full transformer (~60 ms = "17 Hz" reported), but the other 99 ticks just pop a cached action and run at full 30 Hz. The warning firing every ~3 seconds is just the periodic chunk-refresh. The actual loop rate is fine. See [99-gotchas.md](99-gotchas.md).
