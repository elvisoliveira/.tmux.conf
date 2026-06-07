# .tmux.conf
My tmux setup.

```
ln -s (pwd)/.tmux.conf ~/.tmux.conf
```

## Touch scroll on the TTY

On a raw TTY (kernel 5.10+ removed the fbcon scrollback, so `Shift+PgUp`
does nothing), `bin/touch-scroll-daemon` reads the touchscreen via evdev and
turns a vertical drag into `PageUp`/`PageDown`, injected through a virtual
uinput keyboard. `bin/touch-scroll-ctl` (wired to the `client-attached` /
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

Knobs live at the top of `bin/touch-scroll-daemon`: `SCROLL_FRACTION`
(sensitivity), `NATURAL` (drag direction), `TAP_DEADZONE` (tap vs. drag).

## Backlight off (`tela-off`)

`bin/tela-off` turns off the screen backlight until any key is pressed
(chord `Ctrl+G`). It's hardware-level — it writes to the backlight `brightness`
sysfs node, so it does not touch the framebuffer, WiFi, or running processes;
background work keeps going. The bind opens it in a throwaway window so its
`dd` read has its own stdin; the window closes when the script exits.

It needs the `brightness` node to be group-writable. A udev rule grants that
to the `video` group; since the node already exists at boot (an `add` rule
won't fire retroactively), also fix it in place once:

```sh
echo 'SUBSYSTEM=="backlight", ACTION=="add", KERNEL=="intel_backlight", MODE="0664", GROUP="video"' \
  | sudo tee /etc/udev/rules.d/90-backlight.rules
sudo udevadm control --reload-rules
sudo udevadm trigger -s backlight

sudo chgrp video /sys/class/backlight/intel_backlight/brightness
sudo chmod g+w   /sys/class/backlight/intel_backlight/brightness

sudo usermod -aG video "$USER"   # if not already in the video group
```

`~/permitir-backlight.sh` automates exactly this. The group change only
applies to new sessions.

## TTY font size (`font-size`)

`bin/font-size up|down` cycles terminus-font sizes on the raw TTY
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

`~/permitir-setfont.sh` automates exactly this. `bin/font-size` invokes
`sudo -n setfont -C /dev/ttyN ter-vNNn`, which matches the rule above.
