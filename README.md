# .tmux.conf
My tmux setup.

```
ln -s (pwd)/.tmux.conf ~/.tmux.conf
```

## Layout

- `bin/` — generic tmux helpers: status-bar segments, `tmux-swap-pane`,
  `claude-resume-restore`, and the `askpass-*` / `sudo-tty-agent` sudo helpers.
- `bin/notify/` — notify-send → tmux toast bridge (`tmux-notifyd`,
  `tmux-notify-popup`, `pane-notify`).
- `bin/git/` — the floating git dashboard (`tmux-git-*`, prefix-less `C-g`).
- `bin/tmb118/` — bits specific to this laptop (Acer TravelMate Spin B118):
  TTY font size, backlight, touchscreen scroll, keyboard layout toggle.

Scripts are called by absolute path from `.tmux.conf` and find their siblings
via `dirname "$0"`, so nothing here needs to be on `PATH`.

## Touch scroll on the TTY

On a raw TTY (kernel 5.10+ removed the fbcon scrollback, so `Shift+PgUp`
does nothing), `bin/tmb118/touch-scroll-daemon` reads the touchscreen via evdev and
turns a vertical drag into `PageUp`/`PageDown`, injected through a virtual
uinput keyboard. `bin/tmb118/touch-scroll-ctl` (wired to the `client-attached` /
`client-detached` hooks) only starts it while a tmux client is on a real VT
(`/dev/ttyN`); under a graphical terminal (`/dev/pts/*`) the compositor already
handles touch, so the daemon stays off to avoid double scrolling.

It needs Python evdev plus read access to the touchscreen and write access to
`/dev/uinput`. Granting that via group + udev means the daemon runs without
root at runtime:

```sh
# dependency
sudo xbps-install -y python3-evdev

# udev rule: expose /dev/uinput to the `input` group
echo 'KERNEL=="uinput", SUBSYSTEM=="misc", GROUP="input", MODE="0660", OPTIONS+="static_node=uinput"' \
  | sudo tee /etc/udev/rules.d/99-uinput.rules
sudo udevadm control --reload-rules
sudo udevadm trigger /dev/uinput

# join the `input` group (reads the touchscreen, writes uinput)
sudo usermod -aG input "$USER"
```

The group change only applies to new sessions, and the tmux server inherits
the groups of whoever started it — so after `usermod`, run `tmux kill-server`
and start a fresh session (or re-login).

Verify and tune:

```sh
id | grep -o input              # must list `input` before the daemon can run
pgrep -af touch-scroll-daemon   # empty = off; with a PID = running
```

Knobs live at the top of `bin/tmb118/touch-scroll-daemon`: `SCROLL_FRACTION`
(sensitivity), `NATURAL` (drag direction), `TAP_DEADZONE` (tap vs. drag).

## Backlight off (`tela-off`)

`bin/tmb118/tela-off` turns off the screen backlight until any key is pressed
(chord `Ctrl+G`). It's hardware-level — it writes to the backlight `brightness`
sysfs node, so it does not touch the framebuffer, WiFi, or running processes;
background work keeps going. The bind opens it in a throwaway window so its
`dd` read has its own stdin; the window closes when the script exits.

It needs the `brightness` sysfs attribute writable by the `video` group.
udev's `MODE`/`GROUP` keys only apply to `/dev` nodes, and a backlight has
none — `brightness` lives under `/sys` — so the rule must `chgrp`/`chmod` it
explicitly via `RUN+=`, which re-applies on every boot:

```sh
echo 'ACTION=="add", SUBSYSTEM=="backlight", KERNEL=="intel_backlight", RUN+="/bin/chgrp video /sys/class/backlight/%k/brightness", RUN+="/bin/chmod g+w /sys/class/backlight/%k/brightness"' \
  | sudo tee /etc/udev/rules.d/90-backlight.rules
sudo udevadm control --reload-rules
sudo udevadm trigger -c add -s backlight   # apply now (and on every boot)

sudo usermod -aG video "$USER"   # if not already in the video group
```

`~/permitir-backlight.sh` automates this. The group change only applies to new
sessions.

## TTY font size (`font-size`)

`bin/tmb118/font-size up|down` cycles terminus-font sizes on the raw TTY
(`Alt+=` / `Alt+-`). It's a silent no-op on a graphical terminal — it keys off
the real terminal name, which tmux passes in via `TMUX_TERM` since `run-shell`
does not inherit the pane's `TERM`. State persists in `~/.cache/tty-font-size`.

It needs `terminus-font` plus a narrowly-scoped passwordless `setfont`:

```sh
sudo xbps-install -y terminus-font

# NOPASSWD only for `setfont -C /dev/tty[1-6] ter-v...`, nothing broader.
# The filename starts with `zz-` on purpose: sudoers applies the LAST matching
# rule, so this drop-in must sort after `wheel` or a blanket wheel rule would
# override the NOPASSWD.
echo "$(id -un) ALL=(root) NOPASSWD: /usr/bin/setfont -C /dev/tty[1-6] ter-v[0-9]*" \
  | sudo tee /etc/sudoers.d/zz-setfont-nopasswd
sudo chmod 0440 /etc/sudoers.d/zz-setfont-nopasswd
sudo visudo -c -f /etc/sudoers.d/zz-setfont-nopasswd   # validate syntax
```

`~/permitir-setfont.sh` automates exactly this. `bin/tmb118/font-size` invokes
`sudo -n setfont -C /dev/ttyN ter-vNNn`, which matches the rule above.

## Sudo without a tty (`askpass-tmux`, `sudo-tty-agent`)

Claude Code runs under a detached tmux server outside any elogind session, so
interactive `sudo` has no tty and polkit has no agent to route to.
`bin/askpass-tmux` is a `SUDO_ASKPASS` helper: it opens a `display-popup` on
the most recently active tmux client showing the exact command (read from
`/proc` of the calling `sudo`) plus an optional self-declared justification
left in `$XDG_RUNTIME_DIR/sudo-reason` (consumed once, ignored if older than
60s, never verified), and hands the password to `sudo` over a 0600 fifo in
tmpfs. Esc cancels.

```sh
SUDO_ASKPASS=~/.env/.tmux.conf/bin/askpass-tmux sudo -A <command>
```

With no tmux client attached it falls back to `bin/sudo-tty-agent`: run that
in any terminal and leave it in the foreground; each request prints the
command there and prompts for the password. `bin/askpass-tty` is the plain
`SUDO_ASKPASS` counterpart that talks only to the agent.
