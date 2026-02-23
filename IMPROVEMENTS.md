# GSD Codebase Improvement Recommendations

Analysis of [get-shit-done v1.20.5](https://github.com/glittercowboy/get-shit-done) — a meta-prompting and spec-driven development system for Claude Code, OpenCode, and Gemini CLI.

**Codebase stats:** ~7,100 lines JavaScript (CommonJS), ~30 slash commands, 11 sub-agents, 30 workflows, 96 passing tests.

---

## 1. Code Quality & Architecture

### 1a. The install script is a 1,865-line monolith

`bin/install.js` handles argument parsing, runtime conversion (Claude/OpenCode/Gemini), file copying, settings mutation, uninstallation, JSONC parsing, local patch management, manifest generation, permission configuration, and interactive prompts — all in a single file.

**Recommendation:** Split into modules mirroring the lib pattern used in `gsd-tools.cjs`:
- `lib/install-runtime.js` — per-runtime conversion logic (frontmatter, tool names, TOML)
- `lib/install-files.js` — file copying, path replacement, manifest
- `lib/install-settings.js` — settings.json mutation, hook registration
- `lib/install-patches.js` — local patch save/restore
- `lib/install-uninstall.js` — uninstall logic
- `bin/install.js` — thin CLI entry point with arg parsing + prompts

### 1b. No tests for the installer

The installer is the most complex and user-facing part of the codebase, yet it has zero test coverage. The 96 existing tests only cover `gsd-tools.cjs` commands.

**Recommendation:** Add unit tests for:
- `convertClaudeToOpencodeFrontmatter()` — tool name mapping, color conversion
- `convertClaudeToGeminiAgent()` — frontmatter conversion, `${VAR}` escaping
- `convertClaudeToGeminiToml()` — TOML generation
- `parseJsonc()` — comment stripping, trailing comma removal
- `processAttribution()` — removal, replacement, pass-through
- `copyWithPathReplacement()` — path template substitution
- Uninstall logic — selective file removal without touching user files

### 1c. Hand-rolled YAML parser is fragile

`frontmatter.cjs:extractFrontmatter()` is a custom 80-line YAML parser that handles only a subset of YAML syntax. It fails on quoted keys, multi-line strings, anchors/aliases, flow mappings, and edge cases like colons in values.

**Recommendation:** Either:
- Add a lightweight YAML dependency (e.g., `yaml` package, ~50KB) since the project already uses `esbuild` for bundling
- Or add comprehensive test cases for every frontmatter pattern used in GSD templates to document and defend the supported subset

### 1d. Duplicated logic across init commands

`init.cjs` has 12 `cmdInit*` functions that all follow the same pattern: load config, resolve models, check file existence, gather phase info, output JSON. There's significant copy-paste between `cmdInitExecutePhase`, `cmdInitPlanPhase`, `cmdInitPhaseOp`, etc.

**Recommendation:** Extract a shared `buildInitContext(cwd, options)` function that gathers the common context (config, models, file existence, milestone info) and let each init command extend it with workflow-specific fields.

---

## 2. Testing

### 2a. No integration tests for the full workflow

Tests validate individual `gsd-tools` commands in isolation but never test a full workflow (e.g., `new-project` → `plan-phase` → `execute-phase` → `verify-work` → `complete-milestone`).

**Recommendation:** Add at least one end-to-end test that simulates a full project lifecycle using the CLI, verifying that state transitions, file creation, and roadmap updates work together.

### 2b. Missing test coverage for key modules

- `phase.cjs` (873 lines) — only basic tests for phase operations
- `roadmap.cjs` (298 lines) — only basic tests
- `milestone.cjs` (215 lines) — minimal coverage
- `frontmatter.cjs` — no dedicated tests for the YAML parser edge cases
- Hooks (`gsd-statusline.js`, `gsd-context-monitor.js`) — no tests at all

**Recommendation:** Add targeted unit tests for frontmatter parsing edge cases (nested objects, inline arrays, quoted values with colons) and hook output formatting.

### 2c. Tests spawn subprocesses for every assertion

Every test calls `runGsdTools()` which spawns a new Node.js process via `execSync`. This adds ~100ms overhead per test. With 96 tests, this accounts for most of the ~4.7s test duration.

**Recommendation:** For unit-level tests, import the functions directly and test them in-process. Reserve subprocess testing for CLI-level integration tests only.

---

## 3. Error Handling & Robustness

### 3a. Silent `catch {}` blocks throughout

There are 50+ bare `catch {}` blocks across the codebase that swallow errors silently (`core.cjs:108`, `commands.cjs:73-75`, `init.cjs:163`, etc.). While some are intentional (graceful degradation for missing files), many hide real bugs.

**Recommendation:** At minimum, add comments explaining why the error is safe to ignore. For development, consider a debug mode that logs swallowed errors to stderr when `GSD_DEBUG=1` is set.

### 3b. No input validation on CLI arguments

`gsd-tools.cjs` passes `args[1]`, `args[2]`, etc. directly to command handlers without validation. Malformed arguments (e.g., `gsd-tools state patch --field` without a value) can cause confusing errors or silent failures.

**Recommendation:** Add argument validation at the router level — at least check for required arguments before dispatching to command handlers, and provide clear error messages.

### 3c. `execGit` shell injection surface

`core.cjs:127-146` — the `execGit` function uses string concatenation to build shell commands. While it has basic quoting via single-quote escaping, it still passes the command through the shell. The `isGitIgnored` function at line 117 uses a character class filter but doesn't handle all edge cases.

**Recommendation:** Use `execFileSync('git', args)` instead of `execSync('git ' + ...)` to bypass the shell entirely. This is both safer and handles arguments with special characters correctly.

---

## 4. Developer Experience

### 4a. No CI/CD pipeline for tests or linting

The only GitHub Action is `auto-label-issues.yml` for issue triage. Tests aren't run on PRs, there's no linting, and the `prepublishOnly` hook only builds hooks.

**Recommendation:** Add a GitHub Actions workflow that:
- Runs `npm test` on Node 16, 18, 20, 22
- Runs a linter (ESLint with a minimal config)
- Validates that `npm run build:hooks` produces no changes (catches uncommitted builds)

### 4b. The build step is just a file copy

`scripts/build-hooks.js` copies 3 files from `hooks/` to `hooks/dist/` with zero transformation. The `esbuild` devDependency is listed in `package.json` but never used.

**Recommendation:** Either:
- Remove `esbuild` from devDependencies and simplify the build to a plain copy (as it already is)
- Or actually use esbuild to bundle the hooks, which would allow them to share utilities without runtime `require()` paths

### 4c. No TypeScript or JSDoc type annotations

The entire codebase is untyped CommonJS JavaScript. Functions accept generic `options` objects with no documentation of their shape.

**Recommendation:** Add JSDoc `@param` and `@returns` annotations to the public API of each module. Consider a `// @ts-check` header for gradual type checking.

### 4d. `.bak` file committed

`commands/gsd/new-project.md.bak` is committed to the repo.

**Recommendation:** Delete the `.bak` file and add `*.bak` to `.gitignore`.

---

## 5. Usability & Features

### 5a. Update check is version string comparison, not semver

`gsd-check-update.js:48` uses `installed !== latest` for version comparison. This means any difference (including a downgrade) triggers the update notification.

**Recommendation:** Implement proper semver comparison: `latest > installed` should be the trigger, not `latest !== installed`.

### 5b. Context monitor thresholds are hardcoded

The WARNING (35%) and CRITICAL (25%) thresholds in `gsd-context-monitor.js` can't be customized.

**Recommendation:** Read thresholds from `config.json` with the current values as defaults.

### 5c. Statusline hardcodes `~/.claude` path

`gsd-statusline.js:66` uses `path.join(homeDir, '.claude', 'todos')` regardless of the runtime. Todo display only works for Claude Code.

**Recommendation:** Template the config directory path during installation so the statusline works correctly for all runtimes.

### 5d. No `--version` flag on the CLI

`bin/install.js` supports `--help` but not `--version`.

**Recommendation:** Add `--version` / `-v` flag that prints the version and exits.

---

## 6. Documentation

### 6a. No contribution guide

The repo has a README, USER-GUIDE, SECURITY, and CHANGELOG, but no CONTRIBUTING.md.

**Recommendation:** Add a CONTRIBUTING.md covering: repo setup, test commands, architecture overview, PR guidelines.

### 6b. Internal code has minimal comments

Library modules have section headers but very few inline comments explaining non-obvious logic (e.g., phase numbering scheme, `must_haves` block parsing).

**Recommendation:** Add comments to algorithmic sections — especially `comparePhaseNum()`, `parseMustHavesBlock()`, and the frontmatter conversion functions.

---

## Summary — Top 5 Highest-Impact Improvements

| Priority | Improvement | Impact |
|----------|------------|--------|
| 1 | Add CI pipeline (tests + lint on PRs) | Prevents regressions, builds contributor confidence |
| 2 | Add installer tests | Catches conversion bugs across 3 runtimes before users do |
| 3 | Split `bin/install.js` into modules | Makes the most complex file maintainable |
| 4 | Replace `execSync('git ...')` with `execFileSync` | Eliminates shell injection surface |
| 5 | Use proper semver for update checks | Prevents false update notifications |
