# Git Context Snippets for Claude

A collection of ready-to-paste git commands that give Claude useful repo context at the start of a session or before a specific task. Copy the block that fits your situation, run it in your terminal, and paste the output into Claude.

---

## Session Kickoff

Use this at the start of any new Claude session to orient it in the repo.

```bash
echo "=== BRANCH ===" && git branch --show-current && \
echo "=== STATUS ===" && git status -s && \
echo "=== RECENT COMMITS ===" && git log --oneline -10 && \
echo "=== REMOTES ===" && git remote -v
```

**What Claude learns:** which branch you're on, any uncommitted changes, recent history, and the upstream remote.

---

## Before Starting a Feature

Run this before asking Claude to help implement something new. It shows the current state of main and your branching point.

```bash
echo "=== CURRENT BRANCH ===" && git branch --show-current && \
echo "=== DIVERGENCE FROM MAIN ===" && git log --oneline main..HEAD && \
echo "=== FILES CHANGED VS MAIN ===" && git diff --name-status main && \
echo "=== OPEN ISSUES ===" && gh issue list --limit 10
```

**What Claude learns:** how far your branch has drifted from main, which files are already modified, and what issues are open so it can target the right one.

---

## When Debugging a Problem

Give Claude maximum context about what changed recently and what the working tree looks like.

```bash
echo "=== LAST 5 COMMITS WITH DIFFS ===" && git log -5 --stat && \
echo "=== UNCOMMITTED CHANGES ===" && git diff && \
echo "=== STAGED CHANGES ===" && git diff --cached && \
echo "=== STASH LIST ===" && git stash list
```

**What Claude learns:** what you changed recently, what's staged vs unstaged, and whether anything is stashed that might be relevant.

---

## Before Creating a PR

Check that your branch is clean and give Claude a summary of everything going into the PR.

```bash
echo "=== BRANCH ===" && git branch --show-current && \
echo "=== COMMITS IN THIS BRANCH ===" && git log --oneline main..HEAD && \
echo "=== FILES CHANGED ===" && git diff --name-status main && \
echo "=== FULL DIFF STAT ===" && git diff --stat main && \
echo "=== OPEN PRs ===" && gh pr list
```

**What Claude learns:** the full scope of the PR — every commit and every file — so it can write an accurate PR title and body.

---

## Reference: Useful One-Liners

| Purpose | Command |
|---|---|
| Show current branch | `git branch --show-current` |
| Short status | `git status -s` |
| Last 10 commits (one line) | `git log --oneline -10` |
| Commits not yet in main | `git log --oneline main..HEAD` |
| Files changed vs main | `git diff --name-status main` |
| Full diff vs main | `git diff main` |
| Who changed a file | `git log --follow --oneline -- <file>` |
| When was a line introduced | `git log -S "<search string>" --oneline` |
| All branches with last commit | `git branch -v` |
| Upstream tracking info | `git status -sb` |
| Stash list | `git stash list` |
| Show a specific commit | `git show <sha>` |
| Files in a commit | `git show --name-only <sha>` |
| Open GitHub PR for branch | `gh pr view --web` |
| List issues | `gh issue list` |

---

## Tips

**Why paste output rather than ask Claude to run git?**
Claude can run git commands itself, but pasting the output upfront saves a round-trip and means Claude has the context before it starts responding — not after. This leads to better first answers.

**Keep it short when the answer is obvious.**
If you're just asking Claude to write a commit message, `git diff --cached` is enough. Don't dump a full diff of 500 lines when only 20 are relevant.

**Include the branch name explicitly.**
Claude can lose track of which branch you're on mid-conversation. Pasting `git branch --show-current` at the start of a new task resets this.

**Pipe long diffs through head when the context is too large.**
```bash
git diff main | head -200
```

**For firmware or embedded projects, add the build output too.**
If your build fails, paste the error alongside the git diff so Claude can correlate the two.
