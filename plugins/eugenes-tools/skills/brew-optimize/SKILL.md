---
description: Analyze and optimize Homebrew installations - find unused, outdated, and orphaned packages (macOS only)
user-invocable: true
allowed-tools:
  - Bash
  - AskUserQuestion
---

Perform a comprehensive Homebrew optimization audit. Run all analysis steps, then present a summary with actionable recommendations. Ask before making any changes.

## Step 1: Gather Data

Run all of these in parallel:
- `brew leaves` (top-level formulae)
- `brew list --cask` (installed casks)
- `brew outdated` (outdated packages)
- `brew autoremove --dry-run` (orphaned dependencies)
- `brew cleanup --dry-run` (stale downloads/old versions)
- `brew doctor 2>&1` (general health warnings)

## Step 2: Disk Usage

Find the top 10 largest packages by cellar size:
```
du -d1 $(brew --cellar) 2>/dev/null | sort -rn | head -11
```

Also report total cellar size: `du -sh $(brew --cellar)`

## Step 3: Analyze Usage

For each top-level formula from `brew leaves`:
- Get the package prefix: `brew --prefix <pkg>`
- Check last access time of binaries via `stat -f "%Sa" -t "%Y-%m-%d"` on files in the `bin/` directory
- For library-only packages (no bin/), check files in `lib/` or `include/` instead
- Flag packages not accessed in 3+ months as "stale"
- Flag packages not accessed in 6+ months as "likely unused"

For each cask:
- Check last opened date via `mdls -name kMDItemLastUsedDate` on the .app bundle
- Flag apps not opened in 3+ months

## Step 4: Find Orphaned Dependencies

For each outdated package, run `brew uses --installed <pkg>`:
- If no dependents: mark as "orphaned, safe to remove"
- If has dependents: list what depends on it

## Step 5: Detect Duplicate Versions

Check for multiple versions of the same tool:
- `brew list --formula | grep -E '@[0-9]'` to find versioned formulae
- Group by base name (e.g., python@3.11 + python@3.13, llvm@19 + llvm@20)
- For each group, check which versions have dependents via `brew uses --installed`
- Flag versions with no dependents as removal candidates

## Step 6: Present Report

Organize findings into these categories:

1. **Likely Unused** - top-level packages with stale access times and no dependents
2. **Orphaned Dependencies** - outdated packages nothing depends on
3. **Duplicate Versions** - multiple versions of same tool, with dependent info
4. **Outdated (in use)** - packages that are outdated but actively depended upon
5. **Stale Casks** - cask apps not recently opened
6. **Cleanup Savings** - space reclaimable via `brew cleanup`
7. **Doctor Warnings** - issues reported by `brew doctor` (if any)

For each category, show a markdown table with package name, relevant details (last access, version, size, dependents).

End with concrete commands the user can copy-paste:
- `brew uninstall ...` for removals
- `brew upgrade ...` for updates
- `brew cleanup` for cache cleanup

Ask the user which actions they want to take before executing anything.
