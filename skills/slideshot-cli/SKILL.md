---
name: slideshot-cli
description: Use Slideshot to record demo videos of a web application from a natural language flow description. Use it when the user wants an agent to create a new demo video recording of a target web app, demonstrate a feature or specific flow, manage Slideshot login or API-key auth, reuse saved target-app credentials, cancel runs, download demo artifacts, inspect a specific run, or improve the goal passed to `slideshot-cli`.
compatibility: Requires Node.js and internet access to run `npx -y slideshot-cli`
---

# Slideshot CLI

Use this skill when the task is record a demo video of a feature of specified user flow of the web application. Slideshot CLI allows the user to kick off a new video recording run, refine the run goal, manage Slideshot credentials, inspect run status, download artifacts, or report product feedback about a run.

## Core workflow

1. Prefer `npx -y slideshot-cli ...` unless `slideshot` is already installed and known to be the intended binary.
2. Keep the CLI in JSON mode by default. Use `--output text` only when you need to present the result directly to the user.
3. Start with auth preflight when auth state matters:
   - Run `auth status` first if you need to know whether login, a user session, or an API key is already available locally.
   - Prefer `auth login` over `auth set-key`. Login is the default because it stores the user session and usually creates a local API key too, which gives broader CLI coverage than an API key alone.
   - Ask the user whether they already have a Slideshot account or want to set one up before you run the login command.
   - When the installed CLI supports it, prefer `auth login --email <email> --email-only` by default because it matches Slideshot's passwordless flow and can create the account if it does not exist yet.
   - Use email+password login when the user already has a password, explicitly prefers that flow, or the CLI build on the machine does not expose `--email-only`.
   - Use `auth set-key` only when the user explicitly wants API-key-only auth or already has a key and only needs basic run-management commands.
4. Before creating a run, confirm the target URL, whether the app requires login, and ensure the user describes the flow they wish to record in detail.
5. If the user has not already specified video customization, ask once before `runs create` whether they want any of these options:
   - blur visible emails during recording
   - show keyboard shortcuts in `demo.mp4`
   - add a demo-video background (`none`, `solid`, or `gradient`; collect colors if needed)
   If the user has no preference, say you will use the defaults and omit those options.
6. If the demo requires login, do credential preflight before starting the run:
   - Check saved credentials for the target hostname first.
   - Reuse the matching default credential when it exists.
   - Make sure the run target URL uses the same hostname as the saved credential domain. Credential matching is hostname-based.
   - Only ask the user for target-app login details if no suitable saved credential exists or the run later requests OTP or magic-link input.
   - When there is no suitable saved credential and the run requires stable login secrets, suggest the user add the credentials securely in the web app at [app.slideshot.ai](https://app.slideshot.ai) and continue after that, instead of pasting long-lived secrets into chat by default.
7. Write one strong `--goal` that describes a single coherent demo path with a clear end state. If the flow description is not detailed enough, you might want to prompt the user 1-2 times at most to provide more details about certain parts of the flow.
   - If the user needs multiple flows, you can kick off each recording run one by one via Slideshot CLI. The runs will enter the queue and will be processed when it's their turn.
8. After creating a run:
   - Tell the user the run ID.
   - Instruct the user to monitor that specific run in the web app at `https://app.slideshot.ai/?runId=<run-id>`.
   - If it helps, open that specific run URL for them in the system browser with a shell command such as `open "https://app.slideshot.ai/?runId=<run-id>"` on macOS.
   - If you create multiple runs, give the user one per-run web-app URL for each run and, if opening the browser is useful, open each run's URL individually instead of the generic runs list.
   - Do not keep polling or waiting for the run automatically. Stop and wait for the user to tell you what to do next.
   - Expect follow-up requests such as checking a specific run, submitting OTP or magic-link input for a specific run, cancelling a run, or downloading artifacts after success.
9. When the user later asks you to inspect a specific run, branch on the result:
   - `succeeded`: list artifacts and download the requested files.
   - `awaiting_input`: surface the prompt to the user and continue with `runs input`.
   - `failed` or `cancelled`: inspect status, identify the likely cause, and retry with a better goal, better credentials, or different options.
10. If the user asks to stop a run, if a queued or running run is clearly wrong, or if a duplicate run was started by mistake, cancel it with `runs cancel <run-id>`.
11. When the run fails and an issue looks like a product bug or a missing feature, offer to send feedback with the related run ID. The CLI feedback command requires a signed-in user session.
   - If the user is generally frustrated, reports a product problem, or shares a feature request, offer to send that feedback with the CLI feedback command even if it is not tied to a failed run.
12. When the run is complete, the user should download the resulting demo video files or at least be informed that they have the option to download the raw video recording or the editing demo video MP4 files.

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
| Sign in with full account access | `auth login` |
| Store an API key only | `auth set-key` |
| Create or inspect saved target-app credentials | `credentials ...` |
| Start a new demo recording run | `runs create` |
| Tell the user where to monitor run progress | open `https://app.slideshot.ai/?runId=<run-id>` |
| Inspect one run in detail | `runs status` |
| Continue an OTP or magic-link flow | `runs input` |
| Cancel an existing run if the user explicitly asks for it | `runs cancel` |
| Fetch output artifact names or download files | `runs artifacts`, `runs download` |
| Inspect the exact command contract | `schema` |
| Report a bug, feature request, or product gap | `feedback` |

## Goal-writing guidance

A good Slideshot goal gives the runner enough detail to normalize the task into concrete success criteria without guessing.

Include:

- The exact product flow to show, in order.
- The intended starting state if it matters, especially authentication state, workspace, or existing data.
- The visible end state that should be on screen when the demo finishes.
- Any interaction details that are important to the demo, such as opening a modal, using a shortcut, or showing a specific chart or settings panel.
- Visual or capture constraints that materially affect the output, such as blurred emails or shortcut overlays.
- Any specific wait time that might be important for the demo

Avoid:

- Vague requests like "show the app" or "make a demo of onboarding."
- Bundling unrelated journeys into one run when separate runs would be clearer.
- Omitting critical blockers such as required credentials, tenant/workspace choice, or prerequisite records the flow depends on.

If the request is underspecified, you should prompt the user to answer 2-3 follow-up questions to provide more details about the desired flow in the web app before creating the recording run.

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
- When a run is blocked on OTP or magic-link input, continue the existing run with `runs input` instead of starting over.
- Avoid redundant reruns. If a run already succeeded, download artifacts from that run instead of creating another one. If it is awaiting input, continue it instead of replacing it.
- Saved credential matching depends on the target URL hostname matching the credential domain.
- Email-only saved credentials are valid because target-app credential passwords are optional.
- If no customization options were specified, ask once about `video.blur_emails`, `video.shortcuts`, and `video.background` before you create the run.
- After creating a run, point the user to the per-run web-app URL `https://app.slideshot.ai/?runId=<run-id>` instead of the generic runs list, and do the same for each run when multiple runs were started.

See [the reference guide](references/REFERENCE.md) for concrete commands, option JSON examples, web-app monitoring guidance, and failure-recovery patterns.
