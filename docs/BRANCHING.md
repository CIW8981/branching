# Git branching strategy

This guide describes the branching strategy the team uses for multiple deployment environments. It also lists the scenarios that were tested on this repository and what happened in each.

## 1. Branches

| Branch | Purpose | Cut from | Merges into |
|---|---|---|---|
| `main` | Production. Always deployable. No tags. | n/a | n/a |
| `develop` | Staging. Integration of finished features. | n/a | `main` |
| `feature/*` | Development work. | `develop` | `develop` |
| `release/*` | Optional. Ships only the ready work when `develop` also holds unfinished work. | `main` | `main` |
| `sync/*` | Brings `main` back into `develop` after a release. | `main` | `develop` |
| `revert/*` | Removes a small amount of unfinished work from `develop`. | `develop` | `develop` |
| `hotfix/*` | Urgent production fix. | `main` | `main`, `develop`, and `release/*` if one is open |
| `resolve/*` | Resolves a conflict on a `develop → main` PR. | `develop` | `develop` |

Rules:
- `main` and `develop` accept changes only through pull requests. Nobody commits to them directly.
- Merge a conflict on a branch of its own (`resolve/*`, the `sync/*` branch, or a hotfix branch), never on `develop`.

```
Normal:   feature/* --squash--> develop --merge--> main
Release:  main --cut--> release/* <--cherry-pick -x-- develop (ready only)
          release/* --merge--> main --cut--> sync/* --merge--> develop
Revert:   develop --cut--> revert/* --squash--> develop --merge--> main
Hotfix:   main --cut--> hotfix/* --merge--> main
                            |-----merge--> develop
                            |-----merge--> release/*   (only if open)
Conflict: develop --cut--> resolve/* (merge origin/main) --merge--> develop
```

## 2. Merge methods

| Pull request | Method | Reason |
|---|---|---|
| `feature/*` → `develop` | Squash | One commit per feature. A single commit is easy to cherry-pick and to revert. |
| `revert/*` → `develop` | Squash | Same reason. |
| Everything else (`develop → main`, `release/* → main`, `sync/* → develop`, `hotfix/* → main`, `hotfix/* → develop`, `hotfix/* → release/*`, `resolve/* → develop`) | Merge commit | `main` and `develop` keep shared history. GitHub documents that a branch squash-merged and then continued shows its old commits again in later PRs and can conflict repeatedly. |

The repository must allow both squash merges and merge commits. Do not turn on "require linear history" for `main` or `develop`, because merge commits would fail.

## 3. Which path to take

| Situation | Path |
|---|---|
| Everything on `develop` is ready | `develop → main` directly. |
| Several items on `develop` are unfinished | Release branch: cut from `main`, cherry-pick only the ready commits, merge to `main`, then sync. |
| One or two items on `develop` are unfinished | Revert them on `develop`, then `develop → main`. |
| Urgent production bug | Hotfix branch. |

## 4. Rules shown by the tests

These rules come from the strategy decisions and from the scenarios in section 6.

1. **Always sync after a release.** After a release branch merges into `main`, open a `sync/*` PR from `main` into `develop`. Skipping the sync made the next `develop → main` PR conflict.
2. **Fix release bugs on `develop` first.** Merge the fix into `develop` through a PR, then cherry-pick it onto the release branch. A fix committed directly on the release branch made the sync PR conflict.
3. **Ship a dependent feature together with what it depends on.** A ready feature that edits an unfinished feature's file cannot be cherry-picked alone. The cherry-pick conflicts.
4. **Cherry-pick with `-x`.** It records the original commit in the message.
5. **Use one revert PR per feature.** That lets you undo each revert separately.
6. **Keep the original branch if you plan to continue a squashed feature.** Continuing it later gives a mergeable PR.
7. **A hotfix reaches `main` and `develop`.** If a release branch is open, it also goes into the release branch before the release merges.
8. **Resolve a conflicting hotfix → `develop` PR on a branch cut from the hotfix branch.** Never resolve it on `develop`.

## 5. Command cookbook

Replace names in `<angle brackets>`. Find a squash commit's SHA with `gh pr view <n> --json mergeCommit --jq .mergeCommit.oid`.

### 5.1 Normal flow

```
git fetch origin
git checkout -b feature/<name> origin/develop
# work and commit
git push -u origin feature/<name>
gh pr create --base develop --head feature/<name> --title "<title>"
gh pr merge <n> --squash --subject "<title>"

# ship everything on develop
gh pr create --base main --head develop --title "<title>"
gh pr merge <n> --merge
```

### 5.2 Release with only the ready work

```
git fetch origin
git checkout -b release/<id> origin/main
git cherry-pick -x <sha-of-ready-feature-1> <sha-of-ready-feature-2>
git push -u origin release/<id>
gh pr create --base main --head release/<id> --title "<title>"
gh pr merge <n> --merge

# sync main back into develop
git fetch origin
git checkout -b sync/<id> origin/main
git push -u origin sync/<id>
gh pr create --base develop --head sync/<id> --title "<title>"
gh pr merge <n> --merge
```

If the sync PR conflicts, merge `develop` into the sync branch, keep `main`'s version of the conflicted files, commit, push, and then merge the PR:

```
git merge origin/develop
# resolve, then
git add <files> && git commit
git push
```

### 5.3 Direct `develop → main` after unfinished work is finished

Use the normal flow in 5.1 after the sync from 5.2 is done.

If the sync was skipped and the PR conflicts:

```
git fetch origin
git checkout -b resolve/<id> origin/develop
git merge origin/main
# resolve, keeping develop's version, then
git add <files> && git commit
git push -u origin resolve/<id>
gh pr create --base develop --head resolve/<id> --title "<title>"
gh pr merge <n> --merge
# the original develop → main PR is now mergeable
```

### 5.4 Revert unfinished work on `develop`

One branch and one PR per feature:

```
git fetch origin
git checkout -b revert/<feature> origin/develop
git revert --no-commit <squash-sha-of-the-feature>
git commit -m "<title>" -m "Refs: <squash-sha-of-the-feature>"
git push -u origin revert/<feature>
gh pr create --base develop --head revert/<feature> --title "<title>"
gh pr merge <n> --squash --subject "<title>" --body "Refs: <squash-sha-of-the-feature>"
```

Then ship with `develop → main`.

To re-add the work, use either method:

- **Revert the revert.**
  ```
  git checkout -b feature/<feature>-reapply origin/develop
  git revert --no-commit <squash-sha-of-the-revert>
  git commit -m "<title>" -m "Refs: <squash-sha-of-the-revert>"
  ```
  Then push, open a PR to `develop`, and squash-merge it.
- **Continue the original branch.** This needs the branch kept after its first merge.
  ```
  git fetch origin
  git checkout -B feature/<feature> origin/feature/<feature>
  # more commits
  git push
  ```
  Then open a PR to `develop` and squash-merge it.

### 5.5 Hotfix

```
git fetch origin
git checkout -b hotfix/<name> origin/main
# fix and commit
git push -u origin hotfix/<name>

gh pr create --base main --head hotfix/<name> --title "<title>"
gh pr merge <n> --merge
gh pr create --base develop --head hotfix/<name> --title "<title>"
gh pr merge <n> --merge

# only if a release branch is open
gh pr create --base release/<id> --head hotfix/<name> --title "<title>"
gh pr merge <n> --merge
```

If the PR into `develop` conflicts:

```
gh pr close <n> --comment "Conflicts with develop; resolved on a separate branch."
git fetch origin
git checkout -b hotfix/<name>-develop origin/hotfix/<name>
git merge origin/develop
# resolve, then
git add <files> && git commit
git push -u origin hotfix/<name>-develop
gh pr create --base develop --head hotfix/<name>-develop --title "<title>"
gh pr merge <n> --merge
```

Delete temporary branches when they are no longer needed.

## 6. Scenarios covered

All scenarios ran on this repository in one run (`full1`, PRs #53 to #100). Each ended with `main` and `develop` carrying identical content, except where a release leaves unfinished work on `develop` by design (scenario 2).

| # | Scenario | What was done | Result |
|---|---|---|---|
| 1 | Normal flow | Two features squash-merged into `develop`, then one `develop → main` PR. | Both features reached `main`. Each feature is one commit on `develop`. |
| 2a | Release with only the ready work | Five features on `develop`: two ready, two unfinished, and one ready but dependent on an unfinished one. A release branch was cut from `main`, the ready commits were cherry-picked, and the release was merged and synced. | `main` held only the ready work. The sync was clean. |
| 2b | Dependent feature | The cherry-pick of the dependent feature was attempted alone. | It conflicted. The feature must ship with what it depends on. |
| 2c | Fix on the release branch | A fix was committed directly on the release branch, then merged and synced. | The sync PR conflicted. It was resolved on the `sync/*` branch. |
| 2d | Release fix on `develop` first | A fix to a ready feature was merged into `develop`, then cherry-picked onto the release. | Clean. |
| 3a | Direct `develop → main` | After the unfinished features were completed, `develop` was merged straight into `main`. | The PR was mergeable. `main` held all features. |
| 3b | Release without sync | A release skipped the sync, the shipped feature was extended on `develop`, and `develop → main` was opened. | The PR conflicted. A `resolve/*` branch fixed it. |
| 4a | Revert unfinished work | Two unfinished features were reverted through separate PRs, then `develop` was shipped. | `main` had the ready feature and not the reverted ones. |
| 4b | Re-add by reverting the revert | The revert commit was reverted in a new PR. | Mergeable. The original content returned. |
| 4c | Re-add by continuing the original branch | More commits were added to the kept feature branch and a new PR was opened. | Mergeable. Original and new content were both present. |
| 5a | Basic hotfix | Hotfix merged into `main`, then into `develop`. | Both branches got the fix. |
| 5b | Conflicting hotfix | `develop` had changed the same line. The hotfix PR into `develop` was opened. | It conflicted. It was resolved on a branch cut from the hotfix branch, and the next `develop → main` PR was mergeable. |
| 5c | Hotfix during an open release | Hotfix merged into `main`, the open release branch, and `develop`. The release was then merged and synced. | The release PR was mergeable. The hotfix change appeared once on `main` and `develop`. |
