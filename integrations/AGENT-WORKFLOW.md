# In-session agent workflow: worktrees that tear down after merge

This is the canonical procedure an agent follows when its user says "spin up a
worktree and do this" (`/treehouse <task>` in Claude Code or Hermes,
`/treehouse <task>` in Command Code). Each agent's skill/command body is a thin
wrapper that loads this file's steps.

## The lifecycle you are driving

1. **Provision** — acquire a fresh, isolated treehouse worktree for the task.
2. **Work** — do the implementation, tests, and any checks inside that worktree.
3. **PR** — commit on a branch, push it, open a pull request.
4. **Finish** — leave the worktree parked and leased so treehouse (or a later
   sweep) reclaims it **only once the PR is merged**. Never delete or reset a
   worktree whose work has not provably landed.

Your job is to *drive* this lifecycle with the `treehouse` CLI and git — not to
reimplement worktree management.

## Prerequisites

- `treehouse` on PATH (it manages the pool). If missing, stop and tell the user.
- The user is in a git repository that is the **main checkout** (not already
  inside a treehouse worktree).

## Procedure

### 1. Provision a worktree

Run from the repository root:

```sh
TREEHOUSE_BIN="${TREEHOUSE_BIN:-treehouse}"
if ! command -v "$TREEHOUSE_BIN" >/dev/null 2>&1; then
  echo "treehouse not found; install it first" >&2
  exit 127
fi

# Lease a clean worktree for this task. --lease-holder names the agent so
# `treehouse status` shows who holds it. --json gives a machine-readable
# path + lease id. Print ONLY the path to stdout; banners go to stderr.
lease_json="$("$TREEHOUSE_BIN" get --lease --lease-holder "$AGENT_NAME" --json)"
wt_path="$(printf '%s' "$lease_json" | sed -n 's/.*"path"[[:space:]]*:[[:space:]]*"\([^"]*\)".*/\1/p')"
lease_id="$(printf '%s' "$lease_json" | sed -n 's/.*"lease_id"[[:space:]]*:[[:space:]]*"\([^"]*\)".*/\1/p')"

# IMPORTANT: the worktree is handed over in detached HEAD. To have a branch
# that a PR can target (and that the post-merge teardown can verify), create
# one right away:
git -C "$wt_path" checkout -b "feat/<short-task-name>"
```

Record `$wt_path` and `$lease_id` somewhere you can find them later in the
session (e.g. a temp file, or just keep them in context). You will need them at
teardown.

If acquisition fails (pool full, all dirty, etc.), report the error from
`treehouse` and stop — do not fall back to `git worktree add` or a fresh
clone, which would bypass the pool and orphan a directory.

### 2. Work inside the worktree

`cd` into `$wt_path` and do the implementation there. Run tests, builds, and
checks **inside the worktree**, never in the main checkout. The worktree is a
full checkout sharing the repo's object store — commits you make there are
ordinary commits on your `feat/*` branch.

### 3. Commit and push; open a PR

When the work is done and tests pass:

```sh
git -C "$wt_path" add -A
git -C "$wt_path" commit -m "<conventional message>"
git -C "$wt_path" push -u origin "feat/<short-task-name>"

# Open a PR with gh (from the main checkout or the worktree; both reach origin).
gh pr create --base <base-branch> --head "feat/<short-task-name>" \
  --title "<title>" --body "<summary>"
```

`<base-branch>` is the branch the worktree was cut from (the repo default, or
the configured `base_branch`). The worktree stays checked out on your `feat/*`
branch — do **not** delete it or reset it after pushing.

### 4. Finish: park, and let teardown happen only after merge

After the PR is open (or after you've reported the branch for the user to
open one), **leave the worktree exactly where it is**: clean, on its `feat/*`
branch, leased. Do not run `treehouse return`, do not reset, do not delete.

Rationale:

- The lease means no other agent gets this worktree, and `treehouse prune`
  will not touch it while it is leased — so your in-progress or unmerged work
  is protected.
- `treehouse return` would reset the worktree to its base and discard your
  branch's local state, so it must only happen once the work is merged.
- Post-merge reclamation is handled by the `agent-down` teardown policy (run
  by the `agent-up` wrapper, or by a scheduled sweep), which verifies your
  `feat/*` branch is an ancestor of `origin/<base>` before reclaiming.

If you (or the user) want to reclaim the worktree now because the PR was
already merged, run the merge-checked teardown, not a blind reset:

```sh
# Reclaims only if the current branch is merged into origin/<base>.
# Keeps the lease and reports the path otherwise.
agent-down <agent-name> 0
```

If `agent-down` is not installed, the safe equivalent is:

```sh
branch="$(git -C "$wt_path" symbolic-ref --short HEAD)"
# Refresh origin and prove ancestry before destroying anything.
git -C "$wt_path" fetch origin <base>
if git -C "$wt_path" merge-base --is-ancestor "$branch" "origin/<base>"; then
  treehouse destroy "$wt_path" --include-leased --yes
fi
```

## Rules (do not skip)

- **Never reset, `return`, or destroy a worktree whose branch is not merged**
  into its base. That discards committed work that may be on an open PR.
- **Never work in the main checkout** when the user asked for a worktree.
- **Never create ad-hoc worktrees/clones** outside the treehouse pool.
- **Do not commit build artifacts, the pool dir, or `.treehouse/`** — treehouse
  git-ignores its own pool, but keep your `feat/*` commit clean.
- If the task turns out to need no PR (e.g. experimentation), still leave the
  worktree parked and leased, and tell the user it is there — do not silently
  discard it.
