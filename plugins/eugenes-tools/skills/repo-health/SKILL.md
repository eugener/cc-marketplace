---
description: Audit git repository hygiene - large blobs, leaked secrets, stale branches, missing gitignore entries, commit quality, and overall health. Use this when the user asks about repo cleanup, git health check, secret scanning, branch cleanup, or git hygiene.
user-invocable: true
allowed-tools:
  - Bash
  - Grep
  - Glob
  - Read
  - AskUserQuestion
---

Run a comprehensive git repository health audit on the current repo. Present findings as a report with actionable fixes. Ask before making any changes.

## Step 0: Detect Platform

Run `uname -s` to detect OS. Use this throughout:
- **macOS**: `stat -f "%Sa" -t "%Y-%m-%d"` for file timestamps, `date -v-Xm` for date math
- **Linux**: `stat -c "%y"` for file timestamps, `date -d 'X months ago'` for date math

## Step 1: Basic Repo Info

Run in parallel:
- `git rev-list --count HEAD` (total commits)
- `git branch | wc -l` (local branch count)
- `git branch -r | wc -l` (remote branch count)
- `du -sh .git` (git directory size)
- `git log --oneline -1` (last commit)
- `git remote -v` (remotes)

## Step 2: Large Blobs

Find the 10 largest blobs in history. Limit rev-list to avoid hanging on huge repos:
```
git rev-list --objects --all | head -100000 | git cat-file --batch-check='%(objecttype) %(objectname) %(objectsize) %(rest)' | awk '/^blob/ {print $3, $4, $2}' | sort -rn | head -10
```

Flag anything over 1 MB. For files over 5 MB, strongly recommend removal via bfg or git filter-repo.

## Step 3: Binary Files in Git

Find binary files currently tracked that may belong in LFS or .gitignore:
```
git ls-files | while read f; do
  case "$f" in
    *.png|*.jpg|*.jpeg|*.gif|*.ico|*.svg|*.woff|*.woff2|*.ttf|*.eot|*.zip|*.tar|*.gz|*.jar|*.exe|*.dll|*.so|*.dylib|*.pdf|*.mp4|*.mov|*.mp3|*.wav|*.sqlite|*.db|*.wasm|*.a|*.o|*.parquet)
      echo "$f ($(wc -c < "$f" | tr -d ' ') bytes)";;
  esac
done
```

Flag any binary files over 100 KB. Suggest git-lfs for repos with many binary assets.

## Step 4: Secrets Scan

Search the working tree for potential secrets using patterns:
- Files: `.env`, `.env.*` (not `.env.example`), `*.pem`, `*.key`, `credentials.json`, `service-account*.json`
- Content patterns in tracked files:
  - `AKIA[0-9A-Z]{16}` (AWS access keys)
  - `sk-[a-zA-Z0-9]{20,}` (OpenAI/API keys)
  - `sk-ant-[a-zA-Z0-9-]+` (Anthropic keys)
  - `ghp_[a-zA-Z0-9]{36}` (GitHub PATs)
  - `github_pat_[a-zA-Z0-9_]+` (GitHub fine-grained PATs)
  - `npm_[a-zA-Z0-9]+` (npm tokens)
  - `xox[bpors]-[a-zA-Z0-9-]+` (Slack tokens)
  - `sk_live_[a-zA-Z0-9]+` (Stripe secret keys)
  - `rk_live_[a-zA-Z0-9]+` (Stripe restricted keys)
  - `AIza[0-9A-Za-z_-]{35}` (Google API keys)
  - `ya29\.[0-9A-Za-z_-]+` (Google OAuth tokens)
  - `-----BEGIN (RSA |EC |DSA )?PRIVATE KEY-----`
  - `password\s*=\s*['"][^'"]+['"]` (hardcoded passwords)
  - `GITHUB_TOKEN\s*=\s*['"][^'"]+['"]`

Also check if `.gitignore` contains entries for `.env`, `*.pem`, `*.key`. Flag if missing.

## Step 5: Merge Conflict Markers

Search for leftover conflict markers in tracked files:
```
git grep -l '<<<<<<<' -- ':!*.lock'
git grep -l '=======' -- ':!*.lock'
git grep -l '>>>>>>>' -- ':!*.lock'
```

Flag any files with unresolved merge conflicts as critical.

## Step 6: Stale Branches

For each local branch:
- `git log -1 --format="%ai %s" <branch>` to get last commit date
- Flag branches with no commits in 3+ months as stale
- Flag branches marked as `[gone]` on remote (merged/deleted upstream): `git branch -vv | grep ': gone]'`

For remote branches (summarize, don't dump all):
- Count remote branches with no activity in 6+ months
- List up to 10 of the oldest

## Step 7: Gitignore Audit

Detect project type from manifest files and check `.gitignore` for common missing entries:

- **Node/TS**: `node_modules/`, `dist/`, `.env`, `*.tsbuildinfo`, `.DS_Store`
- **Rust**: `target/`, `Cargo.lock` (if library)
- **Go**: vendor/ (if not vendored), binary output
- **Java**: `target/`, `*.class`, `.idea/`, `*.jar`
- **Python**: `__pycache__/`, `*.pyc`, `.venv/`, `dist/`
- **General**: `.DS_Store`, `*.swp`, `.env`, `*.log`, `.idea/`, `.vscode/` (debatable)

Check for tracked files that should be ignored (e.g., `node_modules` or `.env` already committed).

## Step 8: Commit Hygiene

Analyze last 50 commits:
- Flag empty commit messages
- Flag commits with messages under 10 characters
- Check for merge commits vs rebase workflow
- Check for consistent conventional commit format (`feat:`, `fix:`, etc.)
- Report commit frequency (commits per week over last month)

## Step 9: Uncommitted Work

- `git status --porcelain` for dirty working tree
- `git stash list` for forgotten stashes
- Flag stashes older than 1 month

## Step 10: Git Config and Attributes

Check for common configuration issues:
- `git config --get core.autocrlf` -- flag if unset on cross-platform repos (has both Windows and Unix contributors)
- Check if `.gitattributes` exists. If the repo has binary files that should use LFS, flag missing `.gitattributes` with LFS tracking rules
- `git config --get pull.rebase` -- note whether rebase or merge is configured for pulls

## Step 11: Present Report

Organize into sections with severity:

1. **Critical** - leaked secrets, unresolved merge conflicts, large binaries in history
2. **Warning** - stale branches, missing gitignore entries, old stashes, tracked binaries
3. **Info** - repo size, commit stats, branch summary, git config notes

For each finding, provide the exact fix command.

End with a summary of recommended actions, grouped as:
- **Immediate**: security-related fixes, conflict resolution
- **Cleanup**: branch/stash cleanup commands
- **Improvement**: gitignore/commit hygiene suggestions, LFS migration, gitattributes setup

Ask which actions the user wants to take.
