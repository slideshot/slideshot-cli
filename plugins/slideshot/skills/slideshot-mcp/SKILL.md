---
name: slideshot-mcp
description: Use Slideshot to record demo videos of a web application from a natural-language flow description, via the hosted Slideshot MCP server (slideshot tools `create_run`, `get_run`, `list_runs`, `cancel_run`, `submit_run_input`, `list_run_artifacts`, `list_credentials`, `create_credential`, `update_credential`, `set_default_credential`, `delete_credential`, `submit_feedback`). Use it when the user wants the agent to record a new demo video of a target web app, demonstrate a feature or specific flow, manage saved target-app credentials, cancel runs, inspect a specific run, fetch demo artifacts, or send Slideshot product feedback.
compatibility: Requires the `slideshot` MCP server connection (https://api.slideshot.ai/mcp) configured by the host plugin and an authenticated OAuth session.
---

# Slideshot (MCP)

Use this skill when the task is to record a demo video of a feature or specified user flow of a web application. Slideshot's MCP server lets you kick off a new recording run, refine the run goal, manage saved target-app credentials, inspect run status, fetch artifacts, or submit product feedback - all as MCP tool calls instead of shell commands.

This skill complements the per-tool descriptions exposed by the MCP server. The tool descriptions are authoritative for arguments and per-call rules. This skill is authoritative for *workflow*: when to call which tool, what to ask the user before and after each step, and how to handle the lifecycle of a run.

## Core workflow

1. Confirm the MCP server connection is available before assuming the tools are usable. The slideshot MCP tools are namespaced by the connection, so look for tools like `create_run`, `list_runs`, `list_credentials`, etc. exposed by the `slideshot` server.
2. If the slideshot MCP tools are not available, tell the user the Slideshot connector is not connected and instruct them to install/enable the plugin and complete the OAuth sign-in.
3. Before creating a run, confirm:
   - The target URL.
   - Whether the app requires login.
   - The flow the user wants recorded, in enough detail to write a single coherent goal.
4. If the user has not specified video customization, ask once before `create_run` whether they want any of these options. Ask the questions one at a time, do not bundle them, and explain what each option does instead of just naming the field:
   - Login handling: no login, default saved credential for the target hostname, or a specific saved credential.
   - Blur visible emails during recording (`video.blur_emails`).
   - Show keyboard shortcuts in the demo video (`video.shortcuts`).
   - Cursor style (`video.cursor`: `small`, `default`, `large`, or `none`).
   - Video background (`video.background`: `solid` or `gradient`; collect colors and direction if needed).
   - Output video size (`video.size.width`/`height`) and inner content layout (`video.size.content.padding` or `video.size.content.scale`).
   - Export `demo.gif` as well as MP4 (`artifacts.gif`, default `false`).

   If the user has no preference for an option, omit it instead of inventing a value. The MCP tool description for `create_run` requires this confirmation step explicitly; do not skip it for "speed".
5. If the demo requires login, do credential preflight before `create_run`:
   - Call `list_credentials` and look for a saved credential whose `domain` matches the target URL hostname.
   - Prefer the matching default credential when one exists. Use `options.auth = { source: "default" }` so the run resolves it automatically.
   - To pin a specific credential, use `options.auth = { source: "saved", id: "<credential-uuid>" }`.
   - Use `options.auth = { source: "none" }` (or omit `auth`) only for genuinely public flows.
   - If no suitable saved credential exists, prefer asking the user to add it securely in the web app at [app.slideshot.ai](https://app.slideshot.ai) instead of pasting long-lived secrets into chat. Only use `create_credential` from chat when the user explicitly asks for it.
   - Keep the run target URL hostname aligned with the credential domain. Credential matching is hostname-based.
6. Write one strong `goal` per run that describes a single coherent demo path with a clear visible end state. If the description is underspecified, ask 1-2 focused follow-up questions before calling `create_run`.
7. After `create_run`:
   - Tell the user the `run_id` returned by the tool.
   - Tell the user to monitor that specific run in the web app at `https://app.slideshot.ai/?runId=<run-id>`.
   - If multiple runs were created, give one per-run URL for each run, not a generic runs list.
   - Do not poll `get_run` in a loop. Stop and wait for the user to ask you to inspect a run, provide input for a run, cancel it, or fetch artifacts after completion.
8. When the user later asks you to inspect a run, call `get_run` and branch on `status`:
   - `succeeded`: call `list_run_artifacts` and surface the available files. The `download_url` on each artifact is an authenticated, time-limited URL - hand it to the user or use it directly to fetch the file if you have HTTP access.
   - `awaiting_input`: surface `awaiting_input.prompt` to the user and continue with `submit_run_input` on the same `run_id`. This is the right path for OTPs and magic-link codes - never start a new run for these.
   - `failed` or `cancelled`: report `error` and the likely cause, then offer to retry with a better goal, corrected credentials, or different options. Change one thing at a time on retry.
   - `queued`, `running`, anything in progress: report status and stop. Do not poll.
9. To stop a run that is clearly wrong, duplicate, or no longer wanted, call `cancel_run`. Only do this when the user explicitly asks or when the run is obviously off-target.
10. When a run failure looks like a Slideshot bug or product gap, offer to send feedback with `submit_feedback` and include the related `run_ids`. You can also use `submit_feedback` for general feature requests or product frustration even when no specific run failed.
11. Avoid redundant reruns. If a run already succeeded, fetch its artifacts instead of creating another one. If a run is awaiting input, continue it instead of replacing it. Use `list_runs` if you need to find an existing run before starting a fresh one.

## Tool map

Use this escalation pattern. The MCP server's per-tool descriptions are authoritative for arguments; this table is for choosing the right tool.

| Need | Tool |
| --- | --- |
| Find existing runs and avoid redundant reruns | `list_runs` |
| Inspect one specific run before deciding what to do next | `get_run` |
| Start a new demo recording run | `create_run` |
| Continue an OTP or magic-link flow | `submit_run_input` |
| Cancel a queued or running run the user wants stopped | `cancel_run` |
| Surface or fetch the output files for a finished run | `list_run_artifacts` |
| Check whether a saved login already exists for the target hostname | `list_credentials` |
| Add a saved login from chat (only when the user explicitly asks) | `create_credential` |
| Update or correct an existing saved credential | `update_credential` |
| Pick a default credential for a hostname | `set_default_credential` |
| Remove a saved credential the user no longer wants | `delete_credential` |
| Send a bug report, feature request, or product feedback | `submit_feedback` |

## Goal-writing guidance

A good Slideshot goal gives the runner enough detail to normalize the task into concrete success criteria without guessing.

Include:

- The exact product flow to show, in order.
- The intended starting state if it matters, especially authentication state, workspace, or seeded data.
- The visible end state that should be on screen when the demo finishes.
- Important interaction details such as opening a modal, using a keyboard shortcut, or showing a specific chart or settings panel.
- Visual or capture constraints that materially affect the output, such as blurred emails or shortcut overlays.
- Any specific wait time that matters for the demo.

Avoid:

- Vague requests like "show the app" or "make a demo of onboarding".
- Bundling unrelated journeys into one run when separate runs would be clearer.
- Omitting critical blockers such as required credentials, tenant/workspace choice, or prerequisite records the flow depends on.

If the request is underspecified, ask 1-3 focused follow-up questions about the desired flow before calling `create_run`.

### Strong examples

`Sign in to https://app.example.com, open the existing "Q2 Pipeline" workspace, create a report named "Weekly Revenue", apply the last-30-days filter, and end on the analytics dashboard with the revenue chart fully visible.`

`Open the pricing page, scroll through the plan cards, open the enterprise contact modal, fill the work email field, and stop with the modal confirmation state visible. Blur any visible email addresses in the recording.`

### Weak examples

`Show the product.`

`Go through onboarding, settings, billing, analytics, and admin.`

## Operational notes

- The MCP connection is OAuth-protected. If a tool call fails with an auth error, ask the user to re-authenticate the Slideshot connector instead of guessing at credentials.
- `create_run` is billable and produces a user-facing video. Wrong assumptions waste both time and money - confirm customization before calling it.
- Stable public artifact names returned by `list_run_artifacts` are `raw.mp4`, `demo.mp4`, and `plan.json`. `demo.gif` is also returned when the run was created with `options.artifacts.gif=true`.
- `download_url` values on artifacts are authenticated and time-limited. Use them as soon as they are returned, or re-fetch via `list_run_artifacts` if they expire.
- When a run is `awaiting_input`, `awaiting_input.prompt` tells you what to ask the user. The most common cases are authentication OTPs and magic-link codes. Continue the same run with `submit_run_input` instead of starting over.
- Saved credential matching is hostname-based. If the saved credential `domain` is `app.example.com`, prefer a target URL like `https://app.example.com/...` rather than a different hostname that later redirects there.
- Email-only saved credentials are valid; the password field is optional.
- When the user is generally frustrated, reports a product problem, or shares a feature request, `submit_feedback` is appropriate even when no specific run failed.
