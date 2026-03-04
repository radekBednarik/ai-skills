# ai-skills

A collection of AI skill definition files for use with agentic coding assistants.

## Skills

| Skill                                                   | Description                                                                                                   |
| ------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------- |
| [`playwright-test`](./playwright-test/SKILL.md)         | Playwright TypeScript test conventions — Page Object Model, locator strategy, assertions, and best practices  |
| [`quality-engineering`](./quality-engineering/SKILL.md) | Senior QE persona (Alex Mercer) — testing strategy, DORA metrics, shift-left practices, and release readiness |

## Usage

Each skill lives in its own sub-directory as a `SKILL.md` file with a YAML front matter block. Agents load a skill when the user's request matches the trigger conditions described in the `description` field.

## Adding a Skill

1. Create a sub-directory with a short, descriptive name.
2. Add a `SKILL.md` with YAML front matter (`name`, `description`, optional `allowed-tools`).
3. Format with Prettier: `pnpm exec prettier --write .`

## Development

```bash
pnpm install                      # install dependencies
pnpm exec prettier --write .      # format all files
pnpm exec prettier --check .      # check formatting
```
