# Slideshot CLI Reference

## Invocation style

Prefer:

```bash
npx -y slideshot-cli <command>
```

If the `slideshot` binary is already installed and clearly intended, using `slideshot <command>` is equivalent.

If a maintainer explicitly prefers pnpm in their local environment, `pnpm dlx slideshot-cli <command>` is equivalent, but user-facing examples should prefer `npx -y slideshot-cli`.

The CLI defaults to compact JSON output for automation. Add `--output text` only when a human-readable response is more useful than a machine-readable one.

## Authentication

Prefer API-key auth for automated run execution. Use a signed-in user session when you need account-scoped commands such as `runs list` or `feedback`.

Session login:

```bash
npx -y slideshot-cli auth login --email you@example.com
```

Store or inspect an API key:

```bash
npx -y slideshot-cli auth set-key srk_...
npx -y slideshot-cli auth status --output text
npx -y slideshot-cli auth env --shell sh --output text
```

Notes:

- Command auth resolution prefers `--api-key`, then `SLIDESHOT_API_KEY`, then a stored local API key, then a stored user session when the command supports it.
- `runs list` and `feedback` require a signed-in user session.
- `runs create` can bootstrap a local API key from a stored user session if needed.

## Saved credentials for the target app

Create a credential before the run when the target app requires login and the user has already provided valid credentials.

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
- If the run depends on login and no credential exists yet, create the credential first instead of hoping the runner can improvise.

## Waiting, status, and input

Wait for completion:

```bash
npx -y slideshot-cli runs wait <run-id>
```

Terminal exit codes for `runs wait`:

- `0`: succeeded
- `10`: awaiting input
- `11`: failed
- `12`: cancelled

Inspect one run:

```bash
npx -y slideshot-cli runs status <run-id> --output text
```

If the run is waiting for user input, surface the prompt to the user and continue the same run:

```bash
npx -y slideshot-cli runs input <run-id> --value "123456"
```

The most common awaiting-input cases are authentication OTPs or magic-link flows.

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
- `runs.wait`
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
- If the runner reached the wrong area of the app, rewrite the goal so the sequence and end state are explicit.
- If the app needs a very specific workspace, tenant, or seeded record, state that directly in the goal.
- If the run hit an OTP or magic-link prompt, continue it with `runs input` rather than creating a fresh run.

## Feedback

When the problem looks like a Slideshot bug or a product gap, offer to send feedback and include the run ID for context:

```bash
npx -y slideshot-cli feedback "The run stopped after login even though the dashboard loaded." --run-id <run-id>
```

Important:

- The CLI `feedback` command uses the signed-in user session, not just an API key.
- If only an API key is configured, ask the user whether they want to sign in first so the feedback can be submitted from the CLI.
