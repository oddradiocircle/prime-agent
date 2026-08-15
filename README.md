# Prime Agent — Coding-Agent Delegation Skill

A reusable Agent Skill for orchestrating [Prime Agent](https://github.com/PrimeIntellect-ai/prime-agent) from Hermes and other compatible coding agents.

## What it covers

The skill documents Prime Agent's full operating surface: RLM child agents, agent-to-agent messaging, persistent sessions, daemon workers, heartbeats, schedules, goals, bounded autonomous mode, continual-harness refinement, compaction, session branching, Python-backed skills, TypeScript extensions, resource packages, JSON/RPC/ACP integrations, SDK embedding, provider configuration, and evidence-based verification.

## Install

From the public repository:

```bash
npx skills add oddradiocircle/prime-agent --skill prime-agent
```

To inspect the available skill before installing:

```bash
npx skills add oddradiocircle/prime-agent --list
```

For local validation without a remote:

```bash
npx skills add ./ --list
```

The repository also includes `agents/openai.yaml` for compatible OpenAI agent tooling.

## Authorship and attribution

Authored by **Daniel Gómez**. This is an independent orchestration skill and is not affiliated with or endorsed by Prime Intellect. Prime Agent remains the property of Prime Intellect-ai and is linked as the upstream project documented by this skill.

## Community contributions

Issues and pull requests are welcome. Keep changes focused on documented Prime Agent behavior, preserve the safety boundaries in `skills/prime-agent/SKILL.md`, and include a reproducible verification command for documentation changes.

## Publication

The canonical public repository is:

<https://github.com/oddradiocircle/prime-agent>

The repository is prepared for discovery by compatible agent-skill indexes such as [skills.sh](https://skills.sh/). Vercel deployment is not required for skill installation or discovery.

## License

MIT. See [LICENSE](LICENSE).
