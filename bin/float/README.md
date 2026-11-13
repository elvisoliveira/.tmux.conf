# float — programs in a floating pane

Both open in a floating pane at 85% of the window, over the active pane's
directory (no prefix needed):

- `Ctrl+G` — [gitsmart](https://github.com/elvisoliveira/gitsmart), a read-only
  git dashboard, at the repository root. Its keys are in its README; `C-g`
  closes it. Works on the raw TTY and in graphical terminals.
- `Ctrl+E` — nvim in that directory with nvim-tree open; `C-e` again (vimrc mapping
  to `:qa`) closes it, `C-b q` force-quits. `C-b C-e` sends a literal `C-e`.

The outer tmux auto-locks while a float is open, so every key reaches the
program — except `C-z`, bound in .tmux.conf to toggle the float between 85% and
fullscreen.

`tmux-float` is the only script: `tmux-float git|edit` sizes and opens the
float (`--run` is its in-float side, which lifts the outer lock on exit).

Install gitsmart into `~/.local/bin` (needs `cargo`):

```
cargo install --root ~/.local --git https://github.com/elvisoliveira/gitsmart
```
