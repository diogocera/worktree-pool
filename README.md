# worktree-pool

`wt` is a small Bash script that manages a pool of reusable Git worktrees ("slots") inside a repository. Each slot can hold one branch, and each gets its own development port, so you can run several branches side by side without stashing, re-installing or fighting over `localhost:3000`.

```
<repo>/
  worktrees/
    wt-1/   -> http://localhost:3101
    wt-2/   -> http://localhost:3102
    wt-3/   -> http://localhost:3103
```

Slots are created once and recycled: a **FREE** slot sits in detached HEAD; assigning a branch makes it **ACTIVE**; releasing it resets it back to FREE. Git is the only source of truth, so there's no state file to drift.

## Install

Requires Bash and Git (2.23+ for `git switch`).

```bash
git clone https://github.com/diogocera/worktree-pool.git ~/Projects/worktree-pool
ln -s ~/Projects/worktree-pool/wt ~/.local/bin/wt   # any directory on your PATH
```

Then in your repo, ignore the slots folder:

```bash
echo "worktrees/" >> .gitignore
```

## Quick start

```bash
wt init 5                    # create wt-1 … wt-5
wt a feature/payments        # assign a branch to the first free slot
wt list                      # see what's where

wt start wt-1 bin/dev        # run a command inside the slot
open "$(wt url wt-1)"        # http://localhost:3101

cd "$(wt path wt-1)"
wt r                         # done: release the current slot
```

## Commands

| Command | Alias | Description |
|---|---|---|
| `wt init [N]` | | Create N free slots (default 5). Existing ones are skipped. Untracked `.env` / `.env.*` files from the main checkout (except under `scripts/`) are copied into every slot that lacks them. |
| `wt list` | `ls` | Show each slot's status, branch and URL. |
| `wt assign <BRANCH> [--base BASE]` | `a` | Assign BRANCH to the first free slot. |
| `wt release [SLOT]` | `r`, `free` | Reset a slot and return it to FREE. |
| `wt path [SLOT]` | | Print the slot's absolute path. |
| `wt port [SLOT]` | | Print the slot's port. |
| `wt url [SLOT]` | | Print `http://localhost:<port>`. |
| `wt start <SLOT> <CMD...>` | | Run a command with the slot as working directory. |
| `wt doctor` | | Show Git worktree diagnostics. |

Wherever `[SLOT]` is optional, `wt` uses the slot you're currently inside.

### `assign`

`wt assign <BRANCH>` figures out where the branch comes from:

1. Already in a slot → tells you which one.
2. Exists locally → checks it out.
3. Exists on `origin` → creates a local tracking branch (`git fetch` first).
4. Doesn't exist → fetches `origin/<base>` and creates it from there, so it starts at the latest base commit. Falls back to the local base branch if there's no remote copy.

The base is `--base BASE`, else `$WT_BASE`, else `dev`.

### `release`

`wt release` is **destructive to uncommitted work**: it runs `git reset --hard` and `git clean -fd`, then detaches HEAD at the base branch (`$WT_BASE`, else `dev`). Before asking for confirmation it lists uncommitted and untracked files and any unpushed commits. The branch and its commits are kept.

Releasing a slot that is already FREE updates it to the base branch's current commit. If the slot has uncommitted changes or commits on no branch, they are listed and confirmation is asked first.

## Ports

Ports are derived from the slot number: `3100 + N`. Use them to keep parallel servers apart, e.g.:

```bash
PORT=$(wt port) bin/dev
```

## Using it with AI coding agents

[`SKILL.md`](SKILL.md) describes `wt` for agents such as Claude: when to use it, the commands, and the safety rules around `release`. Drop this folder into your agent's skills directory to use it.

## Notes

- A branch can only be checked out in one worktree at a time, including the main one.
- Each slot is a separate working tree: dependencies and untracked config (`.env`, credentials) need setting up per slot.
