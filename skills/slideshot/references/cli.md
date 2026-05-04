# Slideshot CLI Reference

This reference covers concrete commands when running the Slideshot skill via the CLI runtime. It assumes the user has access to a shell and Node.js for `npx -y slideshot-cli`.

## Invocation style

Prefer:

```bash
npx -y slideshot-cli <command>
```

If the `slideshot` binary is already installed and clearly intended, `slideshot <command>` is equivalent. Otherwise prefer `npx -y slideshot-cli`.

The CLI defaults to compact JSON output for automation. Add `--output text` only when a human-readable response is more useful than a machine-readable one.

## Authentication

Prefer API-key setup. The default path is to ask whether the user already has a Slideshot account, send them to [app.slideshot.ai](https://app.slideshot.ai) to create an account or log in if needed, then ask them to create an API key there and paste it so you can configure `auth set-key`.

Preferred API-key setup:

```bash
npx -y slideshot-cli auth set-key srk_...
```

Useful auth commands:

```bash
npx -y slideshot-cli auth status
npx -y slideshot-cli auth env --shell sh --output text
```

Notes:

- `auth status` is the quickest way to discover whether the CLI already has a stored user session, a stored API key, and which auth file path is in use.
- `auth set-key` is the preferred local auth setup flow because it is simpler to guide reliably in chat once the user has an account and can generate a key in the web app.
- Command auth resolution prefers `--api-key`, then `SLIDESHOT_API_KEY`, then a stored local API key, then a stored user session when the command supports it.

## Saved credentials

Before creating a login-required run, list saved credentials and check for a matching hostname first.

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
- Password is optional; email-only credentials are allowed.
- Use `--default` when the same hostname should usually reuse one credential.
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

Opt into GIF export:

```json
{
  "artifacts": { "gif": true }
}
```

Notes:

- Omit `auth` or use `{ "source": "none" }` when no saved credential should be used.

## Status, input, and the web app

Inspect one run:

```bash
npx -y slideshot-cli runs status <run-id> --output text
```

Continue an OTP or magic-link flow on the same run:

```bash
npx -y slideshot-cli runs input <run-id> --value "123456"
```

After creating a run, return the `run_id`, point the user at `https://app.slideshot.ai/?runId=<run-id>`, and optionally open it on macOS:

```bash
open "https://app.slideshot.ai/?runId=<run-id>"
```

## Artifacts

```bash
npx -y slideshot-cli runs artifacts <run-id> --output text
npx -y slideshot-cli runs download <run-id> --artifact demo.mp4 --dir ./artifacts --output text
npx -y slideshot-cli runs download <run-id> --all --dir ./artifacts
```

Stable public artifact names: `raw.mp4`, `demo.mp4`, `plan.json`, plus `demo.gif` when the run was created with `artifacts.gif: true`.

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

## Cancel and feedback

Cancel a run that is no longer wanted or was created with the wrong goal:

```bash
npx -y slideshot-cli runs cancel <run-id>
```

Send feedback when the failure looks like a product bug or gap. Include the related run id for context:

```bash
npx -y slideshot-cli feedback "The run stopped after login even though the dashboard loaded." --run-id <run-id>
```

The CLI `feedback` command works with API-key auth. It is appropriate for product bugs, feature requests, or general frustration even when no specific run failed.
