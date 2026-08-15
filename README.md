# Prime Agent Coding Skill

A reusable Agent Skill for Prime Agent.

## Scope

This skill is agent-agnostic.
It does not depend on a host agent.
It gives one process for Prime Agent coding work.

The skill covers:

- short tasks;
- long tasks;
- child agents;
- agent messages;
- heartbeats;
- schedules;
- persistent goals;
- bounded autonomous runs;
- session memory and compaction;
- skills and extensions;
- packages;
- JSON, RPC, and ACP modes;
- Node.js SDK use; and
- evidence-based checks.

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
