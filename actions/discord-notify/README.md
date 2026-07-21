# discord-notify

Post a CI/CD status message to Discord via webhook. Shared across all `spectrum-labs-tech` repos.

Two ways to use it, one shared secret. The composite action here is the primitive; the
[`discord-cicd-notify.yaml`](../../.github/workflows/discord-cicd-notify.yaml) reusable workflow is a
thin wrapper for the common "ping once when a run fails" case.

## One-time setup: the webhook secret

Store the Discord webhook as a GitHub Actions secret named **`DISCORD_CICD_WEBHOOK`**.

- **Org secret** (recommended — set once, scoped to your private repos; requires GitHub Team):
  `gh secret set DISCORD_CICD_WEBHOOK --org spectrum-labs-tech --visibility private`
- **Per-repo** (GitHub Free, or to override for one repo):
  `gh secret set DISCORD_CICD_WEBHOOK --repo spectrum-labs-tech/<repo>`

The command prompts for the value so the URL never lands in shell history. Rotate by re-running it
(regenerate the webhook in Discord → Channel → Integrations first).

## Usage A — reusable workflow (fires once per run; least boilerplate)

Best default. Add a trailing job that runs only when something upstream failed:

```yaml
jobs:
  build: { ... }
  test: { ... }

  notify:
    needs: [build, test]        # every job whose failure should alert
    if: ${{ failure() }}        # only on failure
    uses: spectrum-labs-tech/.github/.github/workflows/discord-cicd-notify.yaml@main
    secrets: inherit            # passes DISCORD_CICD_WEBHOOK
```

`needs` + a job-level `if: failure()` means one ping per run, not one per matrix leg.

## Usage B — composite action (surgical, in-job)

When you want to alert from inside a specific job/step (e.g. name the step that broke), or a job has
no downstream job to hang a trailing notify on:

```yaml
    steps:
      - run: make build
      - run: make test
      - name: Alert on failure
        if: ${{ failure() }}
        uses: spectrum-labs-tech/.github/actions/discord-notify@main
        with:
          webhook: ${{ secrets.DISCORD_CICD_WEBHOOK }}   # composite actions can't inherit secrets
          status: failure
          message: "build or tests failed"
```

## Inputs (composite action)

| Input     | Required | Default            | Notes                                                        |
| --------- | -------- | ------------------ | ------------------------------------------------------------ |
| `webhook` | yes      | —                  | Discord webhook URL. Always a secret.                        |
| `status`  | no       | `failure`          | `success` / `failure` / `cancelled` drive colour + emoji.    |
| `title`   | no       | derived            | Override the embed title.                                    |
| `message` | no       | —                  | Extra description line.                                      |

The embed auto-includes repo, branch, short commit (linked), actor, workflow, and a link to the run.
Needs `curl` + `jq` on the runner (present on `ubuntu-latest` and the self-hosted homelab runners).
