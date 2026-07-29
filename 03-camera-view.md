# Viewing a camera headless — position/focus it over Tailscale

Stream a camera to your laptop browser to aim and focus it while the Jetson is headless. Uses **ustreamer** (already installed at `/usr/bin/ustreamer`). Needs to be in the `video` group (you are).

## 1. Find the right `/dev/video` node

USB enumeration order is **NOT stable across reboots** — the RealSense and the wrist cam swap node numbers. Always re-check:

```bash
v4l2-ctl --list-devices
```
Groups nodes by physical device, e.g. `Intel(R) RealSense... /dev/video0–5` and `USB2.0_CAM1 /dev/video6–7`.

The RealSense exposes ~6 nodes but only ONE is **color**. Identify by pixel format — color = `YUYV`, depth = `Z16`, IR = `GREY`/`UYVY`:
```bash
for n in 0 1 2 3 4 5; do echo "video$n:"; v4l2-ctl -d /dev/video$n --list-formats | grep "'"; done
```
The node listing `'YUYV'` is the color stream (was `/dev/video4` last check). The wrist USB cam uses `MJPG` — its lower-numbered node is the one to record from.

## 2. Stream it

RealSense color node (YUYV):
```bash
ustreamer --device=/dev/video4 --format=yuyv --resolution=640x480 --desired-fps=15 --host=0.0.0.0 --port=8080
```

Wrist USB cam (MJPG, no `--format` needed):
```bash
ustreamer --device=/dev/video6 --resolution=640x480 --desired-fps=15 --host=0.0.0.0 --port=8081
```
640×480 matches the recording FOV, so what you frame now = what you capture.

## 3. View on your laptop

Get the Jetson's Tailscale IP, then open the URL in your laptop browser:
```bash
tailscale ip -4      # mine: 100.82.20.54
```
```
http://100.82.20.54:8080
```
No SSH tunnel needed over Tailscale. (Plain SSH instead? Forward the port: `ssh -L 8080:localhost:8080 robouser@<host>`, then browse `http://localhost:8080`.)

## 4. Stop it before recording

**ustreamer holds the camera exclusively** — lerobot can't open the device while ustreamer runs. Kill it first:
```bash
pkill -f ustreamer
```

## Streaming DURING teleop is fine (recording is not)

`lerobot-teleoperate` opens **no cameras** (the teleop command has no `--robot.cameras`), so you can run ustreamer on the wrist or overhead cam at the same time — e.g. watch the wrist view in your browser while you drive the arm. No device conflict.

`lerobot-record` **does** open the cameras, so ustreamer must be stopped before recording (step 4 above). Rule of thumb: **teleop = stream freely; record = stop the stream first.**

## Note — the enumeration flip also affects your record command

If the RealSense grabbed `/dev/video0–5` this boot, your **wrist cam is `index_or_path: 6`**, not 0. The RealSense is addressed by serial (unaffected), but the OpenCV wrist cam is addressed by index — double-check it against `v4l2-ctl --list-devices` before recording, or you'll capture the wrong camera.
