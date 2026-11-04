# askpass — sudo password prompts without a tty

Claude Code runs under a detached tmux server outside any elogind session, so
interactive `sudo` has no tty and polkit has no agent to route to. These are
`SUDO_ASKPASS` helpers that put the prompt where a human is.

- `askpass-tmux` — opens a `display-popup` on the most recently active tmux
  client showing the exact command (read from `/proc` of the calling `sudo`)
  plus an optional self-declared justification left in
  `$XDG_RUNTIME_DIR/sudo-reason` (consumed once, ignored if older than 60s,
  never verified), and hands the password to `sudo` over a 0600 fifo in tmpfs.
  Esc cancels. With no tmux client attached it falls back to the agent below.
- `askpass-prompt` — the small script that runs inside that popup (shows the
  command, reads the password, writes it to the fifo).
- `sudo-tty-agent` — resident fallback: run it in any terminal and leave it in
  the foreground; each request prints the command there and prompts for the
  password. Talks over two fifos in `$XDG_RUNTIME_DIR/sudo-tty-agent`.
- `askpass-tty` — the plain `SUDO_ASKPASS` counterpart that talks only to the
  agent.

```sh
SUDO_ASKPASS=~/.env/.tmux.conf/bin/askpass/askpass-tmux sudo -A <command>
```

The password never goes through the caller: it travels fifo → `sudo`.

## bw-tmux — one-shot Bitwarden CLI without a tty

Separate from the sudo helpers above (and from the `bw` fish function used in a
terminal): `bw-tmux <bw subcommand> [args...]` asks for the **master password**
in a tmux popup titled `bitwarden`, unlocks, runs that single command and always
locks the vault again (also on error, Esc or signal). Optional one-line
justification via `BW_REASON='…'`.

- The command shown in the popup is declared by the caller and **not verified**
  (unlike the sudo popup, which reads it from `/proc`). Only type the master
  password when you expect the request.
- The password travels popup → 0600 fifo in tmpfs → environment of `bw unlock`
  only; the session key never leaves the process and is invalidated by `bw lock`.
- Piped stdin (e.g. `bw encode | bw-tmux create item`) is shown in the popup as
  a summary only: size and, for a Bitwarden item, its name, type and which
  fields are present, never their values. Other data: size only.
- `status`, `login`, `logout`, `config`, `lock` pass straight through (no popup);
  `unlock` is refused (one-shot only). No tmux client → fails, no fallback.

```sh
BW_REASON='import ssh key' ~/.env/.tmux.conf/bin/askpass/bw-tmux get item github
```
