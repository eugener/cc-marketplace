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

## Step 0: Detect Platform

Run `uname -s` to detect OS. Use this throughout:
- **macOS**: `date -v-Xw`, `stat -f`
- **Linux**: `date -d 'X weeks ago'`, `stat -c`

## Step 1: Language Breakdown

If `tokei` is available (check with `which tokei`), run:
```
tokei --sort code
```

If not, fall back to counting with `find -print0 | xargs -0 wc -l` grouped by extension (use `-print0`/`xargs -0` for space-safe paths).

Report: total lines, code lines, comment lines, blank lines, breakdown by language.

## Step 2: Project Structure

Run in parallel:
- Count files by top-level directory (exclude `.git`, `node_modules`, `target`, `dist`, `vendor`, `build`, `__pycache__`):
  ```
  find . -not -path './.git/*' -not -path './node_modules/*' -not -path './target/*' -not -path './dist/*' -not -path './vendor/*' -not -path './build/*' -not -path './__pycache__/*' -type f -print0 | xargs -0 -n1 dirname | awk -F/ '{print $2}' | sort | uniq -c | sort -rn
  ```
- Count source files vs test files (detect by `*.test.*`, `*.spec.*`, `*_test.*`, `test_*.*`, files under `test/` or `tests/` or `__tests__/` directories)
- Compute test-to-source file ratio

## Step 3: Monorepo Detection

Check for multiple project manifests:
```
find . -maxdepth 3 -name 'package.json' -o -name 'Cargo.toml' -o -name 'go.mod' -o -name 'pom.xml' -o -name 'pyproject.toml' | grep -v node_modules | grep -v vendor | grep -v target
```

If more than one manifest found, report as monorepo with sub-project breakdown.

## Step 4: Churn Hotspots

Find the 15 most frequently changed files in the last 6 months:
```
git log --since="6 months ago" --name-only --pretty=format: | grep -v '^$' | sort | uniq -c | sort -rn | head -15
```

Flag files with 20+ changes as high-churn.

## Step 5: Recent Activity

Commits per week over last 4 weeks (platform-aware):
```
# macOS
for i in 0 1 2 3; do
  start=$(date -v-$((i+1))w +%Y-%m-%d)
  end=$(date -v-${i}w +%Y-%m-%d)
  count=$(git log --after="$start" --before="$end" --oneline 2>/dev/null | wc -l | tr -d ' ')
  echo "$start..$end: $count commits"
done
```
```
# Linux
for i in 0 1 2 3; do
  start=$(date -d "$((i+1)) weeks ago" +%Y-%m-%d)
  end=$(date -d "$i weeks ago" +%Y-%m-%d)
  count=$(git log --after="$start" --before="$end" --oneline 2>/dev/null | wc -l | tr -d ' ')
  echo "$start..$end: $count commits"
done
```

Use the correct variant based on Step 0 detection. Also run in parallel:
- Top contributors (last 3 months): `git shortlog -sn --since="3 months ago"`
- Recent file changes: `git diff --stat @{7.days.ago} 2>/dev/null || echo "No reflog data for 7 days ago"`

## Step 6: Dependency Analysis

Detect and count dependencies by project type (check all that exist):

- **package.json**: count `dependencies` + `devDependencies` keys via `jq` or manual count
- **Cargo.toml**: count `[dependencies]` + `[dev-dependencies]` entries
- **go.mod**: count `require` entries
- **pom.xml**: count `<dependency>` entries
- **pyproject.toml**: count dependencies entries

Report total direct deps and dev deps separately. For monorepos, report per sub-project.

## Step 7: Code Quality Indicators

Run in parallel:

- Largest source files by line count (top 10, excluding lock files and generated code):
  ```
  find . -type f \( -name "*.ts" -o -name "*.tsx" -o -name "*.js" -o -name "*.jsx" -o -name "*.rs" -o -name "*.go" -o -name "*.java" -o -name "*.py" -o -name "*.zig" -o -name "*.svelte" -o -name "*.vue" \) -not -path '*/node_modules/*' -not -path '*/target/*' -not -path '*/.git/*' -not -path '*/dist/*' -not -path '*/build/*' -print0 | xargs -0 wc -l 2>/dev/null | sort -rn | head -11
  ```
  Flag files over 500 lines as candidates for splitting.

- TODO/FIXME/HACK count: search source files for these markers and report count per file

- Dead code indicators: search for `#[allow(dead_code)]`, `// eslint-disable`, `// nolint`, `@SuppressWarnings`

## Step 8: Present Report

Format as a dashboard:

```
=== Codebase Report: <project-name> ===

Languages:        <tokei summary>
Total Files:      X source / Y test (ratio: Z)
Dependencies:     X direct / Y dev
Repo Size:        X (git: Y)
Monorepo:         Yes/No (N sub-projects)

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
