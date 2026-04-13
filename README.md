# hermes-agent-docker

Automated Docker image builds for [NousResearch/hermes-agent](https://github.com/NousResearch/hermes-agent), published to GitHub Container Registry (GHCR).

Hermes Agent is a self-improving AI agent by [Nous Research](https://nousresearch.com) that learns from experience, supports 200+ models, and connects to multiple messaging platforms.

## Quick Start

```bash
docker pull ghcr.io/h-teske/hermes-agent-docker:latest
```

### Run

```bash
docker run -d \
  --name hermes-agent \
  -v hermes-data:/opt/data \
  --env-file .env \
  ghcr.io/h-teske/hermes-agent-docker:latest
```

### Use a specific version

```bash
docker pull ghcr.io/h-teske/hermes-agent-docker:v2026.3.30
```

Available tags match the upstream [release tags](https://github.com/NousResearch/hermes-agent/releases).

## How the CI works

A GitHub Actions workflow automatically builds and publishes Docker images to GHCR. New upstream releases are detected in two ways:

| Mechanism | How it works |
|-----------|-------------|
| **Scheduled check** | Every 6 hours, the workflow queries the latest upstream release and builds if the image doesn't exist yet in GHCR |
| **Renovate** | As a fallback, [Renovate](https://docs.renovatebot.com/) tracks upstream releases via `upstream-version.json` and creates PRs on new versions |
| **Manual trigger** | The workflow can be dispatched manually with a specific version tag |

Images are tagged with the upstream version (e.g. `v2026.3.30`) and `latest` points to the most recent scheduled/Renovate build.

## Modifications to upstream

This repository applies the following patches to the upstream Dockerfile at build time:

| Patch | Reason |
|-------|--------|
| `apt-get install git` | Required before `npm install` — upstream image omits `git`, causing npm to fail on git-protocol dependencies |
| `apt-get install libolm-dev` | System dependency required for Matrix E2E encryption |
| `pip install 'matrix-nio[e2e]'` | Adds Matrix protocol support with End-to-End Encryption via [matrix-nio](https://github.com/poljar/matrix-nio) |

The patches are injected by the CI workflow after cloning the upstream source, before the Docker image is built.

## Configuration

Hermes Agent is configured via environment variables. See the upstream [`.env.example`](https://github.com/NousResearch/hermes-agent/blob/main/.env.example) for available options.

Data is persisted in the `/opt/data` volume inside the container.

## Links

- **Upstream project:** [NousResearch/hermes-agent](https://github.com/NousResearch/hermes-agent)
- **Container images:** [ghcr.io/h-teske/hermes-agent-docker](https://github.com/h-teske/hermes-agent-docker/pkgs/container/hermes-agent-docker)
- **Upstream releases:** [github.com/NousResearch/hermes-agent/releases](https://github.com/NousResearch/hermes-agent/releases)

## License

This repository only contains CI configuration. The Docker image contents are from [NousResearch/hermes-agent](https://github.com/NousResearch/hermes-agent) which is licensed under the [MIT License](https://github.com/NousResearch/hermes-agent/blob/main/LICENSE).
