---
name: slideshot
description: Use Slideshot to record demo videos of a web application from a natural-language flow description. Use it when the user wants the agent to create a new demo video recording of a target web app, demonstrate a feature or specific flow, manage saved target-app credentials in Slideshot, cancel runs, inspect a specific run, fetch demo artifacts, or send Slideshot product feedback.
last-updated: 2026-05-04
---

# Slideshot

Use this skill when the task is to record a demo video of a feature or specified user flow of a web application. Slideshot lets you kick off a new recording run, refine the run goal, manage saved target-app credentials, inspect run status, fetch artifacts, or send product feedback.

## Choose a runtime

Slideshot exposes the same workflow through two runtimes. Pick one at the start of the session and use it consistently — do not mix invocations between them in a single recommendation.

- **MCP runtime** — if `slideshot.*` MCP tools are available in this session (`create_run`, `get_run`, `list_runs`, `cancel_run`, `submit_run_input`, `list_run_artifacts`, `list_credentials`, `create_credential`, `update_credential`, `set_default_credential`, `delete_credential`, `submit_feedback`), use them. See [`references/mcp.md`](references/mcp.md) for tool argument shapes, OAuth handling, and credential examples.
- **CLI runtime** — otherwise, invoke the public [`slideshot-cli`](https://www.npmjs.com/package/slideshot-cli) npm package via `npx -y slideshot-cli ...`. See [`references/cli.md`](references/cli.md) for command syntax, API-key auth, and JSON option examples.

If neither MCP tools nor a shell are available, tell the user the Slideshot connector is not connected and stop. Do not invent another execution path.

## Core workflow

1. Make sure the user is authenticated into Slideshot before starting. Auth specifics differ between runtimes — see the matching runtime reference.
2. Before creating a run, confirm:
   - The target URL.
   - Whether the app requires login.
   - The flow the user wants recorded, in enough detail to write a single coherent goal.
3. If the user has not specified video customization, ask once before creating the run whether they want any of these options. Ask the questions one at a time, do not bundle them, and explain what each option does instead of just naming the field:
   - Login handling: no login, default saved credential for the target hostname, or a specific saved credential.
   - Blur visible emails during recording.
   - Show keyboard shortcuts in the demo video.
   - Cursor style (`small`, `default`, `large`, or `none`).
   - Video background (`solid` or `gradient`; collect colors and direction if needed).
   - Output video size and inner content layout (padding or scale).
   - Export `demo.gif` as well as MP4 (default `false`, opt-in only).

   If the user has no preference for an option, omit it instead of inventing a value.   
4. If the demo requires login, do credential preflight before creating the run:
   - List saved credentials and look for one whose domain matches the target URL hostname.
   - Prefer the matching default credential when one exists. The auth source for that case is `default`.
   - To pin a specific credential, reference it by id with auth source `saved`.
   - For genuinely public flows, use auth source `none` (or omit auth entirely).
   - If no suitable saved credential exists, prefer asking the user to add it securely in the web app at [app.slideshot.ai](https://app.slideshot.ai) instead of pasting long-lived secrets into chat. Only create credentials from chat when the user explicitly asks for it.
   - Keep the run target URL hostname aligned with the credential domain. Credential matching is hostname-based.
5. Write one strong goal per run that describes a single coherent demo path with a clear visible end state. If the description is underspecified, ask 1–2 focused follow-up questions before creating the run.
   - If the user needs multiple flows, kick off each recording run one by one. The runs enter the queue and are processed in order.
6. After creating a run:
   - Tell the user the `run_id` returned by the runtime.
   - Tell the user to monitor that specific run in the web app at `https://app.slideshot.ai/?runId=<run-id>`.
   - If multiple runs were created, give one per-run URL for each run, not a generic runs list.
   - Do not poll for run status in a loop. Stop and wait for the user to ask you to inspect a run, provide input for a run, cancel it, or fetch artifacts after completion.
7. When the user later asks you to inspect a specific run, branch on its status:
   - `succeeded`: list artifacts and surface or download the requested files.
   - `awaiting_input`: surface the prompt to the user and continue with the same run via the runtime's input mechanism. This is the right path for OTPs and magic-link codes — never start a new run for these.
   - `failed` or `cancelled`: report the error and the likely cause, then offer to retry with a better goal, corrected credentials, or different options. Change one thing at a time on retry.
   - `queued`, `running`, anything in progress: report status and stop. Do not poll.
8. To stop a run that is clearly wrong, duplicate, or no longer wanted, cancel it. Only do this when the user explicitly asks or when the run is obviously off-target.
9. When a run failure looks like a Slideshot bug or product gap, offer to send feedback and include the related run id. You can also send feedback for general feature requests or product frustration even when no specific run failed.
10. Avoid redundant reruns. If a run already succeeded, fetch its artifacts instead of creating another one. If a run is awaiting input, continue it instead of replacing it. List recent runs first if you need to find an existing one before starting a fresh one.

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

If the request is underspecified, ask 1–3 focused follow-up questions about the desired flow before creating the run.

### Strong examples

`Sign in to https://app.example.com, open the existing "Q2 Pipeline" workspace, create a report named "Weekly Revenue", apply the last-30-days filter, and end on the analytics dashboard with the revenue chart fully visible.`

`Open the pricing page, scroll through the plan cards, open the enterprise contact modal, fill the work email field, and stop with the modal confirmation state visible. Blur any visible email addresses in the recording.`

### Weak examples

`Show the product.`

`Go through onboarding, settings, billing, analytics, and admin.`

## Operational notes

- Stable public artifact names are `raw.mp4`, `demo.mp4`, and `plan.json`. `demo.gif` is also a public artifact when the run was created with the GIF artifact opt-in.
- When a run is `awaiting_input`, the runtime surfaces a prompt that tells you what to ask the user. The most common cases are authentication OTPs and magic-link codes. Continue the same run instead of starting over.
- Saved credential matching is hostname-based. If the saved credential domain is `app.example.com`, prefer a target URL like `https://app.example.com/...` rather than a different hostname that later redirects there.
- Email-only saved credentials are valid; the password field is optional.
- Creating a run is billable and produces a user-facing video. Wrong assumptions waste both time and money — confirm customization before starting it.
- After creating a run, point the user to the per-run web-app URL `https://app.slideshot.ai/?runId=<run-id>` instead of the generic runs list, and do the same for each run when multiple runs were started.
- When the user is generally frustrated, reports a product problem, or shares a feature request, sending feedback is appropriate even when no specific run failed.

## References

- [`references/mcp.md`](references/mcp.md) — MCP tool argument shapes, OAuth handling, examples for the MCP runtime.
- [`references/cli.md`](references/cli.md) — `slideshot-cli` command syntax, API-key auth, JSON option examples for the CLI runtime.
