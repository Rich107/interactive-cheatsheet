# tmux quickref

A fast reference for tmux: shell-level commands for managing sessions plus the most useful in-tmux keybindings for windows, panes, and copy mode. The default prefix is `Ctrl-b` — substitute your own (commonly `Ctrl-a`) where shown.

## 1. Start & attach sessions

Create a named, detached session and run the editor in its first window:

```bash
tmux new -s dev -n editor "vim ."
```

Attach to an existing session (or auto-create one if missing):

```bash
tmux new -A -s dev
```

Attach to a known session:

```bash
tmux attach -t dev
```

List active sessions on this server:

```bash
tmux ls
```

## 2. Detach and kill

Detach from the current session (keeps it running in the background):

```text
Ctrl-b d
```

Kill a single session by name:

```bash
tmux kill-session -t dev
```

Kill every session and stop the tmux server entirely:

```bash
tmux kill-server
```

## 3. Windows (tabs)

| Action            | Keys             |
| ----------------- | ---------------- |
| New window        | `Ctrl-b c`       |
| Next window       | `Ctrl-b n`       |
| Previous window   | `Ctrl-b p`       |
| Jump to window N  | `Ctrl-b 0..9`    |
| Rename window     | `Ctrl-b ,`       |
| Kill window       | `Ctrl-b &`       |
| Window picker     | `Ctrl-b w`       |

Rename the active window from the shell:

```bash
tmux rename-window editor
```

## 4. Panes (splits)

| Action                  | Keys                        |
| ----------------------- | --------------------------- |
| Split horizontally      | `Ctrl-b "`                  |
| Split vertically        | `Ctrl-b %`                  |
| Move between panes      | `Ctrl-b ←` `↑` `↓` `→`     |
| Cycle to next pane      | `Ctrl-b o`                  |
| Kill the current pane   | `Ctrl-b x`                  |
| Zoom / unzoom pane      | `Ctrl-b z`                  |
| Show pane numbers       | `Ctrl-b q`                  |
| Swap panes              | `Ctrl-b {` / `Ctrl-b }`     |
| Break pane into window  | `Ctrl-b !`                  |

## 5. Resize panes

Hold the prefix, then use `Ctrl` + arrow to nudge by one cell:

```text
Ctrl-b Ctrl-←   Ctrl-b Ctrl-→   Ctrl-b Ctrl-↑   Ctrl-b Ctrl-↓
```

Resize precisely from the command prompt (`Ctrl-b :`):

```text
:resize-pane -L 10
:resize-pane -R 10
:resize-pane -U 5
:resize-pane -D 5
```

## 6. Layouts

Cycle through the five preset layouts (`even-horizontal`, `even-vertical`, `main-horizontal`, `main-vertical`, `tiled`):

```text
Ctrl-b Space
```

Jump directly to a preset:

```text
Ctrl-b Alt-1    even-horizontal
Ctrl-b Alt-2    even-vertical
Ctrl-b Alt-3    main-horizontal
Ctrl-b Alt-4    main-vertical
Ctrl-b Alt-5    tiled
```

## 7. Copy mode

| Action                     | Keys             |
| -------------------------- | ---------------- |
| Enter copy mode (scroll)   | `Ctrl-b [`       |
| Exit copy mode             | `q`              |
| Start selection (emacs)    | `Space`          |
| Start selection (vi)       | `v`              |
| Copy selection             | `Enter`          |
| Paste most recent buffer   | `Ctrl-b ]`       |
| Search backward            | `?`              |
| Search forward             | `/`              |

List the cut buffers tmux is holding:

```bash
tmux list-buffers
```

## 8. Send keys to a pane from the shell

Run a command in a specific pane without attaching — note the literal `Enter` so the shell actually runs it:

```bash
tmux send-keys -t dev:editor.0 "echo hello" Enter
```

Send Ctrl-C to interrupt whatever is running in that pane:

```bash
tmux send-keys -t dev:editor.0 C-c
```

## 9. Scripting a workspace

Build a detached session, split it, run commands, then attach in one go:

```bash
tmux new -d -s dev "vim ."
tmux split-window -t dev -h "tail -f /var/log/app.log"
tmux split-window -t dev -v "htop"
tmux select-layout -t dev tiled
tmux attach -t dev
```

## 10. Common ~/.tmux.conf tweaks

A minimal, sane starting config:

```text
# Use Ctrl-a as the prefix (closer to your home row, doesn't clash with readline as often)
unbind C-b
set -g prefix C-a
bind C-a send-prefix

# 256-colour + true-colour terminal
set -g default-terminal "tmux-256color"
set -ga terminal-overrides ",xterm-256color:Tc"

# Mouse: click panes/windows, drag to resize, scroll to enter copy mode
set -g mouse on

# Vi-style keys in copy mode
setw -g mode-keys vi

# Start window and pane numbering at 1, renumber on close
set -g base-index 1
setw -g pane-base-index 1
set -g renumber-windows on

# Bigger scrollback buffer
set -g history-limit 50000

# Reload config without restarting
bind r source-file ~/.tmux.conf \; display "Reloaded ~/.tmux.conf"
```

Reload after editing without restarting tmux:

```bash
tmux source-file ~/.tmux.conf
```
