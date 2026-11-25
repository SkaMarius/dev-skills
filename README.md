# dev-skills

The skills I use with AI coding tools to get code I'd actually merge.

Each skill is a folder with a `SKILL.md` file. The format is the open Agent Skills standard, so the same folder works in Claude, Codex, GitHub Copilot, Cursor and other tools that support it.

## What's here

```
skills/
  general/   language-agnostic: code review, testing, commits
  go/        Go, from system design down to single functions
```

Go comes first. Other languages (React is next) will get their own folder when I start on them.

## The Go skills

The Go skills start from the data, keep packages focused, don't add abstractions until you need them, and aim for code people can read.

Each rule is tagged. [Go] means widely agreed Go practice (Effective Go, Code Review Comments). [Opinion] means a deliberate choice that not everyone shares.

Planned, roughly in order:

- `go-design`: designing a service before writing code
- `go-structure`: turning a design into packages and folders
- `go-code`: everyday decisions inside a package
- `go-testing`: tests, benchmarks and profiling

## Using a skill

Copy the skill's folder into the place your tool reads skills from:

| Tool | Personal | Per project |
|---|---|---|
| Claude Code | `~/.claude/skills/` | `.claude/skills/` |
| Codex | `~/.codex/skills/` | `.codex/skills/` |
| GitHub Copilot | `~/.copilot/skills/` | `.github/skills/` |

In the Claude app, zip the skill folder and upload it under Skills in settings.

## Naming

Skill names start with the language (`go-design`, `react-components`) so they don't collide when several are installed together.

## Status

Work in progress. Skills change as I use them and find where they give bad advice.
