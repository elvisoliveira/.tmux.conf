# sudo — password prompts without a tty

Claude Code runs under a detached tmux server outside any elogind session, so
interactive `sudo` has no tty and polkit has no agent to route to. These are
`SUDO_ASKPASS` helpers that put the prompt where a human is.

- `askpass-tmux` — opens a `display-popup` on the most recently active tmux
  client showing the exact command (read from `/proc` of the calling `sudo`)
  plus an optional self-declared justification left in
  `$XDG_RUNTIME_DIR/sudo-reason` (consumed once, ignored if older than 60s,
  never verified), and hands the password to `sudo` over a 0600 fifo in tmpfs.
  Esc cancels. With no tmux client attached it falls back to the agent below.
- `sudo-tty-agent` — resident fallback: run it in any terminal and leave it in
  the foreground; each request prints the command there and prompts for the
  password. Talks over two fifos in `$XDG_RUNTIME_DIR/sudo-tty-agent`.
- `askpass-tty` — the plain `SUDO_ASKPASS` counterpart that talks only to the
  agent.

```sh
SUDO_ASKPASS=~/.env/.tmux.conf/bin/sudo/askpass-tmux sudo -A <command>
```

The password never goes through the caller: it travels fifo → `sudo`.
