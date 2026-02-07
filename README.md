# cc-marketplace

Claude Code plugin marketplace.

## Skills

| Skill | Description |
|---|---|
| `eugenes-tools:brew-optimize` | Analyze and optimize Homebrew installations |
| `eugenes-tools:project-scaffold` | Scaffold projects with best-practice config and tooling |
| `eugenes-tools:repo-health` | Audit git repo hygiene: large blobs, secrets, stale branches, gitignore |
| `eugenes-tools:codebase-stats` | Project analytics: LOC, churn hotspots, deps, code quality indicators |

## Install

```
/plugin marketplace add eugener/cc-marketplace
/plugin install eugenes-tools
```

## Test locally

```
claude --plugin-dir ./plugins/eugenes-tools
```
