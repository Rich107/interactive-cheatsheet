# git rescue — undo & recover

Surgical recovery commands for when something goes wrong in a git repo. Covers undo, history rewrites, reflog recovery, cherry-picks, reverts, bisect, stash, and recovery from a regretted force-push. Captured against git 2.50.1.

## 1. Inspect history and the reflog

See the topology of every branch in one view — fastest way to orient yourself before any rescue:

```bash
git log --oneline --graph --all --decorate -n 30
```

`reflog` records every move `HEAD` has ever made locally — including commits that are no longer reachable from any branch. This is the safety net behind almost every rescue below:

```bash
git reflog --date=iso -n 30
```

## 2. Undo unstaged changes to a file

Modern syntax (git ≥ 2.23) — discards local edits and restores the index version:

```bash
git restore src/app.ts
```

Older equivalent, still works everywhere:

```bash
git checkout -- src/app.ts
```

Restore the file from a specific commit instead of the index:

```bash
git restore --source=a1b2c3d src/app.ts
```

## 3. Unstage a file (keep the edits)

Modern:

```bash
git restore --staged src/app.ts
```

Older equivalent:

```bash
git reset HEAD src/app.ts
```

## 4. Amend the last commit

Edit message AND/OR add staged changes to the previous commit:

```bash
git commit --amend
```

Quietly fold currently-staged changes into the previous commit, keeping the existing message:

```bash
git commit --amend --no-edit
```

Note: amending rewrites the commit SHA. If the commit has already been pushed and others may have pulled it, prefer a new commit or `git revert`.

## 5. Undo the last commit but keep changes

Keep changes **staged** (ready to re-commit with a fixed message):

```bash
git reset --soft HEAD~1
```

Keep changes in the working tree but **unstage** them:

```bash
git reset --mixed HEAD~1
```

(Default mode for `git reset` is `--mixed` when no flag is given.)

## 6. Discard the last commit entirely (DANGEROUS)

Throws away the commit AND any uncommitted edits — destructive, no second prompt:

```bash
git reset --hard HEAD~1
```

If you regret it, the discarded commit is still in your reflog for ~90 days (see step 7).

## 7. Recover a "lost" commit via reflog

Find the SHA you want back:

```bash
git reflog --date=iso -n 30
```

Inspect a candidate without committing to it:

```bash
git checkout a1b2c3d
```

Pin the recovered commit onto a fresh branch so it's safe from GC:

```bash
git branch recovery a1b2c3d
```

## 8. Cherry-pick a commit onto the current branch

Apply a single commit from anywhere in the repo to the tip of your current branch:

```bash
git cherry-pick a1b2c3d
```

Cherry-pick without auto-committing — lets you tweak the change first:

```bash
git cherry-pick --no-commit a1b2c3d
```

## 9. Revert a public commit safely

Creates a NEW commit that undoes the named one — preferred for shared history because it doesn't rewrite anything:

```bash
git revert a1b2c3d
```

Revert a merge commit (pick which parent to keep as mainline):

```bash
git revert -m 1 a1b2c3d
```

## 10. Bisect to find a regression

Start the search, marking the current commit as bad and a known-good earlier commit:

```bash
git bisect start
git bisect bad
git bisect good a1b2c3d
```

Mark each checkout as `good` or `bad`; git binary-searches until it isolates the offender. Finish up:

```bash
git bisect reset
```

Automate it with a script that exits 0 (good) / non-zero (bad):

```bash
git bisect run ./scripts/repro.sh
```

## 11. Stash work in progress

Stash everything (tracked changes) with a label:

```bash
git stash push -m "wip: experimenting"
```

Include untracked files too:

```bash
git stash push -u -m "wip: experimenting"
```

List, peek, and reapply:

```bash
git stash list
git stash show -p stash@{0}
git stash pop
```

Apply without removing from the stash list:

```bash
git stash apply stash@{0}
```

## 12. Recover from a force-push you regret

On the machine that did the force-push, the previous tip is in your local reflog. Find it:

```bash
git reflog --date=iso -n 30
```

Reset the local branch back to the SHA before the force-push:

```bash
git reset --hard a1b2c3d
```

Then push it back with `--force-with-lease` (safer than `--force` — it aborts if someone else has since pushed):

```bash
git push --force-with-lease origin feature/my-branch
```

If your local reflog has already been clobbered, ask a teammate who fetched recently — their `refs/remotes/origin/feature/my-branch` reflog likely still holds the old SHA.
