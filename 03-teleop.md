# Teleop — leader drives follower

## One-liner

```bash
lerobot-teleoperate \
  --robot.type=so101_follower --robot.port=/dev/ttyACM0 \
  --teleop.type=so101_leader --teleop.port=/dev/ttyACM1
```

Move the leader, the follower mirrors. Ctrl+C to stop.

## Notes

- **`--play_sounds=false` is NOT a flag for teleop** — it's record-only. Teleop doesn't make sound calls.
- If the follower jerks badly or one joint refuses to move, the most likely cause is calibration drift on that joint — re-run [calibration](02-calibration.md) for the affected arm.
- If a motor returns `Failed to write 'Torque_Enable' on id_=N ... There is no status packet!`, that motor isn't responding on the serial bus. Check power and reseat the cables at motor N (and the cable upstream of it, since it's a daisy chain). This has bit me twice — once on motor 3 (elbow), once on motor 5 (wrist roll).
- If `/dev/ttyACM0` and `/dev/ttyACM1` swap on you between reboots, that's expected — ACM numbers depend on USB enumeration order. Pin them with `/dev/serial/by-id/<long-id>` if it becomes annoying.
