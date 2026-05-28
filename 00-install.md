# Install — lerobot 0.4.4 on Jetson Orin Nano (Python 3.10, aarch64)

LeRobot's PyPI metadata asks for Python 3.12+, but pip backtracks and installs an older release on 3.10. **0.4.4** is the highest version that supports Python 3.10 — 0.5.0+ requires 3.12 (a much bigger migration on Jetson). Stay on 0.4.4.

## Pre-reqs (one-time)

User must be in the `dialout` group for `/dev/ttyACM*` serial access:
```bash
sudo usermod -aG dialout $USER
# log out + back in (or reboot) for the group change to take effect
```

## Install recipe

1. **Upgrade pip** (default Jetson pip is too old; trips on `hf-libero` 0.1.3 sdist metadata):
   ```bash
   python3 -m pip install --user --upgrade pip
   ```

2. **Pre-build `egl_probe` / `hf-egl-probe`** with a CMake policy override (their CMakeLists declare an ancient min that CMake 4.x rejects):
   ```bash
   python3 -m pip install --no-build-isolation hf-egl-probe
   CMAKE_POLICY_VERSION_MINIMUM=3.5 python3 -m pip install --no-build-isolation egl_probe
   ```

3. **(Only if `pymunk` build fails with cffi version mismatch — happens with the `[all]` extra on 0.4.0; usually skippable on 0.4.4):**
   ```bash
   python3 -m pip install --user --upgrade cffi
   python3 -m pip install --user --no-build-isolation 'pymunk<7.0.0,>=6.6.0'
   ```

4. **Install lerobot**:
   ```bash
   python3 -m pip install --user "lerobot[all]==0.4.4"
   ```
   For pure real-robot use (no simulators) you can skip `[all]` and install the lighter `lerobot[feetech]==0.4.4` instead — avoids the pymunk/robosuite chain entirely.

## CUDA PyTorch (default install gives CPU-only!)

The default install resolves `torch` to a CPU-only build on aarch64. Replace it with the Jetson AI Lab CUDA 12.6 wheel that matches lerobot's `torch>=2.2.1,<2.11` pin:

```bash
python3 -m pip uninstall -y torch torchvision
python3 -m pip install --index-url https://pypi.jetson-ai-lab.io/jp6/cu126 \
    "torch>=2.2.1,<2.11" "torchvision>=0.21,<0.26"
```

This pulls **torch 2.10.0 + torchvision 0.25.0** built for CUDA 12.6.

## Missing system libs (CUDA libraries the wheel expects)

After the above, `import torch` likely complains about `libcudss.so.0` and `libcusparseLt.so.0`. Install them:

```bash
# add CUDA keyring if you haven't:
sudo apt install -y cuda-keyring   # or download from NVIDIA if not present
sudo apt update
sudo apt install -y libcudss0-cuda-12 libcusparselt0
```

Then make `libcudss` discoverable (it lands in a subdirectory):
```bash
echo "/usr/lib/aarch64-linux-gnu/libcudss/12" | sudo tee /etc/ld.so.conf.d/cudss.conf
sudo ldconfig
```

## Verify GPU works

```bash
python3 -c "import torch; print(torch.__version__, torch.cuda.is_available(), torch.cuda.get_device_name(0) if torch.cuda.is_available() else 'CPU only')"
```
Expected: `2.10.0+cu126 True Orin`

## Verify lerobot CLI

```bash
lerobot-record --help | head -5
```

If it prints help, you're good.
