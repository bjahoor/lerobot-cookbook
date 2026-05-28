# tmux — terminal sessions that survive disconnects

The point of tmux: start a long-running command (training, recording, anything), **detach** so your terminal closes but the command keeps running on the machine, reattach later from any new terminal. Without tmux, closing your terminal or dropping SSH kills whatever was running.

## Install (one-time)

```bash
sudo apt install tmux
```

Already on most Jetson + Ubuntu setups by default.

## TL;DR — the 5 commands you actually need

```bash
tmux new -s <NAME>           # start a new session named <NAME>
# (now inside tmux — run your long-running command)
# press Ctrl+B then D        → detach (command keeps running)

tmux ls                      # list all running sessions
tmux attach -t <NAME>        # rejoin a session
tmux kill-session -t <NAME>  # kill a session for good
```

That's enough for "run training overnight, reattach in the morning." Everything below is just nice-to-have.

## The "Ctrl+B" prefix

tmux's own shortcuts start with **`Ctrl+B`** (the "prefix key") so they don't collide with anything you type into the program inside it.

"**Ctrl+B then D**" means: hold Ctrl, press B, release both keys, then press D. It's a two-step combo, not a three-key chord.

(If you've used `screen` before, its prefix is `Ctrl+A` — different. Tmux is `Ctrl+B`.)

## Shortcuts inside tmux (all start with Ctrl+B)

### Sessions

| Keys | What |
|---|---|
| `Ctrl+B D` | **Detach** (leave running, go back to your normal shell) |
| `Ctrl+B S` | List sessions, pick one to switch into |
| `Ctrl+B $` | Rename the current session |

### Windows (like tabs — multiple "screens" in one session)

| Keys | What |
|---|---|
| `Ctrl+B C` | Create a new window |
| `Ctrl+B N` | Next window |
| `Ctrl+B P` | Previous window |
| `Ctrl+B 0`–`9` | Jump to window number 0–9 |
| `Ctrl+B ,` | Rename the current window |
| `Ctrl+B &` | Close the current window (asks to confirm) |

### Panes (splits inside one window)

| Keys | What |
|---|---|
| `Ctrl+B %` | Split current pane **vertically** (side by side) |
| `Ctrl+B "` | Split current pane **horizontally** (top / bottom) |
| `Ctrl+B <arrow>` | Move to the pane in that direction |
| `Ctrl+B Z` | Zoom — make current pane fullscreen / toggle back |
| `Ctrl+B X` | Close the current pane (asks to confirm) |

### Scrollback / copy mode

| Keys | What |
|---|---|
| `Ctrl+B [` | Enter scroll mode (use arrow keys / Page Up to scroll back) |
| `Q` | Exit scroll mode |

Mouse-wheel scrolling inside tmux may or may not work depending on your terminal. Scroll mode is the reliable way to look at output that scrolled off.

## From the shell (no prefix needed)

| Command | What |
|---|---|
| `tmux new -s NAME` | Start a fresh session named NAME |
| `tmux new -d -s NAME 'CMD'` | Start a session running CMD, already detached |
| `tmux ls` | List sessions |
| `tmux attach -t NAME` | Attach to a session by name |
| `tmux attach` | Attach to the last/only session |
| `tmux kill-session -t NAME` | Kill one session |
| `tmux kill-server` | Kill ALL sessions (nuclear option) |
| `tmux rename-session -t OLD NEW` | Rename a session from outside |

## Worked example — running training overnight

```bash
# 1. Start the session
tmux new -s train

# 2. (Now inside tmux — your prompt looks normal, with a green status bar at the bottom)
#    Paste your training command and hit Enter:
lerobot-train ...

# 3. Detach so the run keeps going without you:
#    Press Ctrl+B then D
#    You'll see "[detached (from session train)]" and you're back to your normal shell.

# Now you can: close the terminal, disconnect SSH, shut your laptop, reboot —
# the training keeps running on the server.

# 4. Hours later, from any new terminal:
tmux attach -t train         # back inside, see the live progress bar / output

# 5. When training's finished and pushed:
exit                          # exits the shell inside tmux → closes the session
# (or from outside the session: tmux kill-session -t train)
```

## Things you'll hit

- **The prefix is `Ctrl+B`, not `Ctrl+A`.** If you've used `screen` before, that's the most common reflex error.
- **`Ctrl+C` inside tmux still kills your command** — it goes to the program inside, not to tmux. Detach is `Ctrl+B then D`, never `Ctrl+C`.
- **Closing the terminal does NOT kill the session.** It just disconnects you. The session and its programs keep running. To actually stop everything, attach and `Ctrl+C` your command, then `exit` (or run `tmux kill-session` from outside).
- **You can't `tmux attach` from inside another tmux session.** Use `Ctrl+B S` to switch sessions instead.
- **Multiple sessions are fine.** Have `train`, `record`, `eval` running at once; `tmux ls` shows them all. Use distinct names.
- **Sessions don't survive a reboot.** A power cycle kills everything. If you want truly persistent backgrounding across reboots, you want `systemd` services — different tool.
