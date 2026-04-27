# Slideshot Claude Code Plugin

A Claude Code plugin that bundles:

- The hosted **Slideshot MCP** server connection (`https://api.slideshot.ai/mcp`, Streamable HTTP, OAuth 2.0).
- A workflow **skill** (`slideshot-mcp`) that teaches the agent when and how to call each MCP tool.

Once installed, ask Claude Code to record a demo of a flow and it will use the MCP tools to drive Slideshot.

## What it adds

- MCP tools exposed under the `slideshot` server: `create_run`, `get_run`, `list_runs`, `cancel_run`, `submit_run_input`, `list_run_artifacts`, `list_credentials`, `create_credential`, `update_credential`, `set_default_credential`, `delete_credential`, `submit_feedback`.
- The `slideshot-mcp` skill, which encodes the run lifecycle, credential preflight, customization confirmation, and failure-recovery patterns.

## Install

From the local repo (during development):

```bash
claude plugin install ./plugins/slideshot --scope project
```

From a published source (once available):

```bash
claude plugin install slideshot
```

After installation, run `/mcp` inside Claude Code and authenticate the `slideshot` connection. The OAuth flow opens [app.slideshot.ai](https://app.slideshot.ai) in a browser; sign in (or create an account) and approve the connection.

## Layout

```
plugins/slideshot/
├── .claude-plugin/plugin.json   # Plugin manifest
├── .mcp.json                    # Remote MCP server config
├── README.md
└── skills/slideshot-mcp/
    └── SKILL.md
```

## Relationship to the CLI skill

This repo also publishes the freestanding `slideshot-cli` skill at `skills/slideshot-cli/`, which targets the `slideshot-cli` npm package and uses local API-key auth. The two skills are independent:

- Use the **plugin** (this folder) when you want OAuth-based MCP tool calls inside Claude Code.
- Use the **CLI skill** when you want to drive Slideshot from a shell environment via `npx -y slideshot-cli`.

Do not mix invocations between the two in a single recommendation.
