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
    *.png|*.jpg|*.jpeg|*.gif|*.ico|*.svg|*.woff|*.woff2|*.ttf|*.eot|*.zip|*.tar|*.gz|*.jar|*.exe|*.dll|*.so|*.dylib|*.pdf)
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

## Step 10: Present Report

Organize into sections with severity:

1. **Critical** - leaked secrets, unresolved merge conflicts, large binaries in history
2. **Warning** - stale branches, missing gitignore entries, old stashes, tracked binaries
3. **Info** - repo size, commit stats, branch summary

For each finding, provide the exact fix command.

End with a summary of recommended actions, grouped as:
- **Immediate**: security-related fixes, conflict resolution
- **Cleanup**: branch/stash cleanup commands
- **Improvement**: gitignore/commit hygiene suggestions, LFS migration

Ask which actions the user wants to take.
