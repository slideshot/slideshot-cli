---
name: slideshot-cli
description: Use Slideshot to record demo videos of a web application from a natural-language goal. Use when the user wants an agent to create, retry, or troubleshoot a Slideshot run, manage Slideshot authentication or saved credentials, download demo artifacts, or improve the goal passed to `slideshot-cli`.
compatibility: Requires Node.js and internet access to run `npx -y slideshot-cli`. Designed to work across Agent Skills-compatible agents such as Codex, Claude Code, Cursor, and similar tools.
---

# Slideshot CLI

Use this skill when the task is to create a Slideshot demo run, refine the run goal, manage Slideshot credentials, inspect run status, download artifacts, or report product feedback about a run.

## Core workflow

1. Prefer `npx -y slideshot-cli ...` unless `slideshot` is already installed and known to be the intended binary.
2. Keep the CLI in JSON mode by default. Use `--output text` only when you need to present the result directly to the user.
3. Check whether there is already an active or recent run before starting a duplicate retry.
4. Before creating a run, confirm the target URL, whether the app requires login, and whether the user wants one specific flow or multiple separate demos.
5. If the demo requires login, create or reuse a saved credential for the target hostname before starting the run.
6. Write one strong `--goal` that describes a single coherent demo path with a clear end state.
7. Create the run, wait for completion, then branch on the result:
   - `succeeded`: list artifacts and download the requested files.
   - `awaiting_input`: surface the prompt to the user and continue with `runs input`.
   - `failed` or `cancelled`: inspect status, identify the likely cause, and retry with a better goal, better credentials, or different options.
8. When the issue looks like a product bug or a missing feature, offer to send feedback with the related run ID. The CLI feedback command requires a signed-in user session.

## Prerequisites

Start by checking the CLI surface if you are unsure which command to use:

```bash
npx -y slideshot-cli --help
```

If auth state matters, inspect it first:

```bash
npx -y slideshot-cli auth status --output text
```

## Command map

Use this escalation pattern:

| Need | Command |
| --- | --- |
| Check whether auth is available locally | `auth status` |
| Create or inspect saved target-app credentials | `credentials ...` |
| Start a new demo recording run | `runs create` |
| Poll until the run finishes or needs help | `runs wait` |
| Inspect one run in detail | `runs status` |
| Continue an OTP or magic-link flow | `runs input` |
| Fetch output artifact names or download files | `runs artifacts`, `runs download` |
| Inspect the exact command contract | `schema` |
| Report a bug or product gap | `feedback` |

## Goal-writing guidance

A good Slideshot goal gives the runner enough detail to normalize the task into concrete success criteria without guessing.

Include:

- The exact product flow to show, in order.
- The intended starting state if it matters, especially authentication state, workspace, or existing data.
- The visible end state that should be on screen when the demo finishes.
- Any interaction details that are important to the demo, such as opening a modal, using a shortcut, or showing a specific chart or settings panel.
- Visual or capture constraints that materially affect the output, such as blurred emails or shortcut overlays.

Avoid:

- Vague requests like "show the app" or "make a demo of onboarding."
- Bundling unrelated journeys into one run when separate runs would be clearer.
- Omitting critical blockers such as required credentials, tenant/workspace choice, or prerequisite records the flow depends on.

If the request is underspecified but not blocked, make the smallest reasonable assumption and keep the flow narrow. If the run would likely fail without one missing detail, ask one concise clarification question before creating it.

## Good goal examples

Strong:

`Sign in to https://app.example.com, open the existing "Q2 Pipeline" workspace, create a report named "Weekly Revenue", apply the last-30-days filter, and end on the analytics dashboard with the revenue chart fully visible.`

Strong:

`Open the pricing page, scroll through the plan cards, open the enterprise contact modal, fill the work email field, and stop with the modal confirmation state visible. Blur any visible email addresses in the recording.`

Weak:

`Show the product.`

Weak:

`Go through onboarding, settings, billing, analytics, and admin.`

## Operational notes

- Use `schema` instead of guessing request or response shapes.
- Stable public artifact names are `raw.mp4`, `demo.mp4`, and `plan.json`.
- `runs wait` exits with deterministic terminal codes, so treat it as the main control point for automation.
- When a run is blocked on OTP or magic-link input, continue the existing run with `runs input` instead of starting over.
- Avoid redundant reruns. If a run already succeeded, download artifacts from that run instead of creating another one. If it is awaiting input, continue it instead of replacing it.

See [the reference guide](references/REFERENCE.md) for concrete commands, option JSON examples, and failure-recovery patterns.
