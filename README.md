# Prime Agent Coding Skill

A reusable Agent Skill for Prime Agent.

## Scope

This skill gives host agents and in-session Prime Agent users one secure process
for bounded coding and repository-research work.

It covers:

- permission-bounded short and multi-step tasks;
- practical trust boundaries for repository content;
- safe CLI defaults that disable unreviewed workspace resources;
- read-only child-agent reviews and worktree isolation;
- command review, diff inspection, and evidence-based completion; and
- explicit stop conditions for external or destructive actions.

Runtime installation, package management, uploads, telemetry, releases, and
deployments are intentionally outside the skill.

## Local Check

Run this command from the repository root:

```bash
npx skills add ./ --list
```

The command must find one skill named `prime-agent`.

## Install

Install the current public release with:

```bash
npx skills add oddradiocircle/prime-agent --skill prime-agent
```

## Author and Attribution

Author: **Daniel Gómez**.

This skill is an independent guide.
It is not an official Prime Intellect product.
[Prime Agent](https://github.com/PrimeIntellect-ai/prime-agent) is developed by
Prime Intellect.

## License

MIT.
See [LICENSE](LICENSE).
