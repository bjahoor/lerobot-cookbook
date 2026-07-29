# tmux — keep long runs alive after disconnect

Start a long command (training, recording), **detach** so you can close the terminal or drop SSH while it keeps running on the machine, then reattach later. Without tmux, closing the terminal kills the run.

## The only commands you need

```bash
tmux new -s train            # start a session named "train", run your command inside it
# press Ctrl+B then D        → detach (run keeps going)
tmux ls                      # list running sessions
tmux attach -t train         # rejoin
tmux kill-session -t train   # stop the session for good
```

`Ctrl+B then D` = hold Ctrl, press B, release both, then press D. `Ctrl+B` is tmux's prefix for all its shortcuts — detach is the only one you actually need.

Install if missing: `sudo apt install tmux`.

## Worth knowing

- **`Ctrl+C` inside tmux kills your command**, not tmux. To leave the run alive, detach with `Ctrl+B then D` — never `Ctrl+C`.
- **Closing the terminal does NOT kill the session** — it just disconnects you. Reattach anytime with `tmux attach -t <name>`.
- **Sessions don't survive a reboot.** A power cycle kills everything.
- **Run several at once** (`train`, `record`, `eval`) with distinct `-s` names; `tmux ls` shows them all.
