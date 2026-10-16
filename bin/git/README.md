# git — floating git dashboard

`Ctrl+G` (no prefix, works on the raw TTY and in graphical terminals) opens
[gitsmart](https://github.com/elvisoliveira/gitsmart), a read-only git
dashboard, for the active pane's repository in a floating pane at 85% of the
window. Its keys are in its README; `C-g` closes it. The outer tmux auto-locks
while the float is open, so every key reaches the dashboard — except `C-z`,
bound in .tmux.conf to toggle the float between 85% and fullscreen.

`tmux-git-float` is the only script: it sizes and opens the float (`--run` is
its in-float side, which lifts the outer lock on exit).

Install gitsmart into `~/.local/bin` (needs `cargo`):

```
cargo install --root ~/.local --git https://github.com/elvisoliveira/gitsmart
```
