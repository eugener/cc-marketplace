---
description: Audit git repository hygiene - large blobs, leaked secrets, stale branches, missing gitignore entries, and overall health
user-invocable: true
allowed-tools:
  - Bash
  - Grep
  - Glob
  - Read
  - AskUserQuestion
---

Run a comprehensive git repository health audit on the current repo. Present findings as a report with actionable fixes. Ask before making any changes.

## Step 1: Basic Repo Info

Run in parallel:
- `git rev-list --count HEAD` (total commits)
- `git branch -a` (all branches)
- `du -sh .git` (git directory size)
- `git log --oneline -1` (last commit)
- `git remote -v` (remotes)

## Step 2: Large Blobs

Find the 10 largest blobs in history:
```
git rev-list --objects --all | git cat-file --batch-check='%(objecttype) %(objectname) %(objectsize) %(rest)' | awk '/^blob/ {print $3, $4, $2}' | sort -rn | head -10
```

Flag anything over 1 MB. For files over 5 MB, strongly recommend removal via bfg or git filter-repo.

## Step 3: Secrets Scan

Search the working tree for potential secrets using patterns:
- Files: `.env`, `.env.*` (not `.env.example`), `*.pem`, `*.key`, `credentials.json`, `service-account*.json`
- Content patterns in tracked files:
  - `AKIA[0-9A-Z]{16}` (AWS access keys)
  - `sk-[a-zA-Z0-9]{20,}` (API keys)
  - `-----BEGIN (RSA |EC |DSA )?PRIVATE KEY-----`
  - `password\s*=\s*['"][^'"]+['"]` (hardcoded passwords)
  - `ghp_[a-zA-Z0-9]{36}` (GitHub tokens)
  - `xox[bpors]-[a-zA-Z0-9-]+` (Slack tokens)

Also check if `.gitignore` contains entries for `.env`, `*.pem`, `*.key`. Flag if missing.

## Step 4: Stale Branches

For each local branch:
- `git log -1 --format="%ai %s" <branch>` to get last commit date
- Flag branches with no commits in 3+ months as stale
- Flag branches marked as `[gone]` on remote (merged/deleted upstream)

For remote branches:
- Flag remote tracking branches with no activity in 6+ months

## Step 5: Gitignore Audit

Detect project type from manifest files and check `.gitignore` for common missing entries:

- **Node/TS**: `node_modules/`, `dist/`, `.env`, `*.tsbuildinfo`, `.DS_Store`
- **Rust**: `target/`, `Cargo.lock` (if library)
- **Go**: vendor/ (if not vendored), binary output
- **Java**: `target/`, `*.class`, `.idea/`, `*.jar`
- **Python**: `__pycache__/`, `*.pyc`, `.venv/`, `dist/`
- **General**: `.DS_Store`, `*.swp`, `.env`, `*.log`, `.idea/`, `.vscode/` (debatable)

Check for tracked files that should be ignored (e.g., `node_modules` or `.env` already committed).

## Step 6: Commit Hygiene

Analyze last 50 commits:
- Flag empty commit messages
- Flag commits with messages under 10 characters
- Check for merge commits vs rebase workflow
- Check for consistent conventional commit format (`feat:`, `fix:`, etc.)
- Report commit frequency (commits per week over last month)

## Step 7: Uncommitted Work

- `git status --porcelain` for dirty working tree
- `git stash list` for forgotten stashes
- Flag stashes older than 1 month

## Step 8: Present Report

Organize into sections with severity:

1. **Critical** - leaked secrets, large binaries in history
2. **Warning** - stale branches, missing gitignore entries, old stashes
3. **Info** - repo size, commit stats, branch summary

For each finding, provide the exact fix command.

End with a summary of recommended actions, grouped as:
- **Immediate**: security-related fixes
- **Cleanup**: branch/stash cleanup commands
- **Improvement**: gitignore/commit hygiene suggestions

Ask which actions the user wants to take.
