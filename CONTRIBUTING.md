# Contributing a Skill

This guide walks you through adding a new skill to the MaRDI-skills marketplace.

Skills are agent-agnostic: the same `SKILL.md` files work across providers (Claude Code, GitHub Copilot CLI, Cursor, Codex, Gemini CLI, and others).
Although the general mechanism is supported by all providers, there are some differences with respect to which features they support.
For the specification of the "Agent Skills" standard, see: https://agentskills.io/home.
Skills that are added here should work across providers.

Before we continue, here are some further resources to read up on:

* [Building Skills for Claude](https://resources.anthropic.com/hubfs/The-Complete-Guide-to-Building-Skill-for-Claude.pdf)
* [Claude Code skills documentation](https://docs.claude.com/en/docs/claude-code/skills).
* [Anthropic skill-creator](https://github.com/anthropics/skills/tree/main/skills/skill-creator)
* [awesome claude skills](https://github.com/ComposioHQ/awesome-claude-skills)

## What belongs here

This marketplace collects skills that are useful MaRDI *users*.
This marketplace is **not** for MaRDI-internal tooling.

Possible categories include skills that:

* Interacts with a MaRDI service — e.g. querying the [MaRDI Portal / Knowledge Graph](https://portal.mardi4nfdi.de/).
* Itself constitutes a MaRDI Service, such as skills for reproducibility reviews or FAIRification of research data.
* Simplify usage of MaRDI assets, such as filling out of MaRDI templates.
* ...

## TL;DR

1. Create `skills/<category>/<skill-name>/SKILL.md` with YAML frontmatter.
1. Register the skill in `.claude-plugin/marketplace.json` under the appropriate plugin.
1. Evaluate the skill:
   1. Test the skill to make sure that it works
   1. Ensure that it's not too long by running `uv run internal/count-skill-tokens.py skills/<category>/<skill-name>`
1. Open a pull request.

## Repository layout

```
.
├── .claude-plugin/
│   └── marketplace.json         # Marketplace manifest — lists plugins and their skills
├── skills/
│   └── <category>/
│       └── <skill-name>/
│           ├── SKILL.md         # Skill entry point with YAML frontmatter
│           └── references/      # Optional supporting material
├── internal/
│   └── count-skill-tokens.py    # Validation script for token/line limits
├── CONTRIBUTING.md
└── README.md
```

## Adding a Skill

Place the skill at `skills/<category>/<skill-name>/`, where `<category>` groups related skills and `<skill-name>` is a short, kebab-case identifier prefixed with `mardi-` for consistency.
The directory must contain a `SKILL.md`, but may also contain other folders and files, such as `references/`, `templates/`, or `scripts/`, which the skill an load on demand.

```
skills/paper/mardi-kg-query
├── SKILL.md
├── references/
│   └── checklist.md
└── templates/
    └── review.md
```

### SKILL.md

`SKILL.md` is a Markdown file with a YAML frontmatter block followed by the skill body:

```markdown
---
name: mardi-paper-review
description: Review a MaRDI paper draft for clarity, structure, and citation hygiene. Use when the user asks to review a paper, check a draft, or audit references.
---

<Skill Body>
```

The two metadata fields you have to get right:

* `name` — kebab-case identifier, should match the directory name. Becomes the skill's invocation slug.
* `description` — the trigger text the agent uses to decide whether to load the skill. It is the *only* part of the file always in context, so write it for the model: say what the skill does *and* when it should fire ("Use when…"). Vague descriptions don't get picked up; overly broad ones fire too often. Keep it under 100 tokens.

For the full format and authoring guidance, see the agentskills.io references — start with the quickstart and best-practices pages, then dig deeper as needed:

* [Skill specification](https://agentskills.io/specification) — full metadata schema and file layout.
* [Quickstart](https://agentskills.io/skill-creation/quickstart) — minimal walk-through for a first skill.
* [Best practices](https://agentskills.io/skill-creation/best-practices) — guidance on description writing, scoping, and progressive disclosure via `references/`.
* [Using scripts](https://agentskills.io/skill-creation/using-scripts) — when and how to ship helper scripts.
* [Evaluating skills](https://agentskills.io/skill-creation/evaluating-skills) — how to test that a skill triggers correctly and produces the right output.

### Validate token and line counts

Skills should stay within these limits to keep the agent's context window healthy:

| Field | Limit |
|---|---:|
| `description` | 100 tokens |
| `SKILL.md` | 5,000 tokens / 500 lines |

Run the validator:

```bash
uv run internal/count-skill-tokens.py skills/<category>/<skill-name>
```

The script prints a Markdown table of token and line counts, flagging (⚠️) anything over the limit. Trim or move material into `references/` until it passes. ([`uv`](https://docs.astral.sh/uv/) handles the script's Python dependencies automatically.)

### Registering the skill

Once you have created the skill, you also need to add it to `.claude-plugin/marketplace.json`.
This will also work with the `vercel-labs/skills` cross-platform installer and not only with Claude.
