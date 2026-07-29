# Install — lerobot 0.4.4 on Ubuntu 24.04 PC (Python 3.12, x86_64)

Pin to **0.4.4** to match the Jetson. Otherwise model loading errors out with `DecodingError: unknown field 'use_peft'` or similar — see [99-gotchas.md](99-gotchas.md) #10.

## My setup

- Ubuntu 24.04, system Python 3.12.3, pip 25.3
- RTX 3060 Ti (8 GB), NVIDIA driver 580 (CUDA 13 capable)
- Native install in `~/.local`, **no venv/conda**

## OS deps (one-time, needs sudo password)

```bash
sudo apt install python3-dev build-essential ffmpeg
```

Skip these and you get:
- **No `python3-dev`** → `evdev` (via `pynput`) fails to compile: `fatal error: Python.h: No such file`.
- **No `ffmpeg`** → torchcodec fails to import: `Could not load libtorchcodec_core*.so`.

Ubuntu 24.04 ships `ffmpeg 6.1.1`. torchcodec supports ffmpeg 4–8, so that works fine — no need for the conda `7.1.1` workaround the lerobot docs mention.

## Install lerobot

```bash
python3 -m pip install --break-system-packages "lerobot[smolvla]==0.4.4"
```

Two things about this command:

- **`--break-system-packages` is mandatory on Ubuntu 24.04** — system Python is PEP 668 "externally managed" and plain `pip install` errors out with `error: externally-managed-environment`. See [99-gotchas.md](99-gotchas.md) #13.
- **No `--user` needed.** Without sudo, pip can't write to `/usr` anyway, so it auto-defaults to `~/.local` (prints `Defaulting to user installation`). Same outcome, shorter command.

**Never `sudo pip`.** That writes into `/usr` on top of apt-managed Python packages — exactly what PEP 668 was created to prevent.

For pure ACT training you can use the lighter `lerobot==0.4.4` instead of `lerobot[smolvla]==0.4.4` — saves pulling `transformers` and friends.

## CUDA PyTorch — auto-resolves correctly on x86

Unlike the Jetson, **the default PyPI `torch` wheel on x86_64 is CUDA-enabled**. Just installing lerobot pulls `torch 2.7.1+cu126` (matching lerobot's `torch>=2.2.1,<2.11` pin), which works with the desktop NVIDIA driver.

**Do NOT use the Jetson AI Lab index (`https://pypi.jetson-ai-lab.io/jp6/cu126`) on this machine** — those are ARM wheels and would either fail to install or give you a broken environment.

## Verify

```bash
python3 -c "import lerobot,torch,torchcodec; print('lerobot',lerobot.__version__,'| torch',torch.__version__,'| cuda',torch.cuda.is_available(),'| torchcodec',torchcodec.__version__)"
python3 -c "from torchcodec.decoders import VideoDecoder; print('torchcodec decode: OK')"
```

Expected:
```
lerobot 0.4.4 | torch 2.7.1+cu126 | cuda True | torchcodec 0.5
torchcodec decode: OK
```

If `cuda False`, you might have a CPU-only torch from a botched install — uninstall and reinstall lerobot. (On x86, the default wheel is the right one.)

## HF login — for pushing trained models

```bash
hf auth login         # or: huggingface-cli login (both work in hf-hub 0.35)
```

Paste a token with **write** scope from https://huggingface.co/settings/tokens. A read-only token lets `whoami` succeed but causes a 403 at end-of-training when the push tries.

## wandb login — only if using `--wandb.enable=true`

```bash
wandb login           # paste a key from https://wandb.ai/authorize
```

Setting `--wandb.enable=true` without doing this crashes lerobot at startup. See [99-gotchas.md](99-gotchas.md) #14. For unattended overnight runs, leave `--wandb.enable=false` — simpler.

## What's next

- Comprehensive training playbook → [11-pc-training.md](11-pc-training.md)
- Measured tuning findings (batch, workers, power cap) → [12-pc-tuning.md](12-pc-tuning.md)
