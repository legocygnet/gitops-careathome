# OpenClaw with local Ollama

This stack is self-contained. No onboarding wizard, companion file mount, or
`config set` commands are required. Configuration lives in the gateway's
`OPENCLAW_CONFIG_JSON` block in Compose. Startup validates the JSON and writes
it atomically to `/home/node/.openclaw/declarative.json` in the named state
volume before starting OpenClaw. Edit Compose and redeploy to change settings.
Changes made to that generated file are overwritten on restart. Existing state,
workspace, and auth volumes are preserved; old onboarding configuration is not
the active config. Secrets remain environment references in the generated file.

## Deploy

1. Deploy the repository's Ollama stack first. It creates `ai_backend` and pulls
   both Qwen models. Wait for its model loader to complete.
2. Supply `OPENCLAW_GATEWAY_TOKEN` using Portainer stack environment variables
   or a private `.env` next to this Compose file. Use a long random secret from
   a password manager. `.env.example` lists the optional settings.
3. Deploy `docker/openclaw/compose.yaml` from the repository, or run
   `docker compose up -d` from this directory on the host.

Portainer can deploy this from Git or a pasted Compose document. No relative
bind mounts or companion config files are required. Only the gateway runs by
default; the CLI is an optional administrative service. When deploying from
Git, push the updated Compose file before using Portainer's pull/redeploy action.

All persistence uses named volumes on Docker's configured data drive. No Docker
socket or general-purpose host directories are mounted.

## Access

The default URL is `http://careathome:18789`. Enter the configured gateway token.
This preserves the previous LAN exposure: do not forward this port from the
internet. Plain HTTP exposes tokens and document content to network observers;
use HTTPS or an SSH tunnel before handling sensitive documents.

For SSH-only access, set `OPENCLAW_BIND_ADDRESS=127.0.0.1` and
`OPENCLAW_UI_ORIGIN=http://localhost:18789`, then from your workstation:

```bash
ssh -N -L 18789:127.0.0.1:18789 sean@careathome
```

Open `http://localhost:18789` and enter your gateway token. For access without
an SSH tunnel, configure HTTPS through a reverse proxy or private access layer,
set `OPENCLAW_UI_ORIGIN` to its exact origin, and arrange proxy connectivity.
`OPENCLAW_BIND_ADDRESS` controls the published host interface. Current OpenClaw
supports LAN HTTP pairing, but it remains a downgraded transport. Browser device
approval can still be required and has not been disabled:

```bash
docker compose run --rm openclaw-cli devices list
docker compose run --rm openclaw-cli devices approve <requestId>
```

## Verify

```bash
docker compose ps
docker compose logs --tail=100 openclaw-gateway
docker compose run --rm openclaw-cli config validate
docker compose run --rm openclaw-cli models list
```

The default model is `ollama/qwen3:4b-instruct`; the thinking variant is also
listed. OpenClaw requests a 16K context instead of Ollama's general 8K setting.
Monitor `docker exec ollama ollama ps` after a request: additional context uses
more memory and full GPU residency on the 6 GB card is not guaranteed.

Tools use the minimal profile intentionally: no arbitrary shell or file editing
is enabled yet. PDF extraction/redaction tooling is a separate next step, not
a capability supplied by this configuration. The 4B models are a constrained
local test setup, not a guarantee of reliable complex agent behavior.

The image currently tracks `latest`; pin a tested version or digest before
production use. JSON/YAML syntax can be checked locally, but runtime validation
against the deployed image is required.

References:
- https://docs.openclaw.ai/gateway/configuration
- https://docs.openclaw.ai/providers/ollama
- https://docs.openclaw.ai/web/control-ui
