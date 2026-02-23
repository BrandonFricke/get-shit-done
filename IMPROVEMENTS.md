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

### 1e. Duplicated phase-directory and plan-filtering patterns

The same code to list phase directories and filter plans/summaries appears 6+ times in `phase.cjs` and 4+ times in `verify.cjs`:

```javascript
// Repeated 10+ times across phase.cjs:26,106,166,211,464,495,542,787 and verify.cjs:421,579,622,661
const entries = fs.readdirSync(phasesDir, { withFileTypes: true });
const dirs = entries.filter(e => e.isDirectory()).map(e => e.name).sort((a, b) => comparePhaseNum(a, b));

// Repeated 8+ times across phase.cjs, verify.cjs, commands.cjs
const plans = phaseFiles.filter(f => f.endsWith('-PLAN.md') || f === 'PLAN.md').sort();
const summaries = phaseFiles.filter(f => f.endsWith('-SUMMARY.md') || f === 'SUMMARY.md').sort();
```

**Recommendation:** Extract `getPhaseDirectories(cwd)` and `getPlansAndSummaries(files)` utilities into `core.cjs`.

### 1f. Inconsistent error handling patterns

Three conflicting patterns are used across the codebase:

1. `error('...')` in `core.cjs:45-48` — calls `process.exit(1)`, never returns
2. `output({ error: '...' }, raw)` — returns an error object to the caller (e.g., `state.cjs:158`)
3. Bare `catch {}` — swallows errors silently (50+ occurrences)

Functions that call `error()` never return, making them unpredictable for callers. Functions that use `output({ error })` continue execution.

There is also at least one callsite bug: `state.cjs:51` calls `output(result)` without passing the `raw` parameter, so raw-mode output is broken for that command.

**Recommendation:** Standardize on one pattern. `output({ error })` is the better choice since it allows the CLI to return structured errors. Reserve `process.exit(1)` for truly fatal errors (missing required files, corrupt state). Add `GSD_DEBUG=1` env var to log all swallowed errors. Audit all `output()` calls for missing `raw` parameter.

### 1g. Repeated file reads in hot paths

`phase.cjs:cmdPhaseRemove()` (lines 444-695) reads the phases directory 3 separate times at lines 464, 495, and 542. `verify.cjs:cmdValidateHealth()` reads it 4 times at lines 579, 622, 631, 661. `cmdPhaseComplete()` reads ROADMAP.md, STATE.md, and REQUIREMENTS.md sequentially when they could be batched.

**Recommendation:** Read the directory once, store in a local variable, and pass it to the renumbering steps.

---

## 2. Security

### 2a. `execGit` shell injection surface

`core.cjs:127-146` — the `execGit` function uses string concatenation to build shell commands. While it has basic quoting via single-quote escaping, it still passes the command through the shell. The `isGitIgnored` function at line 117 uses a character class filter but doesn't handle all edge cases.

**Recommendation:** Use `execFileSync('git', args)` instead of `execSync('git ' + ...)` to bypass the shell entirely. This is both safer and handles arguments with special characters correctly.

### 2b. File path traversal in CLI arguments

Multiple functions construct file paths from user-supplied arguments without validating against directory traversal (`init.cjs:121`, `commands.cjs:86`). A crafted phase name like `../../etc` could read files outside `.planning/`.

**Severity:** Low-medium — operations are read-only and only developers run GSD.

**Recommendation:** Validate that resolved paths stay within the project directory:
```javascript
const resolved = path.resolve(cwd, userPath);
if (!resolved.startsWith(path.resolve(cwd))) error('Path traversal detected');
```

### 2c. `JSON.parse()` without try-catch on user input

- `gsd-tools.cjs:261` — `JSON.parse(args[fieldsIdx + 1])` on CLI argument
- `hooks/gsd-statusline.js:15` — `JSON.parse(input)` from stdin
- `hooks/gsd-context-monitor.js:34` — `JSON.parse(input)` from stdin

Malformed input crashes the process with an unhelpful "Unexpected token" error.

**Recommendation:** Wrap each in try-catch with descriptive error messages.

### 2d. Settings.json mutation without backup or atomic write

The installer reads `settings.json`, modifies it in-memory, and writes it back (`install.js:1537-1632`). If the write fails mid-operation, the file could be corrupted. No backup is made before modification.

**Recommendation:** Write to a temp file first, then atomically rename:
```javascript
fs.writeFileSync(settingsPath + '.tmp', JSON.stringify(settings, null, 2));
fs.renameSync(settingsPath + '.tmp', settingsPath);
```

### 2e. Code template injection in update checker

`gsd-check-update.js:25-62` uses `spawn()` with a code template that embeds `JSON.stringify()`'d paths. While `JSON.stringify` escapes strings, paths containing backticks or control characters could theoretically break out of the template literal.

**Recommendation:** Pass data via environment variables or stdin instead of embedding in code strings.

### 2f. No symlink protection on file operations

Node.js `fs` follows symlinks by default. A symlink at `.planning/` could redirect reads/writes to arbitrary locations.

**Recommendation:** Use `fs.lstatSync()` to check for symlinks before operating on critical directories.

---

## 3. Testing

### 3a. No integration tests for the full workflow

Tests validate individual `gsd-tools` commands in isolation but never test a full workflow (e.g., `new-project` -> `plan-phase` -> `execute-phase` -> `verify-work` -> `complete-milestone`).

**Recommendation:** Add at least one end-to-end test that simulates a full project lifecycle using the CLI, verifying that state transitions, file creation, and roadmap updates work together.

### 3b. Missing test coverage for key modules

- `phase.cjs` (873 lines) — only basic tests for phase operations
- `roadmap.cjs` (298 lines) — only basic tests
- `milestone.cjs` (215 lines) — minimal coverage
- `frontmatter.cjs` — no dedicated tests for the YAML parser edge cases
- Hooks (`gsd-statusline.js`, `gsd-context-monitor.js`) — no tests at all
- `bin/install.js` (1,865 lines) — zero test coverage

**Recommendation:** Add targeted unit tests for frontmatter parsing edge cases (nested objects, inline arrays, quoted values with colons) and hook output formatting.

### 3c. Possibly unused test helper export

`tests/helpers.cjs:40` exports `TOOLS_PATH` which does not appear to be imported by any test file. Minor dead code.

**Recommendation:** Verify usage and remove if unused.

### 3d. Tests spawn subprocesses for every assertion

Every test calls `runGsdTools()` which spawns a new Node.js process via `execSync`. This adds ~100ms overhead per test. With 96 tests, this accounts for most of the ~4.7s test duration.

**Recommendation:** For unit-level tests, import the functions directly and test them in-process. Reserve subprocess testing for CLI-level integration tests only.

---

## 4. Installer-Specific Issues

### 4a. JSONC parser has edge-case bugs

The custom `parseJsonc()` function (`install.js:1072-1098`):
- Only handles `\"` escape sequences — fails on `\\`, `\n`, `\t`, `\uXXXX`
- Block comment removal doesn't handle `*/` inside strings
- A URL like `"http://example.com"` followed by `//` can be misinterpreted as a comment start

**Recommendation:** Test against the actual JSONC files generated by Claude Code, OpenCode, and Gemini CLI. Add regression tests for each edge case.

### 4b. Incomplete runtime tool name mappings

`install.js:309-330` — The `claudeToOpencodeTools` mapping is missing entries for common tools. Invalid/unmapped tool names silently default to lowercase, which may not be valid OpenCode tool names. The Gemini mapping at line 357-372 maps `SlashCommand` to lowercase `slashcommand` instead of `skill`.

**Recommendation:** Audit tool name mappings against current OpenCode and Gemini CLI documentation. Add a warning when encountering unmapped tool names.

### 4c. Color conversion silently drops unknown values

`install.js:529-544` — Invalid color names in frontmatter are silently removed during OpenCode conversion with no warning to the user.

**Recommendation:** Log a warning when skipping unrecognized colors.

### 4d. Uninstall leaves orphaned artifacts

- Only cleans `SessionStart` and `PostToolUse` hooks — misses any GSD hooks in other events (`install.js:940-962`)
- OpenCode permission cleanup only handles `read` and `external_directory` types (`install.js:1009-1028`)
- `package.json` removal is fragile — exact-match check on `'{"type":"commonjs"}'` fails if whitespace differs (`install.js:916`)
- Agent file removal only matches `gsd-*.md` pattern (`install.js:881`)

**Recommendation:** Track installed artifacts in the manifest and remove them by reference during uninstall rather than by pattern matching.

### 4e. Local patch restoration command not implemented

`install.js:1348` tells users to run `/gsd:reapply-patches` to restore local modifications, but the command exists only as a markdown definition (`commands/gsd/reapply-patches.md`) with no corresponding workflow implementation.

**Recommendation:** Either implement the workflow or remove the reference and document manual patch restoration.

### 4f. Cross-platform path issues

- Windows: Uses `path.join(os.homedir(), '.config', 'opencode')` instead of `%APPDATA%` (`install.js:92`)
- Windows: Hook command paths use forward slashes which may fail in cmd.exe (`install.js:1541`)
- Non-TTY: Defaults to global install without asking in CI/CD environments (`install.js:1747-1750`)
- `CLAUDE_CONFIG_DIR` env var is case-sensitive in Node.js but case-insensitive on Windows (`install.js:124`)

**Recommendation:** Use platform-appropriate config directories. Add explicit `--global` and `--local` flags to avoid the TTY prompt issue.

---

## 5. Developer Experience

### 5a. No CI/CD pipeline for tests or linting

The only GitHub Action is `auto-label-issues.yml` for issue triage. Tests aren't run on PRs, there's no linting, and the `prepublishOnly` hook only builds hooks.

**Recommendation:** Add a GitHub Actions workflow that:
- Runs `npm test` on Node 16, 18, 20, 22
- Runs a linter (ESLint with a minimal config)
- Validates that `npm run build:hooks` produces no changes (catches uncommitted builds)

### 5b. The build step is just a file copy

`scripts/build-hooks.js` copies 3 files from `hooks/` to `hooks/dist/` with zero transformation. The `esbuild` devDependency is listed in `package.json` but never imported or used anywhere.

**Recommendation:** Remove `esbuild` from devDependencies. The build is already a plain file copy.

### 5c. No TypeScript or JSDoc type annotations

The entire codebase is untyped CommonJS JavaScript. Functions accept generic `options` objects with no documentation of their shape.

**Recommendation:** Add JSDoc `@param` and `@returns` annotations to the public API of each module. Consider a `// @ts-check` header for gradual type checking.

### 5d. `.bak` file committed

`commands/gsd/new-project.md.bak` is committed to the repo.

**Recommendation:** Delete the `.bak` file and add `*.bak` to `.gitignore`.

---

## 6. Usability & Features

### 6a. Update check is version string comparison, not semver

`gsd-check-update.js:48` uses `installed !== latest` for version comparison. This means any difference (including a downgrade) triggers the update notification.

**Recommendation:** Implement proper semver comparison: `latest > installed` should be the trigger, not `latest !== installed`.

### 6b. Context monitor thresholds are hardcoded

The WARNING (35%) and CRITICAL (25%) thresholds in `gsd-context-monitor.js` can't be customized.

**Recommendation:** Read thresholds from `config.json` with the current values as defaults.

### 6c. Statusline hardcodes `~/.claude` path

`gsd-statusline.js:66` uses `path.join(homeDir, '.claude', 'todos')` regardless of the runtime. Todo display only works for Claude Code.

**Recommendation:** Template the config directory path during installation so the statusline works correctly for all runtimes.

### 6d. No `--version` flag on the CLI

`bin/install.js` supports `--help` but not `--version`.

**Recommendation:** Add `--version` / `-v` flag that prints the version and exits.

### 6e. Settings overwritten instead of merged

`install.js:1623-1627` — `finishInstall()` replaces `settings.statusLine` entirely. If the user has custom statusline fields, they're lost.

**Recommendation:** Deep-merge GSD settings with existing user settings instead of replacing top-level keys.

---

## 7. Workflow & Markdown System

### 7a. Two commands lack dedicated workflow files

- `/gsd:debug` (`commands/gsd/debug.md`, ~165 lines) — complex command with agent spawning, fully self-contained in the command file rather than delegating to a workflow
- `/gsd:reapply-patches` (`commands/gsd/reapply-patches.md`, ~111 lines) — complex merge logic without a workflow implementation

**Recommendation:** Extract orchestration logic into workflow files for consistency with the rest of the system.

### 7b. Ambiguous error handling in execute-phase workflow

`workflows/execute-phase.md`:
- **Line 263-264:** If no parent UAT is found, the instruction is to skip — but doesn't clarify what happens if VERIFICATION.md also doesn't exist
- **Line 170:** If a Wave N plan fails and Wave N+1 depends on it, no explicit guidance on automatic failure propagation
- **Line 108-109:** Documents a `classifyHandoffIfNeeded` Claude Code bug workaround that may be stale

**Recommendation:** Add explicit fallback instructions for these edge cases. Verify the Claude Code bug workaround is still needed.

### 7c. Plan-phase revision loop has no termination fallback

`workflows/plan-phase.md` Step 12 says "Max 3 iterations" for plan checker revisions but doesn't specify what happens if the checker keeps finding issues after 3 rounds.

**Recommendation:** Add explicit fallback: present remaining issues to the user and let them decide whether to proceed or abort.

### 7d. Double-negative logic in research flow

`workflows/plan-phase.md` lines 68-72 use "skip if false... without override" which requires careful parsing:
> "Skip if... research_enabled is false... without `--research` override"

**Recommendation:** Rewrite with positive conditions: "Run research if research_enabled is true OR `--research` flag is set."

### 7e. README doesn't document all commands

The README lists most commands but omits `/gsd:cleanup`, `/gsd:reapply-patches`, and `/gsd:research-phase`.

**Recommendation:** Add all 30 commands to the README command reference.

---

## 8. Documentation

### 8a. No contribution guide

The repo has a README, USER-GUIDE, SECURITY, and CHANGELOG, but no CONTRIBUTING.md.

**Recommendation:** Add a CONTRIBUTING.md covering: repo setup, test commands, architecture overview, PR guidelines.

### 8b. Internal code has minimal comments

Library modules have section headers but very few inline comments explaining non-obvious logic (e.g., phase numbering scheme, `must_haves` block parsing, frontmatter conversion). Error messages also lack context — e.g., `error('phase required for init execute-phase')` doesn't explain what the user should do.

**Recommendation:** Add comments to algorithmic sections — especially `comparePhaseNum()`, `parseMustHavesBlock()`, and the frontmatter conversion functions. Include actionable guidance in error messages.

---

## Summary — Top 10 Highest-Impact Improvements

| Priority | Improvement | Category | Impact |
|----------|------------|----------|--------|
| 1 | Add CI pipeline (tests + lint on PRs) | DX | Prevents regressions, builds contributor confidence |
| 2 | Add installer tests | Testing | Catches conversion bugs across 3 runtimes before users do |
| 3 | Split `bin/install.js` into modules | Architecture | Makes the most complex file maintainable |
| 4 | Replace `execSync('git ...')` with `execFileSync` | Security | Eliminates shell injection surface |
| 5 | Standardize error handling patterns | Architecture | Makes behavior predictable, enables `GSD_DEBUG` mode |
| 6 | Implement atomic settings writes | Security | Prevents config corruption on write failure |
| 7 | Extract duplicated phase/plan utilities | Architecture | Reduces 10+ copy-paste sites to 2 shared functions |
| 8 | Fix JSONC parser edge cases | Installer | Prevents silent parse failures on real-world config files |
| 9 | Implement or remove `/gsd:reapply-patches` | Usability | Users are told to run a command that doesn't work |
| 10 | Use proper semver for update checks | Usability | Prevents false update notifications |
