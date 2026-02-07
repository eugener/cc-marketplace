# cc-marketplace

A [Claude Code](https://docs.anthropic.com/en/docs/claude-code) plugin marketplace with developer productivity skills.

## Install

```
/plugin marketplace add eugener/cc-marketplace
/plugin install eugenes-tools
```

## Skills

### `eugenes-tools:brew-optimize`

Comprehensive Homebrew audit. Finds unused packages, orphaned dependencies, duplicate versioned formulae, stale casks, and disk hogs. Presents a categorized report with copy-paste cleanup commands. Runs `brew doctor` for general health warnings. Asks before making any changes.

**Platform:** macOS only

**Example:**
```
/eugenes-tools:brew-optimize
```

---

### `eugenes-tools:project-scaffold`

Scaffolds a new project from scratch. Detects installed toolchains, asks for language/framework/type, researches current best practices via Context7 or web search, then generates a working project with CLAUDE.md, .editorconfig, .gitignore, linter config, and a minimal compilable example with tests. Optionally adds CI, Docker, license, and pre-commit hooks.

**Supported:** Rust, Go, TypeScript (Bun/pnpm/npm), Deno, Java (Maven/Gradle), Zig, Python, Swift

**Example:**
```
/eugenes-tools:project-scaffold my-api
```

---

### `eugenes-tools:repo-health`

Git repository hygiene audit. Scans for large blobs in history, tracked binary files, leaked secrets (AWS, GitHub, Anthropic, Slack, npm tokens, private keys), unresolved merge conflict markers, stale/gone branches, missing .gitignore entries, commit message hygiene, and forgotten stashes. Groups findings by severity (critical/warning/info) with exact fix commands.

**Platform:** macOS and Linux

**Example:**
```
/eugenes-tools:repo-health
```

---

### `eugenes-tools:codebase-stats`

Read-only project analytics dashboard. Uses `tokei` for language breakdown (with fallback), detects monorepos, identifies churn hotspots from git history, tracks weekly commit activity, counts dependencies by manifest type, and flags code quality indicators (large files, TODOs, lint suppressions).

**Platform:** macOS and Linux

**Example:**
```
/eugenes-tools:codebase-stats
```

## Requirements

- [Claude Code](https://docs.anthropic.com/en/docs/claude-code) CLI
- Skills use standard CLI tools (`git`, `brew`, `tokei`, `du`, `stat`). Missing tools are detected and skipped or substituted.
- Optional: [Context7](https://context7.com) MCP server (improves `project-scaffold` research)

## Development

Test a plugin locally without installing:

```
claude --plugin-dir ./plugins/eugenes-tools
```

## License

MIT
