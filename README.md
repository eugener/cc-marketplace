# cc-marketplace

Claude Code plugin marketplace.

## Plugins

| Plugin | Description |
|---|---|
| `brew-optimize` | Analyze and optimize Homebrew installations |
| `project-scaffold` | Scaffold projects with best-practice config and tooling |

## Install

```
/plugin marketplace add eryzhikov/cc-marketplace
/plugin install brew-optimize
/plugin install project-scaffold
```

## Test locally

```
claude --plugin-dir ./plugins/brew-optimize
claude --plugin-dir ./plugins/project-scaffold
```
