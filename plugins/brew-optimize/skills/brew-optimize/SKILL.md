---
description: Analyze and optimize Homebrew installations - find unused, outdated, and orphaned packages
user-invocable: true
allowed-tools:
  - Bash
  - AskUserQuestion
---

Perform a comprehensive Homebrew optimization audit. Run all analysis steps, then present a summary with actionable recommendations. Ask before making any changes.

## Step 1: Gather Data

Run these in parallel:
- `brew leaves` (top-level formulae)
- `brew list --cask` (installed casks)
- `brew outdated` (outdated packages)
- `brew autoremove --dry-run` (orphaned dependencies)
- `brew cleanup --dry-run` (stale downloads/old versions)

## Step 2: Analyze Usage

For each top-level formula from `brew leaves`:
- Check last access time of binaries via `stat -f "%Sa" -t "%Y-%m-%d"` on files in the package's bin directory
- Flag packages not accessed in 3+ months as "stale"
- Flag packages not accessed in 6+ months as "likely unused"

For each cask:
- Check last opened date via `mdls -name kMDItemLastUsedDate` on the .app bundle
- Flag apps not opened in 3+ months

## Step 3: Find Orphaned Dependencies

For each outdated package, run `brew uses --installed <pkg>`:
- If no dependents: mark as "orphaned, safe to remove"
- If has dependents: list what depends on it

## Step 4: Check Disk Usage

Run `brew info <pkg>` for large packages to identify space savings.
Use `du -sh $(brew --cellar)` for total Homebrew disk usage.

## Step 5: Present Report

Organize findings into these categories:

1. **Likely Unused** - top-level packages with stale access times and no dependents
2. **Orphaned Dependencies** - outdated packages nothing depends on
3. **Outdated (in use)** - packages that are outdated but actively depended upon
4. **Stale Casks** - cask apps not recently opened
5. **Cleanup Savings** - space reclaimable via `brew cleanup`

For each category, show a markdown table with package name, relevant details (last access, version, size, dependents).

End with concrete commands the user can copy-paste:
- `brew uninstall ...` for removals
- `brew upgrade ...` for updates
- `brew cleanup` for cache cleanup

Ask the user which actions they want to take before executing anything.
