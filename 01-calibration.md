# Calibration — SO-101 leader + follower

## Find which port is which

Plug in ONE arm at a time and check:
```bash
ls /dev/ttyACM*
```
Or use lerobot's helper:
```bash
lerobot-find-port
```
which tells you to unplug → press Enter → replug, and reports the port.

In this cookbook I assume **follower = `/dev/ttyACM0`, leader = `/dev/ttyACM1`**. ACM numbering depends on plug order and can shuffle across reboots. For stability, use the `/dev/serial/by-id/` path instead.

## Step 1 — Assign motor IDs (one-time per arm)

Each new SO-101 arm ships with all 6 motors at ID=1. You have to assign IDs 1–6 one motor at a time.

**IMPORTANT: do this with the correct power supply** — follower = 12 V, leader = its own (usually 5 V). Using the wrong voltage during ID assignment can damage motors. If you previously had the leader's power adapter plugged in when ID'ing follower motors, redo it with follower's adapter.

```bash
# follower (on /dev/ttyACM0)
lerobot-setup-motors --robot.type=so101_follower --robot.port=/dev/ttyACM0

# leader (on /dev/ttyACM1)
lerobot-setup-motors --teleop.type=so101_leader --teleop.port=/dev/ttyACM1
```

Follow the prompts: plug ONE motor at a time starting from the base (ID=1), press Enter to assign each.

## Step 2 — Calibrate (one-time per arm, or after re-clocking)

The calibration finds each joint's min/max range and "home" (zero) position.

```bash
# follower
lerobot-calibrate --robot.type=so101_follower --robot.port=/dev/ttyACM0

# leader
lerobot-calibrate --teleop.type=so101_leader --teleop.port=/dev/ttyACM1
```

(Skipping `--robot.id` / `--teleop.id` saves to `None.json` which works fine.)

When prompted to set the "middle pose," move each joint to its physical center (not the encoder's 0/4095 seam). When prompted to "sweep through the full range of motion," move each joint slowly through its full physical range a couple of times.

## Calibration error: `Magnitude X exceeds 2047`

Means a joint is sitting past the encoder's wraparound seam (0 ↔ 4095).

**Fix**: physically reposition that joint into the middle of its range BEFORE running calibration. For the elbow specifically, the cleanest fix is:

1. Disconnect the link from the motor horn (the part screwed onto the motor's output shaft).
2. With the motor powered (so you can read positions), run a one-shot read of motor 3:
   ```python
   from lerobot.robots.so_follower.so_follower import SOFollower
   from lerobot.robots.so_follower.config_so_follower import SOFollowerConfig
   f = SOFollower(SOFollowerConfig(port='/dev/ttyACM0', cameras={}))
   f.connect()
   print(f.bus.sync_read("Present_Position"))
   f.disconnect()
   ```
3. Rotate the bare shaft until that motor reads near the center of its range (~1500–2500 for STS3215).
4. Reattach the link/horn so the joint sits at its mid-pose mechanically.
5. Re-run calibration.

This "re-clocks" the horn so the joint's physical mid-pose corresponds to a mid-range encoder value, avoiding the seam.
