---
name: prime-agent
description: "Use when you delegate coding or research work to Prime Agent."
license: MIT
compatibility: "Requires the Prime Agent CLI and a trusted project workspace."
metadata:
  author: Daniel Gómez
  version: "0.3.0"
---

# Prime Agent Coding Skill

Use this skill when you need Prime Agent to do coding or research work.
This skill gives one clear process for short tasks and long tasks.
It covers child agents, durable sessions, goals, schedules, packages, and integrations.
It does not depend on a parent agent or a specific host application.

## Use This Skill When

Use this skill when you:

- ask Prime Agent to inspect, change, test, or review code;
- need a task to continue after the terminal closes;
- need Prime Agent to create child agents;
- need a goal, schedule, or heartbeat;
- need a bounded autonomous run;
- need JSON, RPC, or ACP control; or
- need a Prime Agent skill, extension, or package.

Do not use this skill for a short answer that needs no project work.
Do not use one worktree for two independent implementation agents.

## Safety Rules

Follow these rules before you start a task:

1. Treat Prime Agent as a trusted local process.
2. Do not treat Prime Agent as a security sandbox.
3. Run Prime Agent only in a trusted project.
4. Review third-party skills, extensions, and packages before you load them.
5. Keep credentials out of prompts, files, logs, and command arguments.
6. Do not commit, push, or deploy unless the user authorizes that action.
   Do not run a migration unless the user authorizes it.
7. Use one worktree for one implementation agent.
8. Use separate worktrees for parallel implementation work.
9. Preserve changes that existed before the task.
10. Check the project after every task.

## Requirements

You need these items:

- the `prime-agent` command;
- a valid Prime Agent login or provider setup;
- a trusted project directory;
- a Git worktree for code changes; and
- the project rules and test commands.

Check the command before you start:

```bash
prime-agent --version
prime-agent status
prime-agent doctor
```

Install Prime Agent only when the user requests installation.
You can also install it when the user selects Prime Agent for the task:

```bash
curl -fsSL https://app.primeintellect.ai/prime-agent/install.sh | sh
```

The public installer supports Linux and macOS.
A source build needs Node.js 22.8.0 or newer.

## Prepare the Workspace

Complete these steps before you start a coding task.

1. Set the project directory.
2. Read `AGENTS.md`, `CLAUDE.md`, and the project README when they exist.
3. Check the current branch.
4. Check the current Git status.
5. Read the files that the task can change.
6. Find the project test, lint, build, and check commands.
7. Choose one worktree for the Prime Agent process.

Use a prompt with these items:

- task scope;
- files or symbols in scope;
- files that are out of scope;
- permission to edit;
- permission to commit, push, deploy, or migrate;
- required checks;
- expected output; and
- child-agent permissions.

Example prompt:

```text
Inspect the authentication flow.
Read AGENTS.md and the project README first.
Change only the authentication files and related tests.
Do not commit or push.
Run the targeted tests.
Report each changed file and each command result.
```

## Run a Short Task

Use print mode for a bounded task:

```bash
TASK="Inspect the project. Make the requested change. Run the required checks."
prime-agent -p "$TASK" --cwd /path/to/project
```

Use file arguments when the task needs named files:

```bash
prime-agent -p @README.md @AGENTS.md \
  "Review the plan and list risks. Do not change files." \
  --cwd /path/to/project
```

Set the model and thought level only when the user or project requires them:

```bash
prime-agent -p \
  --provider provider-name \
  --model provider/model-id \
  --thinking high \
  "Implement the change and run the required checks." \
  --cwd /path/to/project
```

Use the Prime Agent login flow or a configured environment variable for credentials.
Do not place an API key in the command.
The `--offline` option disables startup network calls.
It does not make model calls local.

## Run a Long Task

Use an interactive session when the task needs more than one turn:

```bash
TASK="Work through the migration in verified steps. Do not commit."
prime-agent "$TASK" --cwd /path/to/project
```

Prime Agent keeps the session in a worker process.
A closed terminal does not always stop the worker.

Use these commands to find and control the worker:

```bash
prime-agent list --all
prime-agent agents
prime-agent attach "$AGENT_ID"
MESSAGE="Run the targeted checks again."
prime-agent send "$AGENT_ID" "$MESSAGE"
prime-agent stop "$AGENT_ID"
prime-agent status
prime-agent doctor
```

Use `prime-agent shutdown --force` only when the user asks for it.
This command stops all Prime Agent workers.

Resume a session with one of these commands:

```bash
prime-agent -c
prime-agent -r "$SESSION_ID"
```

## Create Child Agents

Prime Agent exposes the `rlm` callable in its Python control environment.
The call admits a child agent and returns a handle.
The call does not return the child answer.

Give each child one small task.
State the project path, the read-first files, the edit rights, and the reply method.

```python
review = await rlm(
    "Review authentication only. Read AGENTS.md first. Do not edit files. "
    "Send file and line findings to the parent.",
    name="auth-reviewer",
)

test_review = await rlm(
    "Find missing regression tests. Do not change production code. "
    "Send file and line findings to the parent.",
    name="test-reviewer",
)
```

A child must send its result with `agent_message` or write a file for the parent.
Use this form for a parent message:

```python
await agent_message.send(
    "The authentication check misses expired tokens in src/auth.ts:88.",
    receiver_role="parent",
)
```

List retained children with:

```python
children = await rlm.list_subagents()
```

Send a follow-up to a retained child with:

```python
await agent_message.send(
    "Check the new regression test.",
    receiver_role="child",
    receiver_name="auth-reviewer",
)
```

Delete a child when you no longer need its context:

```python
await rlm.delete_subagent("auth-reviewer")
```

Use `rlm.find_models()` before you select a child model:

```python
models = await rlm.find_models("coding", limit=8)
```

Use an exact `provider/model` value when you set the child model.
Prime Agent does not replace an unavailable model with another model.
The default child depth is one level.
A root agent can create children.
A child cannot create more children unless you raise the depth limit.

Keep child tasks independent when they run in the same worktree.
Do not let two children edit the same file at the same time.

## Send Messages Between Agents

Use the shell command for a running agent:

```bash
prime-agent send "$AGENT_ID" "Check the latest test result."
```

Use the Python message skill for more control:

```python
roster = await agent_message.list_agents()
receipt = await agent_message.send(
    "Check the latest test result.",
    receiver_role="sibling",
    receiver_name="test-reviewer",
    mode="auto",
)
print(receipt["deliveryStatus"])
```

Use these message modes:

- `auto` sends at once to an idle agent and steers a busy agent;
- `steer` sends at once to a busy agent; and
- `follow_up` waits until the current work ends.

A `delivered` or `queued` receipt proves message delivery.
It does not prove task completion.

## Use Heartbeats and Schedules

Use a heartbeat for a repeated check in the current session:

```text
/heartbeat every 10m Check the deployment and report meaningful changes
/heartbeat status
/heartbeat pause
/heartbeat resume
/heartbeat clear
```

Use `rlm_heartbeat` for more than one agent-owned check:

```python
heartbeat = await rlm_heartbeat.create(
    "Check if the test run finished.",
    interval="5m",
    label="tests",
    delivery_mode="follow_up",
)
await rlm_heartbeat.list()
await rlm_heartbeat.update(
    heartbeat["heartbeat"]["id"],
    status="pause",
)
```

Use a schedule for a future prompt or a cron prompt:

```bash
prime-agent schedule add worker "in 30m" -- "Check the benchmark result."
prime-agent schedule add worker "0 9 * * 1-5" -- "Review open work."
prime-agent schedule list --all
prime-agent schedule cancel "$JOB_ID"
```

Scheduled prompts remain active after the terminal closes.
Check the result of each scheduled task.

## Use Persistent Goals

Use a goal when the user wants one objective to remain active across turns:

```text
/goal Ship the release and check every published artifact
/goal --budget 200000 Complete the repository migration
/goal status
/goal pause
/goal resume
/goal clear
```

Use the Python goal skill to check or complete the goal:

```python
state = await goal.get()
await goal.complete()
```

Only `goal.complete()` marks a goal as complete.
A token limit or a time limit does not prove completion.

## Use Bounded Autonomous Mode

Autonomous mode adds follow-up turns without user input.
Always set a check and resource limits for an unattended task:

```bash
prime-agent -p \
  --autonomous \
  --autonomous-gate "$PROJECT_CHECK" \
  --autonomous-gate-retries 2 \
  --autonomous-gate-timeout-ms 300000 \
  --autonomous-max-continuations 3 \
  --autonomous-max-turns 12 \
  --autonomous-max-tokens 80000 \
  --autonomous-timeout-ms 1800000 \
  "Make the requested change. Run the check. Report the evidence." \
  --cwd /path/to/project
```

Use the project check instead of a fixed command.
A failed gate gives its bounded output to the next turn.
A passed gate allows the run to end.
A turn, token, or time limit does not prove task success.

Use `--goal` for a durable objective.
Use `--autonomous` for bounded follow-up turns.
Do not treat these options as the same feature.

## Keep Context Across Turns

Use the continual harness for small, tested improvements to reusable work patterns:

```text
/refine Record only the reusable lesson from the failed migration check.
/refine status
```

The harness stores notes, memories, and skill descriptions.
It also stores child-agent specifications and refinement records.
A refinement does not change the base system prompt.
Review each refinement before you keep it.

Use compaction when the context grows:

```text
/usage
/context
/compact Preserve the failing tests, exact commands, decisions, and next steps.
```

Compaction summarizes old messages.
It keeps recent messages and the Python state.
It does not stop children, schedules, heartbeats, goals, or autonomous runs.

Use the session tree to test another approach:

```text
/tree
/fork
/clone
/name auth-refresh
/session
/export /path/to/session.html
```

Use `/share` only when the user asks to upload a private session copy.
Use `/reload` after you add or change a resource.
Resources include skills, extensions, prompts, themes, and context files.
Start a new session after you add a Python-backed skill with new dependencies.

## Use Skills, Extensions, and Packages

Choose the smallest resource type that solves the task:

- Use a Markdown skill for instructions.
- Use a Python-backed skill for an importable Python function.
- Use a TypeScript extension for tools, events, commands, or permissions.
  Use it also for providers or UI.
- Use a prompt template for repeated prompt text.
- Use a theme for terminal display.
- Use a package to share several resources.

Use the built-in skill creator for a new Prime Agent skill:

```text
/skill:skill-creator Create a skill for a specific task.
```

Prime Agent finds resources in these locations:

- `~/.prime/agent/skills/`;
- `.prime/agent/skills/`;
- `.agents/skills/`;
- package directories; and
- paths passed with `--skill`.

Use these paths for extensions:

- `~/.prime/agent/extensions/`; and
- `.prime/agent/extensions/`.

Test one extension for one run before you make it persistent:

```bash
prime-agent -e ./path/to/extension.ts
```

Manage packages with these commands:

```bash
prime-agent package install npm:@scope/package@1.2.3
prime-agent package install git:github.com/user/repo@v1 --local
prime-agent package list
prime-agent package update
prime-agent package remove npm:@scope/package
prime-agent update
prime-agent config
```

Pin package versions for repeatable work.
Review package source before you install it.
A package can run code with the same rights as the user.

## Use JSON, RPC, ACP, or the SDK

Use JSON mode for a batch run with machine-readable events:

```bash
prime-agent --mode json "Analyze the repository and report findings." \
  --cwd /path/to/project
```

Read the JSON lines until the run ends.
Use the exit status and the project files as completion checks.

Use RPC mode for a persistent line-based control connection:

```bash
prime-agent --mode rpc --cwd /path/to/project
```

Use `prompt`, `steer`, `follow_up`, and `abort` commands.
A response that accepts a command does not prove task completion.
Keep reading events until the run ends.

Use ACP mode when an external client needs to control a session:

```bash
prime-agent --mode acp --cwd /path/to/project
```

ACP uses one JSON-RPC message per line.
One connection owns one session and one project directory.
Use a new process for a second session.
Keep protocol output on standard output.
Send diagnostics to standard error.

Use the Node.js SDK when a Node.js application must create and control sessions.
Use `createAgentSession` and the documented session events.
Check the project path, model, session state, and end state.
Use the same checks that you use for the CLI.

## Configure the Runtime

Prime Agent reads global settings from `~/.prime/agent/settings.json`.
It reads project settings from `.prime/agent/settings.json`.
Project settings override global settings.

Use `/settings` for interactive changes.
Keep shared project settings in version control only when the project needs them.

Example settings:

```json
{
  "defaultProvider": "provider-name",
  "defaultModel": "provider/model-id",
  "defaultThinkingLevel": "high",
  "compaction": {
    "enabled": true,
    "reserveTokens": 16384,
    "keepRecentTokens": 20000
  },
  "retry": {
    "enabled": true,
    "maxRetries": 3
  },
  "steeringMode": "one-at-a-time",
  "followUpMode": "one-at-a-time"
}
```

Use `/usage` and `/context` before you raise budgets or add children.
Child use counts toward the parent turn.

Treat trace upload and session sharing as external actions:

```text
/traces status
/traces preview
/traces upload-current
/traces off
```

Run these commands only with user approval.

## Procedure

1. Confirm the project path.
2. Read the project rules and README.
3. Check the branch and Git status.
4. Select one worktree.
5. Write a bounded task prompt.
6. Select print, interactive, JSON, RPC, ACP, or autonomous mode.
7. Set a check and a resource limit.
8. Start one Prime Agent process.
9. Check the process, session, or event stream.
10. Check the project files and the task result.
11. Run the required tests and checks.
12. Report changed files, commands, results, limits, and open issues.
13. Stop or detach only the intended worker.

## Quick Reference

```bash
prime-agent --version
prime-agent status
prime-agent doctor
prime-agent -p "$TASK" --cwd /path/to/project
prime-agent "$TASK" --cwd /path/to/project
prime-agent list --all
prime-agent attach "$AGENT_ID"
prime-agent send "$AGENT_ID" "$MESSAGE"
prime-agent stop "$AGENT_ID"
prime-agent -c
prime-agent -r "$SESSION_ID"
prime-agent schedule list --all
prime-agent package list
prime-agent --mode json "$TASK" --cwd /path/to/project
prime-agent --mode rpc --cwd /path/to/project
prime-agent --mode acp --cwd /path/to/project
```

## Limits and Failure Cases

1. Prime Agent does not provide a security sandbox.
2. A child handle proves admission, not a child result.
3. Two writers can change the same file and cause a conflict.
4. A queued message proves delivery, not task completion.
5. A passed gate proves only the command that the gate checks.
6. A limit proves that the run stopped, not that the task succeeded.
7. A detached worker can continue after the terminal closes.
8. A Python skill can fail when the kernel lacks its package.
9. An exact child model can fail when its credentials are not valid.
10. A protocol response can accept a command before the task ends.
11. A trace upload or session share sends data outside the local machine.

## Check Completion

Mark the task complete only when the applicable checks pass:

- The process ends normally or reaches a known end state.
- The project path is correct.
- `git status --short` shows the expected changes.
- The diff contains no unexpected file.
- The project tests, lint, build, or check command passes.
- Each autonomous gate passes.
- Each requested artifact exists.
- Each requested endpoint or deployment has independent evidence.
- No worker or temporary worktree remains by accident.

If a check is missing, report the missing check.
Do not infer success from a Prime Agent message alone.
