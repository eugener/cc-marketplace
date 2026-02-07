---
description: Scaffold a new project with best-practice config, CI, and tooling for any language/framework
user-invocable: true
argument-hint: "[project-name]"
allowed-tools:
  - Bash
  - AskUserQuestion
  - Read
  - Write
  - Edit
  - Glob
  - Grep
  - WebSearch
  - WebFetch
---

Scaffold a new project. The project name is: $ARGUMENTS

## Step 1: Detect Environment

Run in parallel to discover available tools:
- `brew list --formula -1` and `brew list --cask -1`
- `which rustc cargo go deno zig javac python3 dotnet swift kotlin gradle mvn bun pnpm npm yarn`
- `ls ~/.cargo/bin 2>/dev/null`
- Check tool versions for detected tools

Build an internal map of what's available. Do NOT print this to the user.

## Step 2: Ask the User

Ask using AskUserQuestion (all in one call):

1. **Language/framework** -- only offer options that are actually installed on the machine. Include a freeform "Other" option. Examples: Rust, Go, TypeScript (Node), Deno, Java (Maven), Java (Gradle), Zig, Python, Swift, etc.

2. **Project type** -- CLI app, library/crate/module, web API/server, Lambda/serverless function, monorepo, full-stack app

3. **Extras** (multi-select) -- GitHub Actions CI, Docker, license (MIT), pre-commit hooks, README

If $ARGUMENTS is empty, also ask for the project name.

## Step 3: Research Current Best Practices

Based on the user's choices:

1. Use web search for "<language> project setup best practices" with current year

2. Check the latest stable version of key dependencies the project will need

Goal: generate configs that reflect current ecosystem conventions, NOT stale defaults.

## Step 4: Generate Project

Create the project directory and files. Follow these principles:

### Always generate:
- **CLAUDE.md** at project root with:
  - Build/test/lint commands
  - Project structure overview
  - Key conventions for this stack
  - Keep it concise (under 50 lines)
- **.gitignore** appropriate for the language (use gitignore.io or known good defaults)
- **Source files** with a minimal working example (hello world / health endpoint / empty lib with one test)

### Per-language initialization:
Use the native init tool when available:
- Rust: `cargo init` or `cargo new`
- Go: `go mod init`
- Node/TS: `bun init` (prefer bun if available, then pnpm, then npm). When using bun, scaffold pure TypeScript -- all source files must be `.ts`/`.tsx`, no `.js` files. Use bun's native TS support (no separate tsc build step needed for running).
- Deno: `deno init`
- Java: `mvn archetype:generate` or `gradle init`
- Zig: `zig init`
- Python: create `pyproject.toml` with modern config
- Other: create appropriate manifest file

Then layer on additional config files.

### Linter/formatter config:
- Generate config for the standard linter of the ecosystem (clippy, golangci-lint, eslint, deno lint, etc.)
- Use strict/recommended presets
- Only include tools already installed on the machine

### If CI requested:
- `.github/workflows/ci.yml`
- Steps: install deps, lint, test, build
- Use latest stable action versions (look up via web search if unsure)
- Match the language version to what's installed on the machine

### If Docker requested:
- Multi-stage `Dockerfile` optimized for the language
- `.dockerignore` mirroring `.gitignore` plus `.git/`

### If license requested:
- `LICENSE` file with MIT license, current year, user's git name from `git config user.name`

### If pre-commit hooks requested:
- Set up git hooks for fmt/lint check before commit
- Use the language's native tooling (cargo fmt, gofmt, prettier, etc.)

## Step 5: Initialize and Validate

Run sequentially:
1. `git init` in the project directory
2. Run the build command for the language
3. Run the test command
4. Run the linter if configured
5. Stage all files and create initial commit with message "init: scaffold <project-name>"

Report results to the user. If any step fails, show the error and fix it before proceeding.

## Step 6: Summary

Print a tree view of the generated project structure.
List the key commands (build, test, lint, run) the user should know.
If GitHub repo creation is desired, offer to run `gh repo create`.

## Rules

- NEVER generate placeholder or TODO comments. Every file should be functional.
- NEVER add unnecessary dependencies. Start minimal.
- NEVER use deprecated or outdated config formats. Research current standards.
- Prefer the tools and package managers already installed on the machine.
- Keep generated code minimal but compilable and testable.
