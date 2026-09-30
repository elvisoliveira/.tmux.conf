# git — floating git dashboard

`Ctrl+G` (no prefix, works on the raw TTY and in graphical terminals) toggles a
read-only git dashboard for the active pane's repository, in a floating pane.
A float can't be split, so the float runs a NESTED tmux (own socket, own
isolated config) whose pane tree is the dashboard:

```
┌───────────────┬──────────────┐
│ working tree  │              │
├───────────────┤     diff     │
│     log       │   (delta)    │
├───────┬───────┤              │
│branchs│ tags  │              │
└───────┴───────┴──────────────┘
```

Moving through the fzf lists on the left updates the diff on the right.
Unpushed commits get a bold-red hash. Typing filters the list under the
cursor; the working tree keeps its tree shape while filtering, the log drops
the graph.

Keys inside the float (the outer tmux auto-locks while it is open, so `C-b`
reaches the nested tmux):

| key | action |
|---|---|
| `C-b h/j/k/l`, `C-w h/j/k/l`, `Tab` | move between panes |
| `C-j`/`C-k`, `J`/`K`, `C-d`/`C-u` | scroll the diff (drive the list on fzf panes) |
| `C-z` | zoom the pane and grow the float to near-fullscreen |
| `C-r` | re-render every pane |
| `C-g`, `C-b q` | close |

Files:

- `tmux-git-toggle` — bound in the outer tmux; opens the float or closes a
  running one.
- `tmux-git-float` — orchestrator: computes geometry and creates the floating
  pane with `new-pane`.
- `tmux-git-nested` — runs inside the float; boots the nested tmux with
  `tmux-git.tmux.conf` and lays out the panes.
- `tmux-git-pane` — one pane's content (`worktree`, `log`, `branches`, `tags`,
  `diff`); the fzf panes push the selection to the diff pane via
  `tmux-git-push`.
- `tmux-git-worktree`, `tmux-git-tree`, `tmux-git-log` — list renderers, used
  both for the initial load and fzf's `change:reload`.
- `tmux-git-zoom`, `tmux-git-refresh` — the `C-z` and `C-r` actions.

Requires `fzf` and `delta`.
