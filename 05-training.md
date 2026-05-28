# Training ACT — on the PC, not the Jetson

Train on a machine with a real dGPU (mine: RTX 3060 Ti). The Jetson Orin Nano can technically train but it would take ~10× longer.

## Pre-reqs on the PC

Install lerobot at **the same version as the Jetson (0.4.4)** so the trained model loads back without version-mismatch errors:

```bash
python3 -m pip install --user "lerobot==0.4.4"
```

Verify:
```bash
pip show lerobot | grep Version
# Version: 0.4.4
```

Log into HuggingFace (so training auto-pushes the model to the hub):
```bash
huggingface-cli login
```

## Training command

```bash
lerobot-train \
  --dataset.repo_id=bjahoor/so101_yellow_block \
  --policy.type=act \
  --policy.repo_id=bjahoor/act_so101_yellow_block \
  --output_dir=outputs/train/act_yellow_block \
  --job_name=act_so101_yellow_block \
  --policy.device=cuda \
  --wandb.enable=false
```

**`--policy.repo_id` is required** if you want the model auto-pushed to the Hub. Lerobot 0.4.4 has `push_to_hub=True` by default (`lerobot/configs/policies.py:70`) and an explicit check in `lerobot/configs/train.py:138` that errors out at startup with `"'policy.repo_id' argument missing"` if you don't set it. Either set it (recommended — model lands on the Hub ready to pull from the Jetson) OR pass `--policy.push_to_hub=false` to keep it local-only.

Defaults: 100k steps, batch size 8. On a 3060 Ti at 640×480 / 2 cameras, ~5–10 hours for 100k steps; ~1–2 hours for 20k steps.

After training, the checkpoint pushes to whatever you set for `--policy.repo_id`. From the Jetson you reference it with `--policy.path=<that-same-id>`.

## If you want to compare multiple training runs

Give each one a different `--job_name`:
```bash
--job_name=act_so101_yellow_block_v2
```
This pushes to a separate hub repo and writes to a separate `output_dir`, so nothing overwrites.

## Training at a different fps than the dataset

If you ever record at 30 fps but want a policy that runs at 20 fps (for slower hardware), pass:
```bash
--policy.fps=20
```
LeRobot downsamples the dataset on the fly. The runtime then needs `--dataset.fps=20` to match.

## Common errors

- **`DecodingError: unknown field 'use_peft'`** when loading the trained model on the Jetson: PC and Jetson lerobot versions don't match. Get them both on the same patch version (0.4.4).
- **OOM on the PC**: drop `--batch_size` (e.g. `--batch_size=4`).
