# My Notes: What ECC Is and How Hooks Work

Personal reference notes. Not part of the official ECC docs.

## What this repo is

ECC is a **Claude Code plugin**: a large bundle of configuration that shapes how Claude Code behaves.

| Folder | What it holds | Count |
|---|---|---|
| `agents/` | Specialized subagents (planner, code-reviewer, tdd-guide, ...) | 68 |
| `skills/` | Workflow definitions and domain knowledge | 293 |
| `commands/` | Slash commands (`/plan`, `/tdd`, `/code-review`, `/build-fix`, ...) | 94 |
| `rules/` | Always-follow guidelines (security, style, testing) | 23 |
| `hooks/` | Automations that run on Claude Code events | - |
| `mcp-configs/` | MCP server configurations | - |
| `scripts/` | Cross-platform Node.js utilities that back the hooks | - |
| `tests/` | Test suite for the scripts (`node tests/run-all.js`) | - |

Tech: plain CommonJS on Node 18+, no TypeScript. Agents, skills and commands are Markdown with YAML frontmatter. Hooks are JSON plus Node scripts.

## Does it help?

It depends on how I work, and I have not measured any benefit.

**Where it can help**
- Repeatable workflow: `/plan`, `/tdd` and `/code-review` give Claude a consistent process.
- Guardrails: hooks block risky actions (for example `--no-verify` commits, edits to linter configs).
- Memory across sessions: session start and end hooks save and restore context.
- Preloaded conventions: rules and skills mean fewer repeated explanations.

**Where it can hurt**
- Size: 293 skills and 94 commands is a lot to learn and a lot of context to carry. Most people use about 5.
- Overhead: every tool call passes through hook processes, and some add friction.
- Fit: it targets coding workflows, not marketing, sales, docs or spreadsheets.

**Suggested approach:** do not adopt all of it. Use the `minimal` hook profile, try `/plan`, `/tdd` and `/code-review` on a real task, and keep only what saves time.

## What hooks are

Hooks are small scripts that Claude Code runs automatically when a specific event happens. They run on my machine, outside Claude, and Claude cannot skip them.

Why that matters: "always run the formatter" in a prompt or `CLAUDE.md` is only a request. A hook is a guarantee, because Claude Code itself runs the script.

How it works:
1. A settings file says "when this event happens, run this command."
2. When the event fires, Claude Code runs the command and passes details as JSON on stdin (which tool, which file, and so on).
3. The result can change what happens next:
   - Exit code 0: carry on.
   - Exit code 2: block the action and send the script's message back to Claude.
   - Output can also be fed back to Claude as extra context.

Common events:

| Event | When it fires | Typical use |
|---|---|---|
| `PreToolUse` | Just before a tool runs (Bash, Edit, Write, ...) | Block or warn |
| `PostToolUse` | Just after a tool runs | Auto-format, lint, typecheck |
| `PostToolUseFailure` | After a tool fails | Track failures |
| `SessionStart` / `SessionEnd` | Session begins or ends | Load or save context |
| `Stop` | Claude finishes responding | Notifications, evaluation, cost tracking |
| `PreCompact` | Before the conversation is summarized | Save state |

## How hooks work in this repo

Three layers:

### 1. Registration: `hooks/hooks.json`

Claude Code's native format: an event, a `matcher` on the tool name, and a command. Each command is a long inline `node -e` snippet that finds the ECC install (via `CLAUDE_PLUGIN_ROOT`, then `~/.claude`, then plugin and marketplace caches) and hands off to `plugin-hook-bootstrap.js`. That is why the hooks work wherever the plugin is installed.

Examples of what is registered:
- `PreToolUse`: `pre-bash-dispatcher` (Bash), `doc-file-warning` (Write), `config-protection` (Write/Edit), `gateguard-fact-force` (Edit/Write/PowerShell), `mcp-health-check` (`^mcp__`), `observe-runner` (every tool), `governance-capture`.
- `PostToolUse`: a dispatcher on every tool, plus an async one.
- `PostToolUseFailure`: `mcp-health-check`, `skill-run-tracker`.
- `PreCompact`: `pre-compact`.
- `SessionStart`: `session-start-bootstrap`, `plan-canvas-sessions`.
- `Stop` / `SessionEnd`: session persistence, evaluation, cost tracking, metrics, notifications.

### 2. The gate: `scripts/hooks/run-with-flags.js`

Most hooks go through this wrapper. Arguments: `<hookId> <scriptPath> [profiles]`.

What it does:
1. Reads stdin with a size cap (`ECC_HOOK_INPUT_MAX_BYTES` overrides the default).
2. Checks whether the hook is enabled (see controls below). If not, it exits 0 and Claude Code carries on.
3. In dry-run mode, prints a `[DryRun]` line to stderr and does not run the hook.
4. Rejects any script path that resolves outside the plugin root (path traversal guard).
5. Runs the hook. If the script exports `run(rawInput)`, it is `require()`d directly, which saves a process spawn of about 50-100ms. Legacy scripts are spawned as child processes.
6. Passes the hook's output back. A hook can return stdout, stderr and an exit code, or `additionalContext` that is fed back to Claude.

Failure behavior:
- Most hooks **fail open**: errors, missing scripts and malformed input all exit 0, so the tool call proceeds.
- Security-sensitive hooks (`gateguard-fact-force`, `mcp-health-check`) **fail closed** on truncated input and exit 2 to block. Opt out with `ECC_GATEGUARD=off` or `ECC_MCP_HEALTH_FAIL_OPEN=1`.

### 3. Runtime controls: `scripts/lib/hook-flags.js`

| Variable | Effect |
|---|---|
| `ECC_HOOKS_ENABLED=true\|false` | Turn all hooks on or off (default on) |
| `ECC_HOOK_PROFILE=minimal\|standard\|strict` | Pick a profile (default `standard`). A hook runs only if its profile list includes the active one. |
| `ECC_DISABLED_HOOKS=id1,id2` | Disable specific hooks by ID |

If these are unset, it falls back to Claude plugin options, then to a managed `ecc/setup.json`.

### The hook scripts: `scripts/hooks/`

- **Dispatchers** (`pre-bash-dispatcher`, `post-bash-dispatcher`, `posttooluse-dispatcher`) fan one event out to several checks in a single process.
- **Guards**: `block-no-verify`, `config-protection`, `gateguard-fact-force`.
- **Observers**: `observe-runner`, `governance-capture`, `cost-tracker`, `ecc-metrics-bridge`, `session-activity-tracker`.
- Shared helpers live in `scripts/lib/`.

## Conventions when writing hooks here (from `.claude/rules/node.md`)

- Always exit 0 on non-critical errors. A hook must never block work by accident.
- Log to stderr with a `[HookName]` prefix.
- Blocking hooks (PreToolUse, Stop) must be fast: under 200ms, no network calls.
- Async hooks need `"async": true` and a timeout of 30s or less.
- Route every hook through `run-with-flags.js` so profile and disable controls work.
- Keep hook scripts under 200 lines; move helpers to `scripts/lib/`.
- New scripts in `scripts/lib/` need a test in `tests/lib/`. New hooks need an integration test in `tests/hooks/`.
- Before committing: `node tests/run-all.js` and `npx markdownlint-cli '**/*.md' --ignore node_modules`.

## Quick reference

```bash
node tests/run-all.js            # run all tests
node tests/hooks/hooks.test.js   # hook tests only
ECC_HOOK_PROFILE=minimal         # lighter hook set
ECC_DISABLED_HOOKS=pre:observe   # turn off one hook by ID
```
