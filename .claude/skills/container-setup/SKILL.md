---
name: container-setup
description: Use when the user wants to start, stop, configure, or debug the gascity-fedora-container with podman — including Vertex AI setup, ADR pipeline pack, gascity initialization, and session troubleshooting.
---

# Gastown Fedora Container Setup

This skill automates standing up and configuring the gascity-fedora-container
for agentic development with gascity and the ADR pipeline pack.

## Prerequisites

The following environment variables must be set on the **host** before running:

| Variable | Purpose |
|----------|---------|
| `ANTHROPIC_VERTEX_PROJECT_ID` | GCP project ID for Vertex AI |
| `GCP_VERTEX_JSON_PATH` | Absolute path to GCP service account JSON key |
| `GEMINI_API_KEY` | API key for Gemini CLI |
| `GH_TOKEN` | GitHub personal access token (for HTTPS git auth) |

For SSH-based GitHub auth, ensure the SSH private key exists on the host.

For basic (non-Vertex) usage, only `ANTHROPIC_API_KEY` is required.

> **Security:** Use a dedicated GitHub account with a scoped token. The
> container runs Claude Code in YOLO mode — agents execute commands without
> confirmation. Never pass personal credentials.

## Step 1 — Remove any existing container

```bash
podman stop gascity 2>/dev/null; podman rm gascity 2>/dev/null
```

## Step 2 — Start the container

### Basic (direct API keys)

```bash
podman run -d --name gascity --pids-limit=-1 \
  --userns=keep-id:uid=1000,gid=1000 \
  -v ~/Projects:/workspace:Z \
  -v ~/.config:/home/gascity/.config:Z \
  -v ~/.ssh/agent-key:/home/gascity/.ssh/id_ed25519:ro,Z \
  -e ANTHROPIC_API_KEY \
  -e GH_TOKEN \
  ghcr.io/gurnben/gascity-fedora-container:latest \
  sleep infinity
```

### Google Cloud / Vertex AI

```bash
podman run -d --name gascity --pids-limit=-1 \
  --userns=keep-id:uid=1000,gid=1000 \
  -v ~/Projects:/workspace:Z \
  -v ~/.config:/home/gascity/.config:Z \
  -v $GCP_VERTEX_JSON_PATH:/home/gascity/.config/gcloud/application_default_credentials.json:ro,Z \
  -v ~/.ssh/agent-key:/home/gascity/.ssh/id_ed25519:ro,Z \
  -e GOOGLE_APPLICATION_CREDENTIALS=/home/gascity/.config/gcloud/application_default_credentials.json \
  -e CLAUDE_CODE_USE_VERTEX=1 \
  -e CLOUD_ML_REGION=global \
  -e ANTHROPIC_VERTEX_PROJECT_ID \
  -e VERTEXAI_PROJECT=$ANTHROPIC_VERTEX_PROJECT_ID \
  -e VERTEXAI_LOCATION=global \
  -e GOOGLE_CLOUD_PROJECT=$ANTHROPIC_VERTEX_PROJECT_ID \
  -e VERTEX_LOCATION=global \
  -e GOOGLE_CLOUD_LOCATION=global \
  -e GEMINI_API_KEY \
  -e GH_TOKEN \
  ghcr.io/gurnben/gascity-fedora-container:latest \
  sleep infinity
```

### Key flags

- **`--pids-limit=-1`** — gascity needs more than Podman's default 2048 processes
- **`--userns=keep-id:uid=1000,gid=1000`** — maps host UID to container user for file access
- **`sleep infinity`** — keeps the container alive; gascity supervisor starts manually

## Step 3 — Persist environment variables

Write env vars to `/etc/profile.d/` so every `bash -l` session inherits them.

### Vertex AI

```bash
podman exec gascity sudo bash -c 'cat > /etc/profile.d/vertex.sh << "EOF"
export GOOGLE_APPLICATION_CREDENTIALS=/home/gascity/.config/gcloud/application_default_credentials.json
export CLAUDE_CODE_USE_VERTEX=1
export CLOUD_ML_REGION=global
export ANTHROPIC_VERTEX_PROJECT_ID='"$ANTHROPIC_VERTEX_PROJECT_ID"'
export VERTEXAI_PROJECT=$ANTHROPIC_VERTEX_PROJECT_ID
export VERTEXAI_LOCATION=global
export GOOGLE_CLOUD_PROJECT=$ANTHROPIC_VERTEX_PROJECT_ID
export VERTEX_LOCATION=global
export GOOGLE_CLOUD_LOCATION=global
export GEMINI_API_KEY='"$GEMINI_API_KEY"'
EOF'
```

### GitHub token

```bash
podman exec gascity sudo bash -c 'cat > /etc/profile.d/github.sh << "EOF"
export GH_TOKEN='"$GH_TOKEN"'
EOF'
podman exec gascity bash -lc 'gh auth setup-git'
```

## Step 4 — Fix SSH permissions and configure identities

The mounted `.ssh/` directory may be owned by root. Fix ownership and write
the SSH config:

```bash
podman exec gascity sudo bash -c \
  'chown gascity:gascity /home/gascity/.ssh && \
   chmod 700 /home/gascity/.ssh && \
   cat > /home/gascity/.ssh/config << "EOF"
Host github.com
  StrictHostKeyChecking accept-new
  IdentityFile ~/.ssh/id_ed25519
EOF
   chown gascity:gascity /home/gascity/.ssh/config && \
   chmod 600 /home/gascity/.ssh/config'
```

Configure git and dolt identities:

```bash
podman exec gascity bash -lc '
  GH_USER=$(gh api /user | jq -r .login) && \
  GH_ID=$(gh api /user | jq -r .id) && \
  GH_EMAIL="${GH_ID}+${GH_USER}@users.noreply.github.com" && \
  git config --global user.name "$GH_USER" && \
  git config --global user.email "$GH_EMAIL" && \
  git config --global commit.gpgsign true && \
  git config --global gpg.format ssh && \
  git config --global user.signingkey ~/.ssh/id_ed25519 && \
  dolt config --global --add user.name "$GH_USER" && \
  dolt config --global --add user.email "$GH_EMAIL"'
```

## Step 5 — Verify tools and authentication

```bash
podman exec gascity bash -lc 'claude --version && gc version && gh auth status'
podman exec gascity bash -lc 'ssh -T git@github.com 2>&1 || true'
podman exec gascity bash -lc 'claude -p "respond with hello" --output-format text'
```

## Step 6 — Initialize workspace and import the pipeline pack

```bash
podman exec gascity bash -lc 'cd /workspace && gc init --skip-provider-readiness .'
```

Fix a known beads bug where dotted YAML keys are unquoted (causes bd parse errors):

```bash
podman exec gascity bash -lc '
for f in $(find /workspace -name "config.yaml" -path "*/.beads/*"); do
  sed -i "s/^dolt\.auto-start/\"dolt.auto-start\"/; s/^gc\.endpoint_origin/\"gc.endpoint_origin\"/; s/^gc\.endpoint_status/\"gc.endpoint_status\"/; s/^types\.custom/\"types.custom\"/; s/^sync\.remote/\"sync.remote\"/" "$f"
done'
```

Remove the workspace-level mayor (the pipeline pack provides its own):

```bash
podman exec gascity bash -lc 'cd /workspace && rm -rf agents/mayor'
```

Update `pack.toml` to import the pipeline and remove the workspace-level
mayor definition. Choose one:

**Singleton** (1 concurrent pipeline, ~5 GB peak RAM):

```bash
podman exec gascity bash -lc 'cd /workspace && cat > pack.toml << "EOF"
[pack]
name = "workspace"
schema = 2

[imports.pipeline]
source = "/opt/adr-pipeline"
export = true
EOF'
```

**Scaled** (up to 6 concurrent pipelines, ~20 GB peak RAM):

```bash
podman exec gascity bash -lc 'cd /workspace && cat > pack.toml << "EOF"
[pack]
name = "workspace"
schema = 2

[imports.pipeline]
source = "/opt/adr-pipeline-scaled"
export = true
EOF'
```

Then install the import:

```bash
podman exec gascity bash -lc 'cd /workspace && gc import install'
```

## Step 7 — Configure the model

Set Opus 4.6 1M as the default model and increase the startup timeout:

```bash
podman exec gascity bash -lc 'cd /workspace && cat > city.toml << "EOF"
[workspace]
provider = "claude-opus"

[providers.claude-opus]
base = "claude"
args_append = ["--model", "claude-opus-4-6"]

[session]
startup_timeout = "120s"

[daemon]
patrol_interval = "30s"
EOF'
```

## Step 8 — Start the gascity supervisor

```bash
podman exec gascity bash -lc \
  'cd /workspace && gc unregister /workspace 2>/dev/null; \
   tmux new-session -d -s gc "gc start --foreground"'
```

Wait ~30 seconds for sessions to spawn, then dismiss Claude Code onboarding
prompts (theme selector + security notice) that block headless sessions:

```bash
podman exec gascity bash -lc \
  'tmux new-session -d -s dismiss "/opt/adr-pipeline/assets/scripts/dismiss-onboarding.sh"'
```

This runs as a background daemon that continuously watches for new sessions
hitting onboarding prompts, including dynamically scaled pool agents.

## Step 9 — Verify sessions

Wait ~2 minutes for all sessions to start with Vertex AI, then:

```bash
podman exec gascity bash -lc 'cd /workspace && gc session list'
podman exec gascity bash -lc 'cd /workspace && gc doctor'
```

All sessions (mayor, architect, reviewer, planner, qe, senior, dogs) should
show `active` state. If any are `stopped` with stale beads, clean them:

```bash
podman exec gascity bash -lc 'cd /workspace && gc bd close <id>; bd delete <id> --force'
```

The supervisor will recreate fresh sessions automatically.

## Step 10 — Enter and use

```bash
podman exec -it gascity bash -l
gc session attach pipeline.mayor
```

Tell the mayor what to build. Detach with `Ctrl-b d`.

## Troubleshooting

### Sessions flapping (creating → failed-create → creating)

A stale bead from a prior run. Clean it:

```bash
gc bd close <id>
bd delete <id> --force
```

### "resource temporarily unavailable" from bd/dolt

Recreate the container with `--pids-limit=-1`.

### "Permission denied" on mounted files

Recreate with `--userns=keep-id:uid=1000,gid=1000`.

### "systemctl --user daemon-reload: exit status 1"

Expected — containers lack systemd. Use `gc start --foreground`.

### "needs authentication" during gc init

Use `--skip-provider-readiness` with Vertex AI.

### Sessions stuck on Claude Code onboarding screen

Pre-complete the onboarding:

```bash
podman exec gascity bash -lc \
  'mkdir -p ~/.claude && \
   echo "{\"theme\":\"dark\",\"hasCompletedOnboarding\":true}" > ~/.claude/settings.json'
```

### Container lifecycle

```bash
podman stop gascity                       # stop (preserves state)
podman start gascity                      # restart
podman stop gascity && podman rm gascity  # destroy
```

After restart, re-run Step 8 to start the supervisor.
