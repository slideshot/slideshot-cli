# Slideshot Agent Skills

Public agent skill definitions for Slideshot.

This repository currently publishes one canonical skill, `slideshot-cli`, which teaches agent tools how to use the public [`slideshot-cli`](https://www.npmjs.com/package/slideshot-cli) package to record product demo videos from a natural-language goal.

## Install

List the skills in this repository:

```bash
npx skills add <owner>/<repo> --list
```

Install the skill for a specific agent:

```bash
npx skills add <owner>/<repo> --skill slideshot-cli --agent codex
npx skills add <owner>/<repo> --skill slideshot-cli --agent claude-code
npx skills add <owner>/<repo> --skill slideshot-cli --agent cursor
```

Install it globally:

```bash
npx skills add <owner>/<repo> --skill slideshot-cli --agent codex --global
```

## Repository Layout

```text
skills/
  slideshot-cli/
    SKILL.md
    references/
      REFERENCE.md
```

## Agent Compatibility

This repo follows the shared [Agent Skills specification](https://agentskills.io/specification), so the same skill can be installed into Codex, Claude Code, Cursor, and other Agent Skills-compatible tools without maintaining separate copies.

Notes from the current `skills` ecosystem:

- `skills.sh` / `npx skills add` discovers skills in `skills/`, which is why this repository uses that layout.
- Claude Code marketplace manifests such as `.claude-plugin/plugin.json` or `.claude-plugin/marketplace.json` are optional. They are only needed if this repo is later distributed through the Claude plugin marketplace specifically.
- `allowed-tools` support is uneven across agents, so this repo intentionally avoids depending on it.

## Local Validation

Check that the repository is discoverable by the installer:

```bash
npx skills add . --list
```

## Runtime Dependency

The skill itself instructs agents to execute Slideshot through:

```bash
npx -y slideshot-cli <command>
```

That keeps the runtime path simple for end users while still allowing maintainers to use `pnpm dlx slideshot-cli` locally if they prefer.
