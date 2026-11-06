---
name: worktree-pool
description: Use when working in a Git repo that uses the `wt` worktree slot manager — to pick up a branch in its own worktree, find a worktree's path/port/URL, run commands in it, or free the slot when done. Triggers include "wt", "worktree slot", "assign a branch", "release the worktree", or a repo with a `worktrees/wt-N` folder.
---

# worktree-pool (`wt`)

`wt` manages a fixed pool of reusable Git worktrees ("slots") inside a repo, so several branches can be checked out and running side by side, each with its own dev port.

```
<repo>/
  worktrees/
    wt-1/   -> port 3101
    wt-2/   -> port 3102
    ...
```

- A slot is **FREE** when it is in detached HEAD, and **ACTIVE** when a branch is checked out.
- Git is the only source of truth; there is no state file.
- `wt` works from anywhere inside the repo, including from inside a slot.

## Before you start

1. Run `wt list` to see slots, their status, branch and URL.
2. If it reports no slots, ask the user before running `wt init [N]` (default 5). It creates folders and worktrees.
3. Make sure `worktrees/` is git-ignored in the main repo. If it isn't, tell the user rather than editing `.gitignore` yourself.

## Commands

| Command | Alias | What it does |
|---|---|---|
| `wt init [N]` | | Create N free slots (default 5). Existing slots are skipped. |
| `wt list` | `ls` | Show every slot: status, branch, URL. |
| `wt assign <BRANCH> [--base BASE]` | `a` | Put BRANCH into the first free slot. |
| `wt release [SLOT]` | `r`, `free` | Reset the slot and return it to FREE. No SLOT = current worktree. |
| `wt path [SLOT]` | | Absolute path of the slot. |
| `wt port [SLOT]` | | Dev port (3100 + slot number). |
| `wt url [SLOT]` | | `http://localhost:<port>`. |
| `wt start <SLOT> <CMD...>` | | Run CMD with the slot as working directory. |
| `wt doctor` | | Git worktree diagnostics plus `wt list`. |

`[SLOT]` is optional everywhere it's shown in brackets: when you're inside a slot, `wt` detects it. From the main repo you must name it (`wt-3`).

### How `assign` picks the source

1. Branch already assigned to a slot → prints that slot and does nothing.
2. Local branch exists → checks it out.
3. `origin/<BRANCH>` exists → creates a tracking branch.
4. Otherwise → fetches `origin/BASE` and creates a new branch from it (BASE is `--base`, else `$WT_BASE`, else `dev`). Falls back to local BASE if there's no remote copy; BASE must exist in one of the two.

Run `git fetch` first if the branch may only exist on the remote.

## Typical workflow

```bash
wt a feature/payments          # grab a free slot
cd "$(wt path wt-1)"           # or use the slot name wt printed
wt start wt-1 bin/dev          # dev server on port 3101
open "$(wt url wt-1)"
# ...work, commit, push...
wt r                           # from inside the slot, when the branch is done
```

Pass the slot's port to whatever needs it, e.g. `PORT=$(wt port) bin/dev`, so parallel slots don't collide.

## Releasing safely

`release` is destructive: it runs `git reset --hard` and `git clean -fd` in the slot, then detaches HEAD at the base branch's commit (`$WT_BASE`, else `dev`). The branch itself is kept, as are its commits.

- It first prints uncommitted/untracked files and unpushed commits, then asks `Continue? [y/N]` on stdin.
- On an already-FREE slot it moves the slot to the base branch's current commit. It asks first only if the slot has uncommitted changes or commits on no branch.
- **Never release a slot without the user's explicit OK**, and never pipe `y` into it on your own initiative.
- Before asking, commit or stash anything worth keeping, and mention unpushed commits so the user can push first.
- Don't release a slot with a running dev server; stop it first.

## Gotchas

- A branch can only be checked out in one worktree. If `assign` fails because the branch is checked out in the main repo, switch the main repo to another branch first (ask the user).
- "no free slots" → `wt list`, then ask the user whether to release one or `wt init` more.
- Slot names must be `wt-N`; anything else is rejected.
- Each slot has its own working tree, so dependencies (`bundle install`, `node_modules`) and untracked config files (`.env`, `config/master.key`) may need setting up per slot.
