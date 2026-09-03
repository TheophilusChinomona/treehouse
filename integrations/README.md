# treehouse agent integrations

Let Command Code, Claude Code, or Hermes bring up their own isolated
treehouse worktree, work in it, and tear it down once the PR merges — so you
don't end up with a pile of finished worktrees "chilling" in the pool.

The core lifecycle stays in treehouse (leases, resets, destroy, prune). These
two thin scripts add the agent policy around it:

- **`agent-up <agent> [prompt-or-args...]`** — acquires a durable treehouse
  lease, launches the agent with the worktree as its working directory, and
  runs the down policy when the agent exits.
- **`agent-down <agent> [exit-status]`** — the teardown policy. Reclaims the
  worktree once its HEAD is *provably merged* into the base branch; otherwise
  keeps the lease and reports where the worktree is. It is also safe to run
  standalone, and its merge logic is what a scheduled sweep relies on.

## In-session workflow (the `agent-up` alternative)

The wrappers above launch an agent *from outside* and tear the worktree down on
agent exit. If instead you want to sit inside an agent session and tell it to
grab a worktree, the agents each expose a `/treehouse <task>` command that
provisions a leased worktree, does the work and tests there, opens a PR, and
leaves the worktree parked for post-merge reclamation:

- **Claude Code:** `~/.claude/skills/treehouse/SKILL.md`
- **Hermes:** `~/.hermes/skills/software-development/treehouse/SKILL.md`
- **Command Code:** `~/.commandcode/commands/treehouse.md`

All three follow the canonical procedure in
[`AGENT-WORKFLOW.md`](AGENT-WORKFLOW.md). See that file for the exact rules.

## Remote execution with crabbox (throwaway Daytona sandboxes)

Pair an isolated *location* (treehouse worktree) with an isolated *machine*
(crabbox). Crabbox provisions a throwaway sandbox from a Daytona snapshot,
syncs the current repo, runs a command remotely, streams output, and deletes
the box. The repo's `.crabbox.yaml` + `.github/workflows/crabbox-hydrate.yml`
make `crabbox run -- go test ./...` work against any leased host or sandbox.

Each agent also exposes a `/crabbox daytona <task>` command that runs the task
in a throwaway Daytona sandbox:

- **Claude Code:** `~/.claude/skills/crabbox/SKILL.md`
- **Hermes:** `~/.hermes/skills/software-development/crabbox/SKILL.md`
- **Command Code:** `~/.commandcode/commands/crabbox.md`

Setup notes: the `daytona` provider uses the `crabbox-go` snapshot (Debian +
Go preinstalled); `DAYTONA_API_KEY` lives in the shell environment. Static SSH
hosts (e.g. `speccon` over Tailscale) are selected with `--provider ssh`.

## Why it's safe

The down policy **never auto-returns or deletes a worktree whose commits are
not merged into the base.** That would discard an open PR or local-only work.
Concretely it keeps the lease and reports when:

- the agent exited non-zero (didn't finish cleanly),
- the worktree is dirty or has untracked files,
- HEAD is not on a registered branch,
- origin is unreachable (cannot verify merge),
- HEAD is not an ancestor of `origin/<base>` (PR still open).

It only reclaims (`treehouse destroy <path> --include-leased --yes`) when HEAD
is an ancestor of the remote base — the PR landed — and the tree is clean and
idle. treehouse re-verifies the same safety facts at destroy time.

## Requirements

- `treehouse` on `PATH` (or set `TREEHOUSE_BIN`).
- The agent CLI on `PATH` (`cmd`, `claude`, `hermes`), or set a per-agent
  override (below).
- The scripts are bash; they follow the treehouse convention of not assuming a
  specific shell path. On Windows run them under Git Bash / WSL.

## Usage

From your main checkout (the owning repository — not inside a worktree):

```sh
# Interactive agent session in a leased worktree.
integrations/agent-up cmd "Implement the auth refactor described in #42"

# Claude Code
integrations/agent-up claude

# Hermes
integrations/agent-up hermes "fix the flaky test in checkout_test.go"
```

`agent-up` inherits your terminal, so the agent runs interactively. When the
agent exits, the down policy decides the worktree's fate.

### What each outcome looks like

- **PR merged** → the worktree is reclaimed (destroyed) and removed from the
  pool. Nothing is left behind.
- **PR still open / unmerged work** → the lease is kept. Run `treehouse
  status` to see it. The next scheduled sweep, or a later
  `integrations/agent-down <agent>`, will reclaim it once the merge lands.
- **Agent crashed / dirty tree** → the lease is kept and the path is printed,
  so the work is never silently lost.

## Configuration

| Variable | Meaning | Default |
| --- | --- | --- |
| `TREEHOUSE_BIN` | treehouse binary | `treehouse` |
| `TREEHOUSE_BASE` | branch to cut worktrees from | inferred from repo |
| `TREEHOUSE_LEASE_DIR` | where lease state is cached | `${TMPDIR:-/tmp}/treehouse-agent` |
| `TREEHOUSE_NO_SWEEP` / `TREEHOUSE_KEEP` | skip the down policy; keep the lease | unset |
| `TREEHOUSE_DRY_RUN` | (agent-down) only report, don't reclaim | unset |
| `CMD_LAUNCH` / `CLAUDE_LAUNCH` / `HERMES_LAUNCH` | per-agent launcher override | the agent's CLI binary |

Per-agent launch overrides let you wrap an agent with extra flags without
changing the scripts:

```sh
export CLAUDE_LAUNCH="claude --dangerously-skip-permissions"
export CMD_LAUNCH="cmd --yolo"
integrations/agent-up claude "…"
```

## The scheduled sweep

Nothing an agent forgets should leave finished worktrees around forever. Add a
per-machine scheduled job that runs the safe reclaim:

```sh
# Every 15 minutes: reclaim any IDLE, CLEAN, UNLEASED worktree whose PR
# merged, across every pool under the user-level treehouse root.
treehouse prune --all --yes
```

> `prune` only removes slots that are idle, clean, and **unleased** — it skips
> anything still leased (an in-progress agent) or dirty, so it never races a
> live agent or discards uncommitted work. It is a dry run unless you pass
> `--yes`.

Because leased slots are skipped by `prune`, an agent worktree whose PR merged
while it was still leased is only reclaimed by the wrapper's down policy, or by
a standalone `agent-down`. If you prefer the sweep to also handle leased slots
whose work merged, point it at the wrapper instead:

```sh
# Every 15 minutes: for each cached agent lease, run the same merge-checked
# down policy (reclaims only proven-merged, clean, idle leased worktrees).
for agent in cmd claude hermes; do
  integrations/agent-down "$agent" 0 || true
done
```

Wire either into `cron` (Linux), `launchd` (macOS), or the Windows Task
Scheduler.

## How the merge check works

1. The worktree's current branch is resolved via
   `git -C <worktree> symbolic-ref --short HEAD` (the branch the agent created
   for its PR).
2. `git fetch origin <base>` refreshes the remote ref (best-effort).
3. `git merge-base --is-ancestor <branch> origin/<base>` proves the branch's
   commits landed in the base. Only then is the slot reclaimed.

This is the same ancestry test treehouse itself uses to decide whether a slot
is safe to reset, and it deliberately treats squash-merged work as *unmerged*
(the treehouse jj backend does the same) — failing safe rather than discarding.

## Per-agent setup notes

### Command Code

`cmd` is the binary. Run it interactively through `agent-up` for real work, or
headlessly for automation:

```sh
integrations/agent-up cmd -- -p "Summarize this repo" --yolo
#                      ^^ everything after -- is passed to `cmd`
```

`agent-up` inherits stdio, so interactive `cmd` sessions and the TUI work
normally.

### Claude Code

`claude` is the binary. A `~/.claude/settings.json` already exists on this
machine; the agent picks it up automatically inside the worktree.

### Hermes

`hermes` is the binary (or set `HERMES_LAUNCH`). If Hermes is not yet
installed, the wrapper exits `127` and leaves the lease in place rather than
discarding anything.

## Notes / limitations

- Worktrees are handed over in detached HEAD; the **agent must create and push
  a branch** for its PR so the down policy has a branch to check and treehouse
  has registered work to protect. Most agents (incl. Command Code's `/worktree`
  and Claude Code's git flow) do this automatically.
- `agent-up` records the lease holder as the agent name, so `treehouse status`
  shows which agent holds which worktree.
- The cached lease file (default `/tmp/treehouse-agent/<agent>.lease`) is
  how `agent-down` finds the worktree. If you clear `/tmp`, a leased worktree
  stays safely in the pool — reclaim it manually with
  `treehouse status` + `treehouse destroy <path> --include-leased --yes`.
