# Power mode + clocks

## TL;DR

```bash
sudo nvpmodel -m 2 && sudo jetson_clocks
```

Then verify:
```bash
sudo nvpmodel -q
# expect: NV Power Mode: MAXN_SUPER  (ID: 2)
```

## Why this matters

On Jetson Orin Nano "Super" with JetPack 6.2+, the power modes are:

| ID | Mode |
|---|---|
| 0 | 15 W |
| 1 | 25 W |
| 2 | **MAXN_SUPER** ← what you want for ACT inference and recording |

**Common mistake: `nvpmodel -m 0` is the LOW-power 15 W mode, not MAXN.** On older Jetsons mode 0 was MAXN — on this board it isn't. Always check `/etc/nvpmodel.conf` if unsure:
```bash
grep "POWER_MODEL ID" /etc/nvpmodel.conf
```

## What each command does

- **`nvpmodel -m 2`**: sets the SoC power profile to MAXN_SUPER — uncaps the GPU and EMC (memory) clocks. CPU max clock is the same in mode 1 and mode 2, but **GPU and memory are throttled in mode 0 and partly in mode 1**, which slows ACT inference and per-tick data marshalling significantly.
- **`jetson_clocks`**: pins `scaling_min_freq = scaling_max_freq` on every CPU core so the governor can never clock down. Subtle quirk: the governor name stays `schedutil`, which can look like jetson_clocks "didn't take." It did — verify with `cat /sys/devices/system/cpu/cpu*/cpufreq/scaling_min_freq` (all cores should match max).

Both settings revert on reboot. To make permanent, see [NVIDIA's docs](https://docs.nvidia.com/jetson/archives/r36.4.3/DeveloperGuide/SD/PlatformPowerAndPerformance/JetsonOrinNanoSeriesJetsonOrinNxSeriesAndJetsonAgxOrinSeries.html); usually not worth it — just put it in your shell history.

## Verify GPU clock is actually up

```bash
tegrastats --interval 500
```
In MAXN_SUPER you should see `GR3D_FREQ ... ~1020 MHz` during inference bursts.
