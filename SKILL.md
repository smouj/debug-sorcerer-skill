---
name: debug-sorcerer
version: 1.0.0
description: AI-powered debugging assistant that traces errors and generates fixes for JavaScript, TypeScript, Python, and Go.
author: FlickClaw Engineering
homepage: https://kilocode.ai/docs/debug-sorcerer
tags:
  - debugging
  - ai
  - code-analysis
  - developer-tools
license: MIT
dependencies:
  - kilocode-cli>=1.0.0
  - python>=3.8
  - openai>=4.0.0 | anthropic>=0.5.0
  - git
---

# Debug Sorcerer

## Purpose

Debug Sorcerer is an advanced CLI skill that leverages large language models to autonomously debug code. When an error occurs, the skill captures the full context (stack trace, logs, environment), performs deep semantic analysis of the relevant source files, identifies the likely root cause, and generates a minimal patch to resolve the issue. It is designed for developers who want to accelerate bug resolution in JavaScript, TypeScript, Python, and Go projects. The skill integrates with existing git workflows and test suites, ensuring that fixes are safe, reversible, and verified before being applied.

Specific use cases:

- **Crash debugging**: Quickly diagnose runtime errors like `TypeError`, `ReferenceError`, `undefined is not a function`, etc.
- **Flaky test analysis**: When tests fail intermittently, the skill captures the failure context and hypothesizes race conditions or timing issues.
- **Performance bottleneck tracing**: Identify code patterns that cause slowdowns (e.g., unnecessary re-renders, blocking calls).
- **Security vulnerability scanning**: Detect common security anti-patterns such as unvalidated inputs, hardcoded secrets, and unsafe API usage.
- **Code review assistance**: During PR review, the skill can be asked to analyze specific changed files for potential bugs.

## Scope

The skill provides the following CLI commands:

1. `debug-sorcerer trace <error_message> [OPTIONS]`
   - Required: `error_message` (string or file path with `@` prefix, e.g., `@error.txt`)
   - Options:
     - `--file <path>`: Path to the source file where error occurred (if not inferable)
     - `--line <number>`: Line number of the error
     - `--context N`: Include N lines before and after error line (default: 10)
     - `--project-root <dir>`: Override project root detection (default: current directory)
     - `--ai-model <model>`: Specify AI model (e.g., `gpt-4`, `claude-3-opus-20240229`). Default: configured in global settings.
   - Example: `debug-sorcerer trace "UnhandledPromiseRejectionWarning: TypeError: Cannot read property 'id' of undefined" --file src/controllers/user.js --line 45`

2. `debug-sorcerer analyze <file_path> [OPTIONS]`
   - Analyzes a file for potential bugs, anti-patterns, and vulnerabilities.
   - Options:
     - `--checks <comma-separated>`: Limit analysis to specific checks (e.g., `null-safety,async,security`). Default: all enabled.
     - `--output <format>`: Output format (`json`, `text`, `sarif`). Default: `text`.
     - `--severity-threshold <level>`: Minimum severity to report (`low`, `medium`, `high`, `critical`). Default: `low`.
   - Example: `debug-sorcerer analyze src/utils/api.js --checks security --output json > api-issues.json`

3. `debug-sorcerer fix <issue_identifier> [OPTIONS]`
   - Generates a patch for a previously identified issue (from `trace` or `analyze`).
   - Options:
     - `--dry-run`: Show diff without applying changes.
     - `--apply`: Apply fix immediately (use with caution).
     - `--auto-commit`: After successful verification, commit changes automatically.
     - `--branch <name>`: Create a new branch for the fix. Default: `debug-sorcerer/fix-<timestamp>`.
   - Example: `debug-sorcerer fix 7f3a2b --dry-run`

4. `debug-sorcerer test [OPTIONS]`
   - Executes the project's test suite with AI monitoring to catch and diagnose failures.
   - Options:
     - `--watch`: Continuously monitor file changes and re-run tests.
     - `--auto-fix`: Automatically attempt to fix failing tests after each run.
     - `--only-failing`: Only run tests that failed in previous run.
     - `--reporter <name>`: Test reporter to use (e.g., `spec`, `dot`, `json`). Default: project default.
   - Example: `debug-sorcerer test --watch --auto-fix`

5. `debug-sorcerer verify <fix_identifier>`
   - Verifies that a previously applied fix resolves the original issue.
   - Options:
     - `--re-run-tests`: Force re-execution of all tests.
     - `--manual`: Pause for manual validation steps.
   - Example: `debug-sorcerer verify 7f3a2b --re-run-tests`

6. `debug-sorcerer report <format> [OPTIONS]`
   - Generates a comprehensive report of the debugging session.
   - Formats: `html`, `json`, `markdown`, `sarif`.
   - Options:
     - `--output <file>`: Write report to file instead of stdout.
     - `--include-diffs`: Include code diffs in the report.
     - `--session <id>`: Generate report for a specific session (default: latest).
   - Example: `debug-sorcerer report html --include-diffs --output debug-session-2024-01-15.html`

7. `debug-sorcerer config [OPTIONS]`
   - Manage skill configuration.
   - Subcommands: `get <key>`, `set <key> <value>`, `list`, `reset`.
   - Example: `debug-sorcerer config set ai-model gpt-4-turbo`

## Work Process

When a user invokes `debug-sorcerer trace`:

1. **Context Collection**:
   - The skill reads the error message. If the error comes from a file, it loads the file content (and surrounding context lines).
   - It also scans the project for related files (imports, dependencies of the module) to build a call graph.
   - Environment variables from the running process are captured if available (e.g., `NODE_ENV`, `DATABASE_URL`).

2. **Initial AI Analysis**:
   - Constructs a prompt to the AI model containing:
     - The error message and stack trace.
     - The relevant source code snippets (including imports and adjacent functions).
     - A request to identify the root cause and propose 2-3 candidate fixes ranked by confidence.
   - The prompt includes instructions to avoid suggesting changes to third-party libraries or frameworks (only user code).
   - Example internal prompt: `Given the following error in a Node.js Express app: ... Identify the bug and propose fixes.`

3. **Hypothesis Generation**:
   - The AI returns a structured response (JSON) with fields: `root_cause`, `confidence`, `affected_lines`, `suggested_fix`, `side_effects`.
   - The skill parses the response and presents the top hypothesis to the user in a human-friendly format.
   - If confidence is below 0.7, it also shows alternative hypotheses.

4. **Fix Validation**:
   - Before presenting the fix, the skill performs a simulation using Abstract Syntax Tree (AST) parsing to ensure the suggested code is syntactically valid and that the modified code still conforms to the project's ESLint/Prettier rules.
   - It also checks that the fix does not introduce new undefined variables or break import statements.

5. **User Interaction**:
   - The skill prompts the user: `Proposed fix for [issue]? [Y]es, [N]o, [A]lternative, [M]anual edit`. If `--auto` is set, it auto-accepts if confidence > 0.85.
   - If user selects Alternative, the skill fetches the next hypothesis and repeats.
   - If user selects Manual edit, the skill opens the file in the user's configured editor (e.g., `$EDITOR`) with the problematic line highlighted.

6. **Patch Application**:
   - Upon acceptance, the skill creates a git branch (if not already on a branch) named `debug-sorcerer/fix-<timestamp>`.
   - It applies the patch using `git apply` or direct file edit, preserving file permissions and encoding.
   - It records the fix metadata in `.debug-sorcerer/fixes.json` including original issue, applied patch, AI model used, and timestamp.

7. **Verification**:
   - The skill automatically runs the test suite using the project's test runner (detected from `package.json`, `pytest.ini`, `go.mod`, etc.).
   - It captures test output. If tests pass, it marks the fix as `verified`.
   - If tests fail, it attempts to analyze the failure trace to determine if the fix caused a regression. If so, it suggests a rollback or alternative fix.
   - In `--auto-commit` mode, it commits with message: `fix: <auto-generated description> [debug-sorcerer]` and opens a PR if configured.

8. **Reporting**:
   - Generates a session log summarizing the steps, AI interactions, and final outcome.
   - Optionally produces a detailed report in the chosen format.

## Golden Rules

1. **Never Lose Data**: All modifications are preceded by a git commit or stash. If the project is not under version control, the skill fails with an error prompting to initialize a git repo.
2. **User Consent for Destructive Actions**: Committing changes, creating branches, or deleting files requires explicit `--auto` flag or manual confirmation.
3. **Respect Project Conventions**: The skill reads the project's ESLint, Prettier, and formatting configurations and ensures that generated code matches existing style.
4. **Minimal Change Principle**: Patches should only change lines necessary to fix the bug. No refactoring unless explicitly requested via `--refactor`.
5. **No External Secrets**: The skill never logs or stores API keys, passwords, or connection strings. It masks such values in error messages before sending to AI.
6. **Reproducibility**: Every fix includes a `debug-sorcerer-id` in the commit message and in a comment near the change (if language supports comments) for traceability.
7. **Gradual Escalation**: The skill starts with non-invasive analysis. Only after user approval does it write to files.
8. **Language Safety**: For compiled languages (Go, Rust), the skill ensures code compiles before applying fix.
9. **Performance Awareness**: Avoids analyzing huge files (>10K lines) by default; requires `--force` to override.
10. **No Overriding Manual Fixes**: If a developer manually fixes a bug after the AI has suggested something, the skill detects this and avoids duplicate efforts.

## Examples

**Example 1: Trace a runtime error from a log file**

```bash
$ debug-sorcerer trace "Error: Cannot find module 'axios' in src/services/api.js:12" --file src/services/api.js --line 12
```
Output:
```
[Debug Sorcerer] Collecting context...
[Debug Sorcerer] The error indicates that Node.js cannot resolve the module 'axios'. This is typically caused by:
  1. The module is not installed (missing dependency)
  2. The import statement has a typo
  3. The file path is incorrect

Analyzing src/services/api.js around line 12...
  10: import fetch from 'node-fetch';
  11: // import axios from 'axios';  // <- this line is commented out
  12: const apiClient = axios.create({ baseURL: process.env.API_URL });

Root cause: The module 'axios' is not installed and the import line is commented out.
Confidence: 0.95

Suggested fix:
  Option A: Install axios: `npm install axios` and uncomment the import.
  Option B: Replace axios usage with node-fetch (if appropriate).

Which option? [A/B] (default A):
```
If user selects A, the skill runs `npm install axios` and uncomments line 11, then verifies by running tests.

**Example 2: Analyze a file for security issues**

```bash
$ debug-sorcerer analyze src/auth/password.js --checks security --output json > password-audit.json
```

Output (password-audit.json):
```json
{
  "file": "src/auth/password.js",
  "issues": [
    {
      "severity": "high",
      "line": 23,
      "message": "Hardcoded secret detected: 'super-secret-password'",
      "suggestion": "Use environment variable: process.env.ADMIN_PASSWORD"
    },
    {
      "severity": "medium",
      "line": 45,
      "message": "Weak hashing algorithm: MD5",
      "suggestion": "Use bcrypt or Argon2 instead"
    }
  ]
}
```

**Example 3: Auto-fix failing tests in watch mode**

```bash
$ debug-sorcerer test --watch --auto-fix
```
The skill runs `npm test`. Suppose a test fails with "Expected 200 but got 500". The skill:
- Identifies the failing test file.
- Analyzes the test and implementation.
- Suggests adding a null check.
- Prompts: "Fix detected: add guard at src/controllers/user.js:27. Apply? [Y/n]" (auto-accepted if `--auto`).
- Applies fix and re-runs tests. If they pass, it continues watching.

## Rollback Commands

If a fix introduces a regression or is unsatisfactory, the user can rollback:

1. **Rollback a specific fix**:
   ```bash
   $ debug-sorcerer rollback <fix_id>
   ```
   where `<fix_id>` is the identifier shown after applying a fix (e.g., `20240115-143022`). This reverts the changes made by that fix, using git if available or restoring from backup.

2. **Rollback entire session**:
   ```bash
   $ debug-sorcerer rollback --session <session_id>
   ```
   This undoes all fixes applied during a particular debugging session.

3. **Manual git rollback** (if you prefer):
   ```bash
   $ git log --oneline --grep="\[debug-sorcerer\]"  # find commit(s)
   $ git revert <commit_sha>   # revert one fix
   $ git reset --hard HEAD~N   # revert last N commits (if not pushed)
   ```

4. **Cleanup**:
   The skill also offers `debug-sorcerer cleanup` to remove temporary files and branches older than a threshold. By default it keeps 30 days.

## Verification Steps

After applying a fix, verify it's correct:

1. **Re-run the exact scenario** that triggered the error. If the error no longer appears, proceed to step 2.
2. **Run the full test suite**:
   ```bash
   $ npm test   # or pytest, go test, etc.
   ```
   Ensure all tests pass, especially regression tests.
3. **Perform manual functional testing** of the affected feature to confirm behavior is as expected.
4. **Run `debug-sorcerer verify <fix_id>`** to let the skill automatically re-run tests and check for residual errors.
5. **Check metrics** (if applicable) like performance, memory usage, etc. to ensure no degradation.

If any verification fails, immediately rollback and report the issue to the skill's maintainers (by default it opens a GitHub issue).

## Troubleshooting

| Problem | Likely Cause | Solution |
|---------|--------------|----------|
| `Error: No AI API key configured` | Missing `OPENAI_API_KEY` or `ANTHROPIC_API_KEY`. | Set the environment variable or run `debug-sorcerer config set api-key <your_key>`. |
| `Error: Project not a git repository` | The skill needs git for safety but no `.git` found. | Initialize git: `git init` and commit current state. Or use `--no-git` (not recommended). |
| `AI model quota exceeded` | API limits reached. | Wait or upgrade your plan. Switch model with `debug-sorcerer config set ai-model gpt-3.5-turbo` (cheaper). |
| `Analysis too slow on large file` | File exceeds default 5000 lines. | Use `--force` to analyze anyway, or split file. Consider writing a unit test to isolate the problematic part. |
| `Fix conflicts with existing changes` | The file was edited after the trace was captured. | Stash changes before running fix, or resolve merge conflicts manually. The skill will detect and abort. |
| `Unsupported language` | The file extension is not recognized. | Ensure the file is one of the supported languages (js, ts, py, go, rs). Contribute an extension if missing. |
| `Test runner not detected` | The skill cannot find the test script. | Specify with `--test-command "pytest -v"` or set `debug-sorcerer.config.testCommand`. |
| `False positive root cause` | AI misinterpreted the code. | Use `--manual` to provide hints, or adjust the prompt via `config set prompt-override`. Report the issue with the context. |

## Dependencies

- **Kilocode CLI**: Required to run the skill. Install with `npm install -g @kilocode/cli` or via Homebrew.
- **Python**: The skill's core runtime is in Python 3.8+ (required for some AST manipulation). Ensure `python3` is in PATH.
- **AI Provider**: Either OpenAI API (`openai>=4.0.0`) or Anthropic API (`anthropic>=0.5.0`). An API key must be set in `OPENAI_API_KEY` or `ANTHROPIC_API_KEY`.
- **Git**: Recommended for version control safety. If not present, the skill operates in read-only mode (no fixes applied).
- **Node.js / npm / yarn / pnpm**: For JavaScript/TypeScript projects, the appropriate package manager must be available to install missing dependencies suggested by the skill.
- **Language-specific tools**: For Python, `pytest` or `tox`; for Go, `go test`; for Rust, `cargo test`. These must be installed and in PATH.

## Environment Variables

- `DEBUG_SORCERER_API_KEY`: Override the AI API key. If set, takes precedence over global config.
- `DEBUG_SORCERER_LOG_LEVEL`: Controls logging verbosity. Options: `DEBUG`, `INFO`, `WARNING`, `ERROR`. Default: `INFO`.
- `DEBUG_SORCERER_WORKDIR`: Project root directory. Default: current working directory.
- `DEBUG_SORCERER_AI_MODEL`: Default AI model to use. Overrides config. Example: `gpt-4-turbo-preview`.
- `DEBUG_SORCERER_NO_GIT`: If set to `true`, disables git safety checks (use with caution).
- `DEBUG_SORCERER_DRY_RUN`: If set to `true`, never apply any changes automatically; always prompt.
- `DEBUG_SORCERER_TIMEOUT`: Timeout in seconds for AI requests. Default: `60`.

Optional: `DEBUG_SORCERER_DISABLE_TELEMETRY` to opt-out of usage statistics.
```