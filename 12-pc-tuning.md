# PC tuning — measured findings on the RTX 3060 Ti

Empirical conclusions from running 200-step speed tests + live `nvidia-smi dmon` during ACT training. Numbers are specific to my 3060 Ti + i7-6700K + 8 logical CPUs, but the *shape* of the conclusions transfers.

For commands → [11-pc-training.md](11-pc-training.md).

## TL;DR

| Knob | Setting | Why |
|---|---|---|
| `--batch_size` | **8** | GPU already saturates at batch 8; bigger doesn't speed wall-clock. |
| `--num_workers` | **8** | Already keeps the GPU fed (`data_s` ~8 ms). More just causes CPU contention. |
| `--policy.use_amp` | **true** | Default-on. Halves activation memory, costs nothing. |
| `--policy.device` | **cuda** | Obvious. |
| Power limit | **leave at 220 W default** | +2–4% from raising to 230 W isn't worth the longevity trade. |

## Measured step rates (batch 8, augmentation on, AMP on)

| dataset | cameras | resolution | codec | step/s | `data_s` | `updt_s` |
|---|---|---|---|---|---|---|
| green_cap | 1 wrist | 1280×720 | AV1 | **2.9** | ~30 ms | ~340 ms |
| yellow_block | 2 (wrist + overhead) | 480×640 | H.264 | **4.35** | ~8 ms | ~220 ms |

Counter-intuitive but true: **yellow_block is faster despite the extra camera**. Lower total pixel count (614k vs 921k) and H.264 decoding way faster than AV1 outweigh the second backbone pass.

## GPU is compute-bound, not data-bound

Per-step time = `data_s` (dataloader wait) + `updt_s` (GPU compute), overlapped. With 8 workers, `data_s` is single-digit milliseconds while `updt_s` is 200–340 ms — a **~30× ratio**. The GPU is the bottleneck, full stop.

Implications:
- **More workers can't help much.** Theoretical max wall-clock gain if `data_s` → 0 is ~3% (8 ms / 240 ms). And going past 8 workers on an 8-logical-CPU box just causes thread contention.
- **`--num_workers=8` is optimal here.** `=4` would probably also work; `=16` is slower.

## Bigger batch ≠ faster wall-clock

GPU is already at ~95–100% `sm%` at batch 8. A bigger batch (e.g. 16) does ~2× the work per step → **step rate halves, samples/sec stays roughly constant**. Same wall-clock for the same step target — you don't save time, you just do more samples per step.

When IS bigger batch worth it? Only if you specifically want the *training-dynamics* benefit (steadier gradients, larger effective gradient for the optimizer). For wall-clock speedup on a saturated GPU, no.

### Measured batch-16 numbers (yellow_block)

| batch | VRAM used | sm% | step/s | samples/sec |
|---|---|---|---|---|
| 8 | 4.7 / 8 GB | ~95% | 4.35 | ~35 |
| 16 | 7.7 / 8 GB | ~98% | ~2.0 | ~32 |

Batch 16 fits on yellow_block (480×640 × 2 cams) right at the 8 GB cap. Predicted OOM on green_cap (720p × 1 cam, ~10 GB activations) — never tested, didn't need to.

## Power cap: 220 W vendor default = leave it

Card stays power-capped during training (`SW Power Cap: Active` in `nvidia-smi -q -d PERFORMANCE`), pegged at 220 W, ~62–66 °C, ~1950 MHz. **NOT thermal-limited.**

`sudo nvidia-smi -pl 230` (resets on reboot) would unlock the extra 10 W of headroom → small clock bump → ~2–4% faster. I left it at 220 because:

- 220 W is the vendor-validated continuous operating point. 230 W is the *permitted* max, not the intended one.
- ~2–4% gain isn't worth the thermal / longevity / noise trade for unattended overnight runs.
- The card already maxes itself at 220 — it's not being held back.

## ACT defaults that matter — leave alone for a first run

- `--policy.chunk_size=100` — actions predicted per forward pass.
- `--policy.dim_model=512`, `--policy.n_encoder_layers=4`, `--policy.n_decoder_layers=1` — transformer size. Shrinking saves compute but cuts capacity.
- `--policy.vision_backbone=resnet18` — ImageNet1K_V1 weights.
- `--policy.use_vae=true`, `--policy.kl_weight=10.0` — CVAE component.
- Optimizer preset (AdamW, **constant lr=1e-5**) — auto-set via `--use_policy_training_preset=true` (default). The constant LR is why extending steps on a resume is clean — no scheduler to break.

**ACT has no `--policy.resize_shape` / `--policy.crop_ratio`.** Those flags appear in `lerobot-train --help` but belong to other policies (diffusion, smolvla, etc.) — `lerobot-train --help` shows the union of all policy fields. ACT processes images at dataset resolution. For VRAM on ACT, your levers are `--batch_size ↓` and `--policy.use_amp=true` (default on).

## Convergence + overfitting math

50 episodes is small. Most of the loss drop happens in the first ~8–10 epochs:

| dataset | frames | steps / epoch (batch 8) |
|---|---|---|
| green_cap | 29,950 | 3,744 |
| yellow_block | 22,446 | 2,806 |

Empirical: green_cap loss went **3.35 → 0.143** over 40k steps (~10.7 epochs), basically flat by ~30k. So 100k+ on a 50-episode dataset is well into diminishing-returns / overfitting territory.

**Rule of thumb on 50-episode datasets:** aim for ~30–50k steps. **More episodes >> more steps** if you want a better policy.

**Training loss is not the real metric.** On-robot success rate is. See [06-policy-eval.md](06-policy-eval.md).

## One untested speedup option

`--policy.compile_model=true` enables `torch.compile`. Typically gives ~10–30% steady-state speedup on transformer workloads, at a few-min upfront compile cost. Can be finicky (falls back gracefully if it errors). Haven't tried it on my runs yet — if I do, I'll update this section.
