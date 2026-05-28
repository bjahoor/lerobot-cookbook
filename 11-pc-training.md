# Training ACT on the PC — the full working playbook

[05-training.md](05-training.md) is the brief version. This is the actual recipes that work end-to-end on my 3060 Ti, including the parts the brief version skips:
- the **mandatory `--policy.repo_id`** (without it, `lerobot-train` crashes at startup because `push_to_hub` defaults to true — see [99-gotchas.md](99-gotchas.md) #15)
- the **resume-from-checkpoint** dance (non-obvious)
- **live status checks** + GPU monitoring
- the `tr '\r' '\n'` trick for grepping the log

Install: [10-pc-install.md](10-pc-install.md). Tuning conclusions (batch/workers/etc.): [12-pc-tuning.md](12-pc-tuning.md).

## Speed test — measure rate, persist nothing

Always do this on a new dataset before a long run. ~2 min, writes nothing of value, no Hub upload.

```bash
lerobot-train \
  --policy.type=act --dataset.repo_id=<DATASET> \
  --batch_size=8 --steps=200 --num_workers=8 \
  --policy.device=cuda --policy.use_amp=true \
  --dataset.image_transforms.enable=true \
  --save_checkpoint=false --policy.push_to_hub=false \
  --output_dir=outputs/train/<NAME>_speedtest --job_name=<NAME>_speedtest \
  --wandb.enable=false
```

Read steady-state step/s off the progress bar after the first ~30 steps of warmup. Two kill switches: `--save_checkpoint=false` (nothing on disk) and `--policy.push_to_hub=false` (nothing on Hub).

## Fresh ACT training — saves locally + pushes to private Hub repo

```bash
lerobot-train \
  --policy.type=act --dataset.repo_id=<DATASET> \
  --policy.repo_id=bjahoor/act_<TASK> --policy.private=true --policy.push_to_hub=true \
  --batch_size=8 --steps=<N> --save_freq=<S> --num_workers=8 --log_freq=<L> \
  --policy.device=cuda --policy.use_amp=true \
  --dataset.image_transforms.enable=true --save_checkpoint=true \
  --output_dir=outputs/train/<NAME> --job_name=<NAME> \
  --wandb.enable=false
```

Common values:
- `--save_freq=5000` for short runs (≤20k); `=20000` for long runs (40k+) to keep disk in check (~591 MB per checkpoint, kept indefinitely).
- `--log_freq=1000` normal; `=10000` sparse for long unattended runs.

The push happens **only at end of training** (`lerobot_train.py:547`). A mid-run crash → nothing pushed, but local checkpoints survive.

## Resume / continue training to a higher step count

The non-obvious one. Point `--config_path` at the saved `train_config.json` inside the `last` checkpoint, set `--resume=true`, override `--steps` to the new total:

```bash
lerobot-train \
  --config_path=outputs/train/<NAME>/checkpoints/last/pretrained_model/train_config.json \
  --resume=true --steps=<NEW_HIGHER_TOTAL>
# Optional overrides (otherwise inherited from saved config):
# --batch_size=<B>  --save_freq=<S>  --log_freq=<L>
```

Key facts:
- **`--steps` is the absolute total, not additional.** Resuming a 30k run with `--steps=100000` does 70k more steps — see [99-gotchas.md](99-gotchas.md) #16.
- **Progress bar must start at the resumed step, not 0.** If it starts at 0, the resume didn't load and you're training from scratch — abort and check the path.
- Inherits repo_id, private, push_to_hub, batch_size, num_workers, wandb settings, output_dir from the saved config.
- ACT uses constant LR (no scheduler) → extending steps is clean, no scheduler to break.
- Final model re-pushes to the same Hub repo at end, overwriting the prior version.

## Long unattended training — `tmux` is mandatory

```bash
tmux new -s train
# paste the training command
# detach with Ctrl+B then D — run keeps going
# reattach later: tmux attach -t train
```

Without tmux, a closed terminal or dropped SSH kills the run. Worth the habit even for ~1-hour runs.

## Status checks (live)

```bash
ps aux | grep -iE "lerobot[-_]train" | grep -v grep                  # is it running?
nvidia-smi --query-gpu=utilization.gpu,memory.used,power.draw,temperature.gpu --format=csv,noheader
ls -1 ~/outputs/train/<NAME>/checkpoints/                              # what's saved
cat ~/outputs/train/<NAME>/checkpoints/last/training_state/training_step.json
```

Each checkpoint dir contains `pretrained_model/` (model + config + processors) and `training_state/` (optimizer + step counter + rng).

## Live GPU monitor — sample over 30 s while training

```bash
nvidia-smi --query-gpu=utilization.gpu,memory.used,memory.total,power.draw,temperature.gpu,clocks.sm --format=csv,noheader
nvidia-smi dmon -s um -d 2 -c 15

# If you suspect throttling:
nvidia-smi -q -d PERFORMANCE | sed -n '/Clocks Event Reasons/,/Counters/p'
```

Interpret: `sm%` = compute util; `fb` (MB) = VRAM used. ACT pegs `sm%` at ~95–100% on this box. The only "active" throttle reason is normally `SW Power Cap` — the card hitting the 220 W vendor cap, which is fine. If `HW Thermal Slowdown: Active`, fix your case airflow.

## Verify a Hub push went through

```bash
python3 -c "
from huggingface_hub import HfApi
i = HfApi().repo_info('bjahoor/<MODEL>', repo_type='model')
print('private:', i.private, '| last_modified:', i.lastModified)
print('files:', [s.rfilename for s in i.siblings])
"
```

Expect at minimum: `model.safetensors`, `config.json`, `train_config.json`, the pre/post-processor `*.json` + `*.safetensors`. `last_modified` should match end-of-training.

## Extract loss progression from a training log (handles tqdm `\r`)

```bash
tr '\r' '\n' < <LOGFILE> | grep "loss:" | tail -10
```

lerobot's training output is one giant pseudo-line because tqdm overwrites with `\r` (no `\n`). `tr '\r' '\n'` splits them. Each metric line looks like:

```
step:NK smpl:NK ep:N epch:N.NN loss:X.XXX grdn:X.XXX lr:1.0e-05 updt_s:X.XXX data_s:X.XXX
```

`updt_s` is GPU compute time per step; `data_s` is dataloader wait. If `data_s >> 50 ms`, raise `--num_workers`; otherwise you're compute-bound.

## Cleanup — stale interrupted-run output dir

```bash
ls -R ~/outputs/train/<NAME>             # look first — confirm no useful checkpoint
rm -rf ~/outputs/train/<NAME>            # only if nothing valuable inside
```

Lerobot writes checkpoints atomically (whole or absent, never half-written), so Ctrl+C never corrupts existing checkpoints — you only lose work between the last save and the cancel. **But** the `output_dir` must NOT exist when starting a non-resume run — validation refuses with `Output directory already exists and resume is False`. See [99-gotchas.md](99-gotchas.md) #14.

## Disk usage planning

Each ACT checkpoint dir is **~591 MB** (model 207 MB + optimizer state ~380 MB + tiny training-state files). lerobot keeps all of them — there's no rotation. For a 100k run at `save_freq=5000` that's 20 checkpoints ≈ 12 GB. Use `save_freq=20000` for long runs to keep disk in check.

## Sanity numbers (from my runs at batch 8)

| dataset | cams | res / codec | step/s | 100k ETA |
|---|---|---|---|---|
| `bjahoor/so101_green_cap` | 1 wrist | 1280×720 AV1 | **2.9** | ~10 h |
| `bjahoor/so101_yellow_block` | 2 (wrist + overhead) | 480×640 H.264 | **4.35** | ~6.4 h |

More detail and the "why" behind these numbers → [12-pc-tuning.md](12-pc-tuning.md).
