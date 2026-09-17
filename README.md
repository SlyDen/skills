# Skills

My personal collection of skills for Codex and other compatible coding agents.

## Available skills

| Skill | Description |
| --- | --- |
| [Swift Project Architecture](skills/swift-project-architecture/SKILL.md) | Pragmatic architecture for Swift, SwiftUI, Swift Package Manager, and server-side Swift. Covers domain boundaries, persistence, use cases, testing, and concurrency. |

## Install

Requires Node.js, npm, and Git.

Choose which skills to install and which agents to use:

```sh
npx skills add SlyDen/skills
```

Install Swift Project Architecture globally for Codex:

```sh
npx skills add SlyDen/skills --skill swift-project-architecture --agent codex --global
```

Omit `--global` to install in the current project. Add `--yes` to skip prompts.

## Update

Update this skill in your global installation:

```sh
npx skills update swift-project-architecture --global
```

For a project installation, run this from the project directory:

```sh
npx skills update swift-project-architecture --project
```

You can also rerun the original `skills add` command to reinstall from GitHub.

## Use

Ask Codex to use `$swift-project-architecture` to design a feature, review a Swift package, or refactor a backend feature.

## Repository layout

```text
skills/swift-project-architecture/
├── SKILL.md
├── agents/openai.yaml
└── references/architecture-guide.md
```

`SKILL.md` provides the skill name, description, and decision rules. The reference contains detailed guidance. `agents/openai.yaml` supplies Codex metadata.

The [Skills CLI](https://github.com/vercel-labs/skills#creating-skills) discovers this layout directly. No npm manifest or registry publication is required.

Preview available skills without installing:

```sh
npx skills add SlyDen/skills --list
```

## Adding a skill

Create a folder at `skills/<skill-name>/` with a `SKILL.md` containing YAML frontmatter with `name` and `description`. Keep supporting references, scripts, and agent metadata inside that folder. Add the skill to the table above.

Check discovery before publishing:

```sh
npx skills add . --list
```
