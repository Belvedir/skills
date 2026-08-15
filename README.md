# Belvedir skills

Agent skills for [Belvedir](https://belvedir.ai), the observability and self-improvement platform for AI agents. A skill is a markdown guide a coding agent (Claude Code, Cursor, etc.) loads to do a job correctly; each skill here is self-contained.

## belvedir

Everything a coding agent needs to instrument an app with Belvedir: install the SDK (Node or Python), initialize it correctly for the codebase, wrap runs in sessions and tasks, route inference through Belvedir, report outcomes, and debug missing traces.

Install it into your codebase:

```bash
mkdir -p .claude/skills/belvedir && curl -fsSL https://raw.githubusercontent.com/Belvedir/skills/main/belvedir/SKILL.md -o .claude/skills/belvedir/SKILL.md
```

If your agent keeps skills somewhere else, put the file in the equivalent place. The same content is served at [platform.belvedir.ai/agents.md](https://platform.belvedir.ai/agents.md).

You usually don't need to run this yourself: the **Copy prompt** button on the dashboard's Quick Start page produces a prompt that has your coding agent install the skill and follow it.

## Maintenance

This repo is a mirror. The source of truth is `src/lib/agent-skill.ts` in the platform repo; regenerate after changing it:

```bash
# from the platform repo root
npx tsx -e 'import {belvedirSkillMarkdown} from "./src/lib/agent-skill"; process.stdout.write(belvedirSkillMarkdown())' > <this repo>/belvedir/SKILL.md
```

Full documentation: [docs.belvedir.ai](https://docs.belvedir.ai)
