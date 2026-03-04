# AGENTS.md

Guidelines for agentic coding agents working in this repository.

---

## Repository Overview

This is a collection of AI skill definition files. Each skill lives in its own sub-directory as a `SKILL.md` file:

- `playwright-test/SKILL.md` — Playwright TypeScript test conventions
- `quality-engineering/SKILL.md` — Quality Engineering persona and strategy guidelines

There are no application source files. The only tooling at the root level is **pnpm** (package manager) and **Prettier** (formatter).

---

## Package Manager

Always use **pnpm**. Never use `npm` or `yarn`.

```bash
pnpm install        # install dependencies
pnpm add -D <pkg>   # add a dev dependency
```

---

## Formatting

Prettier is the only formatter. Run it before committing:

```bash
pnpm exec prettier --write .   # format all files
pnpm exec prettier --check .   # check without writing
```

No custom Prettier config exists — Prettier defaults apply (double quotes, trailing commas, semicolons, LF line endings).

---

## Skill File Conventions

Each skill is a single Markdown file (`SKILL.md`) with a YAML front matter block followed by the skill content.

### Front matter schema

```yaml
---
name: <kebab-case-identifier>
description: >
  <Multi-line description of when to activate this skill.>
allowed-tools: <optional comma-separated list of tool restrictions>
---
```

- `name` — unique, kebab-case identifier for the skill.
- `description` — written for an AI agent; explains the domain and lists trigger conditions (specific user phrases, topics, or task types that should load this skill).
- `allowed-tools` — optional; restricts which tools the agent may use when the skill is active (e.g. `Bash(pnpm:*), Bash(tsc:*)`).

### Adding a new skill

1. Create a new sub-directory with a short, descriptive name.
2. Add a `SKILL.md` inside it following the front matter schema above.
3. Run `pnpm exec prettier --write .` to format the file.

### Editing an existing skill

- Keep the front matter intact; only modify `description` or `allowed-tools` when the skill's activation scope genuinely changes.
- Prefer clear, imperative language in the skill body — agents execute instructions literally.
- Run `pnpm exec prettier --write .` after editing.

---

## Git & Commits

- Do not commit `node_modules/` (covered by `.gitignore`).
- Format with Prettier before every commit.
