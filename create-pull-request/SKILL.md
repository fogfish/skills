---
name: create-pull-request
description: Create a GitHub pull request following project conventions. Use when the user asks to create a PR, submit changes for review, open a pull request, open a stacked PR (a PR based on another feature branch), or restack/retarget a PR chain. Works from the main checkout or from any git worktree. Handles commit analysis, branch management, stack detection, PR template usage, and PR creation using the gh CLI tool.
model: haiku
---

# Create Pull Request

This skill guides you through creating a well-structured GitHub pull request that follows project conventions and best practices. It supports:

- **Plain PRs** — a feature branch targeting `main`.
- **Stacked PRs** — a feature branch targeting another feature branch (the *parent*), forming a chain `main ← A ← B ← C`.
- **Worktrees** — the PR branch may be checked out in a linked worktree (`git worktree add ...`), possibly with other branches of the same stack checked out in sibling worktrees.

Throughout this skill, the remote is called `github`. If `git remote` shows no `github` remote, use `origin` (or the single remote that exists) instead, everywhere.

## Prerequisites Check

Before proceeding, verify the following:

### 1. Check if `gh` CLI is installed

```bash
gh --version
```

If not installed, inform the user:
> The GitHub CLI (`gh`) is required but not installed. Please install it:
> - macOS: `brew install gh`
> - Other: https://cli.github.com/

### 2. Check if authenticated with GitHub

```bash
gh auth status
```

If not authenticated, guide the user to run `gh auth login`.

### 3. Detect worktree context

```bash
git rev-parse --show-toplevel
git rev-parse --git-dir
git rev-parse --git-common-dir
git worktree list
```

- If `--git-dir` and `--git-common-dir` differ, you are in a **linked worktree**. Run every git and `gh` command from this worktree's top-level directory (`--show-toplevel`), not from the main checkout.
- `git worktree list` shows which branch is checked out where. Remember it: a branch checked out in another worktree **cannot** be checked out here, and must be rebased/edited from its own worktree.
- Repository files such as `.github/pull_request_template.md` are read from the current worktree's top-level directory.

### 4. Verify clean working directory

```bash
git status
```

This checks only the current worktree. If there are uncommitted changes, ask the user whether to:
- Commit them as part of this PR
- Stash them temporarily (note: the stash is shared by all worktrees of the repository)
- Discard them (with caution)

## Gather Context

### 1. Identify the current branch

```bash
git branch --show-current
```

- If empty, the worktree is on a detached HEAD (common right after `git worktree add --detach`). Ask the user for a branch name and create it: `git switch -c <branch>`.
- Ensure you're not on `main`. If so, ask the user to create or switch to a feature branch.

### 2. Find the default branch

```bash
git fetch github
git remote show github | grep "HEAD branch"
```

This is typically `main`. Below, `main` means this default branch.

### 3. Determine the base branch (plain or stacked)

The base of the PR is the **nearest ancestor branch** of the current branch. Use the user's explicit instruction if given (e.g. "stack this on `feature/a`"). Otherwise detect it:

```bash
cur=$(git branch --show-current)
for b in $(git for-each-ref --format='%(refname:short)' refs/heads); do
  [ "$b" = "$cur" ] && continue
  git merge-base --is-ancestor "$b" HEAD || continue
  n=$(git rev-list --count "$b..HEAD")
  [ "$n" -gt 0 ] && echo "$n $b"
done | sort -n | head -5
```

The branch with the smallest count is the candidate parent.

- If the candidate is `main` (or no other branch qualifies), this is a **plain PR**: `BASE=main`.
- If the candidate is another feature branch, this is a **stacked PR**: `BASE=<that branch>`. Confirm with the user if there are ties or if it is not obvious, e.g.:
  > This branch is built on top of `feature/a` (3 commits ahead). Should I open it as a stacked PR targeting `feature/a`, or target `main`?
- Check that the parent has a PR and is pushed:
  ```bash
  gh pr list --head "$BASE" --state all --json number,state,url,baseRefName
  ```
  - **Open PR**: good, record its number and its own base (to build the whole chain).
  - **No PR**: offer to create the parent's PR first (run this skill for the parent branch, from the worktree where it is checked out), or to stack on it anyway once it is pushed.
  - **Merged PR**: the parent is already in `main`. Use `BASE=main` and restack first (see *Maintaining a Stack → Parent was merged*).

Walk up the chain (parent's base, its base, …) until you reach `main`, so you know the complete stack, bottom to top. Also look for children — open PRs whose base is the current branch:

```bash
gh pr list --base "$cur" --state open --json number,headRefName,url
```

### 4. Analyze commits relevant to this PR

Only the commits between the base and HEAD belong to this PR:

```bash
git log github/$BASE..HEAD --oneline --no-decorate
```

If `github/$BASE` does not exist yet (the parent was never pushed), use the local `$BASE` and push the parent before creating the PR.

Review these commits to understand:
- What changes are being introduced
- The scope of the PR (single feature/fix or multiple changes)
- Whether commits should be squashed or reorganized

For a stacked PR, if commits that belong to the parent show up here, the branch is out of date with its parent — restack before continuing.

### 5. Review the diff

```bash
git diff github/$BASE...HEAD --stat
```

This shows which files changed (relative to the base, not to `main`) and helps identify the type of change.

## Information Gathering

Before creating the PR, you need the following information. Check if it can be inferred from:
- Commit messages
- Branch name (e.g., `fix/issue-123`, `feature/new-login`)
- Changed files and their content
- For stacked PRs: the parent PR's description and issue


If any critical information is missing, ask the user:

### Required Information

1. **Related Issue Number**: Look for patterns like `#123`, `fixes #123`, or `closes #123` in commit messages. If no issue number is found but repository uses specification-driven development, add reference to the current spec. For stacked PRs, the issue is often the same as the parent's.
2. **Description**: What problem does this solve? Why were these changes made?
3. **Type of Change**: Bug fix, new feature, breaking change, refactor, cosmetic, documentation, or workflow
4. **Test Procedure**: How was this tested? What could break?



### Example clarifying question

If the issue number is not found:
> I couldn't find a related issue number in the commit messages or branch name. What GitHub issue does this PR address? (Enter the issue number, e.g., "123" or "N/A" for small fixes)

## Git Best Practices

Before creating the PR, consider these best practices:

### Commit Hygiene

1. **Atomic commits**: Each commit should represent a single logical change
2. **Clear commit messages**: Follow conventional commit format when possible
3. **No merge commits**: Prefer rebasing over merging to keep history clean
4. **Small stack layers**: In a stack, each PR should be reviewable on its own; don't move commits between layers without asking

### Branch Management

1. **Rebase on the latest base** (if needed):
   ```bash
   git fetch github
   git rebase github/$BASE
   ```
   For a stacked PR, rebase on the parent branch, never directly on `main` — that would pull the parent's commits into this PR.

2. **Squash if appropriate**: If there are many small "WIP" commits, suggest squashing them. Interactive rebase (`git rebase -i`) is not available in this environment; use a non-interactive equivalent instead:
   ```bash
   git reset --soft github/$BASE
   git commit
   ```
   Only suggest this if commits appear messy and the user agrees.

### Push Changes

Push the parent branches first (bottom of the stack first), then this branch:
```bash
git push -u github HEAD
```

If the branch was rebased, you may need:
```bash
git push github HEAD --force-with-lease
```

## Create the Pull Request

**IMPORTANT**: Read and use the PR template at `.github/pull_request_template.md` (in the current worktree's top-level directory). The PR body format must **strictly match** the template structure. Do not deviate from the template format. If the template is missing, use default `pull_request_template.md` attached in the skill.

When filling out the template:
- Replace `#XXXX` with the actual issue number, or keep as `#XXXX` if no issue exists (for small fixes)
- Fill in all sections with relevant information gathered from commits and context
- Mark the appropriate "Type of Change" checkbox(es)
- Complete the "Pre-flight Checklist" items that apply

### Stack information in the PR body

For a stacked PR, add the stack to the section that links related issues/PRs (`Related Issue(s)` in the default template) — do not add new sections. List the whole chain bottom to top and mark the current PR:

```markdown
Stack (merge bottom to top):
1. #101 feature/a — base `main`
2. #102 feature/b — base #101
3. 👉 this PR — base #102
```

Also state at the start of the description that the PR is stacked, e.g. "Stacked on #102 — review only the commits in this PR."

### Create PR with gh CLI

**Use a temporary file for the PR body** to avoid shell escaping issues, newline problems, and other command-line flakiness. Create it with `mktemp` so PRs prepared in parallel worktrees never overwrite each other's body:

1. Create a unique temporary file and write the PR body into it:
   ```bash
   BODY=$(mktemp "${TMPDIR:-/tmp}/pr-body.XXXXXX")
   ```

2. Create the PR using the file, with the base determined above:
   ```bash
   gh pr create --title "PR_TITLE" --body-file "$BODY" --base "$BASE" --head "$(git branch --show-current)"
   ```

3. Clean up the temporary file:
   ```bash
   rm "$BODY"
   ```

For draft PRs (recommended for upper layers of a stack whose parent is still under review, if the user wants):
```bash
gh pr create --title "PR_TITLE" --body-file "$BODY" --base "$BASE" --draft
```

**Why use a file?** Passing complex markdown with newlines, special characters, and checkboxes directly via `--body` is error-prone. The `--body-file` flag handles all content reliably.

### Update the rest of the stack

After creating a stacked PR, add it to the stack list in the other open PRs of the chain, so every PR shows the full stack:

```bash
gh pr view <N> --json body --jq .body > "$BODY"
# edit only the stack list in $BODY
gh pr edit <N> --body-file "$BODY"
```

Show the user the changes before editing PRs other than the one just created.

## Maintaining a Stack

Use these procedures when the user asks to restack, update, or retarget PRs, or when the checks above find the stack out of date. Each branch must be rebased from the worktree where it is checked out (see `git worktree list`); a branch not checked out anywhere can be rebased from the current worktree.

### Parent got new commits (e.g. review fixes)

Rebase each child onto its updated parent, bottom to top:

```bash
git fetch github
git rebase github/<parent>
git push github HEAD --force-with-lease
```

When the whole stack lives in one worktree, a single rebase of the top branch can move all layers at once:

```bash
git rebase --update-refs github/main
```

`--update-refs` skips branches that are checked out in other worktrees — rebase those from their own worktrees instead.

### Parent was merged

1. Retarget the child PR to the parent's base (usually `main`) **before** the parent branch is deleted, otherwise GitHub closes the child PR:
   ```bash
   gh pr edit <child> --base main
   ```
2. If the parent was squash- or rebase-merged, its commits are not ancestors of `main`. Replay only the child's own commits:
   ```bash
   git fetch github
   git rebase --onto github/main <old-parent-tip> <child-branch>
   git push github <child-branch> --force-with-lease
   ```
   `<old-parent-tip>` is the commit the child was built on (`git merge-base <child-branch> <parent-branch>` while the parent branch still exists locally).
3. Update the stack list in the remaining PRs' bodies.
4. Offer to clean up the merged parent: `git worktree remove <path>` for its worktree (if any), then `git branch -d <parent>`.

## Post-Creation

After creating the PR:

1. **Display the PR URL** so the user can review it. For stacks, show the whole chain with URLs.
2. **Remind about CI checks**: Tests and linting will run automatically (note that some CI workflows only run on PRs targeting `main`)
3. **Suggest next steps**:
   - Add reviewers if needed: `gh pr edit --add-reviewer USERNAME`
   - Add labels if needed: `gh pr edit --add-label "bug"`
   - For stacks: merge bottom to top, and retarget each child before deleting the merged parent branch

## Error Handling

### Common Issues

1. **No commits ahead of base**: The branch has no changes to submit
   - Ask if the user meant to work on a different branch, or if the base was detected wrongly

2. **Branch not pushed**: Remote doesn't have the branch (or the parent of a stacked PR)
   - Push it first: `git push -u github HEAD`, parents before children

3. **PR already exists**: A PR for this branch already exists
   - Show the existing PR: `gh pr view`
   - Ask if they want to update it instead (including its base: `gh pr edit --base <branch>`)

4. **Merge conflicts**: Branch conflicts with base
   - Guide user through resolving conflicts or rebasing onto the base branch

5. **`fatal: '<branch>' is already checked out at '<path>'`**: The branch is in use by another worktree
   - Don't force it; run the command from that worktree (`git -C <path> ...`) or ask the user

6. **Stale worktree entries**: `git worktree list` shows paths that no longer exist
   - Suggest `git worktree prune`

## Summary Checklist

Before finalizing, ensure:
- [ ] `gh` CLI is installed and authenticated
- [ ] Commands run from the correct worktree, which is clean
- [ ] Base branch is correct (`main`, or the parent branch for a stacked PR)
- [ ] Parent branches are pushed and have PRs (for stacked PRs)
- [ ] All commits are pushed
- [ ] Branch is up-to-date with base branch
- [ ] Related issue number is identified, or placeholder is used
- [ ] PR description follows the template exactly, with the stack listed for stacked PRs
- [ ] Other PRs of the stack are updated with the new stack list
- [ ] Appropriate type of change is selected
- [ ] Pre-flight checklist items are addressed
