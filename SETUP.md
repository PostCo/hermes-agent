# PostCo Hermes setup

This repository is PostCo's Hermes runtime. It carries the A2A compatibility
needed for Hermes to call the Mastra `tapir-support` agent. The support
workflow itself lives in
[`PostCo/hermes-postco-support-agent`](https://github.com/PostCo/hermes-postco-support-agent).

The commands below install both pieces without copying files between machines.

## Prerequisites

- macOS or Linux
- Git access to the PostCo repositories
- The Mastra A2A HTTPS endpoint
- `MASTRA_A2A_HERMES_TOKEN` from the team's secret manager
- The model and support-workflow credentials listed in the profile repository

Do not put tokens, API keys, customer data, or production URLs in this runtime
repository.

## Install on a new machine

### 1. Install the PostCo Hermes runtime

Do not start with the official one-line Hermes installer: it clones the
NousResearch repository. Clone this repository into Hermes' standard managed
install path, then run the installer from the checkout:

```bash
mkdir -p "$HOME/.hermes/profiles"

git clone https://github.com/PostCo/hermes-agent.git \
  "$HOME/.hermes/hermes-agent"

bash "$HOME/.hermes/hermes-agent/scripts/install.sh" \
  --dir "$HOME/.hermes/hermes-agent" \
  --skip-setup
```

Confirm that `~/.local/bin` is on `PATH`, then verify the installation:

```bash
hermes version
git -C "$HOME/.hermes/hermes-agent" remote -v
```

`origin` must be `PostCo/hermes-agent`. The installer may add or use
NousResearch as `upstream`, but production updates must come through the
PostCo repository.

### 2. Install the PostCo support profile

```bash
git clone git@github.com:PostCo/hermes-postco-support-agent.git \
  "$HOME/.hermes/profiles/postco-support"

hermes profile alias postco-support
```

This creates the `postco-support` command, equivalent to
`hermes -p postco-support`.

Follow the profile repository's `SETUP.md` to provision its ignored
`config.yaml`, `.env`, Gmail, Sheets, and other credentials. Do not enable its
mail cron job on a second machine unless that machine is intentionally taking
over production mail processing.

### 3. Configure Mastra A2A

Put the bearer token in the profile's ignored environment file:

```dotenv
# ~/.hermes/profiles/postco-support/.env
MASTRA_A2A_HERMES_TOKEN=<retrieve-from-secret-manager>
```

Merge the following into the profile's ignored `config.yaml`. Replace the two
example URLs with the reachable Mastra URLs for the target environment:

```yaml
plugins:
  enabled:
    - platforms/a2a

platform_toolsets:
  cli:
    - a2a
    - hermes-cli

a2a_agents:
  tapir-support:
    url: https://mastra.example.com/api/a2a/tapir-support
    card_url: https://mastra.example.com/api/.well-known/tapir-support/agent-card.json
    auth:
      type: bearer
      token: ${env:MASTRA_A2A_HERMES_TOKEN}
    timeout: 600
    method_style: mastra
    capabilities:
      - project-tapir
      - code-investigation
      - readonly-production-data
```

For a Mastra process on the same machine, `http://127.0.0.1:4111` may be used
instead. Across machines, use a private network address or HTTPS endpoint that
the Hermes machine can reach.

### 4. Verify before enabling mail

Start a fresh profile session:

```bash
postco-support
```

Ask Hermes to call `tapir-support` with a harmless read-only request. Verify
that the result reports a Mastra context ID and task ID and does not return an
authentication error or HTTP 504. Then test with a representative support
email. Enable the gateway and cron only after this smoke test passes.

## Updating an existing machine

The runtime and workflow are separate checkouts and therefore have separate
updates:

```bash
# Runtime and A2A client patches; fetches from PostCo because it is origin.
hermes update

# Support workflow, prompts, skills, and scripts.
git -C "$HOME/.hermes/profiles/postco-support" pull --ff-only
```

After changing runtime code, profile configuration, prompts, or skills,
restart the profile gateway and repeat the A2A smoke test before allowing the
next mail run.

Do not run the official installation command over this checkout. Do not point
`origin` back to NousResearch. Upstream Hermes releases should first be merged
and tested in `PostCo/hermes-agent`; production machines can then receive them
through `hermes update`.

## Troubleshooting

- **A2A tool is unavailable:** confirm `platforms/a2a` is enabled and `a2a` is
  included in the profile's CLI or relevant gateway toolset. Start a new
  session after changing configuration.
- **HTTP 401 or 403:** confirm the same `MASTRA_A2A_HERMES_TOKEN` is configured
  on Hermes and Mastra without printing it to logs.
- **HTTP 504:** confirm the peer uses `method_style: mastra`, has a sufficient
  overall timeout, and that Hermes is polling the returned task rather than
  holding or resubmitting one long request.
- **Updates come from the wrong repository:** run
  `git -C ~/.hermes/hermes-agent remote -v`; `origin` must be the PostCo
  repository.
