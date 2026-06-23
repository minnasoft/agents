# agents

> Because internal config deserves taste too.

agents is minnasoft's internal OpenCode config.

It's not trying to be a generic agent marketplace. It's the little pile of agents, commands, and skills we want available in our projects: opinionated, sharp-edged, and cute enough to keep around.

## Installation

Clone this repo directly into a project's `.opencode` directory:

```bash
git clone https://github.com/minnasoft/agents.git .opencode
```

Run that from the target project root. If the project already has a `.opencode` directory, copy or merge these files instead of overwriting local project configuration.

The repository root should contain OpenCode's expected folders directly:

```text
agents/
commands/
skills/
```

Restart OpenCode after cloning or changing these files. Agents, commands, and skills are loaded at startup.

## What's Inside?

[`skills/elixir`](skills/elixir/README.md), the minnasoft Elixir expert for plan, review, audit, and build loops.

## Permission Model

The role agents use a blacklist-shaped hybrid permission model. Bash is broadly allowed so cross-language projects do not need endless command allowlists. Role boundaries are the blacklisted part: `staff` coordinates, reviews results, and owns git/GitHub publishing; `research` and `review` do not publish or edit; `eng` is the implementation worker and can run formatters, tests, codegen, dependency commands, and other slice-local implementation work.

Git staging, commits, pushes, and external PR/GitHub publishing stay with `staff` after review. This keeps the loop iterative: implementation workers change files, review workers challenge the result, then staff decides what to commit or publish.

## Compatibility

This config is OpenCode-only today. Command filenames use `:` for namespacing, so Windows checkout support is not a goal.

Generated dependencies such as `node_modules`, `package.json`, and lockfiles are ignored and should not be committed.

## License

MIT
