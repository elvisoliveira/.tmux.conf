# .tmux.conf
My tmux setup.

```
ln -s (pwd)/.tmux.conf ~/.tmux.conf
```

Plugins are managed by [tpack](https://github.com/nicholas-fedor/tpack), a
drop-in replacement for tpm: `run 'tpack init'` at the bottom of `.tmux.conf`
installs and sources everything declared with `@plugin`.

## Layout

Scripts are called by absolute path from `.tmux.conf` and find their siblings
via `dirname "$0"`, so nothing here needs to be on `PATH`. Each folder has its
own README.

- `bin/` — generic helpers: status-bar segments (`status-seg`, `battery`,
  `keyboard-layout`, `temperature`) and `tmux-swap-pane`.
- [`bin/notify/`](bin/notify/README.md) — notify-send → tmux toast bridge.
- [`bin/git/`](bin/git/README.md) — read-only git dashboard in a floating pane
  (`Ctrl+G`).
- [`bin/sudo/`](bin/sudo/README.md) — `SUDO_ASKPASS` helpers for sudo without a
  tty (tmux popup, or a resident agent in a terminal).
- [`bin/tmb118/`](bin/tmb118/README.md) — bits specific to this laptop (Acer
  TravelMate Spin B118): TTY font size, backlight, touchscreen scroll, keyboard
  layout toggle.
