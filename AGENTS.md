# OpenCode Config

This repo defines OpenCode's multi-agent configuration — it is **not a software project**. No build/test/install/dev scripts exist.

## Structure

```
opencode.json   — main config: agents, models, permissions, plugin
tui.json        — TUI settings (scroll speed, mouse, diff style)
agents/         — 10 agent definitions (orchestrator is primary `mode: primary`)
commands/       — 4 lane-based slash commands (quick, medium, full, debug)
skills/         — 10 installed skills
skills-lock.json — skill source pins and hashes
package.json    — plugin dependency (@opencode-ai/plugin 1.14.33)
```

## Key Commands

No `npm run` scripts exist. All commands are OpenCode slash commands defined in `commands/`:

- `/quick` — small/single-file changes (research → plan → implement)
- `/medium` — multi-file changes needing structure (adds review + verify)
- `/full` — broad/risky work (adds adversarial review + approval gate)
- `/debug` — bug fixing with hypothesis-driven flow

All route through `orchestrator-agent` as subtasks.

## 10 Agents & Their Models

| Agent | Model | Permissions |
|---|---|---|
| orchestrator-agent | qwen3.6-plus | read, question; delegates to specialists |
| research-agent | deepseek-v4-flash (Go) | read-only, websearch, webfetch |
| research-fast-agent | deepseek-v4-flash-free (Zen) | read-only, local-only (no web) |
| planning-agent | deepseek-v4-pro | read, edit, bash, can invoke research-agent |
| reviewer-agent | glm-5.1 | read-only + git read, question |
| implementation-agent | kimi-k2.6 | read, edit, bash (allow) — **cannot commit** |
| test-fixer-agent | deepseek-v4-pro | read, edit, bash, question |
| verifier-agent | deepseek-v4-pro | read-only + git read, question |
| pr-review-agent | glm-5.1 | read-only, gh CLI, git read |
| prompt-agent | qwen3.6-plus | read, edit, bash, question |

## Critical Constraints

- **Implementation agent must NOT run `git add`/`commit`/`push`** — leaves all changes unstaged
- **Research agent cannot edit files** — read-only with websearch/webfetch
- **Research-fast agent is local-only** — read-only, no web access, for quick file mapping/scouting
- **Reviewer/Verifier agents are read-only** — bash restricted to `cat`/`grep`/`rg`/git-read commands
- Plans are saved to `plans/{feature-name}/plan.md` by planning-agent
- Global `bash: ask` permission (user must approve shell commands); sub-agents may have overrides
- `edit: allow` at root — agents can modify files per their defined permissions

## Execution Lanes

Decoded in orchestrator-agent.md and each command file. Summary:

- **quick**: skip review/verify for mechanical changes; prefers `research-fast-agent` for light scouting
- **medium**: research → plan → review → implement → verify
- **full**: adds adversarial review pass and explicit `[y/N/edit]` approval gate
- **debug**: triage → research (hypothesis) → plan → review → approval → implement → verify

Context: the orchestrator-agent also distinguishes "Fast Lane" (like `/quick` but without plan file), "Standard Lane" (like `/medium`), and "High-Risk Lane" (like `/full`).

## Plugin & Skills

- Plugin: `@warp-dot-dev/opencode-warp`
- 10 skills installed via skills-lock.json from GitHub sources
- Skills auto-activate based on context; explicit invocation via `@skill-name`
- `caveman` skill provides token compression (caveman-lite/caveman-full) for agent-to-agent handoffs
- Lockfile pins specific commits/hashes — update via `opencode skills update`

## Conventions

- Orchestrator reasons in English, responds conversationally in Spanish (all technical artifacts in English)
- Agents use token compression (caveman) for non-critical prose; never compress code, paths, errors, or safety-critical content
- `.gitignore` covers `opencode.local.json`, `state/`, `sessions/`, `cache/`, `auth*.json`, `credentials*.json`
- No additional instruction files (no CLAUDE.md, no .cursor/rules, no copilot-instructions.md)
