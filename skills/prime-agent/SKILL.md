---
name: prime-agent
description: "Delegate coding to Prime Agent with verification."
version: 0.2.0
author: Daniel Gómez
license: MIT
platforms: [linux, macos, windows]
metadata:
  hermes:
    tags: [Coding-Agent, Prime-Agent, RLM, Subagents, Autonomous, Long-Running, Verification]
    related_skills: [claude-code, codex, opencode, hermes-agent]
---

# Prime Agent — Hermes Orchestration Guide

Delegate coding and research work to [Prime Agent](https://github.com/PrimeIntellect-ai/prime-agent), Prime Intellect's RLM-native coding agent. Prime Agent uses a persistent IPython control environment, native recursive subagents, daemon-backed sessions, durable goals, schedules, and bounded autonomous continuations.

This skill covers the standalone `prime-agent` CLI as an external worker. It does not replace Hermes `delegate_task`, and it does not make Prime Agent a security sandbox: its workers and kernels normally run with the same operating-system permissions as the invoking user.

## When to Use

- The user explicitly asks to use Prime Agent.
- A coding task benefits from a persistent or detachable agent session.
- A task needs Prime Agent's native `rlm(...)` child-agent delegation.
- A long task needs bounded autonomous continuations with explicit quality gates.
- You need Prime Agent's JSON or RPC mode for headless integration.

Do not use this skill for a short analysis that Hermes can answer directly. Do not use one shared checkout for concurrent implementation agents; use separate worktrees or serialize edits.

## Prerequisites

- Prime Agent installed and available as `prime-agent`.
- Authentication configured through Prime Agent `/login`, its auth file, or an environment variable already present in the process environment. Never place API keys in prompts, command history, or skill text.
- A deliberate `workdir` pointing at the target repository.
- The target repository's `AGENTS.md`, `CLAUDE.md`, branch policy, and local workflow read before implementation.
- A clean or understood Git state before launching an implementation worker.

Check readiness through the Hermes `terminal` tool:

```text
terminal(command="prime-agent --version")
terminal(command="prime-agent status")
terminal(command="prime-agent doctor")
```

Install only when the user has requested setup or Prime Agent is the selected worker and is absent:

```text
terminal(command="curl -fsSL https://app.primeintellect.ai/prime-agent/install.sh | sh", timeout=120)
```

The public installer currently targets Linux and macOS. A source checkout requires Node.js 22.8.0 or newer, followed by `npm ci` and `./prime-agent.sh`.

## Operating Model

Prime Agent exposes one primary model-facing tool: a persistent IPython kernel. File operations, project commands, skills, and delegation are normally performed from that kernel. The TypeScript host owns provider calls, session state, scheduling, and child-agent lifecycles.

### Recursive subagents

The preloaded `rlm` callable admits a child and returns immediately with a handle; it does not return the child's answer. Children must send requested findings to the parent with `agent_message`, or write an artifact that the parent can inspect.

Use prompts that specify the scope, repository path, read-first files, edit permission, expected artifact, and reply channel:

```python
review = await rlm(
    "Review authentication only. Read AGENTS.md first. Do not edit files. "
    "Return findings to the parent with await agent_message.send(..., receiver_role='parent').",
    name="auth-reviewer",
)
tests = await rlm(
    "Inspect missing regression tests. Do not modify production code. "
    "Reply to the parent with file:line evidence.",
    name="test-reviewer",
)
```

The parent should end or continue its turn according to the requested workflow; it must not treat the admission handle as the child result. Recover retained children with `await rlm.list_subagents()` and follow up with `agent_message.send(..., receiver_role="child", receiver_name=<name>)`. Delete a child only when its context is no longer needed with `await rlm.delete_subagent(selector)`; deletion closes or cancels the runtime and writes a tombstone but does not erase transcript artifacts.

Children inherit the parent model, provider configuration, skills, tools, and session machinery. They run under the same root worker and operating-system permissions. Use child agents primarily for independent analysis, review, test discovery, or narrowly scoped sequential work unless separate worktrees have been established.

When a child needs another model, discover an exact available selector before spawning:

```python
models = await rlm.find_models("coding", limit=8)
handle = await rlm(
    "Review the API and report file:line findings to the parent.",
    name="api-reviewer",
    model="provider/exact-model-id",
)
```

The `model` value must be an exact available `provider/model` selection. If it is unavailable or fails authentication preflight, Prime Agent fails the spawn instead of silently substituting another model. The default maximum recursion depth is 1: root agents may create children, but children do not create grandchildren unless configured otherwise.

## One-Shot Delegation

Use print mode for a bounded task. Always set `workdir` through Hermes rather than changing directories in an unrelated shell:

```text
terminal(
  command="prime-agent -p 'Inspect the repository, implement the requested fix, run the targeted checks, and report exact files and commands. Do not commit or push.'",
  workdir="/path/to/project",
  timeout=600,
)
```

Use `@` file arguments when explicit context is useful:

```text
terminal(
  command="prime-agent -p @README.md @AGENTS.md 'Review the implementation plan and identify risks. Do not edit files.'",
  workdir="/path/to/project",
  timeout=180,
)
```

For model selection and reasoning limits:

```text
terminal(
  command="prime-agent -p --provider openai --model openai/gpt-5.1-codex --thinking high 'Implement and verify the change.'",
  workdir="/path/to/project",
  timeout=600,
)
```

Use the configured Prime Agent auth flow or environment rather than `--api-key` in a command. Prime Agent also supports `--offline` for startup operations when a locally configured model is available; this does not make inference offline.

## Long-Running Sessions

Interactive sessions are daemon-backed. Closing the TUI detaches the client but does not necessarily stop the worker. Start an interactive worker in the background only when multi-turn interaction or later reattachment is needed:

```text
terminal(
  command="prime-agent 'Work through the migration in small verified steps; do not commit.'",
  workdir="/path/to/project",
  background=true,
  pty=true,
)
```

The Hermes `terminal` result provides a process session ID. Monitor it with `process(action="poll", session_id="<id>")` or `process(action="log", session_id="<id>")`. Send a follow-up through `process(action="submit", session_id="<id>", data="...")` only when the agent is asking for input or the workflow explicitly requires steering.

Prime Agent's daemon-level lifecycle commands are useful after the terminal client detaches:

```text
terminal(command="prime-agent list --all")
terminal(command="prime-agent agents")
terminal(command="prime-agent attach <agent>")
terminal(command="prime-agent send <agent> 'Re-run the targeted checks and report failures.'")
terminal(command="prime-agent stop <agent>")
terminal(command="prime-agent status")
terminal(command="prime-agent doctor")
```

Use `prime-agent shutdown --force` only when the user explicitly asks to stop all Prime Agent workers and services. Do not kill a slow worker merely because it has not produced output; inspect logs first.

Resume saved work with `prime-agent -c` for the most recent session or `prime-agent -r [path|id]` for a selected session. Sessions normally persist under Prime Agent's session directory and can outlive the terminal UI.

## Durable Coordination

Use Prime Agent's coordination features when work must continue after the current prompt or terminal disconnect. Keep these distinct:

- **RLM children:** focused subagents created from a parent through `rlm(...)`.
- **Direct messages:** communication between active root agents or retained children.
- **Heartbeats:** recurring prompts attached to a session.
- **Schedules:** one-time or cron prompts owned by an addressable agent.
- **Goals:** durable objectives that remain active until completed, paused, budget-limited, errored, or cleared.
- **Autonomous mode:** bounded host-injected continuations for unattended execution.

### Agent messaging

From the shell, address a running agent:

```text
terminal(command="prime-agent send <agent> 'Recheck the endpoint after the latest edit.'")
```

From IPython, inspect the roster and choose an explicit delivery mode:

```python
roster = await agent_message.list_agents()
receipt = await agent_message.send(
    "Recheck the endpoint after the latest edit.",
    receiver_role="sibling",
    receiver_name="api-reviewer",
    mode="auto",  # auto, steer, or follow_up
)
print(receipt["deliveryStatus"])
```

Use `steer` only when the new instruction should enter active work immediately; use `follow_up` when it must wait for the current work to finish. Broadcast only when every agent in the family roster should receive the message. A `delivered` or `queued` receipt proves delivery state, not task completion.

### Heartbeats and schedules

Use a user heartbeat for one visible recurring instruction:

```text
/heartbeat every 10m Check the deployment and report meaningful changes
/heartbeat status
/heartbeat pause
/heartbeat resume
/heartbeat clear
```

For several agent-owned recurring checks, use the `rlm_heartbeat` Python skill:

```python
heartbeat = await rlm_heartbeat.create(
    "Check whether the test run finished and report failures.",
    interval="5m",
    label="tests",
    delivery_mode="follow_up",
)
await rlm_heartbeat.list()
await rlm_heartbeat.update(heartbeat["heartbeat"]["id"], status="pause")
```

Use `prime-agent schedule` for persistent one-time or cron prompts:

```text
terminal(command="prime-agent schedule add worker 'in 30m' -- 'Check the benchmark result.'")
terminal(command="prime-agent schedule add worker '0 9 * * 1-5' -- 'Review open work.'")
terminal(command="prime-agent schedule list --all")
terminal(command="prime-agent schedule cancel <job-id>")
```

Scheduled work persists while the UI is detached. Due ticks are claimed before delivery and missed ticks are coalesced, so a schedule is not a substitute for verifying every run.

### Persistent goals

Create a goal only when the user wants the objective retained across ordinary turns:

```text
/goal Ship the release and verify every published artifact
/goal --budget 200000 Complete the repository migration
/goal status
/goal pause
/goal resume
/goal clear
```

The kernel-side `goal` skill can inspect or complete the state:

```python
state = await goal.get()
await goal.complete()
```

Only `goal.complete()` marks successful completion. A goal's token budget, elapsed time, continuation count, and error state must be reported separately from autonomous-mode limits.

## Bounded Autonomous Mode

Autonomous mode is a host policy, not proof of completion. Bound it with a quality gate and resource limits:

```text
terminal(
  command="prime-agent -p --autonomous --autonomous-gate 'npm run check' --autonomous-gate-retries 2 --autonomous-gate-timeout-ms 300000 --autonomous-max-continuations 3 --autonomous-max-turns 12 --autonomous-max-tokens 80000 --autonomous-timeout-ms 1800000 'Implement the requested change, run the gate, and report verified evidence. Do not commit or push.'",
  workdir="/path/to/project",
  timeout=2100,
)
```

Use the repository's documented check instead of assuming `npm run check`. Repeat `--autonomous-gate` when multiple gates are required. The documented defaults are 3 continuations, 12 assistant turns, 80,000 accumulated tokens, and 30 minutes; set them explicitly for reproducibility.

A failed gate supplies bounded output to the next continuation. Prime Agent avoids rerunning an unchanged failed gate. Passing a gate allows completion, but reaching a continuation, turn, token, or time limit does not imply that the task succeeded. Hermes must still inspect the real workspace and test output.

Persistent goals are separate from autonomous mode. Use `--goal` only when the user wants an objective retained across turns; autonomous mode controls whether the host injects another continuation.

## Continuity, Harness, and Session Branches

Use the persistent harness to make repeated workflows improve without mutating Prime Agent's immutable base system prompt. `/refine` reviews the current trajectory and can create, update, delete, or roll back supplemental prompt notes, memories, reusable skill descriptions, and subagent specifications. Treat each refinement as a small evidence-backed change and inspect the resulting state; do not use it as a substitute for packaging executable functionality.

```text
/refine Review this run and record only the reusable lesson about the failed migration check.
/refine status
```

Use compaction deliberately on long tasks:

```text
/usage
/context
/compact Preserve the failing tests, exact commands, decisions, and remaining migration steps.
```

Auto-compaction is normally enabled. It summarizes older messages while retaining recent context and the persistent IPython namespace; it is not a completion signal and does not stop children, schedules, heartbeats, goals, or autonomous continuations. When changing branches with `/tree`, accept or provide a focused branch summary so important abandoned-path context survives navigation.

Use session trees to explore alternatives without losing the original path:

```text
/tree
/fork
/clone
/name auth-refresh-experiment
/session
/export /path/to/session.html
/share
```

- `/tree` navigates inside the same JSONL session and can summarize the abandoned branch.
- `/fork` creates a new session from an earlier user message.
- `/clone` duplicates the current active branch into a new session file.
- `/export` creates a local HTML record; `/share` uploads a private GitHub gist and therefore requires explicit user intent.
- `/usage` and `/context` expose token, cost, context, parent, and child information for budget decisions.

After adding or changing context files, skills, extensions, prompts, or themes, use `/reload`; start a fresh session when a Python-backed skill needs kernel dependency installation.

## JSON and RPC Integration

Use JSON event mode when Hermes or a wrapper needs machine-readable lifecycle output:

```text
terminal(
  command="prime-agent --mode json 'Analyze the repository and report findings.'",
  workdir="/path/to/project",
  timeout=600,
)
```

JSON mode emits JSONL events such as `session`, `agent_start`, `turn_start`, `tool_execution_*`, `turn_end`, and `agent_end`. Parse the stream rather than scraping TUI text. The command's exit status and subsequent workspace checks remain authoritative.

Use RPC mode for a persistent stdin/stdout integration:

```text
terminal(
  command="prime-agent --mode rpc",
  workdir="/path/to/project",
  background=true,
  pty=false,
)
```

RPC uses strict LF-delimited JSONL. `prompt`, `steer`, `follow_up`, and `abort` are the main control commands. A successful RPC response means the command was accepted or queued, not that the coding work finished; continue consuming events and verify the workspace afterward. Do not use a generic line reader that splits Unicode line separators as record delimiters.

Use ACP when an external editor or Agent Client Protocol harness should drive Prime Agent interactively:

```text
terminal(
  command="prime-agent --mode acp",
  workdir="/path/to/project",
  background=true,
  pty=false,
)
```

ACP uses JSON-RPC 2.0 over newline-delimited JSON. Its standard methods include `initialize`, `session/new`, `session/prompt`, `session/cancel`, and `session/close`. One ACP connection owns one session and one working directory; start another process for another isolated session. Prime Agent-specific subagent, autonomous, goal, heartbeat, refinement, compaction, and rich-IPython state appears in the reverse-domain `_meta` envelope. Do not write diagnostics to ACP stdout.

Use the Node.js SDK instead of spawning a CLI when embedding Prime Agent in a TypeScript/Node application. The SDK exposes `createAgentSession`, `AgentSession`, `AgentSessionRuntime`, events, prompt queues, resources, tools, extensions, skills, and run modes. Keep SDK embedding separate from shell delegation and verify the same workdir, session lifecycle, provider, and event completion semantics.

## Context and Skills

Prime Agent loads `AGENTS.md` and `CLAUDE.md` from its global and repository-parent context. Treat those files as executable project policy: read them before delegating, and ensure the prompt names the relevant check and branch restrictions.

Prime Agent discovers skills from `~/.prime/agent/skills/`, `.prime/agent/skills/`, `.agents/skills/`, packages, settings, or explicit `--skill` paths. It supports the Agent Skills markdown format and Python-backed skills. Skills are loaded progressively, and `/reload` rediscoveries changes during an interactive session.

For a project-specific Prime Agent skill, use `.prime/agent/skills/<name>/`. For a personal skill, use `~/.prime/agent/skills/<name>/`. Review third-party skills before loading them: Prime Agent's kernel and worker are not a security sandbox.

## Extensibility and Resource Packages

Choose the smallest customization surface that solves the problem:

- **Markdown skill:** reusable instructions and workflow guidance.
- **Python-backed skill:** an importable callable in the persistent IPython kernel; the package includes `SKILL.md`, `pyproject.toml`, and `src/<import_name>/__init__.py`.
- **Extension:** a TypeScript module for custom tools, event interception, permission gates, commands, shortcuts, providers, UI, rendering, or persistent extension state.
- **Prompt template:** reusable prompt expansion.
- **Theme:** terminal presentation.
- **Package:** a distributable npm/git bundle of extensions, skills, prompts, and themes.

Use the built-in skill creator explicitly with `/skill:skill-creator <request>` when creating new Prime Agent skills, and use `/reload` after adding resources. For extensions, use project-local `.prime/agent/extensions/` or global `~/.prime/agent/extensions/`; test a one-off with `-e` before making it persistent. Extensions and Python skills execute with full user permissions.

Manage resource packages explicitly:

```text
terminal(command="prime-agent package install npm:@scope/package@1.2.3")
terminal(command="prime-agent package install git:github.com/user/repo@v1 --local")
terminal(command="prime-agent package list")
terminal(command="prime-agent package update")
terminal(command="prime-agent package remove npm:@scope/package")
terminal(command="prime-agent update")
terminal(command="prime-agent config")
```

Pin third-party package versions for reproducibility, use `--local` for project settings that should travel with a repository, and review package source before installation. A package may run arbitrary TypeScript, install npm dependencies, expose model-callable tools, or instruct the model to execute commands.

For provider/model customization, prefer the documented `/login`, `models.json`, settings, or a trusted extension. An extension can register a provider, but credentials must remain in the host's auth/configuration boundary and never be embedded in a skill or prompt.

## Runtime Configuration and Observability

Prime Agent merges global `~/.prime/agent/settings.json` with project `.prime/agent/settings.json`; project settings override global settings. Use `/settings` for interactive changes and keep shared project policy in the repository only when intended.

Useful settings include:

```json
{
  "defaultProvider": "openai",
  "defaultModel": "openai/gpt-5.1-codex",
  "defaultThinkingLevel": "high",
  "compaction": { "enabled": true, "reserveTokens": 16384, "keepRecentTokens": 20000 },
  "retry": { "enabled": true, "maxRetries": 3 },
  "steeringMode": "one-at-a-time",
  "followUpMode": "one-at-a-time"
}
```

Use `/usage` and `/context` before raising budgets or spawning more children. Prime Agent attributes child usage to the parent turn while keeping child context accounting distinguishable. Consider the daemon's idle eviction policy when relying on retained children for later follow-up.

Trace sharing is opt-in and should be treated as an external side effect:

```text
/traces status
/traces on
/traces preview
/traces upload-current
/traces off
```

Do not upload traces, use `/share`, or enable telemetry changes unless the user explicitly requests it. For privacy-sensitive work, inspect the configured telemetry setting and use the documented global opt-out rather than inventing a local credential or endpoint.

## Repository and Worktree Discipline

- Set the Hermes `terminal` `workdir` explicitly for every Prime Agent invocation.
- Never run two independent implementation agents in the same checkout.
- For parallel edits, create separate Git worktrees and launch one Prime Agent worker per worktree; merge only after reviewing each diff.
- For internal `rlm(...)` children in one root worker, default to independent analysis or serialize edits unless the project explicitly provides isolation.
- Preserve unrelated user changes. Inspect `git status --short` before and after the run.
- Do not ask Prime Agent to commit, push, deploy, or migrate unless the user explicitly authorizes that exact side effect.
- In Skemática, keep `skematica-platform` docs-only and route implementation to the owning runtime repository after reading its local instructions and branch strategy.

## Procedure

1. **Resolve the target.** Identify the exact repository, bounded context, branch policy, and absolute `workdir`.
2. **Read context.** Inspect `AGENTS.md`, `CLAUDE.md`, the repository README, current Git status, and the requested files before launching an implementation worker.
3. **Choose the mode.** Use print mode for bounded work, a daemon-backed interactive session for multi-turn work, JSON/RPC for programmatic integration, and autonomous mode only with explicit gates and limits.
4. **Bound the prompt.** State scope, files or symbols, edit/commit/deploy permissions, required checks, expected report, and whether subagents may edit.
5. **Launch one worker.** Use the Hermes `terminal` tool with explicit `workdir`; use a separate worktree for any concurrent implementation.
6. **Monitor honestly.** For background work, inspect `process` output and Prime Agent lifecycle state. Do not infer completion from process admission, a queued RPC response, or a reached autonomous limit.
7. **Verify the artifact.** Re-run `git status --short`, inspect `git diff --check` and the relevant diff, run the repository's real tests/checks, and confirm generated artifacts or endpoints when applicable.
8. **Report evidence.** Return files changed, commands and actual results, remaining failures, process/session identifiers, and any unverified claims. Never report a Prime Agent summary as verified until the workspace confirms it.
9. **Clean up.** Stop or detach only the intended worker, and remove temporary worktrees after their diffs are reconciled.

## Quick Reference

```text
# Readiness
terminal(command="prime-agent --version")
terminal(command="prime-agent status")
terminal(command="prime-agent doctor")

# One-shot
terminal(command="prime-agent -p '<task>'", workdir="/path/to/project", timeout=600)

# Background interactive
terminal(command="prime-agent '<task>'", workdir="/path/to/project", background=true, pty=true)
process(action="poll", session_id="<id>")
process(action="log", session_id="<id>")
process(action="submit", session_id="<id>", data="<follow-up>")

# Lifecycle
terminal(command="prime-agent list --all")
terminal(command="prime-agent attach <agent>")
terminal(command="prime-agent send <agent> '<message>'")
terminal(command="prime-agent stop <agent>")
terminal(command="prime-agent status")
terminal(command="prime-agent doctor")

# Sessions
terminal(command="prime-agent -c", workdir="/path/to/project")
terminal(command="prime-agent -r <path-or-id>", workdir="/path/to/project")

# Durable coordination
terminal(command="prime-agent send <agent> '<message>'")
terminal(command="prime-agent schedule list --all")
# In interactive mode: /heartbeat, /goal, /refine, /compact, /tree, /fork, /clone

# Resources
terminal(command="prime-agent package list")
terminal(command="prime-agent package install <source> [--local]")
terminal(command="prime-agent config")

# Autonomous, bounded
terminal(command="prime-agent -p --autonomous --autonomous-gate '<check>' --autonomous-max-turns 12 --autonomous-timeout-ms 1800000 '<task>'", workdir="/path/to/project", timeout=2100)

# Headless integration
terminal(command="prime-agent --mode json '<task>'", workdir="/path/to/project", timeout=600)
terminal(command="prime-agent --mode rpc", workdir="/path/to/project", background=true, pty=false)
```

## Pitfalls and Safety

1. **Not a sandbox:** worker and kernel process isolation is for lifecycle and recovery, not privilege reduction.
2. **`rlm()` is admission-only:** the returned handle is not the child's answer; use `agent_message` or files.
3. **Same-checkout collisions:** internal children share the root worker's repository context; avoid concurrent writes without isolation.
4. **Autonomous limits are not success:** gate results, exit status, tests, and workspace inspection are required.
5. **Detached workers persist:** use `list`, `attach`, `stop`, and `shutdown` deliberately to avoid orphaned work.
6. **RPC acceptance is not completion:** keep reading events until the run ends, then verify the workspace.
7. **Context drift:** Prime Agent loads `AGENTS.md` and `CLAUDE.md`; stale or conflicting instructions can alter the result.
8. **Secrets:** prefer `/login`, auth files, or preconfigured environment variables; never echo or paste credentials.
9. **Provider mismatch:** `--offline` disables startup network calls but still requires a configured provider and does not make model inference local.
10. **Cross-repo work:** read the owner repository's instructions and branch strategy before edits; do not implement runtime code in `skematica-platform`.

## Verification

A Prime Agent delegation is verified only when all applicable evidence is present:

- The launch command exited successfully, or the background worker reached a documented end state.
- The actual target workdir is confirmed.
- `git status --short` and the diff account for every changed file.
- The repository's documented tests, lint, build, or check command ran and its real output is recorded.
- Any requested generated artifact, deployment handle, or endpoint was independently checked.
- Autonomous gates passed when configured; a limit or agent self-report alone is insufficient.
- Background processes and temporary worktrees were intentionally retained or cleaned up.

When evidence is incomplete, report the exact missing verification instead of inferring success.
