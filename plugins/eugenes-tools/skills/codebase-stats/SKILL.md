---
description: Project analytics - lines of code, language breakdown, churn hotspots, test coverage ratio, and dependency count
user-invocable: true
allowed-tools:
  - Bash
  - Grep
  - Glob
  - Read
---

Generate a comprehensive codebase analytics report for the current project. All analysis is read-only.

## Step 1: Language Breakdown

If `tokei` is available, run:
```
tokei --sort code
```

If not, fall back to `find` + `wc -l` grouped by extension.

Report: total lines, code lines, comment lines, blank lines, breakdown by language.

## Step 2: Project Structure

- Count files by directory (top 2 levels): `find . -not -path './node_modules/*' -not -path './.git/*' -not -path './target/*' -not -path './dist/*' -not -path './vendor/*' -type f | awk -F/ '{print $2}' | sort | uniq -c | sort -rn`
- Count source files vs test files (detect by `*test*`, `*spec*`, `*_test.*`, `test_*.*` patterns)
- Compute test-to-source file ratio

## Step 3: Churn Hotspots

Find the 15 most frequently changed files in the last 6 months:
```
git log --since="6 months ago" --name-only --pretty=format: | sort | uniq -c | sort -rn | head -15
```

These are complexity/maintenance hotspots. Flag files with 20+ changes as high-churn.

## Step 4: Recent Activity

- Commits per week over last 4 weeks:
  ```
  for i in 0 1 2 3; do
    start=$(date -v-$((i+1))w +%Y-%m-%d)
    end=$(date -v-${i}w +%Y-%m-%d)
    count=$(git log --after="$start" --before="$end" --oneline | wc -l | tr -d ' ')
    echo "$start..$end: $count commits"
  done
  ```
- Top contributors (last 3 months): `git shortlog -sn --since="3 months ago"`
- Files changed in last 7 days: `git diff --stat HEAD~$(git log --oneline --since="7 days ago" | wc -l | tr -d ' ') 2>/dev/null || git diff --stat @{7.days.ago} 2>/dev/null`

## Step 5: Dependency Analysis

Detect and count dependencies by project type:

- **package.json**: count `dependencies` + `devDependencies` keys
- **Cargo.toml**: count `[dependencies]` + `[dev-dependencies]` entries
- **go.mod**: count `require` entries
- **pom.xml**: count `<dependency>` entries
- **pyproject.toml**: count dependencies entries

Report total direct deps and dev deps separately.

## Step 6: Code Quality Indicators

- Largest files by line count (top 10, excluding lock files and generated code):
  ```
  find . -type f \( -name "*.ts" -o -name "*.rs" -o -name "*.go" -o -name "*.java" -o -name "*.py" -o -name "*.zig" \) -not -path '*/node_modules/*' -not -path '*/target/*' -not -path '*/.git/*' | xargs wc -l 2>/dev/null | sort -rn | head -11
  ```
  Flag files over 500 lines as candidates for splitting.

- TODO/FIXME/HACK count: search source files for these markers and report count per file

- Dead code indicators: search for `#[allow(dead_code)]`, `// eslint-disable`, `// nolint`, `@SuppressWarnings`

## Step 7: Present Report

Format as a dashboard:

```
=== Codebase Report: <project-name> ===

Languages:        <tokei summary>
Total Files:      X source / Y test (ratio: Z)
Dependencies:     X direct / Y dev
Repo Size:        X (git: Y)

Activity (last 4 weeks):
  wk1: ██████ 12
  wk2: ████ 8
  ...

Churn Hotspots:
  1. src/foo.ts (34 changes)
  2. ...

Largest Files:
  1. src/bar.ts (820 lines)
  2. ...

Quality Markers:
  TODOs: X | FIXMEs: Y | Lint suppressions: Z
```

Keep it compact. Use inline bar charts (unicode block chars) for visual elements where useful.
