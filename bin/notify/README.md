# notify — libnotify → tmux toast

The TTY-friendly analogue of dunst: any program that calls `notify-send` /
libnotify pops up inside tmux.

- `tmux-notifyd` — ensures a user session bus exists at a fixed address
  (`$XDG_RUNTIME_DIR/bus`), runs `tiramisu --json` on it and feeds each
  notification to the presenter. Idempotent: launched once at load and
  re-kicked on `client-attached`, so it self-heals after a crash. Requires
  `tiramisu` and `dbus-daemon`. The shells export the same fixed
  `DBUS_SESSION_BUS_ADDRESS` so `notify-send` from any pane hits this bus.
- `tmux-notify-popup` — renders ONE notification as an auto-closing toast,
  colored by urgency, on every attached client. Preferred renderer is a
  floating pane via `new-pane -A` (tmux >= 3.8: floats above a zoomed window
  and keeps the zoom). Requires tmux >= 3.8.
- `pane-notify` — bound to `prefix M`: toggles `pipe-pane` on the current pane
  and fires a notification whenever it produces new output, throttled to one
  per cooldown. Useful to get pinged when a long command finishes.
- `notify-selftest` — fires sample notifications straight through the
  presenter on the current server, to check rendering on a new tmux version.

Options (`set -g` in `.tmux.conf`):

```
@notify-popup-duration 4 # seconds a toast stays up
```

Test by hand:

```sh
echo '{"source":"build","summary":"Done","body":"3 passed","hints":{"urgency":"0x02"}}' \
    | ~/.env/.tmux.conf/bin/notify/tmux-notify-popup
```
