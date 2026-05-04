# Slideshot MCP Reference

This reference covers concrete tool calls when running the Slideshot skill via the MCP runtime. The MCP server's per-tool descriptions are authoritative for arguments; this document is for choosing the right tool and surfacing the argument shapes you typically need.

## Connection

Tools are exposed under the `slideshot` MCP server connection. Look for tools like `create_run`, `list_runs`, `list_credentials`, etc.

If the slideshot MCP tools are not available, tell the user the Slideshot connector is not connected and instruct them to install/enable it and complete the OAuth sign-in. Do not guess at credentials or retry blindly.

If a tool call fails with an auth error, ask the user to re-authenticate the Slideshot connector instead of changing arguments.

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

## `create_run` options shape

Pass options as a structured object on the `create_run` call.

Default saved credential plus demo styling:

```json
{
  "auth": { "source": "default" },
  "video": {
    "blur_emails": true,
    "shortcuts": true,
    "background": {
      "type": "gradient",
      "from": "#0B1020",
      "to": "#1D4ED8",
      "direction": "diagonal",
      "scale": 0.95
    }
  }
}
```

Pin a specific saved credential:

```json
{
  "auth": {
    "source": "saved",
    "id": "11111111-2222-3333-4444-555555555555"
  }
}
```

Public flow (no login):

```json
{
  "auth": { "source": "none" }
}
```

Opt into GIF export:

```json
{
  "artifacts": { "gif": true }
}
```

Notes:

- Omit `auth` entirely or use `{ "source": "none" }` for genuinely public flows.
- Use nested `auth` and `video` objects. Legacy flat option keys are not supported.

## Awaiting input

When `get_run` returns `status: "awaiting_input"`, surface `awaiting_input.prompt` to the user verbatim and continue the existing run with `submit_run_input` on the same `run_id`. The most common cases are authentication OTPs and magic-link codes. Never start a new run for these.

## Artifacts and downloads

When `get_run` returns `status: "succeeded"`, call `list_run_artifacts`. The returned `download_url` on each artifact is an authenticated, time-limited URL — hand it to the user or use it directly to fetch the file. If a `download_url` expires, re-fetch the artifact list.

Stable public artifact names returned by `list_run_artifacts`:

- `raw.mp4`
- `demo.mp4`
- `plan.json`
- `demo.gif` (only when the run was created with `artifacts.gif: true`)

## Credentials

`list_credentials` returns saved credentials with `domain` (hostname), `email`, `default` flag, and `id`. Use `domain` for hostname-based matching against the target URL.

`create_credential` accepts `domain`, `email`, optional `password`, optional `label`, and optional `default` flag. Email-only credentials are valid because the password field is optional.

`set_default_credential` pins a credential as the default for its hostname so future runs can resolve it via `auth.source = "default"`.

Prefer asking the user to add credentials securely in the web app at [app.slideshot.ai](https://app.slideshot.ai) when the run requires stable login secrets, instead of pasting long-lived secrets into chat. Only call `create_credential` from chat when the user explicitly asks for it.

## Feedback

`submit_feedback` accepts a free-text `message` and an optional list of related `run_ids`. Use it for product bugs, feature requests, or general frustration even when no specific run failed.
