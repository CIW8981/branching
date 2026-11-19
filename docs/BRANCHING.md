# Git branching strategy

This guide describes the branching strategy the team uses for multiple deployment environments. It also lists the scenarios that were tested on this repository and what happened in each.

## 1. Branches

| Branch | Purpose | Cut from | Merges into |
|---|---|---|---|
| `main` | Production. Always deployable. No tags. | n/a | n/a |
| `develop` | Staging. Integration of finished features. | n/a | `main` |
| `feature/*` | Development work. | `develop` | `develop` |
| `release/*` | Optional. Ships only the ready work when `develop` also holds unfinished work. | `main` | `main` |
| `sync/*` | Brings `main` back into `develop` after anything lands on `main` (a release or a hotfix). | `main` | `develop` |
| `revert/*` | Removes a small amount of unfinished work from `develop`. | `develop` | `develop` |
| `hotfix/*` | Urgent production fix. | `main` | `main` only |
| `resolve/*` | Resolves a conflict on a `develop → main` PR. | `develop` | `develop` |

Rules:
- `main` and `develop` accept changes only through pull requests. Nobody commits to them directly.
- Resolve a conflict on a branch of its own (`resolve/*` or the `sync/*` branch), never on `develop`.
- Always bring `main` back into `develop` through a `sync/*` branch, whether or not there is a conflict. A direct `main → develop` PR is also an accepted practice and a matter of preference. The team uses `sync/*` every time so nobody has to decide which to use, and a conflict can be resolved on the branch without committing to `main` or `develop`.
- A release branch is always cut from the latest `main`. If `main` changes while a release PR is open, close it and cut a new release branch.

```
Normal:   feature/* --squash--> develop --merge--> main
Release:  main --cut--> release/* <--cherry-pick -x-- develop (ready only)
          release/* --merge--> main --cut--> sync/* --merge--> develop
Revert:   develop --cut--> revert/* --squash--> develop --merge--> main
Hotfix:   main --cut--> hotfix/* --merge--> main --cut--> sync/* --merge--> develop
          open release PR: close it, cut a new release/* from main, cherry-pick again
Conflict: develop --cut--> resolve/* (merge origin/main) --merge--> develop
```

## 2. Merge methods

| Pull request | Method | Reason |
|---|---|---|
| `feature/*` → `develop` | Squash | One commit per feature. A single commit is easy to cherry-pick and to revert. |
| `revert/*` → `develop` | Squash | Same reason. |
| Everything else (`develop → main`, `release/* → main`, `hotfix/* → main`, `sync/* → develop`, `resolve/* → develop`) | Merge commit | `main` and `develop` keep shared history. GitHub documents that a branch squash-merged and then continued shows its old commits again in later PRs and can conflict repeatedly. |

The repository must allow both squash merges and merge commits. Do not turn on "require linear history" for `main` or `develop`, because merge commits would fail. "Automatically delete head branches" can be on or off. No flow reuses a branch after its PR merges. The one exception is continuing a reverted feature on its original branch (section 5.4). If auto-delete is on, re-add reverted work with the revert-the-revert method instead.

## 3. Which path to take

| Situation | Path |
|---|---|
| Everything on `develop` is ready | `develop → main` directly. |
| Several items on `develop` are unfinished | Release branch: cut from `main`, cherry-pick only the ready commits, merge to `main`, then sync. |
| One or two items on `develop` are unfinished | Revert them on `develop`, then `develop → main`. |
| Urgent production bug | Hotfix into `main`, then sync into `develop`. |
| `main` changed while a release PR is open | Close the release PR and cut a new release branch from `main` with the same cherry-picks. |

## 4. Rules shown by the tests

These rules come from the strategy decisions and from the scenarios in section 6.

1. **Always sync after anything lands on `main`.** After a release or a hotfix merges into `main`, open a `sync/*` PR from `main` into `develop`, using a merge commit. Skipping the sync made the next `develop → main` PR conflict.
2. **Fix release bugs on `develop` first.** Merge the fix into `develop` through a PR, then cherry-pick it onto the release branch. A fix committed directly on the release branch made the sync PR conflict, and it would be lost if the release branch had to be rebuilt.
3. **Ship a dependent feature together with what it depends on.** A ready feature that edits an unfinished feature's file cannot be cherry-picked alone. The cherry-pick conflicts.
4. **Cherry-pick with `-x`, and list the cherry-picked SHAs in the release PR description.** `-x` records the original commit in the message. The list lets anyone rebuild the release branch.
5. **Use one revert PR per feature.** That lets you undo each revert separately.
6. **Keep the original branch if you plan to continue a squashed feature.** Continuing it later gives a mergeable PR.
7. **A hotfix merges into `main` only.** `develop` gets it through the sync. The hotfix branch is not needed after its merge, so auto-delete is safe.
8. **If `main` changes while a release PR is open, rebuild the release.** Close the PR, delete the branch, cut a new `release/*` from the latest `main`, and cherry-pick the same commits.

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
gh pr create --base main --head release/<id> --title "<title>" \
  --body "Cherry-picks: <sha-of-ready-feature-1> <sha-of-ready-feature-2>"
gh pr merge <n> --merge
```

Then sync, as in 5.6.

### 5.3 Direct `develop → main` after unfinished work is finished

Use the normal flow in 5.1 after the sync from 5.6 is done.

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
```

Then sync, as in 5.6. If a release PR is open, rebuild it, as in 5.7.

### 5.6 Sync `main` into `develop`

Run this after every release or hotfix merge into `main`.

```
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

### 5.7 Rebuild an open release after `main` changed

```
gh pr close <old-release-pr> --comment "main changed. Rebuilding the release from the latest main." --delete-branch
git fetch origin
git checkout -b release/<new-id> origin/main
git cherry-pick -x <same SHAs as the closed release PR>
git push -u origin release/<new-id>
gh pr create --base main --head release/<new-id> --title "<title>" --body "Cherry-picks: <SHAs>"
gh pr merge <n> --merge
```

Then sync, as in 5.6.

Delete temporary branches when they are no longer needed.

## 6. Why this strategy: scenarios tested

All scenarios ran on this repository and behaved as described.

### Scenario 1: `develop` has ready and finished work
- **Problem:** Everything on `develop` is ready and finished. How does it reach production?
- **Strategy:** Features squash into `develop` (staging). `develop` then merges directly into `main` (production) with a merge commit. No release branch is needed.
- **Tested:** All features reached `main`, and each feature is one commit on `develop`.

### Scenario 2: `develop` has ready and unfinished work
- **Problem:** `develop` holds `profile` and `search` (ready) and `payment` and `invoice` (unfinished). Merging `develop` into `main` would ship all four.
- **Strategy:** Cut `release/*` from `main`, because `main` has none of the unfinished work. Cherry-pick only the ready commits, then merge the release into `main`.
- **Tested:** `main` got `profile` and `search`. `payment` and `invoice` stayed on `develop`.
- **Limit found:** `receipt` is ready but edits `payment`'s file. Cherry-picking it alone conflicts, so it ships together with `payment`.

### Scenario 3: `develop` must get the release back
- **Problem:** After the release, `main` holds commits that `develop` does not share history with.
- **Strategy:** A `sync/*` PR merges `main` into `develop` after every release.
- **Tested:** The sync was clean. When we skipped it and then changed a cherry-picked file on `develop`, the next `develop → main` PR conflicted. A `resolve/*` branch fixed it.

### Scenario 4: A bug is found during the release
- **Problem:** The fix has to land on the release and on `develop`.
- **Strategy:** Fix it on `develop` first, then cherry-pick the fix onto the release branch.
- **Tested:** That path was clean. A fix committed directly on the release branch made the sync PR conflict, and we resolved it on the `sync/*` branch.

### Scenario 5: The unfinished work is finished later
- **Problem:** After the release, `payment` and `invoice` are finished on `develop`. Do we need another release branch?
- **Strategy:** No. Once everything on `develop` is ready, merge `develop` into `main` directly, as in scenario 1.
- **Tested:** The PR was mergeable because the sync was done, and `main` held all features.

### Scenario 6: Only one or two items are unfinished
- **Problem:** A release branch is too much work for one or two items.
- **Strategy:** Revert them on `develop`, one PR each, then ship `develop`.
- **Tested:** `main` got the ready work and none of the reverted work. The work came back two ways: reverting the revert, or adding commits to the original feature branch (kept after its first merge) and opening a new PR. Both worked.

### Scenario 7: Urgent bug in production
- **Problem:** The fix cannot wait for `develop`, and `develop` must not lose it.
- **Strategy:** `hotfix/*` from `main` merges into `main` only. A `sync/*` PR then brings it into `develop`, the same step as after a release.
- **Tested:** With "Automatically delete head branches" on, the hotfix branch was deleted after its merge, and the sync still brought the fix into `develop`. The sync was clean, and unfinished work on `develop` stayed off `main`.

### Scenario 8: Urgent bug while a release is open
- **Problem:** The open release branch was cut from the old `main`, so it does not contain the fix.
- **Strategy:** Close the open release PR and cut a new `release/*` from the latest `main`, which already has the fix. Cherry-pick the same commits again, merge, then sync.
- **Tested:** The new release branch contained the hotfix, the same commit cherry-picked cleanly, and the release PR was mergeable. After the sync, the hotfix change appeared once on `main` and once on `develop`.
