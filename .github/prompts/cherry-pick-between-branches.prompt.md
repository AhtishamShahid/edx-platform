# Cherry-Pick Commits Between Branches

Reusable prompt for automated agents cherry-picking commits from a **source branch** into a **target branch**. Designed to be repository-agnostic.

---

## Inputs (provided by the user)

| Variable | Example | Description |
|----------|---------|-------------|
| `SOURCE_BRANCH` | `develop-teak.3` | Branch containing the commits to cherry-pick |
| `TARGET_BRANCH` | `release/ulmo` | Branch that will receive the cherry-picks |
| `CONFLICT_STRATEGY` | `theirs` | How to resolve conflicts: `theirs` (prefer source), `ours` (prefer target), or `manual` |

---

## Step-by-step procedure

### 1. Set up the working branch

```bash
git fetch origin $TARGET_BRANCH $SOURCE_BRANCH
git checkout -b cherry-pick/$SOURCE_BRANCH-into-$TARGET_BRANCH origin/$TARGET_BRANCH
```

> **CRITICAL — Always branch off `origin/$TARGET_BRANCH`.**
> Never branch from `master`, `main`, or any other branch. The new branch
> must share history *only* with the target branch so the PR diff is clean.

### 2. Identify commits to cherry-pick

```bash
# List commits on SOURCE that are NOT on TARGET (oldest first)
git log --oneline --reverse origin/$TARGET_BRANCH..origin/$SOURCE_BRANCH
```

Before cherry-picking, **filter out** commits that should NOT be applied:

| Filter | Command | Why |
|--------|---------|-----|
| **Merge commits** | Exclude lines with more than one parent (`git log --no-merges ...`) | Merge commits bring in entire branch histories and cannot be cleanly cherry-picked. |
| **Already-present changes** | After cherry-picking, if `git cherry-pick` produces an empty commit, skip it with `git cherry-pick --skip`. | Changes may already be in the target branch via a different commit (e.g., a rebase or earlier backport). |

**Recommended: build the filtered list first, then cherry-pick in one pass.**

```bash
# Get non-merge commits only
COMMITS=$(git log --oneline --reverse --no-merges origin/$TARGET_BRANCH..origin/$SOURCE_BRANCH | awk '{print $1}')
```

### 3. Cherry-pick each commit one by one

```bash
for COMMIT in $COMMITS; do
  echo "Cherry-picking $COMMIT ..."
  if ! git cherry-pick "$COMMIT" --no-commit; then
    # Conflict occurred — resolve it
    # (see "Conflict resolution" section below)
  fi

  # Check if the cherry-pick produced any changes
  if git diff --cached --quiet; then
    echo "SKIP: $COMMIT is empty (already in target)"
    git cherry-pick --skip 2>/dev/null || git reset
    continue
  fi

  git commit -C "$COMMIT"  # reuse original commit message
done
```

> **NEVER use `git merge` to resolve divergence or conflicts.**
> `git merge --allow-unrelated-histories` or any merge variant will create
> a merge commit with two parents, pulling hundreds of unrelated commits
> into the PR. This was the #1 mistake in previous attempts.

### 4. Conflict resolution

When a cherry-pick conflicts:

**If `CONFLICT_STRATEGY=theirs` (prefer source branch):**
```bash
git checkout --theirs .
git add .
git commit -C "$COMMIT"
```

**If `CONFLICT_STRATEGY=ours` (prefer target branch):**
```bash
git checkout --ours .
git add .
git commit -C "$COMMIT"
```

**If `CONFLICT_STRATEGY=manual`:**
- List conflicted files: `git diff --name-only --diff-filter=U`
- Resolve each file individually
- `git add <file>` after resolving
- `git commit -C "$COMMIT"`

> **Record every conflict resolution.** Maintain a table of:
> `| Commit | Conflicted files | Resolution | Rationale |`
> This is essential for reviewers.

### 5. Post-cherry-pick review (CRITICAL)

After all cherry-picks are applied, review and fix branch-specific configuration:

| What to check | Why | Example |
|---------------|-----|---------|
| **Release identifiers** | Source branch may have a different release name | `RELEASE_LINE = "teak"` must be reverted to `"ulmo"` on the ulmo branch |
| **Version pins / requirements files** | Dependency versions may differ between releases | `requirements/constraints.txt`, `setup.cfg`, `package.json` |
| **CI/CD configuration** | Workflow triggers or environment variables may be branch-specific | `.github/workflows/*.yml`, `Makefile` |
| **Feature flags / waffle switches** | Source may enable features not ready for target release | Django settings, waffle flag definitions |
| **Branch-name references in code** | Hardcoded branch names in scripts or configs | Deployment scripts, documentation links |

```bash
# Quick check for the most common issue — release line:
git diff origin/$TARGET_BRANCH -- '**/release.py' '**/version.py' '**/release_line*'
```

**Fix any branch-specific values before pushing.** This was a known mistake:
cherry-picking a commit that changed `RELEASE_LINE` from the target release
name to the source release name went unnoticed and would have broken the
target branch.

### 6. Push and create the PR

```bash
git push origin cherry-pick/$SOURCE_BRANCH-into-$TARGET_BRANCH
```

When creating the PR:

- **Set the base branch to `$TARGET_BRANCH`** — NOT `master` or `main`.
  Agents may not be able to change the base branch after PR creation, so
  get this right the first time.
- Use a descriptive title:
  `chore: cherry-pick $SOURCE_BRANCH into $TARGET_BRANCH`
- Include the summary tables (applied, skipped, conflicts) in the PR body.

### 7. Write the PR description

Include these sections in the PR body:

1. **Summary** — one-line description of what was cherry-picked and why.
2. **Applied commits table** — `| # | SHA | Subject |`
3. **Skipped commits table** — `| SHA | Subject | Reason |`
4. **Conflict resolutions table** — `| Commit | Files | Strategy | Rationale |`
5. **Post-cherry-pick fixes** — any branch-specific config that was reverted.
6. **Testing instructions** — which areas to test based on the changes.

---

## Mistakes to avoid (lessons learned)

These are real mistakes from previous cherry-pick sessions. **Do not repeat them.**

### ❌ DO NOT use `git merge` to fix push divergence

**What happened:** The agent used `git merge --allow-unrelated-histories -s ours`
to resolve a push divergence. This created a merge commit with two parents — one
from the cherry-pick chain and one from an unrelated branch. The PR then showed
~200 commits instead of the expected 38.

**Fix:** If your local branch diverges from the remote, use
`git reset --hard origin/<branch>` and redo the work. If force-push is not
available, start over on a new branch. NEVER merge to fix divergence.

### ❌ DO NOT open the PR against the wrong base branch

**What happened:** The PR was opened against `master` instead of `release/ulmo`.
The agent could not change the base branch after creation due to API/firewall
restrictions.

**Fix:** Always specify the correct base branch when creating the PR. Double-check
before creating. If using `gh pr create`, use `--base $TARGET_BRANCH`.

### ❌ DO NOT forget to revert branch-specific configuration

**What happened:** A cherry-picked commit changed `RELEASE_LINE` from `"ulmo"` to
`"teak"`. This was not caught because all conflicts were resolved with `--theirs`
(preferring source).

**Fix:** After cherry-picking, always run the post-cherry-pick review (Step 5).
Diff critical config files against the target branch.

### ❌ DO NOT cherry-pick merge commits

**What happened:** Merge commits were included in the initial commit list. They
cannot be cleanly cherry-picked and bring in entire branch histories.

**Fix:** Always use `git log --no-merges` when building the commit list.

### ❌ DO NOT skip recording conflict resolutions

**What happened:** Conflicts were resolved in bulk with `--theirs` but not
individually documented. Reviewers had no way to verify correctness.

**Fix:** Record every conflict: which commit, which files, what strategy was used,
and why.

### ❌ DO NOT create situations requiring force-push

**What happened:** A bad merge commit required force-push to remove, but the
agent sandbox does not support force-push. A second PR was needed.

**Fix:** Be extremely careful with git operations. Preview every command before
running. If unsure, create a test branch first. Prefer starting over on a new
branch rather than trying to fix a broken history.

---

## Quick-reference checklist

Use this checklist before, during, and after the cherry-pick operation:

### Before starting
- [ ] Confirmed source and target branches exist
- [ ] Built the filtered commit list (`--no-merges`, oldest first)
- [ ] Confirmed conflict resolution strategy with the user

### During cherry-picking
- [ ] Each commit cherry-picked individually (no `git merge`)
- [ ] Empty cherry-picks skipped and recorded
- [ ] Each conflict resolution recorded with files and rationale

### After cherry-picking
- [ ] Reviewed and reverted branch-specific config (release line, versions, flags)
- [ ] Diff against target branch looks correct (`git diff origin/$TARGET_BRANCH --stat`)
- [ ] PR created with correct base branch (`$TARGET_BRANCH`)
- [ ] PR description includes applied/skipped/conflict tables
- [ ] No merge commits in the branch history

---

## Example usage

> **User prompt:**
> Cherry-pick all commits from `develop-teak.3` into `release/ulmo`.
> Resolve conflicts by preferring `develop-teak.3`.

The agent should follow Steps 1–7 above with:
- `SOURCE_BRANCH=develop-teak.3`
- `TARGET_BRANCH=release/ulmo`
- `CONFLICT_STRATEGY=theirs`
