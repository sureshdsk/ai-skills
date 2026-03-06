# AI Skills

A collection of skill files for AI coding agents (Claude Code, Codex, Windsurf, Cursor, etc.) to reference when working with specific frameworks and tools.

## What Are Skills?

Each skill is a self-contained markdown file that gives an AI agent accurate, opinionated guidance for a specific technology. Skills exist to correct common agent mistakes, capture framework-specific patterns, and reduce hallucination for tools that diverge from popular conventions.

## Available Skills

| Skill | Folder | Description |
|-------|--------|-------------|
| Django Bolt | `skills/django-bolt/` | Rust-powered API framework for Django. Not DRF, not Django Ninja — its own routing, serializers, auth, and server. |

---

## Installing with skills.sh

The easiest way to install a skill into Claude Code:

```bash
npx skills add sureshdsk/ai-skills --skill django-bolt
```

Or browse at [skills.sh/sureshdsk/ai-skills](https://skills.sh/sureshdsk/ai-skills).

---

## Installing Manually in Claude Code

```bash
git clone https://github.com/sureshdsk/ai-skills.git
SKILL=django-bolt
mkdir -p ~/.claude/skills/$SKILL
cp ai-skills/skills/$SKILL/SKILL.md ~/.claude/skills/$SKILL/SKILL.md
```

The skill will appear in Claude Code immediately — no restart needed. Verify with `/skills`.

---

## Using with Other Agents

The `SKILL.md` files are plain markdown and work with any agent (Codex, Windsurf, Cursor, etc.):

- **Paste into context** — copy `skills/django-bolt/SKILL.md` into your agent's system prompt
- **Reference in project config** — add to `CLAUDE.md`, `.cursorrules`, or equivalent

---

## Repository Structure

```
ai-skills/
└── skills/
    └── django-bolt/
        └── SKILL.md
```

---

## Contributing

Each skill should:
- Start with a clear description of what the technology is and what it is NOT
- Cover installation, core patterns, and common mistakes
- Include working code examples
- End with a "common mistakes" table or section
- Stay focused — one skill per framework/tool

To add a new skill, create `skills/<name>/SKILL.md` with YAML frontmatter:

```markdown
---
name: my-skill
description: "Trigger description for when the agent should use this skill."
---

# Skill content here...
```
