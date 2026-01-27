# git-stack

A Git stacked branches management tool for organizing and synchronizing dependent branches.

## Overview

`git-stack` helps you manage **stacked branches** — a workflow where feature branches are built on top of each other rather than all branching from `main`. This is useful for:

- Breaking large features into smaller, reviewable chunks
- Maintaining a logical progression of changes
- Keeping dependent work moving while waiting for code review

### Key Features

- **Works with branches, not patches** — Uses standard Git branches that you can push, pull, and manage with any Git tool. No special commit formats or patch files required.
- **Works completely offline** — All operations are local. No server, no account, no network connection needed.
- **Zero dependencies** — Single Python script using only the standard library. Just copy and run.
- **Transparent metadata** — Stack relationships stored in standard git config, easily inspectable and portable.

## Table of Contents

- [Overview](#overview)
- [Installation](#installation)
- [Quick Start](#quick-start)
- [User Journeys](#user-journeys)
- [How It Works](#how-it-works)
- [Commands](#commands)
- [Prior Art](#prior-art)
- [Requirements](#requirements)
- [License](#license)

## Installation

1. Clone the repository:
   ```bash
   git clone https://github.com/ershov/git-stack.git
   ```

2. Copy the `git-stack` script to a directory in your `PATH`:
   ```bash
   cp git-stack/git-stack ~/bin/
   ```

3. (Optional) Set up git aliases for convenience:
   ```bash
   git config --global alias.stack '!git-stack'
   git config --global alias.stk '!git-stack'
   ```

4. The tool can be invoked as:
   ```bash
   git-stack <command>
   # or (if aliased):
   git stack <command>
   git stk <command>
   ```

## Quick Start

```bash
# Start from main and create a stack of features
git checkout main
git stack create feature/step1
# ... make changes, commit ...

git stack create feature/step2  # Creates step2 stacked on step1
# ... make changes, commit ...

git stack create feature/step3  # Creates step3 stacked on step2
# ... make changes, commit ...

# View your stack
git stack ls

# When main is updated, sync all branches
git stack sync-all
```

## User Journeys

### Journey 1: Creating a New Feature Stack

Start from main and create a series of dependent changes:

```bash
git checkout main
git stack create feature/step1
# ... make changes for step 1 ...
git commit -m "Step 1: Add base infrastructure"

git stack create feature/step2     # Creates step2 stacked on step1
# ... make changes for step 2 ...
git commit -m "Step 2: Add API layer"

git stack create feature/step3     # Creates step3 stacked on step2
# ... make changes for step 3 ...
git commit -m "Step 3: Add UI components"
```

**Result:** `main <- feature/step1 <- feature/step2 <- feature/step3`

### Journey 2: Adopting Existing Branches

Organize existing branches into a stack:

```bash
git checkout feature/api-refactor
git stack adopt feature/api-refactor main
# Now feature/api-refactor is marked as stacked on main

git checkout feature/api-tests
git stack adopt feature/api-tests feature/api-refactor
# Now feature/api-tests is marked as stacked on feature/api-refactor
```

### Journey 3: Syncing After Upstream Changes

When main or a base branch gets updated:

```bash
# Update just the current branch from its immediate parent
git checkout feature/step2
git stack sync1

# Recursively sync current branch (sync all ancestors first)
git checkout feature/step3
git stack sync-up

# Sync only the dependencies (not the current branch)
git checkout feature/step3
git stack sync-deps

# Sync all stacked branches in the repo
git stack sync-all
```

### Journey 4: Nested Stacks (Stack on Stack)

Create sub-features that branch off stacked branches:

```bash
git checkout feature/step2
git stack create feature/step2-hotfix
# ... make hotfix changes ...
git commit -m "Hotfix for step 2"
```

**Result:**
```
main <- step1 <- step2 <- step2-hotfix
                     \<- step3
```

Both `step2-hotfix` and `step3` are based on `step2`. When `step2` is updated, `git stack sync-all` will rebase both onto the new `step2`.

### Journey 5: Changing the Base of a Stack

```bash
git checkout feature/step3
git stack base
# Shows: feature/step2

git stack base feature/step1
# Changes step3 to be based on step1 instead of step2

git stack rebase main
# Rebases step3 directly onto main
```

### Journey 6: Reviewing and Pushing Stacks

```bash
# View the current stack structure
git stack ls
#   main
#   └── feature/step1 (2 commits ahead)
#       └── feature/step2 (1 commit ahead)
#           ├── feature/step2-hotfix (1 commit ahead)
#           └── feature/step3 (3 commits ahead)

# View commits in the current stacked branch
git checkout feature/step2
git stack log

# View diff against base
git stack diff

# Push all branches in the stack
git stack push-all
```

### Journey 7: Completing and Cleaning Up

After `feature/step1` is merged to main:

```bash
git checkout main
git pull
git stack delete feature/step1
# Deletes step1 and updates step2's base to point to main
```

Or manually update the base:

```bash
git checkout feature/step2
git stack base main
git stack sync-up
```

## How It Works

Stack relationships are stored in git config:

```bash
git config branch.<branch>.stackbase <base-branch>
```

For example:
```bash
git config branch.feature/step2.stackbase feature/step1
```

This metadata is local to your repository and doesn't affect remote branches.

## Commands

| Command | Alias | Description |
|---------|-------|-------------|
| **Viewing** |||
| `ls` | `list`, `st`, `status` | List stacked branches as a tree |
| `log` | `l` | Show commits in current branch (relative to base) |
| `diff` | `df` | Show diff of current branch against its base |
| **Creating & Managing** |||
| `create <branch>` | `cr` | Create a new stacked branch from current branch |
| `adopt <branch> [base]` | `ad` | Mark existing branch as stacked on base |
| `set-parent [branch] <parent>` | `sp`, `sb` | Set the parent/base of a branch |
| `base [new-base]` | `b` | Show or change the base of current branch |
| `delete <branch>` | `d` | Delete a stacked branch and update dependents |
| `rename [old] <new>` | `rn` | Rename a branch and update all references |
| `split <branch> [parent] [child]` | `spl` | Split a commit into a new branch |
| **Syncing** |||
| `sync1` | `s1` | Sync current branch with its immediate parent |
| `sync-up` | `su` | Sync current branch with all its ancestors |
| `sync-deps` | `sd` | Sync only dependencies of current branch |
| `sync-all` | `sa` | Sync all stacked branches in the repository |
| **Conflict Handling** |||
| `continue` | `c` | Continue sync after resolving conflicts |
| `abort` | `ab` | Abort current rebase, continue with remaining |
| `abort-all` | `aa` | Abort and restore all branches to original state |
| `rebase-reset` | `rr` | Clear rebase state without restoring branches |
| **Rebasing** |||
| `rebase <new-base>` | `rb` | Rebase current branch onto a new base |
| **Pushing** |||
| `push` | `p` | Push current stacked branch to remote |
| `push-all` | `pa` | Push all stacked branches to remote |
| `push-remotes` | `pr` | Push only branches that have remote tracking |
| **Import/Export** |||
| `export [file]` | | Export stack config to JSON |
| `import <file>` | | Import stack config from JSON (replaces existing) |
| `import-update <file>` | | Import stack config (merges with existing) |
| `zap` | | Remove all stack configuration from repo |
| **Help** |||
| `help` | `h`, `-h`, `--help` | Show help message |

---

### Viewing Stacks

#### `git stack ls` (aliases: `list`, `st`, `status`)

List stacked branches as a tree, showing sync status and remote tracking info.

```bash
git stack ls
# Output:
#   main
#   └── feature/step1 (2 commits ahead, origin/feature/step1)
#       └── feature/step2 (1 commit ahead, needs sync)
#           ├── feature/step2-hotfix (1 commit ahead)
#           └── feature/step3 (3 commits ahead, origin/feature/step3 1 ahead)
```

#### `git stack log` (alias: `l`)

Show commits in current branch that are not in its base. Passes additional arguments to `git log`.

```bash
git stack log
# Shows commits in current branch relative to its base

git stack log --oneline
# Compact format

git stack log -p
# Show patches
```

#### `git stack diff` (alias: `df`)

Show diff of current branch against its base. Passes additional arguments to `git diff`.

```bash
git stack diff
# Shows all changes in current branch relative to its base

git stack diff --stat
# Show diffstat only

git stack diff -- src/
# Diff only files in src/ directory
```

---

### Creating & Managing Stacks

#### `git stack create <branch>` (alias: `cr`)

Create a new stacked branch from current branch.

```bash
git checkout main
git stack create feature/auth
# Created stacked branch 'feature/auth' based on 'main'

git stack create feature/auth-tests
# Created stacked branch 'feature/auth-tests' based on 'feature/auth'
```

#### `git stack adopt <branch> [base]` (alias: `ad`)

Mark an existing branch as stacked on a base branch. If base is not specified, uses current branch.

```bash
# Adopt branch onto current branch
git checkout main
git stack adopt feature/existing
# Set parent of 'feature/existing' to 'main'

# Adopt branch onto specified base
git stack adopt feature/tests feature/api
# Set parent of 'feature/tests' to 'feature/api'

# Adopt current branch (use '.' for current)
git checkout feature/my-branch
git stack adopt . main
# Set parent of 'feature/my-branch' to 'main'
```

#### `git stack set-parent [branch] <parent>` (aliases: `sp`, `sb`, `set-base`)

Set the parent/base of a branch without rebasing.

```bash
# Set parent of current branch
git stack set-parent feature/api
# Set parent of 'feature/current' to 'feature/api'

# Set parent of specified branch
git stack set-parent feature/tests feature/api
# Set parent of 'feature/tests' to 'feature/api'
```

#### `git stack base [new-base]` (alias: `b`)

Show or change the base of current branch.

```bash
# Show current base
git stack base
# feature/step1

# Change base (updates metadata only, does not rebase)
git stack base main
# Changed base of 'feature/step2' from 'feature/step1' to 'main'
```

#### `git stack delete <branch>` (alias: `d`)

Delete a stacked branch and update dependents to point to deleted branch's base.

```bash
# Given: main <- step1 <- step2 <- step3
git stack delete feature/step2
# Updated 'feature/step3' to be based on 'feature/step1'
# Deleted stacked branch 'feature/step2'
# Result: main <- step1 <- step3
```

#### `git stack rename [old] <new>` (alias: `rn`)

Rename a stacked branch and update all references (base relationships and dependents).

```bash
# Rename current branch
git stack rename feature/better-name
# Renamed branch 'feature/old-name' to 'feature/better-name'

# Rename specified branch
git stack rename feature/old feature/new
# Renamed branch 'feature/old' to 'feature/new'
#   Updated stack base: 'feature/new' -> 'main'
#   Updated dependent: 'feature/child' -> 'feature/new'
```

#### `git stack split <branch> [parent] [child]` (alias: `spl`)

Split a commit into a new stacked branch between parent and child. Creates a new branch at the current commit.

```bash
# Given: main <- feature/big-change (with multiple commits)
# You're at a commit in the middle that you want to split out

# Auto-detect parent and child (when unambiguous)
git stack split feature/part1
# Created stacked branch 'feature/part1' at current commit
#   main <- feature/part1 <- feature/big-change

# Explicit parent and child
git stack split feature/middle feature/step1 feature/step2
# Created stacked branch 'feature/middle' at current commit
#   feature/step1 <- feature/middle <- feature/step2
```

---

### Syncing Branches

#### `git stack sync1` (alias: `s1`)

Sync current branch with its immediate parent (single rebase).

```bash
git checkout feature/step2
git stack sync1
# Syncing 'feature/step2' onto 'feature/step1'...
# Sync complete.
```

#### `git stack sync-up` (alias: `su`)

Sync current branch with all its ancestors (recursive). Syncs from root to current.

```bash
# Given: main <- step1 <- step2 <- step3 (you're on step3)
git stack sync-up
# Syncing 'feature/step1' onto 'main'...
# Syncing 'feature/step2' onto 'feature/step1'...
# Syncing 'feature/step3' onto 'feature/step2'...
# Sync complete.
```

#### `git stack sync-deps` (alias: `sd`)

Sync only dependencies of current branch (ancestors), but not the current branch itself.

```bash
# Given: main <- step1 <- step2 <- step3 (you're on step3)
git stack sync-deps
# Syncing 'feature/step1' onto 'main'...
# Syncing 'feature/step2' onto 'feature/step1'...
# Sync complete.
# (step3 is not rebased)
```

#### `git stack sync-all` (alias: `sa`)

Sync all stacked branches in the repository in dependency order.

```bash
git stack sync-all
# Syncing 'feature/step1' onto 'main'...
# Syncing 'feature/step2' onto 'feature/step1'...
# Syncing 'feature/step2-hotfix' onto 'feature/step2'...
# Syncing 'feature/step3' onto 'feature/step2'...
# Sync complete.
```

---

### Handling Conflicts

#### `git stack continue` (alias: `c`)

Continue sync after resolving conflicts.

```bash
# After resolving conflicts during a sync operation
git add .
git stack continue
# Continuing rebase...
# Syncing 'feature/step3' onto 'feature/step2'...
# Sync complete.
```

#### `git stack abort` (alias: `ab`)

Abort current rebase, skip the current branch, and continue with remaining branches.

```bash
git stack abort
# Aborting current rebase...
# Skipping 'feature/step2'...
# Continuing with remaining branches...
# Syncing 'feature/step3' onto 'feature/step2'...
# Sync complete (with some branches skipped).
```

#### `git stack abort-all` (alias: `aa`)

Abort and restore all branches to their original state before the sync started.

```bash
git stack abort-all
# Aborting current rebase...
# Restoring all branches to original state...
# Restoring 'feature/step1' to abc1234...
# Restoring 'feature/step2' to def5678...
# Sync aborted. All branches restored to original state.
```

#### `git stack rebase-reset` (alias: `rr`)

Clear rebase state without restoring branches. Useful when branches are in a good state but sync state is stuck.

```bash
git stack rebase-reset
# Rebase state cleared.
```

---

### Rebasing

#### `git stack rebase <new-base>` (alias: `rb`)

Rebase current branch onto a new base, updating the stack base metadata.

```bash
git checkout feature/step3
git stack rebase main
# Rebasing 'feature/step3' onto 'main'...
# Rebase complete.
# (feature/step3 is now based on main instead of step2)
```

---

### Pushing to Remote

#### `git stack push` (alias: `p`)

Push current stacked branch to remote with upstream tracking.

#### `git stack push-all` (alias: `pa`)

Push all stacked branches to remote.

#### `git stack push-remotes` (alias: `pr`)

Push only branches that already have remote tracking branches.

```bash
git stack push-remotes
# Pushing 2 branch(es) with remotes:
#   feature/step1
#   feature/step3

git stack push-remotes --force-with-lease
# Force push with lease
```

---

### Import/Export

#### `git stack export [file]`

Export stack configuration to JSON. Outputs to stdout if no file specified.

#### `git stack import <file>`

Import stack configuration from JSON, replacing all existing configuration.

#### `git stack import-update <file>`

Import stack configuration, merging with existing (does not remove existing relationships).

#### `git stack zap`

Remove all stack configuration from the repository.

---

### Help

#### `git stack help` (alias: `h`, `-h`, `--help`)

Show help message with all available commands.

## Prior Art

There are several existing tools for managing stacked changes and dependent branches:

| Tool | Language | Description |
|------|----------|-------------|
| [Graphite](https://graphite.dev/) | TypeScript | Commercial tool with CLI and web UI for stacked PRs, integrates with GitHub |
| [ghstack](https://github.com/ezyang/ghstack) | Python | Facebook's tool for stacked diffs, creates separate PRs for each commit |
| [git-branchless](https://github.com/arxanas/git-branchless) | Rust | Suite of tools for working with DAG-structured branches, includes `git move` and `git sync` |
| [git-town](https://www.git-town.com/) | Go | High-level Git workflow automation with stacked changes support |
| [spr](https://github.com/ejoffe/spr) | Go | Stacked Pull Requests on GitHub, one commit per PR |
| [git-machete](https://github.com/VirtusLab/git-machete) | Python | Tool for organizing and visualizing branch dependencies |
| [Stacked Git (StGit)](https://stacked-git.github.io/) | Python | Manage patches as a stack on top of a branch |
| [Sapling](https://sapling-scm.com/) | Rust | Meta's source control system with native stacked commits support |
| [Phabricator/Arcanist](https://www.phacility.com/phabricator/) | PHP | Code review platform with stacked diffs workflow |
| [Gerrit](https://www.gerritcodereview.com/) | Java | Code review system with dependent changes support |
| [git-ps](https://github.com/uptech/git-ps) | Rust | Git Patch Stack for managing patches as a stack |
| [git-revise](https://github.com/mystor/git-revise) | Python | Efficiently update, split, and rearrange Git commits |
| [git-absorb](https://github.com/tummychow/git-absorb) | Rust | Automatically absorb staged changes into ancestor commits |

### Why git-stack?

`git-stack` aims to be:

- **Simple**: Single Python script with no external dependencies — just copy and run
- **Offline-first**: All operations are local; no server, account, or network connection required
- **Branch-based**: Works with standard Git branches, not patches or special commit formats
- **Transparent**: Uses standard git config for metadata storage — easily inspectable and portable
- **Non-invasive**: Works with existing branches; no need to change your commit workflow
- **Flexible**: Supports both linear stacks and branching (multiple children per branch)

### Comparison with Other Tools

| Feature | git-stack | Graphite | ghstack | git-branchless | StGit |
|---------|-----------|----------|---------|----------------|-------|
| Works offline | ✅ | ❌ (requires server) | ❌ (requires GitHub) | ✅ | ✅ |
| No installation | ✅ (single script) | ❌ | ❌ | ❌ | ❌ |
| Uses branches | ✅ | ✅ | ❌ (commits) | ✅ | ❌ (patches) |
| No account required | ✅ | ❌ | ❌ | ✅ | ✅ |
| Works with any remote | ✅ | ❌ (GitHub only) | ❌ (GitHub only) | ✅ | ✅ |
| Branching stacks | ✅ | ❌ | ❌ | ✅ | ❌ |

## Requirements

- Python 3.6+
- Git 2.0+

## License

GPLv3 License - See [LICENSE](LICENSE) for details.
