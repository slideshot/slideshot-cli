# Slideshot CLI Reference

## Invocation style

Prefer:

```bash
npx -y slideshot-cli <command>
```

If the `slideshot` binary is already installed and clearly intended, using `slideshot <command>` is equivalent. Otherwise, agent should prefer `npx -y slideshot-cli`.

The CLI defaults to compact JSON output for automation. Add `--output text` only when a human-readable response is more useful than a machine-readable one.

## Authentication

Prefer login over API-key-only setup. A stored login gives the user session and usually also a local API key, which covers more CLI workflows than an API key by itself.

Before running a login command, ask whether the user already has a Slideshot account or wants to set one up now.

Preferred login flow when supported by the installed CLI:

```bash
npx -y slideshot-cli auth login --email you@example.com --email-only
```

Use email+password when the user already has a password, explicitly wants that flow, or the installed CLI build does not expose `--email-only`:

```bash
npx -y slideshot-cli auth login --email you@example.com
printf '%s' 'secret' | npx -y slideshot-cli auth login --email you@example.com --password-stdin
```

Store or inspect an API key only when the user explicitly wants key-only auth or already has a key:

```bash
npx -y slideshot-cli auth set-key srk_...
npx -y slideshot-cli auth status
npx -y slideshot-cli auth env --shell sh --output text
```

Notes:

- `auth status` is the quickest way to discover whether the CLI already has a stored user session, a stored API key, and which auth file path is in use.
- `auth login` stores the user session locally and creates a local API key by default unless `--skip-api-key` is passed.
- Default to `--email-only` when available because it matches Slideshot's passwordless behavior and can create the account if it does not already exist.
- If the installed CLI help on the machine does not show `--email-only`, use the best available login flow on that build and mention the mismatch.
- Command auth resolution prefers `--api-key`, then `SLIDESHOT_API_KEY`, then a stored local API key, then a stored user session when the command supports it.
- `runs list` and `feedback` require a signed-in user session.
- `runs create` can bootstrap a local API key from a stored user session if needed, but API-key-only auth still lacks account-scoped commands like `runs list`.

## Saved credentials for the target app

Before creating a login-required run, list saved credentials and check for a matching hostname first. Prefer the default matching credential for that hostname.

```bash
npx -y slideshot-cli credentials list
```

Create a credential only when you have no suitable saved credential and the user explicitly wants to add one from the CLI:

```bash
npx -y slideshot-cli credentials create \
  --label Main \
  --domain app.example.com \
  --email you@example.com \
  --password secret \
  --default
```

Useful commands:

```bash
npx -y slideshot-cli credentials list --output text
npx -y slideshot-cli credentials set-default <credential-id>
npx -y slideshot-cli credentials delete <credential-id>
```

Notes:

- `--domain` should be the hostname only, such as `app.example.com`.
- Password is optional, so email-only credentials are allowed.
- Use `--default` when the same hostname should usually reuse one credential.
- The run target URL hostname should match the saved credential domain. Credential matching is hostname-based.
- Example: if the saved credential domain is `app.example.com`, prefer a target URL like `https://app.example.com/...` instead of a different hostname that later redirects there.
- Only ask the user for login details if no matching saved credential exists or the run later requires OTP or magic-link input.
- When the run requires login and no suitable credential exists, prefer suggesting that the user add credentials securely in the web app at [app.slideshot.ai](https://app.slideshot.ai) instead of pasting long-lived secrets into chat by default.

## Creating a run

Basic run:

```bash
npx -y slideshot-cli runs create https://app.example.com \
  --goal "Sign in, create a project, and end on the analytics dashboard."
```

If you need structured options, prefer a JSON file:

```bash
npx -y slideshot-cli runs create https://app.example.com \
  --goal "Sign in, create a project, and end on the analytics dashboard." \
  --options @options.json
```

Example `options.json` using the default saved credential plus demo styling:

```json
{
  "auth": {
    "source": "default"
  },
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

Example using one specific saved credential:

```json
{
  "auth": {
    "source": "saved",
    "id": "11111111-2222-3333-4444-555555555555"
  }
}
```

Important details:

- Omit `auth` or use `{ "source": "none" }` when no saved credential should be used.
- Legacy flat option keys are rejected; use nested `auth` and `video` objects.
- Keep the target URL hostname aligned with the credential domain when the flow depends on saved credentials.
- If the run depends on login and no credential exists yet, either create the credential intentionally or send the user to [app.slideshot.ai](https://app.slideshot.ai) to add it securely before you create the run.
- If the user has not specified any video styling choices yet, ask once before creating the run whether they want:
  - `video.blur_emails`
  - `video.shortcuts`
  - `video.background` with `none`, `solid`, or `gradient`
- If the user says they have no preference, keep the defaults by omitting those keys instead of inventing style choices.

## Status, input, and the web app

Inspect one run:

```bash
npx -y slideshot-cli runs status <run-id> --output text
```

If the run is waiting for user input, surface the prompt to the user and continue the same run:

```bash
npx -y slideshot-cli runs input <run-id> --value "123456"
```

The most common awaiting-input cases are authentication OTPs or magic-link flows.

After creating a run:

1. Return the `run_id` to the user.
2. Tell the user to monitor that specific run in the web app at `https://app.slideshot.ai/?runId=<run-id>`.
3. If useful, open that specific run URL in the system browser. Example on macOS:

```bash
open "https://app.slideshot.ai/?runId=<run-id>"
```

4. If multiple runs were created, provide one per-run URL for each run and open each individual URL if browser opening is useful.
5. Do not keep polling automatically. Wait for the user to ask you to inspect a specific run, provide input for a specific run, cancel it, or download artifacts after completion.

## Artifacts

List public artifacts:

```bash
npx -y slideshot-cli runs artifacts <run-id> --output text
```

Download the polished demo video:

```bash
npx -y slideshot-cli runs download <run-id> \
  --artifact demo.mp4 \
  --dir ./artifacts \
  --output text
```

Download everything public:

```bash
npx -y slideshot-cli runs download <run-id> --all --dir ./artifacts
```

Stable public artifact names:

- `raw.mp4`
- `demo.mp4`
- `plan.json`

## Using `schema` to avoid guessing

Inspect the exact contract when an integration needs request or response details:

```bash
npx -y slideshot-cli schema runs.create --output text
npx -y slideshot-cli schema credentials.create --output text
npx -y slideshot-cli schema feedback --output text
```

Useful command IDs include:

- `runs.create`
- `runs.status`
- `runs.list`
- `runs.cancel`
- `runs.input`
- `runs.artifacts`
- `runs.download`
- `credentials.list`
- `credentials.create`
- `credentials.update`
- `credentials.set-default`
- `credentials.delete`
- `feedback`

## Failure recovery

When a run fails:

1. Check `runs status` to see the terminal state and any error details.
2. Confirm whether the target flow was blocked by missing credentials, missing OTP input, or an overly vague goal.
3. Retry with one improvement at a time instead of changing goal, credentials, and options all at once.

Common fixes:

- If login failed, create or correct the saved credential and retry.
- If the app needs a very specific workspace, tenant, or seeded record, state that directly in the goal.
- If the run hit an OTP or magic-link prompt, continue it with `runs input` rather than creating a fresh run.
- If the run is no longer wanted or was created with the wrong goal, cancel it instead of waiting for it to finish:

```bash
npx -y slideshot-cli runs cancel <run-id>
```

## Feedback

When the problem looks like a Slideshot bug or a product gap, offer to send feedback and include the run ID for context:

```bash
npx -y slideshot-cli feedback "The run stopped after login even though the dashboard loaded." --run-id <run-id>
```

Important:

- The CLI `feedback` command uses the signed-in user session, not just an API key.
- If only an API key is configured, ask the user whether they want to sign in first so the feedback can be submitted from the CLI.
- If the user is generally frustrated or has a feature request, you can still use `feedback` even if the issue is not tied to a failed run.
